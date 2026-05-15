# Prompt
## Context

Um incidente está em andamento durante pico de tráfego. Doc Brown (**CTO**) entrou na call e precisa de um postmortem técnico em 20 minutos para decidir entre rollback do deploy v2.48.0 (que subiu ontem) e scaling emergencial (aumento de limits do **RDS** e do pool de conexões). Os artefatos disponíveis para análise são os seguintes. Evento do deploy anterior (ontem, 18:42 **UTC**):

```
Deploy chronos-api: v2.47.0 -> v2.48.0
Argo CD sync: **2026**-04-23 18:42:11 **UTC**
Changelog:
- Adicionado endpoint **POST** /v2/transactions/batch
- Refatorado cliente do Ledger (pool de conexoes movido para nova biblioteca interna)
- Bump de psycopg 3.1.18 -> 3.2.0
- Reduzido timeout do Ledger de 5s para 2s

```

Métricas do Beacon nos últimos 30 minutos:
```
timestamp                p99_latency_ms   req_rate_s   err_rate_pct
2026-04-24 13:30 UTC     420              1200         0.2
2026-04-24 13:45 UTC     510              1450         0.3
2026-04-24 14:00 UTC     780              1780         0.8
2026-04-24 14:10 UTC     2400             2100         4.5
2026-04-24 14:15 UTC     5200             2400         8.2
2026-04-24 14:20 UTC     8100             2650         11.7
```


Trecho do log do pod `chronos-api-79c4d8b9-xk2jp`:

```
2026-04-24 14:19:48 [ERROR] [ledger-client] connection pool exhausted (max=20, active=20, waiting=147)
2026-04-24 14:19:49 [WARN]  [ledger-client] query timeout after 2000ms: SELECT ... FROM transactions WHERE ...
2026-04-24 14:19:49 [ERROR] [handler] POST /v2/transactions/batch failed: context deadline exceeded
2026-04-24 14:19:50 [ERROR] [ledger-client] connection reset by peer
2026-04-24 14:19:51 [WARN]  [circuit-breaker] ledger-client OPEN (threshold 50%, current 87%)
2026-04-24 14:19:52 [ERROR] [reactor] failed to publish message: chronos-api upstream error
```

Estado do Reactor (fila `chronos-transactions`):

- 50.**127** mensagens acumuladas, crescendo a ~**800**/min.
- Consumer lag atual: 18 minutos e aumentando.
Estado do cluster:

