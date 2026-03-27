# 📦 Inventário Empresarial via WhatsApp — n8n + IA

> Sistema de controle de estoque operado inteiramente por WhatsApp. Funcionários registram entradas e saídas em linguagem natural — a IA interpreta, confirma com o operador e grava no banco automaticamente.

---

## Sobre o projeto

Sistema de automação de inventário desenvolvido em **n8n**, integrando um agente de IA com fluxo de confirmação em etapas, cache de produtos, deduplicação de alertas e notificação automática ao gerente.

Não exige nenhuma interface adicional — o operador interage apenas pelo WhatsApp.

> 🚧 Em desenvolvimento — os nós HTTP Request serão migrados para nós nativos do n8n em versões futuras.

---

## Arquitetura

```
WhatsApp (API Oficial Meta)
        │
        ▼
   Webhook n8n
        │
        ▼
Verificar e Extrair Webhook
(valida assinatura HMAC-SHA256 + extrai phone/mensagem)
        │
        ▼
  Tem Mensagem Válida?
        │
        ▼
Redis ──► Sessão GET + Etapa GET + Cache Produtos GET
        │
        ├── Cache existe? ──► Sim: usa cache (TTL 5 min)
        │                └── Não: Supabase → busca produtos ativos → grava cache
        │
        ▼
  Montar Contexto IA
(monta prompt com lista de produtos + sessão + etapa atual)
        │
        ▼
IA - Agente de Inventário (Gemini)
+ Parser de Saída Estruturada
        │
        ▼
  Switch - Status IA
   │         │         │         │
pendente  confirmando aguardando cancelado
   │
   ▼
Tem Produto Novo?
   ├── Sim: avisa operador para cadastrar o item
   └── Não: Preparar Movimentações
              │
              ├─► Supabase - Inserir Movimentações
              ├─► Supabase - Buscar Itens Críticos
              ├─► Supabase - Buscar Alertas Hoje (deduplicação)
              ├─► Há Alertas Novos?
              │       └── Sim: WhatsApp - Alerta Gerente
              │                └── Supabase - Inserir Alertas Log
              └─► WhatsApp - Resposta Confirmado
```

---

## Funcionalidades

| Recurso | Descrição |
|---|---|
| **Registro em linguagem natural** | Operador envia "entrada 10 caixas de leite" e a IA interpreta |
| **Fluxo de confirmação em etapas** | IA gera resumo, operador confirma antes de gravar |
| **Máquina de estados por sessão** | Cada telefone tem etapa independente salva no Redis (`pendente`, `confirmando`, `aguardando`, `cancelado`) |
| **Cache de produtos** | Lista de produtos ativos cacheada no Redis (TTL 5 min) — evita consulta ao Supabase a cada mensagem |
| **Proteção contra produto não cadastrado** | Se a IA detectar `produto_novo: true`, bloqueia a gravação e pede cadastro prévio |
| **Alertas de estoque crítico** | Após cada movimentação, verifica itens abaixo do mínimo e notifica o gerente |
| **Deduplicação de alertas** | Mesmo produto não gera alerta duas vezes no mesmo dia (via tabela `alertas_log`) |
| **Validação de assinatura** | Webhook valida HMAC-SHA256 (`x-hub-signature-256`) antes de processar qualquer mensagem |
| **Resposta imediata** | Retorna HTTP 200 ao webhook instantaneamente e processa em segundo plano |

---

## Stack tecnológica

- **n8n** — orquestração do workflow
- **WhatsApp Business API (Meta)** — receber e enviar mensagens
- **Google Gemini** — interpretação de mensagens e geração de saída estruturada
- **Redis** — sessões por usuário + cache de produtos (TTL configurável)
- **Supabase** — banco de dados de produtos, movimentações e log de alertas

---

## Como importar no n8n

1. Acesse seu n8n → **Workflows** → **Import from file**
2. Selecione o arquivo `workflow_inventario_whatsapp_supabase.json`
3. Configure as variáveis de ambiente e credenciais (seções abaixo)
4. Ative o workflow

---

## Variáveis de ambiente

Configure no painel do n8n em **Settings → Environment Variables**:

| Variável | Descrição |
|---|---|
| `SUPABASE_URL` | URL do seu projeto Supabase |
| `SUPABASE_SERVICE_ROLE_KEY` | Chave service role do Supabase (com permissão de escrita) |
| `WHATSAPP_API_BASE_URL` | URL base da API do WhatsApp (ex: `https://graph.facebook.com/v19.0`) |
| `WHATSAPP_PHONE_NUMBER_ID` | ID do número de telefone na API Meta |
| `WHATSAPP_TOKEN` | Token de acesso da API do WhatsApp |
| `WHATSAPP_APP_SECRET` | Secret do app Meta (usado na validação HMAC) |
| `INVENTORY_MANAGER_PHONE` | Número do gerente que recebe os alertas (formato internacional) |

---

## Credenciais no n8n

| Credencial | Nós que usam |
|---|---|
| Redis | Todos os nós `Redis - *` |
| Google Gemini API | Nó `Gemini - Chat Model` |

---

## Banco de dados Supabase

Execute o script `inventario_supabase.sql` no SQL Editor do seu projeto para criar as tabelas necessárias. As tabelas utilizadas pelo workflow são:

| Tabela / View | Uso |
|---|---|
| `produtos` | Catálogo de produtos ativos (campos: `id`, `nome`, `sku`, `unidade`, `categoria`, `ativo`) |
| `movimentacoes` | Registro de entradas e saídas (campos: `produto_id`, `tipo`, `quantidade`, `origem`, `operador`, `observacao`, `numero_nfe`) |
| `alertas_log` | Log de alertas enviados para deduplicação diária (campos: `produto_id`, `tipo_alerta`, `referencia_dia`) |
| `v_itens_criticos` | View que retorna produtos com estoque abaixo do mínimo |

---

## Fluxo de uma movimentação

```
Operador: "saída 5 unidades de arroz"
    │
    ▼
IA interpreta → status: "pendente"
    │
    ▼
Bot: "Confirma? Saída de 5 un. Arroz Integral 5kg (SKU: ARR001)"
    │
Operador: "sim"
    │
    ▼
Grava em movimentacoes no Supabase
    │
    ▼
Verifica estoque crítico → deduplica → envia alerta ao gerente (se necessário)
    │
    ▼
Bot: "✅ Movimentação registrada com sucesso."
```

---

## Sessões e estados Redis

Cada operador tem duas chaves independentes no Redis:

| Chave | Valor | TTL |
|---|---|---|
| `inv:session:{phone}` | JSON com payload da sessão atual | 600s (10 min) |
| `inv:step:{phone}` | Etapa atual (`pendente`, `confirmando`, `aguardando`, `cancelado`) | 600s (10 min) |
| `inv:cache:produtos` | JSON com lista de produtos ativos | 300s (5 min) |

---

## Autor

Desenvolvido por **Kauã B** — estudante de Engenharia de Software com foco em automação de processos para pequenos e médios negócios.

- Stack principal: n8n · Supabase · Redis · Google Gemini · WhatsApp API

---

## Licença

MIT — livre para uso e adaptação com atribuição.
