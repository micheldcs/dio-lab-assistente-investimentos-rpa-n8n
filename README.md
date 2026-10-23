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
├── 📁 rpa/
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
4. No notebook `rpa/extrair_clientes.ipynb`, troque `COLE_AQUI_A_URL_DO_SEU_WEBHOOK` pela **Production URL** do nó Webhook (`.../webhook/clientes`).
5. Rode o notebook no Google Colab. A resposta deve vir com `Status n8n: 200`.
6. Confira a execução no n8n em **Executions** (execução verde).

> **Teste x Produção no n8n:** a URL `/webhook-test/` só funciona após clicar em *Listen for test event* e mostra os dados no canvas. A URL `/webhook/` funciona com o workflow ativo e os resultados ficam na aba *Executions*.

---

## Decisões técnicas

### 1. RPA em Python (Colab)
O script faz o que um humano faria: abre a página, lê a tabela e envia os dados para outro sistema. Mantive o código do enunciado e apenas ativei o envio ao Webhook (`requests.post` com `json={"clientes": clientes}`), seguindo o protocolo do lab de não tratar os dados na origem — o tratamento acontece no n8n.

### 2. Leitura do CSV no n8n
O `data.csv` é lido por HTTP Request e convertido por um nó Code. O código:
- detecta o separador automaticamente (`,` ou `;`);
- converte o campo `minimo` para número, aceitando formatos como `1000`, `1.000,50` ou `R$ 500,00`.

### 3. Junção dos dados (Merge por posição)
O Webhook entrega 1 item (lista de clientes) e o CSV entrega 1 item (lista de investimentos). O Merge em *combine by position* junta os dois em um único item, simplificando o cruzamento no passo seguinte.

### 4. Regra de recomendação (determinística)
Para cada cliente:
1. filtra os produtos do **mesmo perfil** (comparação sem acento e sem diferenciar maiúsculas);
2. ordena pelo valor mínimo;
3. se o saldo cobre algum produto → escolhe o de **maior mínimo que o saldo cobre** (`saldo_ok`);
4. se não cobre nenhum → indica o de menor mínimo e quanto falta (`saldo_insuficiente`);
5. se não há produto para o perfil → `sem_produto_para_perfil`.

A recomendação é feita **por regra em código, não pela IA**. Assim a IA nunca inventa produto, valor mínimo ou rentabilidade — ela só redige o texto a partir de dados já validados.

### 5. MVP: mensagens estáticas por perfil
Cada cliente recebe 3 variações de mensagem (uma é sorteada), acrescidas do tom do perfil exigido no enunciado:
- **Conservador:** foco em renda fixa e segurança;
- **Moderado:** mix equilibrado entre renda fixa e variável;
- **Arrojado:** ênfase em ações e maior potencial de retorno.

### 6. IA generativa (Google Gemini)
- Modelo **gemini-3.5-flash-lite**: plano gratuito, rápido e suficiente para e-mails curtos.
- O nó *Preparar mensagem/contexto IA* quebra a lista em **1 item por cliente** e envia junto regras de escrita (`copy_rules`): idioma pt-BR, tom amigável e profissional, 4–6 linhas, sem promessas de ganho, sem garantias e sem linguagem agressiva.
- A IA recebe a mensagem estática como **referência** e devolve um JSON com `subject`, `text_body` e `html_body`.

### 7. Normalização e plano B
Modelos podem responder fora do formato pedido (ex.: JSON dentro de ```` ```json ````). O nó *Normalizar e-mails*:
- extrai o texto da resposta independentemente do formato do nó (Gemini ou OpenAI);
- remove marcações e converte para objeto;
- se a IA falhar, usa a **mensagem estática do MVP** como fallback — o fluxo nunca quebra e o campo `meta.origem_texto` indica se o texto veio da IA ou do fallback.

### 8. Validação antes do envio
Um nó **If** valida o destinatário com a regex `^[^@\s]+@[^@\s]+\.[^@\s]+$`. E-mails vazios ou mal formados não são enviados.

### 9. Segurança
- A URL do Webhook **não** está publicada no notebook (placeholder).
- As credenciais (Gemini e Gmail) **não** são incluídas no `workflow.json` exportado.
- Os clientes são fictícios. Nos testes, o envio foi direcionado a um e-mail próprio, já que `email.com` é um provedor real e os endereços da base poderiam existir.

---

## Entregáveis

### MVP (Mínimo Viável)

- [x] Repositório forkado com o workflow N8N implementado
- [x] Workflow N8N exportado (`n8n/workflow.json`) com mensagens estáticas
- [x] Script de RPA integrado ao Webhook do N8N
- [x] Print ou vídeo demonstrando o fluxo funcionando de ponta a ponta

### Desafio Completo

- [x] Todos os itens do MVP
- [x] Integração com Agente de IA no N8N (Google Gemini)
- [x] Mensagens geradas dinamicamente via LLM
- [x] Documentação explicando as decisões técnicas

---

## Evidências

- Colab retornando `Status n8n: 200` com os resultados por cliente.
- Execução de produção com sucesso na aba *Executions* do n8n.
- E-mails gerados pela IA recebidos na caixa de entrada.

*(adicione aqui os prints ou o link do vídeo)*

---

## Tecnologias

| Etapa | Ferramenta |
|---|---|
| Hospedagem dos dados | GitHub Pages |
| Extração (RPA) | Python + BeautifulSoup (Google Colab) |
| Orquestração | n8n (self-hosted) |
| IA generativa | Google Gemini (gemini-3.5-flash-lite) |
| Envio | Gmail (OAuth2) |
