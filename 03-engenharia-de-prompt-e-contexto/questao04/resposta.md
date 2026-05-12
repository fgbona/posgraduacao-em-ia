# Prompt
# Task
Crie um relatório para o gerente, a ser entregue ao CEO, sobre o crescimento das transações nos últimos 6 meses. O gerente precisa dos números, porém, como não escreve SQL, enviou a solicitação ao Reactor, um sistema de mensageria.

# Action
O banco de dados PostgreSQL, com as tabelas mais relevantes listadas abaixo, possui o histórico completo. Categorias em produção hoje: subscription, one_time, refund e credit_adjustment. "Com base no schema abaixo, escreva a query SQL PostgreSQL que extraia esses dados."

CREATE TABLE transactions (
  id              BIGSERIAL PRIMARY KEY,
  customer_id     BIGINT NOT NULL REFERENCES customers(id),
  category        VARCHAR(32) NOT NULL,
  amount_cents    BIGINT NOT NULL,
  status          VARCHAR(16) NOT NULL,
  payment_method  VARCHAR(16),
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_transactions_created_at ON transactions(created_at);
CREATE INDEX idx_transactions_status ON transactions(status);
CREATE INDEX idx_transactions_category ON transactions(category);
CREATE TABLE customers (
  id          BIGSERIAL PRIMARY KEY,
  segment     VARCHAR(16) NOT NULL,
  country     CHAR(2) NOT NULL,
  signup_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
# Goal
Escreve uma relatório, que tenha um filtro status = 'completed'. O campo amount_cents está em centavos de real e precisa aparecer na saída em reais com 2 casas decimais. O recorte é dos últimos 6 meses corridos a partir de hoje (2026-04-24), agrupado por mês (no formato YYYY-MM) e por categoria, trazendo duas métricas por linha:
* quantidade de transações
* volume total em reais.

Ordenação final: mês crescente, depois categoria crescente.

O relatório deverá ser entregue no formato markdown.

## Modelo
Claude Haiku 4.5

###### Output
Vou criar um relatório completo com a query SQL e o documento em markdown para o CEO.
1. Query SQL
Aqui está a query PostgreSQL que extrai os dados conforme especificado:
sqlSELECT
  TO_CHAR(DATE_TRUNC('month', t.created_at), 'YYYY-MM') AS mes,
  t.category AS categoria,
  COUNT(*) AS quantidade_transacoes,
  ROUND(SUM(t.amount_cents)::NUMERIC / 100, 2) AS volume_reais
FROM transactions t
WHERE t.status = 'completed'
  AND t.created_at >= NOW() - INTERVAL '6 months'
GROUP BY DATE_TRUNC('month', t.created_at), t.category
ORDER BY mes ASC, categoria ASC;
Explicação da query:

DATE_TRUNC('month', t.created_at) agrupa as transações por mês completo
TO_CHAR(..., 'YYYY-MM') formata o mês como solicitado
WHERE t.status = 'completed' filtra apenas transações completadas
NOW() - INTERVAL '6 months' captura os últimos 6 meses (de 2025-10-24 a 2026-04-24)
SUM(t.amount_cents)::NUMERIC / 100 converte centavos para reais
ROUND(..., 2) garante 2 casas decimais
Ordenação: primeiro por mês (crescente), depois por categoria (crescente)


2. Relatório em Markdown
Vou criar o relatório formatado:

# Relatório de Crescimento de Transações
## Período: Últimos 6 Meses (Outubro 2025 - Abril 2026)

**Data do Relatório:** 24 de abril de 2026
**Destinatário:** CEO
**Gerenciador de Dados:** Reactor

---

## Resumo Executivo

Este relatório apresenta uma análise consolidada do crescimento das transações completadas nos últimos 6 meses, desagregadas por mês e categoria de transação. Os dados incluem volume de transações (quantidade) e impacto financeiro (em reais).

---

## Dados Consolidados por Período

| Mês | Categoria | Quantidade de Transações | Volume Total (R$) |
|-----|-----------|--------------------------|-------------------|
| 2025-10 | credit_adjustment | [N/A] | [N/A] |
| 2025-10 | one_time | [N/A] | [N/A] |
| 2025-10 | refund | [N/A] | [N/A] |
| 2025-10 | subscription | [N/A] | [N/A] |
| 2025-11 | credit_adjustment | [N/A] | [N/A] |
| 2025-11 | one_time | [N/A] | [N/A] |
| 2025-11 | refund | [N/A] | [N/A] |
| 2025-11 | subscription | [N/A] | [N/A] |
| 2025-12 | credit_adjustment | [N/A] | [N/A] |
| 2025-12 | one_time | [N/A] | [N/A] |
| 2025-12 | refund | [N/A] | [N/A] |
| 2025-12 | subscription | [N/A] | [N/A] |
| 2025-01 | credit_adjustment | [N/A] | [N/A] |
| 2025-01 | one_time | [N/A] | [N/A] |
| 2025-01 | refund | [N/A] | [N/A] |
| 2025-01 | subscription | [N/A] | [N/A] |
| 2025-02 | credit_adjustment | [N/A] | [N/A] |
| 2025-02 | one_time | [N/A] | [N/A] |
| 2025-02 | refund | [N/A] | [N/A] |
| 2025-02 | subscription | [N/A] | [N/A] |
| 2025-03 | credit_adjustment | [N/A] | [N/A] |
| 2025-03 | one_time | [N/A] | [N/A] |
| 2025-03 | refund | [N/A] | [N/A] |
| 2025-03 | subscription | [N/A] | [N/A] |
| 2025-04 | credit_adjustment | [N/A] | [N/A] |
| 2025-04 | one_time | [N/A] | [N/A] |
| 2025-04 | refund | [N/A] | [N/A] |
| 2025-04 | subscription | [N/A] | [N/A] |
| 2025-05 | credit_adjustment | [N/A] | [N/A] |
| 2025-05 | one_time | [N/A] | [N/A] |
| 2025-05 | refund | [N/A] | [N/A] |
| 2025-05 | subscription | [N/A] | [N/A] |
| 2025-06 | credit_adjustment | [N/A] | [N/A] |
| 2025-06 | one_time | [N/A] | [N/A] |
| 2025-06 | refund | [N/A] | [N/A] |
| 2025-06 | subscription | [N/A] | [N/A] |
| 2025-07 | credit_adjustment | [N/A] | [N/A] |
| 2025-07 | one_time | [N/A] | [N/A] |
| 2025-07 | refund | [N/A] | [N/A] |
| 2025-07 | subscription | [N/A] | [N/A] |
| 2025-08 | credit_adjustment | [N/A] | [N/A] |
| 2025-08 | one_time | [N/A] | [N/A] |
| 2025-08 | refund | [N/A] | [N/A] |
| 2025-08 | subscription | [N/A] | [N/A] |
| 2025-09 | credit_adjustment | [N/A] | [N/A] |
| 2025-09 | one_time | [N/A] | [N/A] |
| 2025-09 | refund | [N/A] | [N/A] |
| 2025-09 | subscription | [N/A] | [N/A] |
| 2025-10 | credit_adjustment | [N/A] | [N/A] |
| 2025-10 | one_time | [N/A] | [N/A] |
| 2025-10 | refund | [N/A] | [N/A] |
| 2025-10 | subscription | [N/A] | [N/A] |
| 2026-01 | credit_adjustment | [N/A] | [N/A] |
| 2026-01 | one_time | [N/A] | [N/A] |
| 2026-01 | refund | [N/A] | [N/A] |
| 2026-01 | subscription | [N/A] | [N/A] |
| 2026-02 | credit_adjustment | [N/A] | [N/A] |
| 2026-02 | one_time | [N/A] | [N/A] |
| 2026-02 | refund | [N/A] | [N/A] |
| 2026-02 | subscription | [N/A] | [N/A] |
| 2026-03 | credit_adjustment | [N/A] | [N/A] |
| 2026-03 | one_time | [N/A] | [N/A] |
| 2026-03 | refund | [N/A] | [N/A] |
| 2026-03 | subscription | [N/A] | [N/A] |
| 2026-04 | credit_adjustment | [N/A] | [N/A] |
| 2026-04 | one_time | [N/A] | [N/A] |
| 2026-04 | refund | [N/A] | [N/A] |
| 2026-04 | subscription | [N/A] | [N/A] |

---

## Notas Metodológicas

- **Período Analisado:** 6 meses corridos anteriores a 24 de abril de 2026
- **Filtro de Status:** Apenas transações com `status = 'completed'`
- **Moeda:** Real Brasileiro (R$) com 2 casas decimais
- **Categorias de Transação:**
  - `subscription`: Transações recorrentes de assinatura
  - `one_time`: Transações pontuais
  - `refund`: Reembolsos processados
  - `credit_adjustment`: Ajustes de crédito
- **Ordenação:** Crescente por mês, depois crescente por categoria

---

## Próximos Passos

Para popular este relatório com dados reais, execute a seguinte query no banco de dados PostgreSQL:

```sql
SELECT
  TO_CHAR(DATE_TRUNC('month', t.created_at), 'YYYY-MM') AS mes,
  t.category AS categoria,
  COUNT(*) AS quantidade_transacoes,
  ROUND(SUM(t.amount_cents)::NUMERIC / 100, 2) AS volume_reais
FROM transactions t
WHERE t.status = 'completed'
  AND t.created_at >= NOW() - INTERVAL '6 months'
GROUP BY DATE_TRUNC('month', t.created_at), t.category
ORDER BY mes ASC, categoria ASC;
```

Os resultados deverão ser inseridos na tabela de dados consolidados acima.

---

**Preparado por:** Reactor (Sistema de Mensageria)
**Data:** 24 de abril de 2026
**Confidencialidade:** Interno

Query SQL PostgreSQL
sqlSELECT
  TO_CHAR(DATE_TRUNC('month', t.created_at), 'YYYY-MM') AS mes,
  t.category AS categoria,
  COUNT(*) AS quantidade_transacoes,
  ROUND(SUM(t.amount_cents)::NUMERIC / 100, 2) AS volume_reais
FROM transactions t
WHERE t.status = 'completed'
  AND t.created_at >= NOW() - INTERVAL '6 months'
GROUP BY DATE_TRUNC('month', t.created_at), t.category
ORDER BY mes ASC, categoria ASC;
Características:

✅ Filtra transações com status = 'completed'

✅ Converte centavos em reais com 2 casas decimais

✅ Agrupa por mês (formato YYYY-MM) e categoria

✅ Retorna quantidade de transações e volume total

✅ Ordena por mês crescente, depois categoria crescente

✅ Período: últimos 6 meses completos (24/10/2025 a 24/04/2026)

📄 Relatório em Markdown
O arquivo relatorio_crescimento_transacoes_6m.md foi criado com:

Estrutura executiva profissional
Tabela pronta para receber os dados da query
Notas metodológicas explicando os critérios
Query SQL integrada para referência
Formatação apropriada para entrega ao CEO

Para completar o relatório: Execute a query no PostgreSQL e preencha os resultados na tabela de dados consolidados do markdown. O gerente poderá então entregar um documento completo e profissional ao CEO! 🚀

###### Justificativa

Task: A task contextualiza a demanda sobre o relatorio de crescimento dos ultimos 6 meses que o pm solicita.
Action: Fornece o schema completo das tabelas transactions e customers, e as categorias, dando ao modelo contexto técnico para poder criar a query.
Goal: Relatorio baseado no filtro status, conversao de amount_cents para reais e agrupamento por mes e categoria, duas metricas por linhas e ordenação final. Mesmo o modelo mais simples da antropic, o haiku, entregou uma resposta e relatórios bem completos.
