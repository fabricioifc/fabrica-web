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
