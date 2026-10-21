# Laboratório de CSS Grid — hospedagem própria

Aplicação estática (um único `index.html`) servida por nginx em Docker.

```
laboratorio-css-grid/
├── site/index.html              # a aplicação
├── nginx.conf                   # configuração do nginx
├── Dockerfile
├── docker-compose.yml
└── laboratorio-css-grid.html    # versão do Artifact (não é usada pelo Docker)
```

## 1. Subir o container

Copie a pasta para o servidor de casa e rode, dentro dela:

```bash
docker compose --env-file .env -f infra/docker-compose.yml up -d
```

Teste no próprio servidor:

```bash
curl http://localhost:8080/healthz
```

Deve responder `ok`.

## 2. Cloudflare Tunnel (acesso público dos alunos)

**Se o túnel é gerenciado pelo painel (Zero Trust):**
Networks → Tunnels → seu túnel → *Public Hostname* → *Add a public hostname*

- Subdomain: `grid` (por exemplo) · Domain: seu domínio
- Service: `HTTP` · URL: `localhost:8080`

**Se o túnel usa `config.yml`:** adicione a regra *antes* da regra final `http_status:404`:

```yaml
ingress:
  - hostname: grid.seudominio.com.br
    service: http://localhost:8080
  # ... outras regras ...
  - service: http_status:404
```

Depois reinicie: `sudo systemctl restart cloudflared`.

> Se o `cloudflared` roda **em um container**, `localhost` aponta para o próprio
> container dele. Nesse caso coloque os dois na mesma rede Docker e use
> `http://laboratorio-css-grid:80`, ou use o IP do host (ex.: `http://172.17.0.1:8080`).

## 3. Tailscale (acesso pela sua rede privada)

A porta 8080 já fica acessível pelo Tailscale: `http://<nome-do-servidor>:8080`.

Para ter HTTPS dentro da tailnet (necessário para o botão "Copiar" do editor funcionar):

```bash
sudo tailscale serve --bg 8080
```

Fica disponível em `https://<nome-do-servidor>.<sua-tailnet>.ts.net`.

## Atualizar a aplicação

Edite `site/index.html` e rode de novo:

```bash
docker compose --env-file .env -f infra/docker-compose.yml up -d
```

O nginx envia o HTML com `Cache-Control: no-cache`, então os alunos recebem a versão nova ao recarregar.
Se o Cloudflare estiver com cache agressivo no domínio, faça *Purge Cache* após atualizar.

## Deploy automático (GitHub Actions + Tailscale)

A cada `git push` na branch `main`, o workflow `.github/workflows/deploy.yml`:

1. constrói a imagem e publica no GitHub Container Registry (`ghcr.io/<usuario>/<repo>:sha-xxxxxxx`);
2. entra na sua tailnet como uma máquina temporária com a tag `tag:ci`;
3. acessa o servidor por SSH (Tailscale SSH), copia `deploy/docker-compose.yml` e roda `docker compose pull && up -d`;
4. confere se `/healthz` respondeu.

Nenhuma porta precisa ser aberta no roteador.

### Configuração (uma vez só)

**Tailscale — políticas de acesso** (admin console → Access controls). Acrescente:

```jsonc
"tagOwners": {
  "tag:ci":       ["autogroup:admin"],
  "tag:servidor": ["autogroup:admin"]
},
"ssh": [
  {
    "action": "accept",
    "src":    ["tag:ci"],
    "dst":    ["tag:servidor"],
    "users":  ["deploy"]          // usuário Linux usado no deploy
  }
]
```

O Tailscale só permite SSH de uma máquina com tag para outra **com tag**, então
o servidor precisa da `tag:servidor`: no admin console, em Machines → servidor →
*Edit ACL tags*. Máquinas com tag não expiram a chave, o que é bom para um servidor.

Se você usa regras `acls` (e não só o padrão "tudo liberado"), libere também
`tag:ci` → servidor nas portas `22` e `8080`.

**Tailscale — OAuth client** (Settings → OAuth clients → *Generate*): escopo
`auth_keys` (escrita) com a tag `tag:ci`. Guarde o *client ID* e o *secret*.

**No servidor de casa:**

```bash
sudo tailscale set --ssh
```

```bash
sudo usermod -aG docker deploy
```

Se o repositório for **privado**, a imagem no GHCR também será. Faça login uma vez
no servidor, como o usuário `deploy`, com um token clássico do GitHub com escopo `read:packages`:

```bash
docker login ghcr.io -u SEU_USUARIO_GITHUB
```

(Alternativa: em GitHub → Packages → a imagem → *Package settings*, torne-a pública.)

**No GitHub** (repo → Settings → Environments → *New environment* `producao`):

| Tipo | Nome | Valor |
|---|---|---|
| Secret | `TS_OAUTH_CLIENT_ID` | client ID do OAuth do Tailscale |
| Secret | `TS_OAUTH_SECRET` | secret do OAuth do Tailscale |
| Variable | `DEPLOY_HOST` | nome do servidor no Tailscale (ex.: `servidor-casa`) |
| Variable | `DEPLOY_USER` | `deploy` |
| Variable | `DEPLOY_DIR` | opcional; padrão `laboratorio-css-grid` (relativo à home) |

Antes do primeiro deploy automático, pare o container criado manualmente
(`docker compose down` na pasta antiga), pois os dois usam o mesmo nome e a porta 8080.

### Voltar para uma versão anterior

Cada deploy usa uma tag com o hash do commit. No servidor:

```bash
cd ~/laboratorio-css-grid && echo IMAGE=ghcr.io/<usuario>/<repo>:sha-abc1234 > .env && docker compose up -d
```

Ou, no GitHub, rode de novo (*Re-run*) o workflow de um commit antigo.

## Observações

- O código e o progresso de cada aluno ficam no **navegador dele** (localStorage),
  separados por endereço: o progresso salvo em `grid.seudominio.com.br` não aparece
  no endereço do Tailscale, e vice-versa. Não há banco de dados nem login.
- As fontes vêm do Google Fonts. Sem internet, a página continua funcionando com fontes do sistema.
- Para servir só para o próprio servidor (sem acesso pela rede local/Tailscale),
  troque a porta no `docker-compose.yml` por `"127.0.0.1:8080:80"`.
