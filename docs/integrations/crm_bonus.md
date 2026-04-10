---
title: CRM Bonus (Cashback)
feature: crm_bonus_cashback
module: cashback
engine: vtex
analysis_type: integration_analysis
analysis_date: 2026-04-10
confidence: medium
status: needs_review
source: codex_analysis
---

# 1. Resumo executivo
A integração CRM Bonus está funcional para aplicação de cashback no carrinho, mas não cobre funcionalidades completas como extrato e histórico.

# 2. Visão geral da feature
- Objetivo: permitir uso de cashback
- Aplicado diretamente no carrinho
- Baseado em integração com VTEX

# 3. Arquitetura e funcionamento
- UseCase → Repository → DataSource
- Integração via endpoints VTEX
- Parsing baseado em estruturas genéricas

# 4. Comportamento por engine
## VTEX
- Implementação funcional
- Dependente de endpoints específicos

# 5. Status de implementação

| Parte | Status |
|------|--------|
| Aplicação de cashback | Total |
| Extrato | Não implementado |
| Histórico | Não implementado |

# 6. Uso atual (clientes)
- Ativado via configuração
- Uso identificado em clientes VTEX

# 7. Parâmetros e configuração
- cashback_enabled
- cashback_type=crmBonus

# 8. Providers e strategies
- Provider baseado em DataSource específico

# 9. Gaps e limitações
- Falta de extrato
- Falta de histórico
- Forte acoplamento com VTEX

# 10. Impacto e riscos
- Experiência incompleta para o usuário
- Limitação de expansão

# 11. Conclusão
A integração é funcional, mas incompleta e com limitações estruturais.
