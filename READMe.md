# WAHA Local - Ambiente de Testes

Repositório simples para subir a API do WAHA localmente utilizando Docker.

> 🎓 **Uso exclusivamente educacional.** Este projeto foi criado para ensino aos alunos do curso **Geração Tech 4.0 - IA Generativa**. Ele serve apenas para estudo e testes em ambiente local, não para uso em produção.

## 📁 Estrutura do Projeto

```
waha/
├── .env.sample          # Modelo das variáveis de ambiente
├── agentes.md           # Contexto e regras para agentes de IA que auxiliam no projeto
├── docker-compose.yml   # Define e sobe o container do WAHA
├── Dockerfile           # Imagem baseada em devlikeapro/waha, com as variáveis e a porta 3000
└── README.md            # Esta documentação
```

| Arquivo | Para que serve |
|---|---|
| `.env.sample` | Modelo com as variáveis do WAHA (`WAHA_API_KEY`, `WAHA_LOG_LEVEL`, `WAHA_DASHBOARD_USERNAME`, `WAHA_DASHBOARD_PASSWORD`). Copie para `.env` e ajuste os valores. |
| `docker-compose.yml` | Usa a imagem `devlikeapro/waha:latest`, cria o container `waha`, lê as variáveis do `.env` e publica a porta `3000`. |
| `Dockerfile` | Parte da imagem oficial do WAHA, declara as mesmas variáveis de ambiente e expõe a porta `3000`. |
| `agentes.md` | Instruções para assistentes de IA: o que eles podem ajudar a fazer (requisições HTTP, rotas no n8n, scripts de teste) e o que não devem alterar. |

Depois de subir o container, a API e o dashboard ficam disponíveis em `http://localhost:3000`.

## 🚀 Como Executar

### Pré-requisitos
- Docker e Docker Compose instalados na máquina.

### 🐳 O que é Docker?
Docker é uma ferramenta que empacota uma aplicação e tudo o que ela precisa para rodar em um **container**. Assim, ela funciona do mesmo jeito em qualquer computador, sem precisar instalar dependências manualmente.

- **Imagem**: o "pacote" pronto da aplicação (ex.: `devlikeapro/waha`).
- **Container**: a imagem em execução.
- **Docker Compose**: sobe e configura containers a partir de um arquivo (`docker-compose.yml`).

### 💻 Como instalar o Docker
**Windows**
1. Baixe o **Docker Desktop** em https://docs.docker.com/desktop/setup/install/windows-install/
2. Execute o instalador e mantenha a opção **WSL 2** marcada.
3. Reinicie o computador, se solicitado.
4. Abra o Docker Desktop e aguarde o status **Engine running**.

**macOS**: https://docs.docker.com/desktop/setup/install/mac-install/

**Linux**: https://docs.docker.com/engine/install/

O Docker Desktop já inclui o Docker Compose.

Para confirmar a instalação, rode no terminal:
```bash
docker --version
docker compose version
```

> ⚠️ O Docker Desktop precisa estar **aberto e rodando** para os comandos `docker` funcionarem.

📚 Documentação oficial: https://docs.docker.com/

### Passo a Passo
1. Clone o repositório ou baixe os arquivos.
2. Abra o terminal na pasta do projeto.
3. Execute o comando:
   ```bash
   docker compose up -d
   ```