# Traefik atrás da Cloudflare

Proxy reverso (Traefik v3) para uma VPS com o domínio na Cloudflare. A origem só aceita tráfego da Cloudflare, em quatro camadas:

1. firewall do provedor liberando 80/443 só para as [faixas da Cloudflare](https://www.cloudflare.com/ips/);
2. `ipAllowList` (`cloudflare-only`) nas entradas 80 e 443;
3. **Authenticated Origin Pulls** (mTLS): o 443 exige o certificado de cliente que só a sua conta da Cloudflare tem;
4. `sniStrict`: hostname desconhecido é recusado no TLS.

Também traz: imagens fixadas por digest, socket do Docker atrás de um proxy somente leitura (rede interna), `cap_drop: ALL`, `no-new-privileges`, IP real do visitante (`forwardedHeaders.trustedIPs`), `aliasHeadersStrategy=delete`, HSTS, TLS ≥ 1.2, healthcheck e log de acesso em JSON. WebSocket funciona sem configuração extra.

O compose é autossuficiente (configuração dinâmica e CA pública embutidas em `configs`), para deploy por URL. Só o `.env` é copiado à mão.

## Deploy

1. Copie `.env.example` para `.env` no diretório do projeto e preencha `ACME_EMAIL`.
2. Na Cloudflare, **antes** de subir o Traefik, configure o que está abaixo (principalmente o Authenticated Origin Pulls).
3. `docker compose up -d` (ou deploy pela URL deste `docker-compose.yml`).

## Cloudflare

| Onde | Configuração |
|---|---|
| SSL/TLS → Overview | **Full (strict)** |
| SSL/TLS → Edge Certificates | **Always Use HTTPS: OFF** (o Traefik já redireciona; ligado, quebra o desafio HTTP do Let's Encrypt) · Minimum TLS **1.2** |
| SSL/TLS → Origin Server → Authenticated Origin Pulls → **Zone-level** | **Upload certificate** com o certificado de cliente e a chave dele (assinados pela CA embutida no compose) e ligue o toggle |
| DNS | registro **A** para o IPv4 da VPS, **Proxied**; **sem AAAA** para a VPS (o Docker sem IPv6 troca o IP de origem e o `ipAllowList` bloquearia) |
| Network | **WebSockets: ON** |
| Rules → Transform Rules → Modify Request Header | **Set** `X-Real-IP` = dinâmico `ip.src` |
| Speed / Scrape Shield | **Rocket Loader: OFF** · **Email Obfuscation: OFF** |

Com o toggle do Authenticated Origin Pulls desligado, o HTTPS da origem recusa a Cloudflare (erro 525/526).

## Publicar um serviço

O serviço entra na rede `proxy` e declara as labels:

```yaml
services:
  app:
    image: exemplo/app
    networks: [proxy]
    labels:
      - traefik.enable=true
      - traefik.http.routers.app.rule=Host(`app.exemplo.com`)
      - traefik.http.services.app.loadbalancer.server.port=8080

networks:
  proxy:
    external: true
```

A entrada `websecure`, o certificado e os middlewares `cloudflare-only` e `secure-headers` já valem por padrão. Não publique `ports` no serviço.

## Manutenção

- **Faixas da Cloudflare:** ao mudarem, atualize o `sourceRange` em `configs.traefik-dynamic` e o padrão de `CF_TRUSTED_IPS` no `command`.
- **mTLS:** o certificado de cliente vence em 2028-10-23 (a Cloudflare avisa 30 e 14 dias antes). Emita outro com a mesma CA e troque na Cloudflare. A chave da CA fica fora do servidor e fora deste repositório.
- **Outra CA:** gere a sua (veja a [documentação da Cloudflare](https://developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/set-up/zone-level/)) e troque o bloco `configs.cloudflare-aop-ca`.
- **Atualizar o Traefik:** troque a tag e o digest da imagem.
