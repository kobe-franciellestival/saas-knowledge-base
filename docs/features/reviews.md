---
title: Reviews (Avaliações)
feature: reviews
module: pdp
engine: agnostic
analysis_type: feature_analysis
analysis_date: 2026-04-10
confidence: medium
status: valid
source: codex_analysis
suggested_question: O que já existe hoje para avaliações de produto?
display_priority: 8
---

# 1. Resumo executivo
A feature cobre exibição de avaliações, página dedicada, fluxo de criação (opcional) e interações como voto útil, com impacto direto na conversão via prova social. Atua como módulo transversal, configurável por provider e flags, sem dependência forte de engine.

---

# 2. Visão geral da feature

## Cobertura da feature
- Exibição de nota média e volume de reviews em card de produto, PLP e PDP
- Página de avaliações com filtros/ordenação
- Fluxo de criação de review no app (opcional)
- Destaque de review útil (positivo/negativo) e voto útil (provider-dependente)

## Valor de negócio
- Aumenta confiança e conversão em PDP/PLP
- Reduz fricção na decisão de compra via prova social
- Permite estratégia de pós-compra (UGC e retenção), quando o provider suporta criação

## Encaixe na plataforma
- Módulo transversal, desacoplado do engine na maior parte
- Ativado por configuração (provider + flags), não por hardcode de cliente :contentReference[oaicite:0]{index=0}

---

# 3. Arquitetura e funcionamento

## Componentes principais
- Configuração: `RatingConfig` e `RatingConfigRegistrations`
- Orquestração: `RatingInitializer`, `RatingController`, `RatingRepository`
- Providers (datasources):
  - TrustVox
  - Judge.me
  - Konfidency
  - BazaarVoice
  - YourViews
  - CloudCommerce (VTEX/Magento)
- Enriquecimento de busca: `RatingSearch` (decora `SearchService`)

## Dependências

### Externas
- APIs de reviews (TrustVox, Judge.me, Konfidency, Bazaar, YourViews)

### Internas
- `OrderRepository` (validação de elegibilidade de review)
- `SearchService`
- `ConfigManager`

### Dependência de cloud commerce
- Apenas no provider `cloudCommerce` (VTEX/Magento)

## Inputs / Outputs

### Input
- ID do produto (estratégia configurável)
- Página
- Ordenação
- Credenciais do provider

### Output
- `RatingModel` com:
  - média
  - total
  - reviews
  - paginação
- Estados de UI no controller :contentReference[oaicite:1]{index=1}

---

# 4. Comportamento por engine

## Regra geral
- TrustVox, Judge.me, Konfidency, Bazaar, YourViews são selecionados por chave de configuração
- Podem operar em qualquer engine
- `cloudCommerce` só é instanciado para VTEX ou Magento

## Shopify
- Suportado com providers externos (ex.: Judge.me, Konfidency, potencialmente TrustVox)
- TrustVox pode ser usado em Shopify (sem bloqueio no inicializador)
- Para Shopify com `productId`, há normalização de GID (`gid://...`) para ID numérico

## Diferenças por engine
- VTEX/Magento podem usar provider `cloudCommerce`
- Shopify não usa `cloudCommerce`; depende de provider externo

## Configurações necessárias
- Credencial do provider escolhido
- Estratégia de ID (`rating_product_id_property`)
- Flags de UI/fluxo:
  - `rating_is_active_product_card`
  - `enable_product_review_flow`
  - etc. :contentReference[oaicite:2]{index=2}

---

# 5. Status de implementação

## Exibição de ratings em catálogo/PDP
- Totalmente implementada
- `RatingSearch` enriquece produtos em lote
- UI presente em card, PDP e página de reviews

## Consulta de reviews por provider
- Parcialmente implementada
- Leitura básica em todos os datasources
- Funcionalidades avançadas variam por provider

## Criação de review no app
- Parcialmente implementada
- Implementada em:
  - `VtexRatingDataSource`
  - `KonfidencyRatingDataSource`
- Não implementada nos demais providers

## Review útil e voto útil
- Parcialmente implementada
- TrustVox implementa ambos
- Outros providers: não implementado ou retorno nulo

## Provider cloud commerce
- Parcialmente implementada
- Apenas VTEX/Magento
- Outros engines não recebem datasource cloud commerce :contentReference[oaicite:3]{index=3}

---

# 6. Uso da feature (clientes)

## Método de apuração
Precedência:
1. Remote Config (não vazio)
2. args.properties
3. default

## Base analisada
- `customizations/*`

## Adoção geral (clientes não demo)
- Total: 133
- Com provider ativo: 45 (33,8%)
- Sem provider: 88 (66,2%)

