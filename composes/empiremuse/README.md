# EmpireMuse atrás do Traefik

Receita de implantação em arquivo único para o [EmpireMuse](https://github.com/EmpireCommunity/EmpireMuse) (bot de música multi-instância para Discord com Lavalink embutido, relé RPC e painel web administrativo), operando em conjunto com o [Traefik com Cloudflare](../traefik) deste cookbook.

Esta configuração foi desenhada para VPS e gerenciadores de contêineres que não suportam múltiplos arquivos compose (`-f`) ou `env_file` (como o **Docker Manager da Hostinger**):
- **Configuração Inline**: Todas as opções de runtime, intents, skins, codecs e regras do bot já estão pré-configuradas no `docker-compose.yml` com os padrões recomendados do projeto.
- **`.env` Enxuto**: Apenas segredos, tokens e o domínio FQDN são parametrizados no `.env`.
- **Compatibilidade com Limites de Painel**: Compatível com formulários web que impõem limite de 256 caracteres por variável e recusam comentários em linha.
- **Suporte a Multi-bot**: Distribuição de múltiplos tokens de bot em variáveis dedicadas (`TOKEN`, `TOKEN_2`..`TOKEN_5`), detectadas nativamente pelo escaneamento do pool.
- **Segurança Reforçada**: Contêiner com `read_only: true`, `tmpfs: [/tmp]`, `cap_drop: [ALL]`, `no-new-privileges: true`, rotação de logs em JSON (`10m` x 3) e supervisão do processo Java com reinício automático.

---

## Pré-requisitos

1. **Traefik em Execução**: A stack do Traefik deste cookbook ([`composes/traefik`](../traefik)) deve estar rodando na mesma máquina, gerenciando a entrada 443 e a terminação TLS.
2. **Rede Externa `proxy`**: O Traefik e o EmpireMuse comunicam-se pela rede interna Docker `proxy`. Caso ainda não exista, crie-a com:
   ```bash
   docker network create proxy
   ```
3. **Zona DNS e Cloudflare**:
   - Registro DNS do tipo **A** apontando para o IPv4 da VPS com proxy da Cloudflare ativado (**Proxied / Nuvem Laranja**).
   - SSL/TLS definido como **Full (strict)** com **Always Use HTTPS** ativado.
   - Em **Network**, certifique-se de que **WebSockets** está ativado (**ON**).
   - O domínio FQDN deve estar coberto pelo certificado Origin CA configurado no Traefik (ex.: `*.seudominio.com` ou nome exato).

---

## Configurações no Discord Developer Portal

Antes de iniciar a stack, prepare as aplicações no [Discord Developer Portal](https://discord.com/developers/applications):

1. **Privileged Gateway Intents (Obrigatório)**:
   - Em cada aplicação de bot do seu pool, acesse a aba **Bot**.
   - Na seção **Privileged Gateway Intents**, ative:
     - **Message Content Intent** (obrigatório para comandos prefixados, links de áudio e suporte ao chat de pedidos).
     - **Server Members Intent** (recomendado para sincronização de DJ e permissões).
   - Clique em **Save Changes**.

2. **Redirect URI do Painel Web (OAuth2)**:
   - Na aplicação principal do bot (ou na aplicação dedicada ao painel), acesse a aba **OAuth2 > General**.
   - Em **Redirects**, clique em **Add Redirect** e adicione a URL de callback exata:
     ```
     https://<domínio>/painel/callback
     ```
     *(Substitua `<domínio>` pelo valor configurado em `EMPIREMUSE_DOMAIN`, ex.: `https://muse.seudominio.com/painel/callback`)*.
   - Guarde o **Client ID** e o **Client Secret** dessa aplicação para preencher `PANEL_OAUTH_CLIENT_ID` e `PANEL_OAUTH_CLIENT_SECRET`.

---

## Deploy

### Opção A: No Docker Manager da Hostinger (Painel Web)

1. No painel VPS da Hostinger, acesse a seção **Docker** e crie uma nova aplicação Docker Compose.
2. No campo de configuração do **Compose (`docker-compose.yml`)**, cole o conteúdo do arquivo [`docker-compose.yml`](docker-compose.yml).
3. No campo de **Variáveis de Ambiente (`.env`)**, declare as variáveis baseadas no [`.env.example`](.env.example):
   - **Atenção**: Cole apenas as declarações no formato `CHAVE=valor`. Não inclua comentários nas linhas de valores, pois o formulário da Hostinger pode interpretá-los incorretamente.
   - Cada valor não deve exceder **256 caracteres**. Para múltiplos bots, informe um token por linha (`TOKEN=...`, `TOKEN_2=...`, etc.).
4. Clique em **Deploy** ou **Salvar e Iniciar**.

### Opção B: Via Linha de Comando (CLI)

1. Crie um diretório para o projeto no servidor:
   ```bash
   mkdir -p /opt/empiremuse && cd /opt/empiremuse
   ```
2. Baixe o `docker-compose.yml` e o `.env.example`:
   ```bash
   curl -fsSL https://raw.githubusercontent.com/Or4cu1o/docker-compose-cookbook/main/composes/empiremuse/docker-compose.yml -o docker-compose.yml
   curl -fsSL https://raw.githubusercontent.com/Or4cu1o/docker-compose-cookbook/main/composes/empiremuse/.env.example -o .env
   ```
3. Edite o `.env` informando seus dados reais:
   ```bash
   nano .env
   ```
4. Suba o serviço:
   ```bash
   docker compose up -d
   ```
5. Verifique os logs da partida do bot e do Lavalink:
   ```bash
   docker logs -f empiremuse
   ```

---

## YouTube OAuth (Anti-Bot)

Para evitar bloqueios de reprodução do YouTube (*"Sign in to confirm you're not a bot"*), o plugin do YouTube embutido no Lavalink suporta autenticação via OAuth (cliente de TV):

> [!WARNING]
> Use **SEMPRE uma conta Google secundária / descartável** (*throwaway account*). Nunca utilize sua conta pessoal principal, pois o YouTube pode restringir contas que executam streaming automatizado.

1. Com o bot iniciado em um servidor Discord de suporte/gerenciamento, execute o comando de barra:
   ```
   /ytoauth
   ```
2. O bot responderá com uma URL de verificação do Google (`https://google.com/device`) e um código alfanumérico de pareamento.
3. Acesse o link no navegador conectado à sua **conta descartável** e confirme a autorização.
4. Após autorizar, o bot gerará o refresh token seguro. Copie o valor e adicione-o à variável `YOUTUBE_OAUTH_REFRESH_TOKEN` no seu `.env`.
5. Reinicie o contêiner para que o Lavalink carregue o token na inicialização.

---

## Variáveis do `.env`

| Variável | Obrigatória | Descrição |
| :--- | :---: | :--- |
| `EMPIREMUSE_DOMAIN` | **Sim** | FQDN público roteado pelo Traefik (ex.: `muse.seudominio.com`). |
| `TOKEN` | **Sim** | Token do bot principal do Discord gerado no Developer Portal. |
| `TOKEN_2`..`TOKEN_5` | Não | Tokens de bots secundários para operar em pool distribuído (até 5 instâncias). |
| `OWNER_IDS` | **Sim** | IDs de usuário dos administradores no Discord (separados por espaço). |
| `MONGO` | Não | URI de conexão com o MongoDB (ex.: `mongodb+srv://...`). Se vazio, utiliza banco local TinyMongo em `/data`. |
| `SPOTIFY_CLIENT_ID` | Não | Client ID da API do Spotify para resolução de faixas e playlists. |
| `SPOTIFY_CLIENT_SECRET` | Não | Client Secret da API do Spotify. |
| `LASTFM_KEY` | Não | API Key do Last.fm para busca de faixas e scrobbling. |
| `LASTFM_SECRET` | Não | Secret da API do Last.fm. |
| `PANEL_OAUTH_CLIENT_ID` | Não | Client ID para autenticação dos usuários no painel web via Discord. |
| `PANEL_OAUTH_CLIENT_SECRET` | Não | Client Secret para o OAuth do painel web. |
| `PANEL_COOKIE_SECRET` | Não | String aleatória segura (64+ caracteres) para assinatura de cookies de sessão. |
| `RPC_SERVER_SECRET` | Não | Segredo para conexão de bots com o relé RPC (vazio gera efêmero). |
| `YOUTUBE_OAUTH_REFRESH_TOKEN`| Não | Refresh token da conta Google descartável para contornar bloqueios anti-bot. |
| `BOT_ADD_REMOVE_LOG` | Não | Webhook do Discord para avisos de entrada e saída do bot em servidores. |
| `ERROR_REPORT_WEBHOOK` | Não | Webhook do Discord para o botão de envio manual de relatórios de erro. |
| `AUTO_ERROR_REPORT_WEBHOOK` | Não | Webhook do Discord para encaminhamento automático de erros e tracebacks. |

---

## Manutenção e Atualização

- **Persistência**: Todos os dados persistentes (banco local, sessões, caches e logs do Lavalink) ficam salvos no volume Docker `empiremuse-data` montado em `/data`.
- **Atualização**: Para atualizar para uma nova versão da imagem oficial, atualize a tag da imagem no `docker-compose.yml` (ex.: de `2.0.0` para `2.0.1` ou use a tag de série `2`) e recrie o contêiner:
  ```bash
  docker compose pull && docker compose up -d
  ```
