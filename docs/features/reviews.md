---
title: Reviews (Avaliações)
feature: reviews
module: pdp
engine: agnostic
analysis_type: feature_analysis
analysis_date: 2026-04-10
confidence: medium
status: valid
source: manual
suggested_question: O que já existe hoje para avaliações de produto?
display_priority: 8
---

# 1. Resumo executivo
O módulo de reviews permite exibir avaliações de produtos na PDP, com suporte a múltiplos providers.

# 2. Visão geral da feature
- Exibição de avaliações na PDP
- Suporte a diferentes providers
- Integração via camada de abstração

# 3. Arquitetura e funcionamento
- Camada de provider
- Strategy por fornecedor
- Renderização via widget na PDP

# 4. Status de implementação

| Parte | Status |
|------|--------|
| Exibição de reviews | Total |
| Providers múltiplos | Parcial |

# 5. Gaps e limitações
- Dependência de provider externo
- Diferença de payload entre providers

# 6. Conclusão
A feature é funcional, mas depende da integração com providers específicos.
