# Projeto 1 — Qualificador de Leads com IA

> Stack: n8n · OpenAI/Gemini · Google Sheets · Gmail/SMTP  
> Tempo estimado: 3–4 horas  
> Valor de venda: $300–600

---

## O que esse projeto faz

1. Cliente preenche um formulário (Typeform, Tally ou página HTML simples)
2. n8n recebe os dados via webhook
3. IA analisa e classifica o lead (quente / morno / frio) com justificativa
4. Resultado é salvo automaticamente no Google Sheets
5. Se o lead for **quente**, dispara email de notificação imediata para o dono do negócio

---

## Estrutura do repositório

```
qualificador-leads/
├── docker-compose.yml
├── .env.example
├── .env                     ← criado por você, nunca commitar
├── workflows/
│   └── qualificador.json    ← workflow exportado do n8n
├── form/
│   └── index.html           ← formulário HTML simples (opcional)
└── README.md
```

---

## Pré-requisitos

- Docker + Docker Compose instalados
- Conta Google (para Sheets + Gmail)
- Chave de API da OpenAI **ou** Google Gemini (ambas têm free tier)
- Conta no [Tally.so](https://tally.so) (gratuito) — opcional, para o formulário

---

## Passo 1 — Subir o n8n com Docker

### `docker-compose.yml`

```yaml
version: "3.8"

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: qualificador-leads-n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_PASSWORD}
      - N8N_HOST=${N8N_HOST:-localhost}
      - N8N_PORT=5678
      - N8N_PROTOCOL=${N8N_PROTOCOL:-http}
      - WEBHOOK_URL=${WEBHOOK_URL:-http://localhost:5678}
      - GENERIC_TIMEZONE=America/Recife
    volumes:
      - n8n_data:/home/node/.n8n
    env_file:
      - .env

volumes:
  n8n_data:
```

### `.env.example`

```env
# n8n auth
N8N_USER=admin
N8N_PASSWORD=troque_isso_aqui

# Usado quando fizer deploy em produção
N8N_HOST=localhost
N8N_PROTOCOL=http
WEBHOOK_URL=http://localhost:5678

# Chaves de API (adicione apenas a que for usar)
OPENAI_API_KEY=sk-...
GEMINI_API_KEY=AIza...
```

### Subindo o ambiente

```bash
# Clone ou crie a pasta do projeto
mkdir qualificador-leads && cd qualificador-leads

# Copie o .env
cp .env.example .env
# Edite o .env com suas credenciais

# Sobe o n8n
docker compose up -d

# Acesse em: http://localhost:5678
```

---

## Passo 2 — Criar o formulário de captação

### Opção A — Tally.so (recomendado, zero código)

1. Acesse [tally.so](https://tally.so) e crie conta gratuita
2. Crie um formulário com os campos abaixo
3. Em **Integrations → Webhooks**, cole a URL do webhook do n8n (gerada no Passo 3)

### Opção B — HTML simples (`form/index.html`)

Use quando quiser embutir o formulário no site do cliente.

```html
<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <title>Fale com a gente</title>
  <style>
    body { font-family: sans-serif; max-width: 480px; margin: 60px auto; padding: 0 1rem; }
    input, select, textarea { width: 100%; padding: 10px; margin: 6px 0 16px; border: 1px solid #ccc; border-radius: 6px; box-sizing: border-box; }
    button { background: #4f46e5; color: white; padding: 12px 24px; border: none; border-radius: 6px; cursor: pointer; width: 100%; font-size: 16px; }
    label { font-weight: 500; font-size: 14px; }
  </style>
</head>
<body>
  <h2>Quero saber mais</h2>
  <form id="leadForm">
    <label>Nome completo</label>
    <input type="text" name="nome" required>

    <label>Email</label>
    <input type="email" name="email" required>

    <label>Empresa</label>
    <input type="text" name="empresa">

    <label>Cargo</label>
    <input type="text" name="cargo">

    <label>Quantos funcionários tem sua empresa?</label>
    <select name="tamanho_empresa">
      <option value="1-10">1 a 10</option>
      <option value="11-50">11 a 50</option>
      <option value="51-200">51 a 200</option>
      <option value="200+">Mais de 200</option>
    </select>

    <label>Qual sua principal necessidade?</label>
    <textarea name="necessidade" rows="4" placeholder="Descreva brevemente..."></textarea>

    <label>Qual é seu orçamento mensal para essa solução?</label>
    <select name="orcamento">
      <option value="menos-500">Menos de R$500</option>
      <option value="500-2000">R$500 a R$2.000</option>
      <option value="2000-5000">R$2.000 a R$5.000</option>
      <option value="5000+">Mais de R$5.000</option>
    </select>

    <button type="submit">Enviar</button>
  </form>

  <script>
    const WEBHOOK_URL = 'COLE_AQUI_A_URL_DO_WEBHOOK_N8N';

    document.getElementById('leadForm').addEventListener('submit', async (e) => {
      e.preventDefault();
      const data = Object.fromEntries(new FormData(e.target));
      await fetch(WEBHOOK_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data)
      });
      e.target.innerHTML = '<p>✅ Recebemos sua mensagem! Entraremos em contato em breve.</p>';
    });
  </script>
</body>
</html>
```

---

## Passo 3 — Montar o workflow no n8n

Acesse `http://localhost:5678` e crie um novo workflow. Monte os nós na ordem abaixo.

---

### Nó 1 — Webhook (trigger)

- Tipo: **Webhook**
- HTTP Method: `POST`
- Path: `qualificador-leads`
- Authentication: `None` (para teste; adicione Basic Auth em produção)
- Response Mode: `When Last Node Finishes`

> A URL gerada será algo como:  
> `http://localhost:5678/webhook/qualificador-leads`  
> Cole essa URL no formulário.

---

### Nó 2 — Formatar dados recebidos

- Tipo: **Code** (JavaScript)
- Descrição: normaliza os dados antes de mandar pra IA

```javascript
const body = $input.first().json.body ?? $input.first().json;

return [{
  json: {
    nome: body.nome ?? body.name ?? 'Não informado',
    email: body.email ?? 'Não informado',
    empresa: body.empresa ?? body.company ?? 'Não informado',
    cargo: body.cargo ?? body.role ?? 'Não informado',
    tamanho_empresa: body.tamanho_empresa ?? body.company_size ?? 'Não informado',
    necessidade: body.necessidade ?? body.message ?? 'Não informado',
    orcamento: body.orcamento ?? body.budget ?? 'Não informado',
    recebido_em: new Date().toLocaleString('pt-BR', { timeZone: 'America/Recife' })
  }
}];
```

---

### Nó 3 — Classificar com IA

#### Opção A — OpenAI

- Tipo: **OpenAI**
- Resource: `Chat`
- Model: `gpt-4o-mini` (mais barato, ótimo pra isso)
- System Message:

```
Você é um especialista em qualificação de leads de vendas B2B.
Analise os dados do lead e retorne APENAS um JSON válido, sem markdown, sem explicações.
```

- User Message (montar com expressão):

```
Analise este lead e retorne um JSON com a seguinte estrutura:
{
  "classificacao": "QUENTE" | "MORNO" | "FRIO",
  "score": número de 0 a 100,
  "justificativa": "explicação em 2-3 frases",
  "proxima_acao": "o que o time de vendas deve fazer",
  "urgencia": "ALTA" | "MEDIA" | "BAIXA"
}

Dados do lead:
- Nome: {{ $json.nome }}
- Empresa: {{ $json.empresa }}
- Cargo: {{ $json.cargo }}
- Tamanho da empresa: {{ $json.tamanho_empresa }}
- Necessidade: {{ $json.necessidade }}
- Orçamento: {{ $json.orcamento }}

Critérios de classificação:
- QUENTE: orçamento acima de R$2.000, cargo de decisão (CEO, diretor, gerente), necessidade clara
- MORNO: orçamento médio ou cargo intermediário, necessidade definida
- FRIO: orçamento baixo, cargo operacional, necessidade vaga
```

#### Opção B — Google Gemini (via HTTP Request)

Se preferir Gemini, use um nó **HTTP Request**:

- Method: `POST`
- URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key={{ $env.GEMINI_API_KEY }}`
- Body (JSON):

```json
{
  "contents": [{
    "parts": [{
      "text": "Analise este lead B2B e retorne APENAS um JSON válido (sem markdown):\n{\"classificacao\": \"QUENTE|MORNO|FRIO\", \"score\": 0-100, \"justificativa\": \"...\", \"proxima_acao\": \"...\", \"urgencia\": \"ALTA|MEDIA|BAIXA\"}\n\nLead:\nNome: {{ $json.nome }}\nEmpresa: {{ $json.empresa }}\nCargo: {{ $json.cargo }}\nTamanho: {{ $json.tamanho_empresa }}\nNecessidade: {{ $json.necessidade }}\nOrçamento: {{ $json.orcamento }}"
    }]
  }]
}
```

---

### Nó 4 — Parsear resposta da IA

- Tipo: **Code** (JavaScript)
- Descrição: extrai o JSON retornado pela IA

```javascript
// Pega o texto da resposta (adapte conforme a IA usada)
// Para OpenAI:
const texto = $input.first().json.choices[0].message.content;

// Para Gemini, troque por:
// const texto = $input.first().json.candidates[0].content.parts[0].text;

// Remove blocos de markdown se a IA insistir em colocar
const limpo = texto.replace(/```json|```/g, '').trim();

let analise;
try {
  analise = JSON.parse(limpo);
} catch (e) {
  analise = {
    classificacao: 'ERRO',
    score: 0,
    justificativa: 'Falha ao parsear resposta da IA: ' + texto,
    proxima_acao: 'Verificar manualmente',
    urgencia: 'MEDIA'
  };
}

// Mescla com os dados originais do lead
const lead = $('Formatar dados recebidos').first().json;

return [{
  json: {
    ...lead,
    ...analise
  }
}];
```

---

### Nó 5 — Salvar no Google Sheets

- Tipo: **Google Sheets**
- Operation: `Append`
- Spreadsheet: crie uma planilha chamada `Leads Qualificados`
- Sheet: `Página1`
- Columns (mapeie os campos):

| Coluna na planilha | Valor do n8n |
|---|---|
| Data | `{{ $json.recebido_em }}` |
| Nome | `{{ $json.nome }}` |
| Email | `{{ $json.email }}` |
| Empresa | `{{ $json.empresa }}` |
| Cargo | `{{ $json.cargo }}` |
| Orçamento | `{{ $json.orcamento }}` |
| Necessidade | `{{ $json.necessidade }}` |
| Classificação | `{{ $json.classificacao }}` |
| Score | `{{ $json.score }}` |
| Justificativa | `{{ $json.justificativa }}` |
| Próxima Ação | `{{ $json.proxima_acao }}` |
| Urgência | `{{ $json.urgencia }}` |

---

### Nó 6 — Verificar se lead é QUENTE (condicional)

- Tipo: **IF**
- Condition: `{{ $json.classificacao }}` **equals** `QUENTE`

---

### Nó 7 (branch QUENTE) — Enviar email de alerta

- Tipo: **Gmail** ou **Send Email (SMTP)**
- To: email do dono do negócio
- Subject: `🔥 Lead QUENTE: {{ $json.nome }} — Score {{ $json.score }}`
- Body (HTML):

```html
<h2>🔥 Novo lead quente identificado!</h2>

<table style="border-collapse: collapse; width: 100%; font-family: sans-serif;">
  <tr><td style="padding: 8px; font-weight: bold; background: #f5f5f5;">Nome</td><td style="padding: 8px;">{{ $json.nome }}</td></tr>
  <tr><td style="padding: 8px; font-weight: bold; background: #f5f5f5;">Email</td><td style="padding: 8px;">{{ $json.email }}</td></tr>
  <tr><td style="padding: 8px; font-weight: bold; background: #f5f5f5;">Empresa</td><td style="padding: 8px;">{{ $json.empresa }}</td></tr>
  <tr><td style="padding: 8px; font-weight: bold; background: #f5f5f5;">Cargo</td><td style="padding: 8px;">{{ $json.cargo }}</td></tr>
  <tr><td style="padding: 8px; font-weight: bold; background: #f5f5f5;">Orçamento</td><td style="padding: 8px;">{{ $json.orcamento }}</td></tr>
  <tr><td style="padding: 8px; font-weight: bold; background: #f5f5f5;">Necessidade</td><td style="padding: 8px;">{{ $json.necessidade }}</td></tr>
</table>

<h3 style="color: #4f46e5;">Análise da IA</h3>
<p><strong>Score:</strong> {{ $json.score }}/100</p>
<p><strong>Justificativa:</strong> {{ $json.justificativa }}</p>
<p><strong>Próxima ação recomendada:</strong> {{ $json.proxima_acao }}</p>
<p><strong>Urgência:</strong> {{ $json.urgencia }}</p>

<hr>
<p style="font-size: 12px; color: #888;">Qualificado automaticamente em {{ $json.recebido_em }}</p>
```

---

## Passo 4 — Exportar o workflow

Após montar e testar:

1. No n8n, clique nos **3 pontos** do workflow → **Download**
2. Salve o arquivo como `workflows/qualificador.json`
3. Isso permite versionar no Git e reimportar em qualquer instância

---

## Passo 5 — Testar localmente

```bash
# Teste com curl simulando o formulário
curl -X POST http://localhost:5678/webhook/qualificador-leads \
  -H "Content-Type: application/json" \
  -d '{
    "nome": "João Silva",
    "email": "joao@empresa.com.br",
    "empresa": "Tech Solutions Ltda",
    "cargo": "CEO",
    "tamanho_empresa": "11-50",
    "necessidade": "Preciso automatizar meu processo de vendas e qualificação de leads",
    "orcamento": "2000-5000"
  }'
```

Verifique:
- [ ] Dados apareceram na planilha do Google Sheets
- [ ] Email de alerta chegou (se lead for QUENTE)
- [ ] Classificação faz sentido para os dados enviados

---

## Passo 6 — Deploy em produção (para entregar ao cliente)

### Opção A — Railway (recomendado, gratuito no início)

```bash
# Instale o CLI do Railway
npm install -g @railway/cli

# Login e deploy
railway login
railway init
railway up
```

Configure as variáveis de ambiente no painel do Railway (as mesmas do `.env`).

### Opção B — Render

1. Crie conta em [render.com](https://render.com)
2. New → Web Service → conecte o repositório Git
3. Build Command: (vazio)
4. Start Command: `docker compose up`
5. Adicione as env vars no painel

> **Atenção:** em produção, troque `WEBHOOK_URL` pela URL pública gerada pelo Railway/Render.

---

## Diferenciais para vender no Upwork

Quando for mostrar esse projeto, destaque:

- **Sem código no cliente** — basta preencher o formulário
- **Planilha atualizada em tempo real** — acesso imediato via Google Sheets
- **Alerta instantâneo** — nenhum lead quente fica esperando
- **IA configurável** — os critérios de classificação são ajustáveis por negócio
- **Escalável** — funciona para 10 ou 10.000 leads sem custo adicional de infra

---

## Possíveis expansões (cobrar à parte)

- Integrar com CRM (HubSpot, Pipedrive) em vez de Google Sheets — **+$150**
- Adicionar notificação no Slack além do email — **+$50**
- Dashboard de analytics dos leads no Metabase — **+$200**
- Versão com WhatsApp (via Evolution API + n8n) — **+$300**

---

## Perguntas frequentes do cliente

**"E se a IA errar a classificação?"**  
A planilha tem todos os dados brutos. O time pode revisar e corrigir manualmente. Com o tempo, ajustamos os critérios do prompt.

**"Meus dados ficam seguros?"**  
Sim. Os dados vão direto do formulário para sua planilha. A IA processa apenas o texto do lead, sem armazenar nada.

**"Posso mudar as perguntas do formulário?"**  
Sim. O formulário e o prompt da IA são totalmente customizáveis para cada negócio.
