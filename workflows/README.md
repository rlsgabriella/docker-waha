# Agente Financeiro WAHA

Fluxo do n8n que recebe mensagens do WhatsApp (pelo WAHA), entende se a pessoa está registrando um gasto ou um ganho e salva o lançamento no Supabase. O usuário recebe uma resposta no próprio WhatsApp.

O fluxo aceita **texto**, **áudio** e **foto de comprovante**.

## Visão geral

```
WhatsApp -> WAHA -> Webhook -> Filtra e Normaliza -> Tipo de Mídia
                                                       |
            +--------------+---------------+-----------+----------+
            | texto        | imagem        | áudio     | outros   |
            v              v               v           v          |
      Prepara Texto   Baixa + Analisa  Baixa + Transcreve  Avisa Formato
            |              |               |                      |
            +--------------+---------------+                      |
                           v                                      |
                  Agente Financeiro (Gemini + Memória)            |
                           v                                      |
                    Valida Transação                              |
                           v                                      |
                   Transação Válida? --não--+                     |
                           | sim            |                     |
                           v                |                     |
                  Grava no Supabase         |                     |
                    |ok         |erro       |                     |
                    v           v           v                     v
              Confirma     Erro ao      Responde WhatsApp (WAHA) <--+
              Gravação     Gravar
```

## Nó por nó

### 1. WAHA Webhook
Porta de entrada. Toda mensagem que chega no WhatsApp é enviada pelo WAHA para este nó. Ele responde "recebido" ao WAHA na hora e segue com o processamento, para o WAHA não achar que falhou e reenviar a mensagem.

### 2. Filtra e Normaliza
Descarta o que o fluxo não precisa e entrega os dados já organizados. Fica logo depois do webhook para que só mensagens úteis sigam adiante.

Ele **ignora**:
- eventos que não são mensagens (como confirmação de leitura);
- mensagens enviadas pelo próprio bot (evita o bot responder a si mesmo);
- mensagens de grupos, listas de transmissão e canais;
- mensagens repetidas (guarda o ID das últimas 200);
- números que não estão na lista de autorizados (se a lista estiver preenchida).

Ele **entrega** apenas: número do contato, tipo da mensagem (`texto`, `imagem`, `audio` ou `outros`), o texto, o tipo do arquivo e o endereço do arquivo de mídia.

No topo do código do nó há três configurações: o endereço interno do WAHA, a lista `NUMEROS_AUTORIZADOS` e o tamanho máximo do texto. Se a lista de números ficar vazia, qualquer pessoa pode usar o agente.

### 3. Filtra Tipo de Mídia
Um roteador. Olha o tipo definido no passo anterior e manda a mensagem para um de quatro caminhos: texto, imagem, áudio ou outros.

### 4. Caminho do texto
- **Prepara Texto**: pega o texto da mensagem e o coloca no campo `entrada`, que é o que o agente lê.

### 5. Caminho da imagem
- **Baixa Imagem**: baixa a foto do WAHA.
- **Analisa Imagem**: o Gemini descreve o comprovante (estabelecimento, valor, data, itens), sem inventar nada.
- **Prepara Imagem**: junta a descrição (e a legenda, se houver) no campo `entrada`.

### 6. Caminho do áudio
- **Baixa Áudio**: baixa o áudio do WAHA.
- **Transcreve Áudio**: o Gemini transforma a fala em texto.
- **Prepara Áudio**: coloca a transcrição no campo `entrada`.

### 7. Avisa Formato
Para vídeo, documento e mensagens vazias. Monta uma resposta pedindo para o usuário enviar texto, áudio ou foto de comprovante. Essa mensagem vai direto para o envio, sem passar pela IA e sem gravar nada.

### 8. Agente Financeiro Gemini
O "cérebro". Lê a mensagem e decide se ela é uma movimentação financeira real. Regras do agente:
- só trata como lançamento se houver um **valor explícito**;
- nunca inventa valor, categoria ou descrição;
- se faltar informação, pergunta o que falta;
- se a conversa não for financeira (por exemplo, um "oi"), responde de forma curta;
- usa o histórico para completar dados dados antes;
- usa a data de hoje e uma lista fixa de categorias.

