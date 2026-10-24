# Traefik atrás da Cloudflare

Proxy reverso (Traefik v3) para uma VPS com o domínio na Cloudflare. Só a porta **443** fica aberta, e a origem só aceita tráfego da Cloudflare, em quatro camadas:

1. firewall do provedor liberando a 443 só para as [faixas da Cloudflare](https://www.cloudflare.com/ips/);
2. `ipAllowList` (`cloudflare-only`) na entrada 443;
3. **Authenticated Origin Pulls** (mTLS): a 443 exige o certificado de cliente que só a sua conta da Cloudflare tem;
4. `sniStrict`: hostname desconhecido é recusado no TLS.

Sem porta 80: o redirecionamento HTTP→HTTPS é feito na borda da Cloudflare, e o certificado de origem é o **Cloudflare Origin CA** (sem ACME/Let's Encrypt).

Também traz: imagens fixadas por digest, socket do Docker atrás de um proxy somente leitura (rede interna), `cap_drop: ALL`, `no-new-privileges`, IP real do visitante (`forwardedHeaders.trustedIPs`), `aliasHeadersStrategy=delete`, HSTS, TLS ≥ 1.2, healthcheck e log de acesso em JSON. WebSocket funciona sem configuração extra.

A configuração dinâmica e a CA pública do mTLS vão embutidas no compose (`configs`), para deploy por URL. Copiados à mão para o diretório do projeto: o `.env` (opcional) e os certificados de origem (um por domínio).

## Deploy

1. **Certificados de origem (um por domínio/zona):** em cada zona, Cloudflare → SSL/TLS → Origin Server → **Create Certificate** (hostnames `dominio.com` e `*.dominio.com`). Salve no diretório do projeto como `origin/cert1.pem` + `origin/cert1.key`, `origin/cert2.*` e `origin/cert3.*` (chaves com `chmod 600`; nunca vão para o Git). O Traefik escolhe o certificado pelo hostname. Para mais domínios, acrescente `cert4`… na lista `tls.certificates` do compose.
2. **Authenticated Origin Pulls, em cada zona:** Cloudflare → SSL/TLS → Origin Server → Authenticated Origin Pulls → **Zone-level** → **Upload certificate** com o certificado de cliente e a chave dele (assinados pela CA embutida no compose; o mesmo par serve para todas as zonas) e ligue o toggle.
3. Ajuste o resto da Cloudflare (tabela abaixo, **em cada zona**) e o firewall do provedor (só 443, só faixas da Cloudflare).
4. `docker compose up -d` (ou deploy pela URL deste `docker-compose.yml`).

Faça os passos 1 e 2 **antes** de subir: sem o certificado de origem ou com o toggle do Authenticated Origin Pulls desligado, a Cloudflare mostra erro 525/526.

## Cloudflare (em cada zona)

| Onde | Configuração |
|---|---|
| SSL/TLS → Overview | **Full (strict)** |
| SSL/TLS → Edge Certificates | **Always Use HTTPS: ON** (substitui a porta 80) · Minimum TLS **1.2** |
| SSL/TLS → Origin Server | certificado de origem (passo 1) e Authenticated Origin Pulls **Zone-level** ligado (passo 2) |
| DNS | registro **A** para o IPv4 da VPS, **Proxied**; **sem AAAA** para a VPS (o Docker sem IPv6 troca o IP de origem e o `ipAllowList` bloquearia) |
| Network | **WebSockets: ON** |
| Rules → Transform Rules → Modify Request Header | **Set** `X-Real-IP` = dinâmico `ip.src` |
| Speed / Scrape Shield | **Rocket Loader: OFF** · **Email Obfuscation: OFF** |

## Publicar um serviço

O serviço entra na rede `proxy` e declara as labels:

```yaml
services:
  app:
    image: exemplo/app
    networks: [proxy]
    labels:
      - traefik.enable=true
      - traefik.http.routers.app.rule=Host(`app.dominio.com`)
      - traefik.http.services.app.loadbalancer.server.port=8080

networks:
  proxy:
    external: true
```

A entrada 443, o TLS e os middlewares `cloudflare-only` e `secure-headers` já valem por padrão. Não publique `ports` no serviço. O hostname precisa estar coberto pelo certificado de origem.

## Manutenção

- **Faixas da Cloudflare:** ao mudarem, atualize o `sourceRange` em `configs.traefik-dynamic` e o padrão de `CF_TRUSTED_IPS` no `command`.
- **Certificados de origem:** valem o prazo escolhido na criação (até 15 anos); para trocar, substitua os arquivos em `origin/` (o Traefik recarrega a configuração dinâmica; se não pegar, reinicie).
- **mTLS:** o certificado de cliente vence em 2028-10-23 (a Cloudflare avisa 30 e 14 dias antes). Emita outro com a mesma CA e troque na Cloudflare. A chave da CA fica fora do servidor e fora deste repositório.
- **Outra CA:** gere a sua (veja a [documentação da Cloudflare](https://developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/set-up/zone-level/)) e troque o bloco `configs.cloudflare-aop-ca`.
- **Atualizar o Traefik:** troque a tag e o digest da imagem.
