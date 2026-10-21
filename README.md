# Laboratório de CSS Grid — hospedagem própria

Aplicação estática (um único `index.html`) servida por nginx em Docker.

## Desenvolvimento local

Requer Docker Desktop em execução. Na raiz do projeto, inicie o modo de
desenvolvimento:

```bash
docker compose -f infra/docker-compose.yml up --build
```

Acesse http://localhost:8080. O diretório `site/` é montado no container:
salvar alterações em `site/index.html` atualiza a página sem reconstruir a
imagem. Este modo não usa GHCR nem precisa da variável `IMAGE`.

Para parar, pressione `Ctrl+C` e rode:

```bash
docker compose -f infra/docker-compose.yml down
```

```
laboratorio-css-grid/
├── site/index.html                    # aplicação
├── infra/
│   ├── docker-compose.yml             # desenvolvimento local
│   ├── docker-compose.prod.yml        # produção
│   ├── Dockerfile
│   └── nginx.conf
└── examples/laboratorio-css-grid.html # versão alternativa
```

## 1. Subir o container

Copie a pasta para o servidor de casa e rode, dentro dela:

```bash
docker compose --env-file infra/.env -f infra/docker-compose.prod.yml up -d
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
docker compose --env-file infra/.env -f infra/docker-compose.prod.yml up -d
```

O nginx envia o HTML com `Cache-Control: no-cache`, então os alunos recebem a versão nova ao recarregar.
Se o Cloudflare estiver com cache agressivo no domínio, faça *Purge Cache* após atualizar.

## Deploy automático (GitHub Actions + Tailscale)

A cada `git push` na branch `main`, o workflow `.github/workflows/deploy.yml`:

1. constrói a imagem e publica no GitHub Container Registry (`ghcr.io/<usuario>/<repo>:sha-xxxxxxx`);
2. entra na sua tailnet como uma máquina temporária com a tag `tag:ci`;
3. acessa o servidor por SSH com chave, copia `infra/docker-compose.prod.yml` e roda `docker compose -f docker-compose.prod.yml pull` e `up -d`;
4. confere se `/healthz` respondeu.

Nenhuma porta precisa ser aberta no roteador.

### Configuração (uma vez só)

**Tailscale — políticas de acesso** (admin console → Access controls). A tag `tag:ci`
precisa ter acesso TCP ao servidor nas portas SSH configurada no servidor e `8080`
(verificação de saúde). Por exemplo, se o SSH usa a porta `2222`, permita TCP nas
portas `2222` e `8080` de `tag:ci` para `tag:servidor`.

O servidor precisa ter a tag `tag:servidor`: no admin console, em Machines →
servidor → *Edit ACL tags*. A conexão SSH do workflow usa o `sshd` normal do
servidor e uma chave SSH; ela não usa Tailscale SSH.

**Tailscale — OAuth client** (Settings → OAuth clients → *Generate*): escopo
`auth_keys` (escrita) com a tag `tag:ci`. Guarde o *client ID* e o *secret*.

**No servidor de casa:** instale a chave pública correspondente à chave privada
do GitHub no `~deploy/.ssh/authorized_keys` (ou no diretório home do usuário
configurado). Gere o par de chaves em uma máquina confiável:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f github-actions-deploy
```

Cadastre `github-actions-deploy.pub` no servidor e guarde o conteúdo da chave
privada `github-actions-deploy` no secret `DEPLOY_SSH_PRIVATE_KEY`. Não envie a
chave privada ao repositório. O usuário também precisa ter acesso ao Docker:

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
| Secret | `DEPLOY_SSH_PRIVATE_KEY` | chave privada SSH autorizada no servidor |
| Variable | `DEPLOY_HOST` | nome do servidor no Tailscale (ex.: `servidor-casa`) |
| Variable | `DEPLOY_USER` | `deploy` |
| Variable | `DEPLOY_SSH_PORT` | porta SSH do servidor (padrão `22`) |
| Variable | `DEPLOY_DIR` | opcional; padrão `laboratorio-css-grid` (relativo à home) |

Antes do primeiro deploy automático, pare o container criado manualmente
(`docker compose down` na pasta antiga), pois os dois usam o mesmo nome e a porta 8080.

### Voltar para uma versão anterior

Cada deploy usa uma tag com o hash do commit. No servidor:

```bash
cd ~/laboratorio-css-grid && echo IMAGE=ghcr.io/<usuario>/<repo>:sha-abc1234 > .env && docker compose -f docker-compose.prod.yml up -d
```

Ou, no GitHub, rode de novo (*Re-run*) o workflow de um commit antigo.

## Observações

- O código e o progresso de cada aluno ficam no **navegador dele** (localStorage),
  separados por endereço: o progresso salvo em `grid.seudominio.com.br` não aparece
  no endereço do Tailscale, e vice-versa. Não há banco de dados nem login.
- As fontes vêm do Google Fonts. Sem internet, a página continua funcionando com fontes do sistema.
- Para servir só para o próprio servidor (sem acesso pela rede local/Tailscale),
  troque a porta em `infra/docker-compose.prod.yml` por `"127.0.0.1:8080:80"`.
