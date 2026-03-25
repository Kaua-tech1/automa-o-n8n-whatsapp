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
        ├─── [Texto] ──► Set Fields ──► Filtro fromMe
        │                               │
        └─── [Áudio] ──► base64 → Groq Whisper ──► texto
                                        │
                               Redis (debounce 1s)
                                        │
                               Redis Memory (TTL 1h)
                                        │
                               AI Agent (o4-mini)
                               ├── Tool: agendamento ──► Google Calendar
                               │                    └──► Supabase
                               └── Output
                                        │
                               ┌────────┴────────┐
                           [texto]           [MEDIA]
                               │                 │
                         Evolution API     Google Drive
                         (enviar msg)      (tabela de preços)
                               │
                         Google Sheets (log de conversas)
```

---

##  Funcionalidades

| Recurso | Descrição |
|---|---|
| **Atendimento 24/7** | Responde automaticamente no WhatsApp via Evolution API |
| **Transcrição de áudio** | Converte áudios do cliente em texto via Groq Whisper |
| **Memória de contexto** | Mantém histórico da conversa por sessão (Redis, TTL 1h, 20 mensagens) |
| **Debounce de mensagens** | Agrupa mensagens rápidas antes de processar (evita respostas duplicadas) |
| **Agendamento inteligente** | Sub-agente verifica disponibilidade e registra no Google Calendar |
| **Banco de dados de clientes** | Salva nome, telefone e data no Supabase após cada agendamento |
| **Envio de tabela de preços** | Envia imagem via Google Drive quando o cliente solicita preços |
| **Log de conversas** | Registra cada interação no Google Sheets para análise |
| **Bloqueio anti-loop** | Ignora mensagens enviadas pelo próprio bot (flag `fromMe`) |

---

##  Stack tecnológica

- **n8n** — orquestração do workflow
- **Evolution API** — integração WhatsApp Business
- **OpenAI** — o4-mini (agente principal) + GPT-4.1 (agente de agendamento)
- **Groq** — transcrição de áudio (Whisper large-v3-turbo)
- **Redis** — memória de conversa + debounce + bloqueio de sessão
- **Supabase** — banco de dados de agendamentos (tabela `Coversas`)
- **Google Calendar** — agenda da barbearia
- **Google Drive** — armazenamento da tabela de preços
- **Google Sheets** — log de conversas para análise

---

##  Como importar no n8n

1. Acesse seu n8n → **Workflows** → **Import from file**
2. Selecione o arquivo `banco_de_dados_e_agendamento__0_1_.json`
3. Configure as credenciais (veja seção abaixo)
4. Ative o workflow

---

##  Credenciais necessárias

Antes de usar, configure as seguintes credenciais no n8n:

| Credencial | Onde criar | Usado em |
|---|---|---|
| `Evolution API` | Painel da sua instância Evolution | Receber e enviar mensagens |
| `Groq API` | [console.groq.com](https://console.groq.com) | Transcrição de áudio |
| `OpenAI API` | [platform.openai.com](https://platform.openai.com) | Agentes de IA |
| `Redis` | Instância Redis (local ou cloud) | Memória e debounce |
| `Supabase API` | [supabase.com](https://supabase.com) | Banco de dados de agendamentos |
| `Google Calendar OAuth2` | Google Cloud Console | Gerenciar agenda |
| `Google Drive OAuth2` | Google Cloud Console | Tabela de preços |
| `Google Sheets OAuth2` | Google Cloud Console | Log de conversas |

### Banco de dados Supabase

Crie a tabela `Coversas` com as colunas:

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
| Calendar da barbearia | Nós `registraAgendamento1` e `verificaAgendamento1` | Gmail do dono |
| Tabela de preços | Nó `Google Drive Download1` (fileId) | ID fixo de exemplo |
| Duração do agendamento | System prompt do `agendamento1` | 2 horas |
| Janela de debounce | Nó `Wait1` | 1 segundo |
| TTL de bloqueio (anti-loop) | Nó `chave block1` | 300 segundos |
| TTL da memória | Nó `memoria1` | 3600 segundos (1 hora) |

---

##  Padrões técnicos implementados

- **Dual-agent pattern**: agente orquestrador + sub-agente especializado com tools próprias
- **Debounce assíncrono**: Redis list + Wait + comparação de última mensagem
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
Chama sub-agente de agendamento (agentTool)
    │
    ▼
verificaAgendamento → Google Calendar (próximos 7 dias)
    │
    ├─── Horário ocupado → retorna {"status": "ocupado"}
    │
    └─── Horário livre + confirmação = true
              │
              ├─► registraAgendamento → Google Calendar
              └─► Registro de Agendamento → Supabase
              └─► retorna {"status": "confirmado"}
```

---

##  Estrutura do repositório

```
 chatbot-barbearia-n8n
├── banco_de_dados_e_agendamento__0_1_.json   # Workflow principal (importar no n8n)
└── README.md
```

---

##  Autor

Desenvolvido por **Kauã B** — estudante de Engenharia de Software com foco em automação com n8n e desenvolvimento de agentes de IA para pequenos negócios.

- Foco: automação comercial para barbearias e clínicas
- Stack principal: n8n · Python · OpenAI API · Redis · Supabase

---

##  Licença

MIT — livre para uso e adaptação com atribuição.
