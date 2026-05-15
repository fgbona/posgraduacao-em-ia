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
cobrir os passos iniciais de diagnóstico, com os comandos específicos a rodar
verificação esperada ao final de cada passo
critérios objetivos para escalar para o tiime sênior
critério para encerrar o incidente.
o playbook deverá estar no formato markdown para download.

# Modelo
Gemini 3.1 Pro

# Output

# Justificativa
