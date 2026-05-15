# Prompt
## Role
Você é um engenheiro cloud especialista em Kubernetes, Docker e AWS sênior.

## Input
Beacon (nosso sistema de observabilidade) toda semana em média 4 vezes, dispara o mesmo alerta no canal de plantão: [CRITICAL] High memory usage on Chronos API pods (>85% for 10min). Quem assume o plantão gasta de 30 a 40 minutos até resolver, e o tempo varia muito porque não existe procedimento documentado, nenhum playbook ou documentação auxiliar.

## Steps
Crie um playbook que detalhe o passo a passo para resolver o problema do aplicativo Chronos e que seja executável pela equipe técnica, sendo o técnico tendo do nível mais baixo de conhecimento ao mais alto:

* Chronos roda no EKS, namespace production, 6 réplicas com HPA configurado (min 4, max 12, CPU target 70%).
* Deploy via Argo CD a partir do repositório hvt/chronos-api.
* Dependências diretas: Ledger (PostgreSQL) e Reactor (filas SQS).
* Observabilidade: métricas expostas em /metrics, logs centralizados no Beacon, dashboards em Grafana.
* Ferramentas disponíveis para o plantão: kubectl, aws cli, argocd cli.
* Canal de plantão: #oncall-chronos no Slack.
* Time sênior de escalação: @chronos-core (SLA de resposta: 15 minutos em horário comercial, 30 fora).

## Expectation
O playbook deverá:
* cobrir os passos iniciais de diagnóstico, com os comandos específicos a rodar
* verificação esperada ao final de cada passo
* critérios objetivos para escalar para o tiime sênior
* critério para encerrar o incidente.

# Modelo
Gemini 3.1 Pro

# Output
# 📖 Playbook de Incidente: High Memory Usage - Chronos API

**Alerta:** `[CRITICAL]` High memory usage on Chronos API pods (>85% for 10min)  
**Aplicação:** Chronos API  
**Namespace:** production  
**SLA Escalation:** 15m (horário comercial) / 30m (fora do horário)

---

## 0. Comunicação Inicial (Acknowledge)

Antes de iniciar o troubleshooting, avise o time que você está atuando no incidente para evitar sobreposição de esforços.

**Ação:** Envie no canal do Slack `#oncall-chronos`:

```
Assumindo o alerta de memória do Chronos API. Iniciando investigação.
```

---

## 1. Triagem e Diagnóstico Inicial

O objetivo desta etapa é entender o estado atual do cluster e identificar se os pods estão sofrendo OOMKilled (Out of Memory) ou apenas rodando no limite.

### Passo 1.1: Verificar o status dos Pods

Vamos checar se há pods reiniciando frequentemente (indicativo de OOMKilled).

**Comando:**

```bash
kubectl get pods -n production -l app=chronos-api
```

> **Nota:** Assumindo que a label base seja `app=chronos-api` ou similar

**Verificação Esperada:**

- **Saudável:** Status `Running` e a coluna `RESTARTS` com valor `0` (ou um número baixo e antigo).
- **Crítico:** Status `CrashLoopBackOff` ou `OOMKilled`, e a coluna `RESTARTS` subindo rapidamente.

### Passo 1.2: Verificar o consumo real de recursos

Vamos olhar o consumo exato de CPU e Memória de cada pod.

**Comando:**

```bash
kubectl top pods -n production -l app=chronos-api
```

**Verificação Esperada:** Identificar quais pods estão passando dos 85% do limit de memória configurado.

> **Atenção:** O HPA do Chronos está configurado para escalar baseado em CPU (Target 70%). Se o problema for vazamento de memória (Memory Leak), a CPU pode ficar baixa, o HPA não vai escalar, e o pod vai morrer por falta de memória.

### Passo 1.3: Analisar os Logs no Beacon ou Terminal

Verifique se a aplicação está gerando erros de conexão com as dependências (Ledger/PostgreSQL ou Reactor/SQS) que possam estar represando dados na memória.

**Comando (via kubectl para as últimas linhas):**

```bash
kubectl logs --tail=100 -l app=chronos-api -n production | grep -iE "error|exception|timeout"
```

**Verificação Esperada:**

- Procure por logs como `Connection refused` (banco fora) ou `OutOfMemoryError`.
- Verifique também os dashboards do Grafana para ver se a métrica da fila SQS está acumulando (backlog) ou se a latência do banco subiu.

---

## 2. Ações de Mitigação

Se o uso de memória está crítico (>95%) ou os pods estão em `CrashLoopBackOff`, a ação imediata de plantão para restabelecer o serviço é forçar a reciclagem dos pods.

### Passo 2.1: Restart Controlado (Rollout)

Reiniciar os pods vai limpar a memória e restaurar o serviço temporariamente enquanto a causa raiz é investigada no dia seguinte.

**Comando:**

```bash
kubectl rollout restart deployment chronos-api -n production
```

**Verificação Esperada:**

Acompanhe o status da recriação dos pods:

```bash
kubectl rollout status deployment chronos-api -n production
```

Deve retornar: `deployment "chronos-api" successfully rolled out`.

### Passo 2.2: Verificar sincronia no Argo CD

Garanta que não houve um deploy quebrado ou configuração manual sobrescrevendo o repositório (`hvt/chronos-api`).

**Comando:**

```bash
argocd app get chronos-api
```

**Verificação Esperada:** O status deve constar como `Synced` e `Healthy`. Se estiver `OutOfSync`, avise o time e sincronize:

```bash
argocd app sync chronos-api
```

---

## 3. Critérios de Escalonamento

Se a mitigação acima não funcionar, você deve acionar o time sênior.

**Escale imediatamente marcando `@chronos-core` no canal `#oncall-chronos` se atender a QUALQUER um destes critérios:**

- **Falha no Restart:** O comando `rollout restart` foi executado, mas os novos pods continuam morrendo por `OOMKilled` em menos de 5 minutos.

- **Dependência Fora:** Os logs indicam claramente que o Ledger (PostgreSQL) ou o Reactor (SQS) estão indisponíveis, e não apenas o Chronos.

- **HPA Esgotado:** O número de réplicas bateu o limite máximo do HPA (12) e o sistema continua instável ou caindo.

- **Tempo Esgotado:** Você está há 15 minutos (horário comercial) ou 30 minutos (fora do horário) tentando diagnosticar e o alerta não normalizou.

**Mensagem padrão de escalonamento:**

```
@chronos-core Escalonando incidente. Pods do Chronos API com high memory/OOMKilled. 
Rollout restart executado, mas o consumo voltou a subir rapidamente. 
Dependências (PostgreSQL/SQS) parecem OK. Preciso de apoio para analisar 
possível memory leak no código.
```

---

## 4. Critério para Encerrar o Incidente

O incidente só pode ser considerado Resolvido (e o plantonista liberado) quando **TODOS** os critérios abaixo forem atendidos:

- **Estabilidade de Pods:** Todos os pods do Chronos API (mínimo de 4) estão com status `Running` por pelo menos 15 minutos consecutivos sem nenhum novo restart.

- **Métricas Normalizadas:** O consumo de memória reportado pelo `kubectl top pods` ou no Grafana estabilizou abaixo de 75%.

- **Argo CD:** A aplicação encontra-se `Synced` e `Healthy`.

- **Comunicação Final:** Uma mensagem de resolução foi enviada no Slack com um breve resumo do que foi feito. Exemplo:

  ```
  Alerta resolvido. Feito o rollout restart do deployment. Memória estabilizada 
  em 50% há mais de 15 minutos. Criarei um ticket no Jira para o time analisar 
  possível vazamento de memória na sprint.
  ```
# Justificativa
