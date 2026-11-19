# WAHA + n8n com Docker (local e AWS EC2)

Repositório para subir a API do WhatsApp **WAHA** e o **n8n** com Docker, tanto na máquina local quanto em uma instância EC2 na AWS, e conectar o n8n ao Kiro por MCP.

> 🎓 **Uso exclusivamente educacional.** Este projeto foi criado para ensino aos alunos do curso **Geração Tech 4.0 - IA Generativa**. Serve para estudo e testes, não para produção.

## 📁 Estrutura do projeto

```
docker-waha/
├── .env.example         # Modelo das variáveis (sem valores reais)
├── .env                 # Suas credenciais reais (NÃO vai para o GitHub)
├── .gitignore           # Protege .env, mcp.json, chaves .pem, dados do n8n
├── mcp.example.json     # Modelo da conexão MCP com o n8n (sem token)
├── mcp.json             # Sua conexão MCP real (NÃO vai para o GitHub)
├── agentes.md           # Regras para agentes de IA que ajudam no projeto
├── docker-compose.yml   # WAHA + n8n para rodar localmente
├── Dockerfile           # Imagem baseada em devlikeapro/waha
└── README.md            # Esta documentação
```

| Arquivo | Para que serve |
|---|---|
| `.env.example` | Lista todas as variáveis do projeto: WAHA, n8n, MCP e AWS. Copie para `.env` e preencha. |
| `.env` | Valores reais. Está no `.gitignore`. |
| `mcp.example.json` | Exemplo de configuração do servidor MCP do n8n, com um texto no lugar do token. |
| `mcp.json` | Cópia local da sua configuração MCP, com o token real. Está no `.gitignore`. |
| `docker-compose.yml` | Sobe o container `waha` (porta 3000) e o container `n8n` (porta 5678). |
| `Dockerfile` | Parte da imagem oficial do WAHA e expõe a porta 3000. |

## 🔐 Credenciais e segurança

Nenhuma credencial fica no código. Tudo vive no `.env`, que o `.gitignore` bloqueia.

1. Copie o modelo: `Copy-Item .env.example .env` (PowerShell) ou `cp .env.example .env`.
2. Preencha os valores no `.env`.
3. Antes de cada commit, confira com `git status` que `.env`, `mcp.json` e arquivos `.pem` não aparecem.

Variáveis do `.env`:

| Grupo | Variáveis | O que são |
|---|---|---|
| WAHA | `WAHA_API_KEY`, `WAHA_DASHBOARD_USERNAME`, `WAHA_DASHBOARD_PASSWORD`, `WAHA_SWAGGER_*`, `WAHA_LOG_LEVEL` | Chave da API e login do dashboard e do Swagger |
| n8n | `N8N_URL`, `N8N_OWNER_EMAIL`, `N8N_OWNER_PASSWORD`, `N8N_ENCRYPTION_KEY`, `N8N_LOCAL_API_KEY` | Endereço, login do usuário dono, chave que criptografa as credenciais salvas e API key do n8n local |
| MCP | `N8N_MCP_URL`, `N8N_MCP_TOKEN` | Endereço e token do servidor MCP do n8n |
| AWS | `AWS_REGION`, `EC2_INSTANCE_ID`, `EC2_PUBLIC_IP`, `EC2_ALLOCATION_ID`, `EC2_SECURITY_GROUP`, `EC2_KEY_NAME`, `EC2_KEY_FILE`, `EC2_SSH_USER` | Identificadores e chave SSH da instância |

> ⚠️ Se algum valor já foi enviado ao GitHub por engano, apague-o do histórico e **gere um novo** (troque senha, API key e token). Remover o arquivo em um commit novo não basta.

## 💻 Rodar localmente

Pré-requisito: Docker com Docker Compose.

```bash
docker compose up -d
```

- WAHA: `http://localhost:3000`
- n8n: `http://localhost:5678`

> O Docker Desktop é pesado. Uma alternativa é instalar só o Docker Engine dentro do WSL2 (Ubuntu), que usa menos memória e roda o mesmo `docker-compose.yml`.

## ☁️ Instância na AWS (EC2)

Para não sobrecarregar a máquina local, WAHA e n8n também foram colocados em uma instância EC2.

### O que foi criado

| Recurso | Configuração |
|---|---|
| Região | `us-east-1` |
| Instância | `t3.micro` (2 vCPU, 1 GB de RAM), Amazon Linux 2023 |
| Disco | 20 GB gp3 |
| Memória extra | 2 GB de swap, porque 1 GB de RAM é pouco para WAHA e n8n juntos |
| IP fixo | 1 Elastic IP associado à instância |
| Chave SSH | Key pair RSA `.pem`, salva em `~/.ssh` (fora do repositório) |
| Security group | Porta 22 (SSH) só para o IP de quem criou. Portas 3000 (WAHA) e 5678 (n8n) abertas para a internet |
| Docker | Docker Engine + plugin Compose, instalados pelo *user data* na criação |

### O que roda nela

Um `docker-compose.yml` próprio da instância, em `/opt/app`, com dois containers:

