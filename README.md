# Assistente de Investimentos com RPA + n8n + IA Generativa

Projeto do lab DIO **"Criando um Assistente de Investimentos com RPA e IA Generativa"**.

Pipeline que coleta clientes de uma página web (RPA em Python), cruza o perfil de cada um com uma base de investimentos (CSV) no n8n, gera um e-mail personalizado com IA (Google Gemini) e envia via Gmail.

---

## Arquitetura

```
[Google Colab - Python/BeautifulSoup]
        │  raspa a tabela de clientes (docs/index.html)
        │  POST JSON { clientes: [...] }
        ▼
[n8n] Webhook (POST /clientes)
   ├─► HTTP Request - Coletar investimentos (docs/data.csv)
   │        └─► Organizar dados (Code: CSV → lista de investimentos)
   ▼
Unificar e tabelar dados (Merge - combine by position)
   ▼
Criar mensagem personalizada (Code: cruza perfil × saldo × produto + mensagem estática)
   ▼
Preparar mensagem/contexto IA (Code: 1 item por cliente + regras de escrita)
   ▼
Message a model (Google Gemini - gemini-3.5-flash-lite)
   ▼
Normalizar e-mails (Code: converte resposta da IA em subject / text_body / html_body)
   ▼
If (valida formato do e-mail via regex)
   ├─ true  ─► Send a message (Gmail)
   └─ false ─► descartado (não envia)
   ▼
Respond to Webhook (retorna o resultado ao Colab)
```

---

## Estrutura do repositório

```
📁 dio-lab-assistente-investimentos-rpa-n8n/
├── 📄 README.md
├── 📁 src/
│   └── 📄 extrair_clientes.ipynb   # RPA: raspa os clientes e envia ao Webhook do n8n
├── 📁 n8n/
│   └── 📄 workflow.json            # Workflow exportado do n8n
└── 📁 docs/
    ├── 📄 index.html               # Página de clientes (fonte do RPA)
    └── 📄 data.csv                 # Opções de investimento por perfil
```

---

## Como executar

1. **Importe o workflow** no n8n: *Workflows → Import from File → `n8n/workflow.json`*.
2. **Configure as credenciais** (não vão no export):
   - **Google Gemini (PaLM) API**: chave gratuita gerada no Google AI Studio.
   - **Gmail OAuth2**: app OAuth criado no Google Cloud (Gmail API ativada, usuário de teste cadastrado e Redirect URI do n8n autorizada).
3. **Ative o workflow** (toggle *Active* / *Publish*).
4. No notebook `src/extrair_clientes.ipynb`, troque `COLE_AQUI_A_URL_DO_SEU_WEBHOOK` pela **Production URL** do nó Webhook (`.../webhook/clientes`).
5. Rode o notebook no Google Colab. A resposta deve vir com `Status n8n: 200`.
6. Confira a execução no n8n em **Executions** (execução verde).

> **Teste x Produção no n8n:** a URL `/webhook-test/` só funciona após clicar em *Listen for test event* e mostra os dados no canvas. A URL `/webhook/` funciona com o workflow ativo e os resultados ficam na aba *Executions*.

---

## Decisões técnicas

### 1. RPA em Python (Colab)
O script faz o que um
