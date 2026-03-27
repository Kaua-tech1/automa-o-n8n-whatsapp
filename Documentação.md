#  Chatbot WhatsApp para Barbearia — n8n + IA Dual-Agent

> Sistema de automação de atendimento para barbearias com IA, capaz de responder clientes, transcrever áudios e realizar agendamentos automaticamente.

---

##  Sobre o projeto

Sistema de atendimento automatizado desenvolvido em **n8n** para barbearias, integrando dois agentes de IA em cadeia:

- **Agente principal** (OpenAI o4-mini): recebe mensagens, mantém contexto da conversa e decide quando acionar o agendamento
- **Agente de agendamento** (GPT-4.1): sub-agente especializado, operacional e estruturado — verifica disponibilidade, registra e cancela agendamentos com precisão

---

##  Arquitetura

```
WhatsApp (Evolution API)
        │
        ▼
    Webhook n8n
        │
        ├─── [Texto] ──► Set Fields ──► mensagem enviada pelo bot?
        │                               │
        └─── [Áudio] ──► base64 → HTTP Request (Groq Whisper) ──► texto
                                        │
                               Redis (debounce + rate limit)
                                        │
                               Redis Memory (TTL 1h)
                                        │
                               AI Agent (o4-mini)
                               ├── Tool: agendamento ──► verifica Agendamento (Google Calendar)
                               │                    ├──► registra Agendamento (Google Calendar)
                               │                    ├──► deleta Agendamento (Google Calendar)
                               │                    └──► Registro de Agendamento (Supabase)
                               └── Output
                                        │
                               ┌────────┴────────┐
                           [texto]           [MEDIA]
                               │                 │
                         Enviar texto      Google Drive Download
                         (Evolution API)   (tabela de preços)
                                                 │
                                           Enviar Imagem
                                           (HTTP Request)
```

---

##  Funcionalidades

| Recurso | Descrição |
|---|---|
| **Atendimento 24/7** | Responde automaticamente no WhatsApp via Evolution API |
| **Transcrição de áudio** | Converte áudios do cliente em texto via Groq Whisper (HTTP Request) |
| **Memória de contexto** | Mantém histórico da conversa por sessão (Redis, TTL 1h, 20 mensagens) |
| **Debounce de mensagens** | Agrupa mensagens rápidas antes de processar (evita respostas duplicadas) |
| **Rate limiting** | Controla número de requisições por usuário via Redis (RATE LIMIT - Increment) |
| **Bloqueio anti-loop** | Ignora mensagens enviadas pelo próprio bot (nó `mensagem enviada pelo bot?`) |
| **Agendamento inteligente** | Sub-agente verifica disponibilidade e registra no Google Calendar |
| **Cancelamento de agendamento** | Sub-agente consegue deletar agendamentos via `deleta Agendamento` |
| **Banco de dados de clientes** | Salva nome, telefone e data no Supabase após cada agendamento |
| **Envio de tabela de preços** | Baixa imagem do Google Drive e envia via HTTP Request quando o cliente solicita |

---

##  Stack tecnológica

- **n8n** — orquestração do workflow
- **Evolution API** — integração WhatsApp Business
- **OpenAI** — o4-mini (agente principal) + GPT-4.1 (agente de agendamento)
- **Groq** — transcrição de áudio (Whisper large-v3-turbo) via HTTP Request
- **Redis** — memória de conversa + debounce + bloqueio de sessão + rate limiting
- **Supabase** — banco de dados de agendamentos (tabela `Agendamento`)
- **Google Calendar** — agenda da barbearia
- **Google Drive** — armazenamento da tabela de preços

---

##  Como importar no n8n

1. Acesse seu n8n → **Workflows** → **Import from file**
2. Selecione o arquivo `projeto de atendimento e agendamento-n8n.json`
3. Configure as credenciais (veja seção abaixo)
4. Ative o workflow

---

##  Credenciais necessárias

