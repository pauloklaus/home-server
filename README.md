# HomeServer

Stack Docker para infraestrutura doméstica: reverse proxy com TLS, DNS filtrado, observabilidade de logs, monitoramento de banda e gestão de containers. Apps de mídia ficam em composes separados: Immich em `media/` e Navidrome em `music/`.

## Arquitetura

| Serviço | Função | URL / porta |
| --- | --- | --- |
| **Traefik** | Reverse proxy e certificados Let's Encrypt (DNS challenge Cloudflare) | `https://traefik.${DOMAIN}` |
| **AdGuard Home** | DNS filtrado (portas 53/tcp+udp) e painel web | `https://dns.${DOMAIN}` · UI local `:8080` |
| **Loki + Promtail** | Agregação e coleta de logs (containers e host) | interno (rede `monitoring`) |
| **Grafana** | Visualização de logs (datasource Loki provisionado) | `https://logs.${DOMAIN}` |
| **Speedtest Tracker** | Histórico de testes Ookla do uplink (a cada 6 h) | `https://speedtracker.${DOMAIN}` |
| **Portainer** | Gestão de containers Docker | `https://portainer.${DOMAIN}` |
| **Immich** (`media/`) | Galeria de fotos/vídeos | `https://media.${DOMAIN}` |
| **Navidrome** (`music/`) | Streaming da biblioteca de música (Subsonic/Chora) | `https://music.${DOMAIN}` |

Redes Compose: `monitoring` (stack principal) e `proxy` (externa; usada pelo Immich, pelo Navidrome e pelo Traefik).

## Estrutura do repositório

```
.
├── docker-compose.yml          # Stack principal
├── .env.template               # Variáveis do stack principal (copie para .env)
├── adguard/
│   └── conf/AdGuardHome.yaml.template   # Modelo de config; copie para AdGuardHome.yaml
├── traefik/letsencrypt/        # Persistência ACME (acme.json, runtime)
├── loki/loki-config.yaml       # Configuração do Loki (retenção 14 dias)
├── promtail/promtail.yaml      # Scraping de logs Docker e /var/log do host
├── grafana/provisioning/       # Datasource Loki pré-configurado
├── speedtest-tracker/data/     # Dados persistentes do Speedtest Tracker
├── media/
│   ├── docker-compose.yml      # Immich (server, ML, Redis/Valkey, Postgres)
│   └── .env.template           # Variáveis do Immich
└── music/
    ├── docker-compose.yml      # Navidrome
    └── .env.template           # Variáveis do Navidrome
```

Arquivos e diretórios de runtime (`.env`, dados, certificados, workdirs) estão no `.gitignore` e não devem ser versionados.

## Pré-requisitos

- Docker Engine e Docker Compose v2
- Rede Docker externa `proxy`:

  ```sh
  docker network create proxy
  ```

- Domínio gerenciado na Cloudflare (DNS challenge ACME)
- Token de API Cloudflare com permissão de edição DNS na zona

## Configuração

1. Copie e preencha os templates de ambiente:

   ```sh
   cp .env.template .env
   cp media/.env.template media/.env
   cp music/.env.template music/.env
   ```

   Variáveis principais (`.env`):

   | Variável | Uso |
   | --- | --- |
   | `DOMAIN` | Domínio base das regras Host do Traefik |
   | `CF_DNS_API_TOKEN` | Token Cloudflare para ACME |
   | `ACME_EMAIL` | E-mail da conta Let's Encrypt |
   | `GRAFANA_ADMIN_PASSWORD` | Senha do admin do Grafana |
   | `SPEEDTEST_APP_KEY` | Chave da aplicação (`echo "base64:$(openssl rand -base64 32)"`) |

2. (Opcional) Preparar o AdGuard a partir do template:

   ```sh
   cp adguard/conf/AdGuardHome.yaml.template adguard/conf/AdGuardHome.yaml
   ```

   Ajuste credenciais e políticas conforme necessário. Na primeira execução, o AdGuard também pode concluir o setup pela UI.

## Importante: DNS do host e AdGuard

Para o AdGuard escutar na porta **53** do host, o resolvedor DNS local do sistema precisa ser desativado. Sem isso, o binding do container em `:53` falha.

**Ordem crítica:** faça o pull das imagens **antes** de desativar o DNS. Depois de desativar o resolvedor do sistema, não haverá resolução de nomes até o AdGuard estar em execução — o que impede `docker compose pull` e downloads em geral.

```sh
docker compose pull
docker compose -f media/docker-compose.yml pull
docker compose -f music/docker-compose.yml pull
```

Em seguida:

1. Edite o Netplan e comente os `nameserver` (caminho típico em cloud-init):

   ```sh
   sudo nano /etc/netplan/50-cloud-init.yaml
   sudo netplan generate
   sudo netplan apply
   ```

2. Pare e desabilite o `systemd-resolved`:

   ```sh
   sudo systemctl stop systemd-resolved
   sudo systemctl disable systemd-resolved
   ```

3. Substitua o stub de `/etc/resolv.conf` por um arquivo apontando para o loopback (onde o AdGuard passará a servir):

   ```sh
   sudo rm -f /etc/resolv.conf
   echo 'nameserver 127.0.0.1' | sudo tee /etc/resolv.conf
   ```

4. Suba o stack imediatamente para restaurar a resolução DNS:

   ```sh
   docker compose up -d
   ```

Clientes da rede local devem usar o IP deste servidor como DNS (ou DHCP apontando para ele).

## Operação

Stack principal:

```sh
docker compose up -d
docker compose ps
docker compose logs -f <serviço>
```

Immich (a partir de `media/`):

```sh
cd media
docker compose up -d
```

Navidrome (a partir da raiz do HomeServer):

```sh
mkdir -p /storage/music music/data
# Ajuste o dono se PUID/PGID no .env não for o seu usuário
chown -R 1000:1000 music/data
docker compose -f music/docker-compose.yml up -d
```

Coloque os arquivos de áudio em `MUSIC_LOCATION` (`/storage/music` no `.env`). O volume é somente leitura.

Na primeira subida, abra `https://music.${DOMAIN}` e crie o usuário admin. Depois, em Settings → Users, crie uma conta por pessoa da família. No Chora, a URL do servidor é essa mesma HTTPS, com usuário e senha do Navidrome.

Playlists: crie no web UI ou no app; marque como pública para os outros usuários verem. Links `/share/...` (WhatsApp etc.) exigem o túnel Cloudflare no hostname `music`.

Túnel `hometech` (mesmo padrão do Immich): acrescente o hostname `music.${DOMAIN}` → `http://localhost:8202` no `config.yml` do cloudflared (ver [Exemplos Docker/cloudflared](../Exemplos%20Docker/cloudflared/)) e publique o DNS:

```sh
sudo cloudflared tunnel route dns hometech music.pauloklaus.com.br
sudo systemctl restart cloudflared
```

Endpoints HTTPS esperados (com `DOMAIN` configurado e registros DNS na Cloudflare):

- `traefik.${DOMAIN}` · `dns.${DOMAIN}` · `logs.${DOMAIN}`
- `speedtracker.${DOMAIN}` · `portainer.${DOMAIN}` · `media.${DOMAIN}` · `music.${DOMAIN}`
