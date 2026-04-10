---
title: Brinde no carrinho (Shopify)
feature: cart_gift
module: cart
engine: shopify
analysis_type: feature_analysis
analysis_date: 2026-04-10
confidence: medium
status: needs_review
source: codex_analysis
---

# 1. Resumo executivo
A feature de brinde no carrinho não está implementada no app para Shopify. Existe suporte parcial via checkout webview, mas o app não controla as regras de aplicação.

# 2. Visão geral da feature
- Objetivo: aumentar conversão e ticket médio
- Atua no carrinho e checkout
- No Shopify depende do backend/checkout

# 3. Arquitetura e funcionamento
- UI: não existe implementação nativa
- Checkout: webview externo
- Regra de brinde não controlada pelo app

# 4. Comportamento por engine
## Shopify
- Não implementado no app
- Possível via checkout externo

# 5. Status de implementação

| Parte | Status |
|------|--------|
| App | Não implementado |
| Checkout webview | Parcial |

# 6. Uso atual (clientes)
- Não há evidência de uso funcional no app

# 7. Parâmetros e configuração
- Não identificado

# 8. Providers e strategies
- Dependente do engine Shopify

# 9. Gaps e limitações
- App não controla regra
- Dependência do checkout externo
- Experiência inconsistente

# 10. Impacto e riscos
- Pode gerar inconsistência na experiência
- Risco de venda incorreta da feature

# 11. Conclusão
A feature não está implementada no app para Shopify e depende de soluções externas.
