# Contexto do Projeto para Agentes de IA

Este repositório fornece um ambiente local para execução da API não oficial do WhatsApp (WAHA) via Docker Compose.

## Diretrizes de Infraestrutura (NÃO ALTERAR)
- O arquivo `docker-compose.yml` já está configurado e **não deve ser modificado**.
- O container expõe o serviço WAHA na porta `3000`.
- O dashboard está configurado para iniciar sem autenticação via chave ou senha (`WAHA_DASHBOARD_NO_PASSWORD=true` e `WAHA_NO_API_KEY=true`).

## Escopo de Atuação da IA
- **Permitido:** Auxiliar os alunos a criarem requisições HTTP, rotas no n8n ou scripts de teste que consumam a API rodando em `http://localhost:3000`.
- **Proibido:** Modificar as portas do container, alterar variáveis do arquivo `.env` sem aviso ou sugerir mudanças no `Dockerfile`/`docker-compose.yml`.