- **waha** (`devlikeapro/waha`, motor NOWEB), porta 3000, com API key e senha do dashboard e do Swagger ativas. As sessões do WhatsApp ficam no volume `waha_data`.
- **n8n** (`n8nio/n8n`), porta 5678, com `N8N_HOST`, `WEBHOOK_URL` e `N8N_SECURE_COOKIE=false`, porque o acesso é por HTTP. Os dados ficam no volume `n8n_data`.

Os dois usam `restart: always`, então sobem sozinhos quando a instância liga.

### Dados do n8n importados

As credenciais e os workflows do n8n que rodava localmente (via `npx n8n`) foram levados para o n8n da instância:

1. Os arquivos `config` (com a `encryptionKey`) e `database.sqlite` da pasta `~/.n8n` foram empacotados.
2. Foram enviados por `scp` e extraídos dentro do volume `n8n_data`.
3. O container foi reiniciado.

A `encryptionKey` precisa ser a mesma. Sem ela, as credenciais importadas não descriptografam. Os dados antigos da instância ficaram guardados em `_prev` dentro do volume. As sessões do WhatsApp **não** foram migradas: é preciso escanear o QR code de novo no WAHA.

### Acessos

Os endereços e senhas estão no `.env`.

- WAHA: `http://<EC2_PUBLIC_IP>:3000` (usuário e senha do dashboard)
- n8n: `http://<EC2_PUBLIC_IP>:5678` (usuário dono)
- SSH: `ssh -i <EC2_KEY_FILE> ec2-user@<EC2_PUBLIC_IP>`

### Ligar, desligar e custos

Desligar a instância para de cobrar o processamento. O disco e o IP continuam cobrando valores pequenos.

```powershell
aws ec2 stop-instances  --region us-east-1 --instance-ids <EC2_INSTANCE_ID>
aws ec2 start-instances --region us-east-1 --instance-ids <EC2_INSTANCE_ID>
```

- O IP fixo sobrevive ao desligamento. Se você liberar o Elastic IP, o endereço muda ao religar e é preciso atualizar `N8N_HOST` e `WEBHOOK_URL` no compose da instância.
- Custo aproximado, com preços de tabela: `t3.micro` cerca de US$ 0,0104/hora, IPv4 público US$ 0,005/hora, disco de 20 GB cerca de US$ 1,60/mês. Confira os valores reais em **Billing → Cost Explorer** e o saldo em **Billing → Credits**.

### Excluir tudo

```powershell
aws ec2 terminate-instances --region us-east-1 --instance-ids <EC2_INSTANCE_ID>
aws ec2 wait instance-terminated --region us-east-1 --instance-ids <EC2_INSTANCE_ID>
aws ec2 release-address --region us-east-1 --allocation-id <EC2_ALLOCATION_ID>
aws ec2 delete-security-group --region us-east-1 --group-id <EC2_SECURITY_GROUP>
aws ec2 delete-key-pair --region us-east-1 --key-name <EC2_KEY_NAME>
```

### Riscos desta configuração

- O acesso é **HTTP sem HTTPS**: senha, API key e token trafegam sem criptografia.
- As portas 3000 e 5678 estão abertas para a internet. Use só para testes e desligue ou exclua depois.
- Crie o usuário dono do n8n assim que subir, antes que outra pessoa o faça.

## 🔌 Conectar o n8n ao Kiro (MCP)

O n8n tem um servidor MCP de instância. Ele deixa o Kiro listar e executar os workflows que você liberar.

1. No n8n, abra **Settings → Instance-level MCP** e confirme que está **Enabled**.
2. Em **Workflows enabled**, marque os workflows que o Kiro pode usar. Com 0 marcados, o Kiro conecta mas não vê nenhum.
3. Clique em **Connect**, abra a aba **API key** e copie o token. O n8n mostra o token uma vez só.
4. Coloque o token em `N8N_MCP_TOKEN` no `.env`.
5. Adicione o servidor ao arquivo de configuração do Kiro (`~/.kiro/settings/mcp.json`, ou `.kiro/settings/mcp.json` no workspace), usando `mcp.example.json` como modelo:
   ```json
   {
     "mcpServers": {
       "n8n-mcp": {
         "type": "http",
         "url": "http://<EC2_PUBLIC_IP>:5678/mcp-server/http",
         "headers": { "Authorization": "Bearer <N8N_MCP_TOKEN>" }
       }
     }
   }
   ```
6. Salve. O Kiro reconecta sozinho. Confira no painel **MCP Servers** do Kiro e em **Connected clients** no n8n.

O arquivo `mcp.json` na raiz deste repositório é só uma cópia local e o Kiro **não** o lê dali. Para valer, o conteúdo precisa estar em um dos dois caminhos do passo 5. Nunca envie o `mcp.json` real ao GitHub.

O servidor `n8n-mesa-certa`, que já existia na configuração do Kiro, aponta para o n8n local (`127.0.0.1:5678`) e é independente deste.

## 🐳 Noções de Docker

- **Imagem**: o pacote pronto da aplicação (ex.: `devlikeapro/waha`).
- **Container**: a imagem em execução.
- **Volume**: pasta que guarda os dados fora do container, para sobreviverem a reinícios.
- **Docker Compose**: sobe vários containers a partir de um arquivo (`docker-compose.yml`).

Documentação oficial: https://docs.docker.com/