- Chronos: 12/12 pods running (**HPA** no máximo).
- **CPU** médio dos pods: 62%.
- Memória média dos pods: 71%.
- Conexões ativas ao Ledger: **240**/**250** (limite do **RDS**).

## Action

Com base nos artefatos acima, analise a correlação entre o deploy v2.48.0 e a degradação observada, identifique a causa raiz provável e recomende objetivamente entre rollback e scaling emergencial.

## Result

Documento com causa raiz provável, recomendação entre rollback vs scaling, e justificativa técnica

## Example

O documento de post-mortem deverá estar no formato markdown, contendo:

- timeline, 
- root cause, 
- impacto 
- Decisão recomendada.
# Modelo
## Claude Opus 4.7 Adaptative Thinking

# Output
# Post-mortem técnico — Incidente chronos-api (em andamento)

**Status:** ATIVO
**Severidade:** P1
**Início da degradação:** ~14:00 UTC (2026-04-24)
**Deploy suspeito:** v2.47.0 → v2.48.0 (2026-04-23 18:42 UTC)
**Janela para decisão:** 20 minutos

---

## Timeline

| Horário (UTC) | Evento |
|---|---|
| 23/04 18:42 | Deploy v2.47.0 → v2.48.0 via Argo CD (4 mudanças, ver "Root Cause") |
| 24/04 13:30 | p99 = 420ms, err = 0.2% — baseline ainda saudável, tráfego subindo |
| 24/04 13:45 | p99 = 510ms, err = 0.3% — degradação leve acompanha rampa de tráfego |
| 24/04 14:00 | p99 = 780ms, err = 0.8% — joelho da curva; início da contenção |
| 24/04 14:10 | p99 = 2.4s, err = 4.5% — HPA satura em 12/12 pods |
| 24/04 14:15 | p99 = 5.2s, err = 8.2% — pool de conexões esgota (20/20, 147 waiting) |
| 24/04 14:19 | Circuit breaker do `ledger-client` abre em 87% |
| 24/04 14:20 | p99 = 8.1s, err = 11.7%, lag do Reactor = 18min e crescendo |

---

## Root Cause (hipótese forte)

A causa raiz é **arquitetural no client-side**, não de capacidade computacional. O deploy v2.48.0 introduziu **três mudanças que interagem destrutivamente sob carga de pico**:

1. **Pool de conexões refatorado** (nova lib interna) — o tamanho efetivo do pool por pod é **20**. Com HPA no teto (12 pods), o total cluster-wide é `12 × 20 = 240` conexões — exatamente o que está sendo visto no RDS (240/250). O pool foi dimensionado pensando em carga média, não em pico.
2. **Timeout do Ledger reduzido de 5s → 2s** — sob contenção do pool, queries que antes drenavam a fila em 3–4s agora morrem em 2s, **sem liberar a conexão de forma limpa** (vide `connection reset by peer`). Cada timeout vira pressão extra: o cliente faz retry, a fila de waiters cresce, e o circuit breaker é forçado a abrir.
3. **Novo endpoint `POST /v2/transactions/batch`** — agrega N transações por request, mas no client do Ledger isso vira N queries sequenciais segurando a mesma conexão por mais tempo. Amplifica o consumo de pool por requisição.

### Evidência convergente

- HPA no máximo com **CPU 62% e memória 71%** → não é gargalo de compute, é gargalo de I/O bloqueado esperando conexão.
- 147 waiters por pod × 12 pods ≈ **1.764 requests enfileirados** esperando conexão livre — bate com a explosão de p99.
- Conexões RDS 240/250 com pool client-side esgotado simultaneamente → o teto é o **pool do cliente**, não o RDS. Aumentar `max_connections` do RDS sem mexer no client não muda nada.
- Bump psycopg 3.1.18 → 3.2.0 é candidato secundário (mudanças de comportamento em prepared statements / connection lifecycle), mas isoladamente não explicaria o padrão. Entra como suspeito de reforço.

### Por que só agora, ~20h após o deploy?

A v2.48.0 subiu **fora do pico** (18:42 UTC de ontem). O bug é latente e só se manifesta quando a taxa de requisições ultrapassa ~1.700 req/s — exatamente o que aconteceu às 14:00 UTC de hoje.

---

## Impacto

- **Usuário final:** 11.7% de erro e p99 de 8.1s no endpoint crítico de transações. Inaceitável em pico.
- **Reactor:** 50.127 mensagens acumuladas, +800/min, lag de 18min e crescendo → **risco de SLA de processamento assíncrono** e, se persistir, risco de retenção / TTL na fila.
- **Ledger / RDS:** próximo do teto de conexões (240/250). Qualquer outro consumidor do RDS vai começar a falhar em cascata se não estabilizar.
- **Blast radius:** todo fluxo que depende de `chronos-api` → `Ledger`, incluindo a publicação no Reactor (já falhando).

---

## Decisão recomendada: **ROLLBACK para v2.47.0**

### Justificativa comparativa

| Critério | Rollback v2.47.0 | Scaling emergencial (RDS + pool) |
|---|---|---|
| Tempo até recuperação | **~2–5 min** (Argo CD sync) | 15–30 min (mudança de parameter group do RDS pode exigir reboot; pool size exige novo deploy) |
| Risco da operação | **Baixo** — volta a estado conhecido e estável | Alto — mexer em RDS sob 240/250 conexões e ~12k req em 5 min é cirurgia em paciente sangrando |
| Cobre todas as mudanças suspeitas? | **Sim** — reverte pool novo, psycopg, timeout e endpoint batch de uma vez | Não — não corrige timeout agressivo nem o comportamento do novo client; só empurra o teto |
| Previsibilidade do resultado | **Alta** — v2.47.0 rodava em produção até ontem | Baixa — não sabemos qual das 4 mudanças é o vilão dominante; pode persistir |
| Reversibilidade | Trivial (re-sync no Argo) | Mudanças de RDS são lentas de reverter |

**Comando sugerido:** rollback via Argo CD apontando para a tag/revisão da v2.47.0.

### Por que scaling não é a resposta agora

- Aumentar `max_connections` do RDS **sem** aumentar o pool do client não muda nada — o gargalo está no cliente (pool exausto antes de chegar no RDS).
- Aumentar o pool do client de 20 → 40 daria `12 × 40 = 480` conexões, **excedendo o atual limite do RDS** mesmo se for elevado. Exigiria fazer as duas mudanças coordenadas, sob fogo, em pico.
- O timeout de 2s vai continuar matando queries legítimas — scaling não resolve.
- Cada minuto debugando isso é mais ~800 mensagens acumulando no Reactor.

---

## Plano pós-rollback (próximas 2h)

1. Confirmar p99 e err_rate voltando ao baseline em ~10 min após o rollback.
2. Drenar lag do Reactor — monitorar consumer group.
3. Postmortem completo (blameless) com bisect das 4 mudanças do v2.48.0 em staging com **carga sintética de pico** (load test que cubra >1.700 req/s — o que faltou no rollout).
4. Antes de re-deploy:
   - Ajustar tamanho do pool no client baseado em `pods_max × pool_size ≤ rds_max_connections × 0.8`.
   - Reverter ou justificar com SLO o timeout de 2s.
   - Validar comportamento de `connection reset` da nova lib de pool sob timeout.

---

## TL;DR para Doc Brown

Não é capacidade, é regressão arquitetural client-side da v2.48.0 que só aparece em pico. **Rollback resolve em 5 min com risco baixo. Scaling resolve em 30 min com risco alto e talvez não resolva. Rollback agora, autópsia depois.**

# Justificativa