A resposta do agente é sempre um JSON com: se é transação, tipo, valor, categoria, descrição, nível de confiança e uma frase para o usuário.

Dois nós ligados a ele:
- **Google Gemini Chat Model**: o modelo de IA usado pelo agente.
- **Memória da Conversa**: guarda as últimas 10 mensagens de cada contato, para o agente lembrar do contexto (por exemplo: "gastei 50" e depois "foi no mercado"). Esta memória fica no próprio n8n e é perdida quando ele reinicia.

### 9. Valida Transação
A trava de segurança. A IA pode errar, então este nó confere o resultado antes de qualquer gravação. Ele só aprova se:
- o JSON puder ser lido;
- o agente tiver marcado como transação;
- o tipo for RECEITA ou DESPESA;
- o valor for maior que zero e no máximo R$ 100.000;
- a descrição tiver pelo menos 3 caracteres;
- a confiança não for baixa.

Se a categoria não estiver na lista, ela vira "Outros". Quando algo é reprovado, o nó monta uma pergunta para o usuário (por exemplo: "Quanto foi?").

### 10. Transação Válida?
Um "se". Se a validação aprovou, segue para a gravação. Se não, vai direto para o envio da resposta, e nada é salvo.

### 11. Grava Transação Supabase
Insere uma linha na tabela `transacoes` com: `user_phone`, `tipo`, `valor`, `categoria` e `descricao`. Os campos `id` e `created_at` são preenchidos pelo próprio banco. Este nó tem duas saídas: sucesso e erro.

### 12. Confirma Gravação e Erro ao Gravar
- **Confirma Gravação**: prepara a mensagem de confirmação (por exemplo: "Despesa registrada: R$ 35,00 em Alimentação").
- **Erro ao Gravar**: se o banco falhar, avisa o usuário em vez de confirmar um registro que não existe.

### 13. Responde WhatsApp (WAHA)
Ponto final de todos os caminhos. Envia o texto da resposta para o contato pelo WAHA.

## Tabela `transacoes` (Supabase)

| Coluna | Tipo | Observação |
|---|---|---|
| id | uuid | gerado pelo banco |
| user_phone | text | número do contato, só dígitos |
| tipo | text | `RECEITA` ou `DESPESA` |
| valor | numeric | maior que zero |
| categoria | text | uma das categorias fixas |
| descricao | text | texto curto |
| created_at | timestamptz | gerado pelo banco |

Recomendações para o banco: ativar o RLS na tabela e criar restrições (`check`) para `tipo`, `valor` e `descricao`, como segunda barreira além da validação do fluxo.

## Como usar

1. No n8n, importe o arquivo `agente-financeiro-waha.example.json`. Ele é um modelo: não traz credenciais nem endereços reais.
2. Depois de importar, abra os nós que usam credenciais e selecione as suas: Google Gemini (modelo, análise de imagem e transcrição de áudio), Supabase e a de acesso ao WAHA (download de mídia e envio de resposta). Os valores ficam guardados no n8n, não neste repositório.
3. No nó **Filtra e Normaliza**, troque `SEU_WAHA_HOST` pelo endereço do seu WAHA e preencha a lista `NUMEROS_AUTORIZADOS`.
4. No WAHA, aponte o webhook de mensagens para a URL de produção do nó **WAHA Webhook**.
5. Ative o fluxo.

## Cuidados

- Nunca publique chaves de API, tokens ou o endereço da sua instância neste repositório.
- O JSON deste repositório é um exemplo sem credenciais. Se você exportar o seu fluxo do n8n, ele traz o nome e o ID das credenciais, o endereço do WAHA e o ID do webhook. Remova esses dados antes de publicar.
- Este fluxo lida com dados financeiros. Mantenha a lista de números autorizados preenchida e proteja o WAHA com chave de API.