Antes de usar, configure as seguintes credenciais no n8n:

| Credencial | Onde criar | Usado em |
|---|---|---|
| `Evolution API` | Painel da sua instância Evolution | Receber e enviar mensagens |
| `Groq API` | [console.groq.com](https://console.groq.com) | Transcrição de áudio (HTTP Request) |
| `OpenAI API` | [platform.openai.com](https://platform.openai.com) | Agentes de IA |
| `Redis` | Instância Redis (local ou cloud) | Memória, debounce e rate limiting |
| `Supabase API` | [supabase.com](https://supabase.com) | Banco de dados de agendamentos |
| `Google Calendar OAuth2` | Google Cloud Console | Gerenciar agenda |
| `Google Drive OAuth2` | Google Cloud Console | Tabela de preços |

### Banco de dados Supabase

Crie a tabela `Agendamento` com as colunas:

```sql
CREATE TABLE "Coversas" (
  id TEXT PRIMARY KEY,
  "Data" TEXT,
  "Cliente" TEXT,
  "Telefone" TEXT
);
```

---

##  Configurações para personalizar

| Item | Onde alterar | Padrão |
|---|---|---|
| Calendar de registro/verificação | Nós `registra Agendamento` e `verifica Agendamento` | `exemplo@gmail.com` |
| Calendar de cancelamento | Nó `deleta Agendamento` | Calendar separado (grupo) |
| Tabela de preços | Nó `Google Drive Download` (fileId) | ID fixo de exemplo |
| Duração do agendamento | System prompt do `agendamento` | 2 horas |
| Janela de debounce | Nó `Wait1` | 10 segundos |
| TTL da memória | Nó `memoria1` | 3600 segundos (1 hora) |
| Janela de contexto | Nó `memoria` | 20 mensagens |

---

##  Padrões técnicos implementados

- **Dual-agent pattern**: agente orquestrador (`AI Agent`) + sub-agente especializado (`agendamento`) com tools próprias
- **Debounce assíncrono**: Redis list (`Armazenar mensagem no buffer`) + `Wait1` + comparação de última mensagem (`if`)
- **Rate limiting por usuário**: Redis increment com expiração (`RATE LIMIT - Increment`)
- **Session key composta**: `{Instance}_{RemoteJid}` para isolar conversas por instância e usuário
- **Structured output**: sub-agente de agendamento retorna sempre JSON (`status`, `Start`, `End`)
- **Error handling no prompt**: o agente de agendamento tem tratamento explícito de `data_invalida` e `falha_ferramenta`

---

##  Fluxo de agendamento

```
Cliente: "quero marcar para amanhã às 14h"
    │
    ▼
Agente principal extrai intenção + confirmação
    │
    ▼
Chama sub-agente de agendamento (agendamento)
    │
    ▼
verifica Agendamento → Google Calendar (próximos 7 dias)
    │
    ├─── Horário ocupado → retorna {"status": "ocupado"}
    │
    └─── Horário livre + confirmação = true
              │
              ├─► registra Agendamento → Google Calendar
              └─► Registro de Agendamento → Supabase
              └─► retorna {"status": "confirmado"}

Cliente: "quero cancelar"
    │
    ▼
Sub-agente chama deleta Agendamento → Google Calendar
    └─► retorna {"status": "cancelado"}
```

---

##  Estrutura do repositório

```
 chatbot-barbearia-n8n
├── projeto de atendimento e agendamento-n8n.json   # Workflow principal (importar no n8n)
└── Documentação.md
```

---

##  Autor

Desenvolvido por **Kauã B** — estudante de Engenharia de Software com foco em automação com n8n e desenvolvimento de agentes de IA para pequenos negócios.

- Foco: automação comercial para barbearias e clínicas
- Stack principal: n8n · Google Calendar · OpenAI API · Redis · Supabase

---

##  Licença

MIT — livre para uso e adaptação com atribuição.