## Distribuição por provider
- TrustVox: 21
- Konfidency: 9
- YourViews: 9
- CloudCommerce: 3
- Judge.me: 2
- Bazaar: 1

## Shopify (não demo, 6 clientes)
- Judge.me: amaro, haoma
- Konfidency: havaianas, plie
- Sem provider: baianao, patbo

## Adoção Shopify
- 4/6 com provider ativo (66,7%)

## Flags de uso
- `enable_product_review_flow=true`: 1 cliente
- `rating_is_active_product_card=true`: 4 clientes
- `rating_upload_media_enabled=true`: 1 cliente :contentReference[oaicite:4]{index=4}

---

# 7. Parâmetros e configuração

## Obrigatórios por provider
- `trustvox_store_key`
- `rating_judge_me_api_token`
- `rating_konfidency_customer`
- `yourviews_store_key`
- `rating_bazaar_voice_api_key`
- `rating_is_cloud_commerce` (VTEX/Magento)

## Funcionais / UI
- `rating_product_id_property`
- `rating_is_active_product_card`
- `enable_product_review_flow`
- `show_approved_reviews_only`
- `rating_upload_media_enabled`
- `ratings_page_style`
- `pdp_star_rating_small_widget_layout`

## Observação crítica
- Provider escolhido por prioridade fixa:
  - bazaar > judgeMe > konfidency > trustvox > yourviews > cloudCommerce
- Se múltiplas credenciais existirem, a prioridade define o provider :contentReference[oaicite:5]{index=5}

---

# 8. Providers e strategies

## Providers
- trustVox
- judgeMe
- konfidency
- bazaar
- yourViews
- cloudCommerce (VTEX/Magento)

## Strategies de ID
- `ProductIdForRatingStrategy`
- `GenericIdForRatingStrategy`
- `SkuIdForRatingStrategy` (default Konfidency)
- `ReferenceCodeIdForRatingStrategy` (default Bazaar)

## Seleção de provider
- Determinada exclusivamente por credenciais/flags
- Não depende de código por cliente :contentReference[oaicite:6]{index=6}

---

# 9. Gaps e limitações

## Cobertura funcional desigual
- Criação de review não universal
- Voto útil não universal
- “Most helpful” não universal

## Problema de elegibilidade
- `canUserReviewProduct` exige status `completed`
- Shopify usa `delivered`, `invoiced`, etc.
- Pode bloquear review flow

## Inconsistência de configuração
- Uso legado de `bazaarvoice_store_key`
- Módulo espera `rating_bazaar_voice_api_key`

## Dependência de configuração
- Conflito entre Remote Config e args.properties dificulta diagnóstico :contentReference[oaicite:7]{index=7}

---

# 10. Impacto e riscos

## Impacto no negócio
- Sem provider → perda de prova social
- Sem review flow → não gera novo conteúdo

## Riscos técnicos
- Seleção errada de provider
- Estratégia de ID incorreta
- Regressão silenciosa por falta de suporte no datasource

## Dependências críticas
- APIs externas
- Configuração consistente
- Mapeamento correto de status de pedido :contentReference[oaicite:8]{index=8}

---

# 11. Requisitos para completar

## Para cobertura completa
- Implementar:
  - `createProductReview`
  - `manageReviewVote`
  - `getMostHelpful`
- Ajustar elegibilidade para engines (ex.: Shopify)
- Normalizar keys de provider
- Criar observabilidade por provider

## Integrações necessárias
- Endpoints de criação/voto
- Fallback para indisponibilidade de provider

## Arquitetura
- Opcional: matriz de capabilities por provider :contentReference[oaicite:9]{index=9}

---

# 12. Critérios de avaliação para agente de IA

## Pré-requisitos
- Provider efetivo resolvido corretamente
- Credenciais válidas
- Estratégia de ID compatível
- Flags coerentes

## Checks funcionais
- PLP com rating enriquecido
- PDP com reviews paginados
- Review flow funcional (se ativo)
- Voto útil funcional (se exibido)

## Sinais de problema
- Ratings vazios com provider configurado
- Divergência de provider
- Shopify sem elegibilidade
- Uso de key legada :contentReference[oaicite:10]{index=10}

---

# Referências técnicas
- `rating_initializer.dart`
- `rating_config.dart`
- `rating_config_registrations.dart`
- `rating_controller.dart`
- `rating_search.dart`
- `trustvox_rating_datasource.dart`
- `judge_me_rating_datasource.dart`
- `konfidency_rating_datasource.dart`
- `vtex_rating_datasource.dart`
- `shopify_orders_service.dart`
- `config_manager.dart`
