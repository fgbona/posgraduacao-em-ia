# Avaliação de Prompts — Desafio IAOps (Playbook de IA Operacional da Aegis)

A entrega deste módulo é um repositório próprio, porque o desafio pede que o repositório **seja** o playbook: biblioteca de prompts no formato do template `prompt-registry`, testes em promptfoo ao lado de cada prompt e pipeline de CI gateando regressão.

**Repositório da entrega:** https://github.com/fgbona/aegis-playbook

| Checkpoint | O que foi entregue | Onde ler |
|---|---|---|
| 01 | Prompt de triagem de pods | [doc](https://github.com/fgbona/aegis-playbook/blob/main/docs/checkpoints/01-triagem-de-pods.md) |
| 02 | Prompt de nota de triagem | [doc](https://github.com/fgbona/aegis-playbook/blob/main/docs/checkpoints/02-nota-de-triagem.md) |
| 03 | Prompt de causa-raiz do Cerebro | [doc](https://github.com/fgbona/aegis-playbook/blob/main/docs/checkpoints/03-causa-raiz.md) |
| 04 | Prompt de decisão de backpressure do Relay | [doc](https://github.com/fgbona/aegis-playbook/blob/main/docs/checkpoints/04-decisao-backpressure.md) |
| 05 | Cadeia de três prompts da migração do Forge | [doc](https://github.com/fgbona/aegis-playbook/blob/main/docs/checkpoints/05-migracao-forge.md) |
| 06 | Prompt de endurecimento da NetworkPolicy do Sentinel, com iterações | [doc](https://github.com/fgbona/aegis-playbook/blob/main/docs/checkpoints/06-networkpolicy-sentinel.md) |
| 07 | Biblioteca no formato prompt-registry | [doc](https://github.com/fgbona/aegis-playbook/blob/main/docs/checkpoints/07-biblioteca-vira-codigo.md) |
| 08 | Testes determinísticos com promptfoo | [doc](https://github.com/fgbona/aegis-playbook/blob/main/docs/checkpoints/08-testes-deterministicos.md) |
| 09 | Gate de qualidade com LLM-as-judge | [doc](https://github.com/fgbona/aegis-playbook/blob/main/docs/checkpoints/09-llm-as-judge.md) |
| 10 | Pipeline em GitHub Actions | [doc](https://github.com/fgbona/aegis-playbook/blob/main/docs/checkpoints/10-pipeline.md) |

Provedores usados: Google (Gemini 3.8 Flash, 3.1 Pro) e Anthropic (Claude Haiku 4.5, Sonnet 4.6, Sonnet 5.5 no meta-prompting). Todos os outputs documentados são execuções reais via promptfoo, com latência e custo registrados.
