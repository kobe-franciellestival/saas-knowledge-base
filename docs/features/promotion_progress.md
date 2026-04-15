---
title: Régua de Frete e Promoções (Promotion Progress Card)
feature: promotion_progress_card
module: cart, pdp, checkout
engine: agnostic
analysis_type: consolidated_analysis
analysis_date: 2026-04-13
confidence: high
status: valid
source: multi_document_merge
suggested_question: Como funciona a régua de frete e promoções no SaaS?
display_priority: 9
---

# 1. Resumo executivo

## Visão consolidada da integração

A régua de frete e promoções é implementada no SaaS por um único componente base chamado `PromotionProgressCard`, utilizado em três contextos de exibição: `cart` (carrinho), `pdp` (página de detalhe de produto) e `snackbar` (barra flutuante em Home/Search/PDP). Não existe implementação de minicart nem de régua visual no checkout — apenas bloqueio lógico de avanço por valor mínimo (`isBlockInitCheckout`).

A feature possui quatro fontes possíveis de dados, com precedência assíncrona e distribuída: Remote Config (baseline), CMS (Contentful ou VTEX CMS, sobrescreve a baseline quando campanha válida), HintUp (personalização dinâmica por estado/região, exclusivo para cliente Biostevi) e payload da engine de e-commerce via `getShippingBarData` (implementado apenas para Magento).

A arquitetura é centrada no modelo `PromotionProgressCardConfig` no `Environment`, com resolução dinâmica no `BaseCartController` e renderização em múltiplos contextos de UI. A feature está funcional e reutilizável, mas heterogênea: a integração de dados da engine é efetiva apenas no Magento, o Shopify CMS retorna `null` (TODO), e coexistem flags legadas (`show_promotion_progress_on_cart`) sem uso no runtime atual.

Há forte evidência de uso em produção em clientes relevantes (gang, mizuno, umbro, savegnago, osklen e ~18 outros com flag ativa), com configuração por Remote Config e/ou CMS.

## Principais capacidades

- Régua visual de progresso até benefício (tipicamente frete grátis) no carrinho, PDP e snackbar.
- Suporte a múltiplas metas (primeira e segunda etapa) com textos configuráveis.
- Quatro fontes de dados com precedência definida: Remote Config, CMS, HintUp e engine de e-commerce.
- Bloqueio de avanço de checkout por valor mínimo não atingido (`isBlockInitCheckout`).
- Dois modos visuais: `contrast` e `highlight`.
- Três bases de cálculo do valor atual: `subTotalValue`, `totalValue`, `priceItems`.
- Integração dinâmica com dados da engine (Magento) para sobrescrever meta, textos e status.
- Personalização dinâmica por estado/região via HintUp (Biostevi).
- Alta capacidade de configuração por cliente via Remote Config e CMS.

## Principais limitações

- Integração de dados da engine (`getShippingBarData`) implementada apenas para Magento; VTEX, Shopify, Salesforce e Wake retornam `null`.
- Shopify CMS retorna `null` com TODO — campanha progressiva via CMS não disponível para clientes Shopify.
- Minicart: sem implementação identificada.
- Checkout visual: sem régua visual; apenas bloqueio lógico de avanço.
- PDP e snackbar não passam `CheckoutService`, portanto não recebem payload dinâmico da engine.
- Flag legada `show_promotion_progress_on_cart` presente em muitos arquivos de configuração, mas sem leitura no runtime atual do app.
- Mensagem de bloqueio no Contentful hardcoded ("Valor mínimo não atingido"), sem campo dedicado.
- Precedência entre fontes de dados é assíncrona e distribuída, com risco de inconsistência entre mensagem exibida e regra real de frete/promoção.
- HintUp restrito ao cliente Biostevi; não é extensão genérica disponível para todos os clientes.

---

# 2. Visão geral da integração

## O que a integração cobre

- Comunicação visual do progresso do usuário até um benefício (tipicamente frete grátis) durante a jornada de compra.
- Exibição em três contextos: carrinho, PDP e snackbar (Home/Search/PDP).
- Bloqueio funcional de avanço para checkout quando o valor mínimo não é atingido.
- Configuração de campanha promocional progressiva com múltiplas metas, textos e ícone via CMS.
- Personalização dinâmica de thresholds/textos por estado ou região via HintUp (Biostevi).
- Sobrescrita de meta, mensagens e status por dados em tempo real da engine de e-commerce (Magento).

## O que não está no escopo atual

- Régua visual no checkout (apenas bloqueio lógico).
- Régua no minicart.
- Campanha progressiva via Shopify CMS.
- Dados dinâmicos de shipping bar para engines VTEX, Shopify, Salesforce e Wake.

## Objetivo dentro do SaaS

- Aumentar conversão e ticket médio por incentivo de valor mínimo para benefício.
- Oferecer configuração de campanha promocional progressiva por cliente sem desenvolvimento adicional.
- Bloquear avanço de checkout quando regra de valor mínimo não é atingida (configurável).

---

# 3. Arquitetura e funcionamento

## Componentes principais

| Tipo | Nome | Caminho | Responsabilidade |
|---|---|---|---|
| Inicialização | App init | `app/lib/initializers/app_initializer.dart` | Carrega Remote Config, dispara fetch CMS assíncrono e registra provider |
| Inicialização | RemoteConfig mapping | `app/lib/initializers/remote_config_values_initializer.dart` | Mapeia chaves `promotion_progress_*` |
| Modelo | Card config | `app/lib/modules/config/models/promotion_progress_card_config.dart` | Contrato unificado da configuração da régua |
| UI | Promotion card | `app/lib/modules/cart/pages/promotion/promotion_progress_card.dart` | Renderização, animação, fallback e integração shipping bar |
| UI | Cart insertion | `app/lib/modules/cart/widgets/cart_list_view_widget.dart` | Insere card no carrinho com `CheckoutService` |
| UI | PDP insertion | `app/lib/modules/design_system/components/product_edit/product_edit.dart` | Insere card na PDP (sem CheckoutService) |
| UI | Snackbar mixin | `app/lib/modules/cart/widgets/promotion_snackbar/promotion_progress_snackbar_mixin.dart` | Observa triggers e exibe snackbar (sem CheckoutService) |
| Lógica | Base cart controller | `app/lib/modules/cart/controller/base_cart_controller.dart` | Resolve config dinâmica e calcula valor atual |
| Bloqueio | Checkout gate | `app/lib/util/cart_utils.dart` | Bloqueia `onCloseOrder` por valor mínimo |
| CMS adapter | Campaign adapter | `app/lib/adapters/cms_adapter/model/shipping_progress_bar/shipping_progress_bar_campaign_adapter.dart` | Contrato de campanha CMS |
| CMS Contentful | Delivery impl | `app/lib/modules/contentful_api/content_delivery.dart` | Busca campanha e mapeia promoções |
| CMS VTEX | Delivery impl | `app/lib/modules/vtex_cms_api/vtex_content_delivery.dart` | Busca campanha no VTEX CMS |
| CMS Shopify | Delivery impl | `app/lib/modules/shopify_cms/shopify_content_delivery.dart` | Retorna `null` (TODO) |
| E-commerce contract | CheckoutService API | `app/lib/adapters/modular_adapter/services/checkout_service.dart` | Contrato `getShippingBarData()` |
| E-commerce Magento | Shipping bar service | `app/lib/modules/magento_api/services/magento_shipping_bar_service.dart` | Consulta GraphQL `getShippingBar` |
| HintUp provider | HintUp provider | `app/lib/modules/promotion_progress/hint_up/hint_up_provider.dart` | Personalização dinâmica por UF/região |
| HintUp Biostevi | Biostevi service | `app/lib/custom_modules/biostevi/hint_up/domain/services/biostevi_hint_up_service.dart` | Implementação HintUp para Biostevi |

## Fontes de verdade e precedência

A resolução de configuração da régua segue precedência assíncrona e distribuída, com quatro camadas:

| Ordem | Fonte | O que define | Quando sobrescreve |
|---|---|---|---|
| 1 (baseline) | Remote Config | Enable, textos, metas, tipo de valor, modo visual, bloqueio de checkout, páginas de inserção | Sempre (base inicial) |
| 2 (async) | CMS (Contentful / VTEX CMS) | Ativação, páginas, ícone, tipo, promoções (1+ metas), bloqueio | Sobrescreve Remote Config se campanha válida (`enabled=true` e `firstValue>0`) |
| 3 (runtime) | HintUp (Biostevi) | Thresholds, textos, checkpoints, mixed-values por UF/região | Ajusta config dinamicamente após update do orderForm |
| 4 (runtime cart) | Payload engine (Magento) | `goal`, mensagens, `status` | Sobrescreve meta e textos no card do carrinho; pode desabilitar card se `status` indicar |

**Risco de precedência:** a busca CMS é disparada sem `await` no init, podendo a UI renderizar com config do Remote Config antes de a campanha CMS ser carregada. Isso cria janela de inconsistência entre mensagem exibida e regra real.

## Fluxo detalhado

```
1. App inicializa Remote Config → monta Environment.promotionProgress*
2. App dispara (sem await) busca CMS da campanha progressiva (Contentful ou VTEX CMS)
3. Se campanha CMS válida (enabled=true e firstValue>0): sobrescreve config no Environment
4. Provider de promoção registrado: default ou HintUp (se saasClientId=biostevi)
5. Controllers de carrinho chamam resolvePromotionProgressCardConfigs() no orderFormManager.onUpdate
6. Provider ajusta config dinamicamente (HintUp: por UF/região)
7. UI cart/PDP/snackbar lê config reativa do controller
8. currentValue calculado por typeValue (priceItems / priceTotal / subtotal sem frete)
9. No cart: PromotionProgressCard tenta checkoutService.getShippingBarData()
   → Magento: retorna payload com goal/mensagens/status → sobrescreve config local
   → VTEX/Shopify/Salesforce/Wake: retorna null → mantém config local
10. Bloqueio de checkout: CartUtils.onCloseOrder() verifica isBlockInitCheckout
    → também verificado no ShippingChooseUnifiedController
```

## Terminologia do código

| Termo | Contexto |
|---|---|
| `PromotionProgressCard` | Componente UI principal |
| `PromotionProgressCardConfig` | Modelo de configuração unificado |
| `promotion progress snackbar` | Contexto snackbar |
| `shipping progress bar campaign` / `shipping bar` | Campanha CMS |
| `progressive-promotional-campaign` | Content type CMS |
| `getShippingProgressBarCampaign` | Método de busca de campanha no CMS |
| `getShippingBarData` | Método de dados dinâmicos da engine |
| `ShippingProgressBarCampaignAdapter` | Adapter de campanha CMS |
| `ShippingProgressPromotionAdapter` | Adapter de promoção individual |
| `TotalValueType` | Enum: `subTotalValue`, `totalValue`, `priceItems` |
| `showInCart`, `showInPdp`, `showInSnackbar` | Páginas de inserção |
| `shouldBlockCheckoutWithoutMinValue` | Flag de bloqueio de checkout |
| `blockCheckoutWithoutMinValueErrorMessage` | Mensagem de erro de bloqueio |
| `show_promotion_progress_on_cart` | Flag legada (sem uso no runtime atual) |
| `IPromotionProgressCardProvider` | Interface de provider (extensão por cliente) |

---

# 4. Comportamento por engine

## VTEX

- **Status:** Parcial.
- **Fonte de dados:** Remote Config / CMS / HintUp.
- `getShippingBarData()` retorna `null` — sem fonte dinâmica de shipping bar da engine.
- Régua visual funciona com dados estáticos de Remote Config ou CMS (Contentful ou VTEX CMS).
- VTEX CMS suporta campanha progressiva via `shipping_progress_bar_extension` (implementado).

## Shopify

- **Status:** Parcial.
- **Fonte de dados:** Remote Config apenas (CMS retorna `null`).
- `getShippingBarData()` retorna `null`.
- Shopify CMS: método `getShippingProgressBarCampaign` retorna `null` com TODO — campanha progressiva via CMS não disponível.
- Régua visual funciona apenas com dados de Remote Config.

## Salesforce Commerce (SFCC)

- **Status:** Parcial.
- **Fonte de dados:** Remote Config / CMS (Contentful, se configurado) / HintUp.
- `getShippingBarData()` retorna `null` (B2C e B2B).
- Sem fonte dinâmica de shipping bar da engine.

## Magento

- **Status:** Implementado (mais completo).
- **Fonte de dados:** Remote Config / CMS / HintUp + Magento GraphQL `getShippingBar`.
- `MagentoCheckoutService` e `MagentoShippingBarService` implementam `getShippingBarData()` via GraphQL.
- Card do carrinho pode sobrescrever `firstValue`, mensagens e `enabled` com payload Magento.
- **Limitação:** integração dinâmica Magento disponível apenas no card do carrinho; PDP e snackbar não passam `CheckoutService`.

## Wake Commerce

- **Status:** Parcial.
- **Fonte de dados:** Remote Config / CMS / HintUp.
- `getShippingBarData()` retorna `null`.
- Sem fonte dinâmica de shipping bar da engine.

## Diferenças relevantes por engine

| Engine | getShippingBarData | CMS de campanha | Fonte dinâmica |
|---|---|---|---|
| VTEX | null | VTEX CMS / Contentful | Não |
| Shopify | null | Não implementado (TODO) | Não |
| Salesforce | null | Contentful (se configurado) | Não |
| Magento | Implementado (GraphQL) | Contentful (se configurado) | Sim (cart apenas) |
| Wake | null | Contentful (se configurado) | Não |

---

# 5. Features suportadas

## Régua visual no carrinho (Cart)

- Renderização de `PromotionProgressCard` no `CartListViewWidget`.
- Animação de progresso até a meta.
- Suporte a primeira e segunda etapa com textos distintos.
- Texto de prefixo, sufixo e sucesso configuráveis.
- Dois modos visuais: `contrast` e `highlight`.
- Ícone configurável via CMS.
- Integração com `CheckoutService.getShippingBarData()` para sobrescrita dinâmica (Magento).
- Bloqueio de checkout por valor mínimo (`isBlockInitCheckout`).

## Régua visual na PDP

- Renderização de `PromotionProgressCard` via `product_edit.dart`.
- Sem integração com `CheckoutService` (sem dados dinâmicos da engine).
- Configurável de forma independente do cart via flags `*_pdp_enabled` e `*_pdp_first_value`/`*_max_value`.

## Snackbar (Home/Search/PDP)

- `PromotionProgressSnackbarMixin` observa triggers e exibe snackbar reativo.
- Sem integração com `CheckoutService`.
- Configurável de forma independente via flags `*_snackbar_enabled` e `*_snackbar_first_value`/`*_max_value`.
- Modelo `PromotionProgressSnackbarConfig` existe, mas sem uso efetivo no fluxo atual.

## Bloqueio de checkout

- Bloqueia `CartUtils.onCloseOrder()` quando `isBlockInitCheckout=true` e `currentValue < firstValue`.
- Também verificado no `ShippingChooseUnifiedController`.
- Mensagem de erro configurável via Remote Config ou CMS (Contentful: hardcoded; campo dedicado ausente).
- Sem régua visual dedicada no checkout; apenas bloqueio lógico.

## Múltiplas metas

- Suporte a primeira e segunda etapa com thresholds e textos distintos.
- Via CMS: array de `promotions` com `minValue`, `inactiveMessage`, `activeMessage`, `textSuffix` por etapa.
- Via Remote Config: `first_value`/`max_value` e `second_value` com textos separados.

## Cálculo do valor atual

- `TotalValueType`: `subTotalValue` (subtotal sem frete), `totalValue` (total com frete) ou `priceItems` (preço dos itens).
- Calculado em `BaseCartController` a cada atualização do orderForm.

## Personalização por cliente (HintUp)

- Thresholds, textos, checkpoints e mixed-values configuráveis por UF/região.
- Suporte explícito apenas para `saasClientId=biostevi`.
- Interface `IPromotionProgressCardProvider` disponível como ponto de extensão para novos provedores.

## Minicart

- Não implementado.

## Checkout visual

- Não implementado. Apenas bloqueio lógico de avanço.

---

# 6. Status de implementação

| Feature / Contexto | Engine / CMS | Status | Observação |
|---|---|---|---|
| Régua visual no carrinho | Todos (Remote Config) | Implementado | Funcional com dados estáticos |
| Régua visual no carrinho | Magento (engine data) | Implementado | Sobrescrita dinâmica via GraphQL |
| Régua visual no carrinho | VTEX / Shopify / Salesforce / Wake (engine data) | Parcial | `getShippingBarData()` retorna `null` |
| Régua visual na PDP | Todos (Remote Config / CMS) | Implementado | Sem dados dinâmicos da engine |
| Snackbar | Todos (Remote Config / CMS) | Implementado | `PromotionProgressSnackbarConfig` existe sem uso efetivo |
| Bloqueio de checkout | Todos | Implementado | Lógico; sem UI visual no checkout |
| Régua visual no checkout | Todos | Não implementado | Apenas bloqueio lógico |
| Régua no minicart | Todos | Não implementado | Sem widget/contexto de minicart |
| Campanha progressiva via CMS | Contentful | Implementado | Apenas 1 campanha (`limit:1`); mensagem de bloqueio hardcoded |
| Campanha progressiva via CMS | VTEX CMS | Implementado | Ordem de promoções depende de `sections` |
| Campanha progressiva via CMS | Shopify CMS | Não implementado | Retorna `null` com TODO |
| Múltiplas metas | Todos | Implementado | Via CMS (array) ou Remote Config (first/second value) |
| Segunda meta | Remote Config | Implementado | `second_value` e textos associados |
| HintUp (personalização por região) | Biostevi | Implementado | Exclusivo para `saasClientId=biostevi` |
| HintUp (outros clientes) | — | Não implementado | Interface extensível disponível |
| Flag legada `show_promotion_progress_on_cart` | — | Legado | Presente em configs; sem leitura no runtime atual |
| `PromotionProgressSnackbarConfig` | — | Parcial | Modelo existe; sem uso efetivo no fluxo atual |
| Analytics de promoção/frete | — | Não mapeado | Sem evidência de eventos de analytics específicos da régua |

---

# 7. Uso atual (clientes)

## Clientes com evidência forte de uso

| Cliente | Flags ativas | Escopo | Observação |
|---|---|---|---|
| gang | `card_enabled`, `pdp_enabled`, `snackbar_enabled` = true | Cart + PDP + Snackbar | Evidência forte — configuração ativa em `customizations/gang/args_remote.json` |
| mizuno | Idem | Cart + PDP + Snackbar | Evidência forte — `customizations/mizuno/args_remote.json` |
| umbro | Idem | Cart + PDP + Snackbar | Evidência forte — `customizations/umbro/args_remote.json` |
| savegnago | `card_enabled=true`, `is_block_init_checkout=true` | Cart + bloqueio checkout | Também em `scripts/remote_config/output.json` com parâmetro legado |
| osklen | `card_enabled=true`, `show_promotion_progress_on_cart=true` | Cart | Mistura flag atual + legado |

## Clientes com indício de uso (flag ativa, sem prova de publicação em produção)

americanas, vix, angeloni, atacadao, b_blend, casas_pedro, levis, luz_da_lua, paulistao_atacadista, rissul, saint_james, highstil, granado, farmacias_sao_joao, haoma, underarmour, drogarias_minas_mais (~18 clientes com `promotion_progress_card_enabled=true` em `args_remote.json`).

## Padrões de uso identificados

- Configuração predominantemente via Remote Config.
- Clientes que ativam a régua tendem a habilitar os três contextos simultaneamente (cart + pdp + snackbar).
- Bloqueio de checkout (`is_block_init_checkout=true`) presente em configuração de clientes com regime de frete grátis condicional.
- Coexistência de flag atual e flag legada em alguns clientes (osklen, savegnago).

---

# 8. Parâmetros e configuração

## Parâmetros de Remote Config

### Habilitação por contexto

| Parâmetro | Tipo | Contexto | Default |
|---|---|---|---|
| `promotion_progress_card_enabled` | bool | Cart | `false` |
| `promotion_progress_card_pdp_enabled` | bool | PDP | `false` |
| `promotion_progress_snackbar_enabled` | bool | Snackbar | `false` |

### Metas e textos — Cart

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `promotion_progress_card_first_value` | number | Meta primária do cart; usa `max_value` quando `first_value==0` |
| `promotion_progress_card_max_value` | number | Fallback de meta quando `first_value==0` |
| `promotion_progress_card_second_value` | number | Segunda meta (0 = desabilitada) |
| `promotion_progress_card_text_promotion_prefix` | string | Texto antes do valor |
| `promotion_progress_card_text_promotion_sufix` | string | Texto após o valor |
| `promotion_progress_card_text_promotion_success` | string | Texto de sucesso ao atingir meta |
| `promotion_progress_card_second_text_promotion_*` | string | Textos da segunda meta |
| `promotion_progress_card_mode` | string enum | Modo visual: `contrast` / `highlight` (default: `contrast`) |

### Metas e textos — PDP

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `promotion_progress_card_pdp_first_value` | number | Meta primária da PDP |
| `promotion_progress_card_pdp_max_value` | number | Fallback de meta PDP |

### Metas e textos — Snackbar

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `promotion_progress_snackbar_first_value` | number | Meta primária do snackbar |
| `promotion_progress_snackbar_max_value` | number | Fallback de meta snackbar |

### Tipo de valor e bloqueio

| Parâmetro | Tipo | Descrição | Default |
|---|---|---|---|
| `type_value` | string enum | Base de cálculo: `subTotalValue`, `totalValue`, `priceItems` | `subTotalValue` |
| `promotion_progress_card_is_block_init_checkout` | bool | Bloqueia checkout abaixo da meta | `false` |
| `promotion_progress_card_message_error_is_block_init_checkout` | string | Mensagem do bloqueio | vazio |

### Flag legada (sem uso no runtime atual)

| Parâmetro | Observação |
|---|---|
| `show_promotion_progress_on_cart` | Presente em muitos arquivos de config; sem leitura no runtime do app atual |

**Localização de mapeamento:** `remote_config_values_initializer.dart:1851` — `_setupPromotionProgressCardConfigs`.

## Parâmetros de CMS (Contentful / VTEX CMS)

| Campo CMS | Tipo | Descrição |
|---|---|---|
| `enabled` | bool | Habilita campanha |
| `showInCart` / `showInPdp` / `showInSnackbar` | bool | Páginas de inserção |
| `type` | string | Tipo de campanha |
| `icon` | asset | Ícone exibido na régua |
| `beginDate` / `endDate` | datetime | Período de vigência |
| `blockCheckout` / `shouldBlockCheckoutWithoutMinValue` | bool | Bloquear checkout |
| `promotions[].minValue` | number | Threshold de cada etapa |
| `promotions[].inactiveMessage` | string | Mensagem antes de atingir meta |
| `promotions[].activeMessage` | string | Mensagem ao atingir meta |
| `promotions[].textSuffix` | string | Sufixo de texto |
| Mensagem de bloqueio | — | Hardcoded em Contentful ("Valor mínimo não atingido"); sem campo dedicado |

**Content type:** `progressive-promotional-campaign`. Contentful busca apenas 1 campanha (`limit:1`).

## Parâmetros de engine de e-commerce (Magento / `getShippingBarData`)

| Campo | Tipo | Descrição |
|---|---|---|
| `achieve_message` | string | Mensagem ao atingir meta |
| `below_goal_message` | string | Mensagem abaixo da meta |
| `first_message` | string | Primeira mensagem |
| `goal` | number | Meta dinâmica da engine |
| `status` | mixed | Status do shipping bar; pode desabilitar card |

Default/fallback: `null` → mantém config local (Remote Config ou CMS).

**Limitação:** payload disponível apenas no card do carrinho; PDP e snackbar não passam `CheckoutService`.

## Parâmetros de HintUp (Biostevi)

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `hint_up_tokens` | JSON | Tokens de autenticação HintUp |
| `hint_up_insertion_pages` | list | Páginas habilitadas para HintUp |
| Thresholds / textos / checkpoints / mixed-values | — | Configurados via serviço HintUp por UF/região |

---

# 9. Providers e strategies

## Seleção de provider

O app registra um provider de `IPromotionProgressCardProvider` no startup:
- **Default:** resolução padrão via config do `Environment` (Remote Config + CMS).
- **HintUp:** ativo quando `saasClientId=biostevi` e módulo HintUp configurado; ajusta config dinamicamente por UF/região no `orderFormManager.onUpdate`.

## Providers de CMS

| CMS | Implementação | Capacidades |
|---|---|---|
| Contentful | `content_delivery.dart` | Busca campanha `progressive-promotional-campaign`, mapeia promoções, datas, bloqueio |
| VTEX CMS | `vtex_content_delivery.dart` | Busca via `shipping_progress_bar_extension`; mapeia por `sections` |
| Shopify CMS | `shopify_content_delivery.dart` | Retorna `null` (TODO — não implementado) |

## Providers de engine de e-commerce

| Engine | Implementação | Capacidades |
|---|---|---|
| Magento | `magento_shipping_bar_service.dart` | GraphQL `getShippingBar`; retorna goal, mensagens, status |
| VTEX | `checkout.dart` | Retorna `null` |
| Shopify | `shopify_checkout_service.dart` | Retorna `null` |
| Salesforce | — | Retorna `null` (B2C e B2B) |
| Wake | — | Retorna `null` |

## Pontos de extensão

- **Nova fonte CMS:** implementar `getShippingProgressBarCampaign` no `ContentDelivery` da engine CMS desejada.
- **Nova fonte de engine:** implementar `getShippingBarData` no `CheckoutService` da engine.
- **Novo comportamento por cliente:** implementar `IPromotionProgressCardProvider` (modelo HintUp).
- **Novos contextos de UI:** ampliar enum `PromotionProgressCardInsertionPage` (`cart`, `pdp`, `snackbar`) e adicionar pontos de renderização.

---

# 10. Gaps e limitações

## Gaps técnicos

| Gap | Localização | Impacto |
|---|---|---|
| Shopify CMS retorna `null` | `shopify_content_delivery.dart` | Clientes Shopify sem campanha progressiva via CMS |
| `getShippingBarData` não implementado (VTEX) | `checkout.dart` | Sem dados dinâmicos da engine para VTEX |
| `getShippingBarData` não implementado (Shopify) | `shopify_checkout_service.dart` | Idem para Shopify |
| `getShippingBarData` não implementado (Salesforce) | — | Idem para Salesforce |
| `getShippingBarData` não implementado (Wake) | — | Idem para Wake |
| PDP e snackbar sem `CheckoutService` | `product_edit.dart`, snackbar mixin | Dados dinâmicos da engine indisponíveis nesses contextos |
| Régua visual no checkout | Ausente | Usuário bloqueado sem referência visual de progresso |
| Régua no minicart | Ausente | Contexto de minicart sem suporte |
| Mensagem de bloqueio hardcoded | Contentful | Sem campo dedicado; não configurável via CMS |
| Flag legada `show_promotion_progress_on_cart` | `customizations/*` | Presente em configs mas sem leitura no runtime; risco de confusão |
| `PromotionProgressSnackbarConfig` sem uso efetivo | — | Modelo existe sem fluxo funcional completo |
| HintUp restrito a Biostevi | `biostevi_hint_up_service.dart` | Personalização dinâmica por região não disponível para outros clientes sem extensão |

## Limitações de arquitetura

- **Precedência assíncrona:** Remote Config é carregado sincronicamente; CMS é buscado sem `await`. Existe janela de inconsistência entre mensagem exibida e campanha real enquanto o CMS não respondeu.
- **Enum de inserção incompleto:** `PromotionProgressCardInsertionPage` cobre apenas `cart`, `pdp` e `snackbar`. Checkout e minicart não são contemplados.
- **Cobertura de testes:** concentrada em Magento service; pouco teste de fluxo ponta a ponta de UI/config por engine.
- **Status Magento pode desabilitar card:** se `status` retornado pelo `getShippingBarData` indicar inativo, o card é desabilitado mesmo com config ativa no Remote Config/CMS — comportamento que pode surpreender operadores.

## Riscos identificados

- Divergência entre mensagem exibida na régua e regra real de frete/promoção (causada pela precedência assíncrona).
- Falsa percepção de configuração: clientes com `show_promotion_progress_on_cart=true` podem acreditar que a flag está ativa no runtime quando não está.
- Clientes Shopify sem CMS de campanha podem ter régua com dados estáticos desatualizados se Remote Config não for atualizado manualmente.

---

# 11. Impacto e riscos

## Impacto de negócio

- Régua de frete grátis é feature de alta conversão e ticket médio — gaps de implementação por engine reduzem eficácia do incentivo.
- Bloqueio de checkout sem UI visual no checkout pode causar confusão e abandono não intencional do usuário.
- Inconsistência entre mensagem exibida e regra real de promoção pode gerar reclamações e perda de confiança.

## Riscos técnicos

- **Precedência assíncrona:** possível inconsistência de UX entre o que o Remote Config exibe e o que a campanha CMS deveria mostrar.
- **Flag legada ativa:** `show_promotion_progress_on_cart` configurada em clientes sem efeito no runtime pode levar times de produto a assumir que a feature está ativa quando não está.
- **Status Magento sobrescreve config:** pode desabilitar silenciosamente o card sem aviso para o operador.
- **Shopify CMS TODO:** clientes Shopify que dependem de campanha progressiva via CMS não têm suporte atual.
- **Cobertura de testes insuficiente:** fluxo ponta a ponta por engine/contexto com baixa cobertura aumenta risco de regressão.

## Clientes mais expostos aos riscos

- **Clientes Shopify** com necessidade de campanha progressiva via CMS: sem suporte atual.
- **Clientes com bloqueio de checkout ativo** (`savegnago`): UI de checkout sem régua visual pode causar confusão.
- **Clientes com flag legada ativa** (`osklen`): flag `show_promotion_progress_on_cart` sem efeito no runtime.

---

# 12. Requisitos para completar

## Hardening de precedência e consistência

- Formalizar e documentar a precedência entre fontes de dados (Remote Config → CMS → HintUp → engine).
- Considerar `await` no fetch CMS durante o init ou exibir estado de loading enquanto CMS não responde, eliminando janela de inconsistência.
- Centralizar resolução de configuração em um único serviço em vez de distribuir entre init, controller e provider.

## Implementações ausentes críticas

- **Shopify CMS:** implementar `getShippingProgressBarCampaign` em `shopify_content_delivery.dart` (remover TODO).
- **`getShippingBarData` para VTEX, Shopify, Salesforce, Wake:** avaliar viabilidade de API dinâmica de shipping bar por engine e implementar onde aplicável.
- **PDP e snackbar com dados dinâmicos:** avaliar passar `CheckoutService` para esses contextos ou criar mecanismo alternativo de atualização.

## Melhorias de UX e configuração

- **Régua visual no checkout:** implementar exibição da régua no fluxo de checkout, especialmente quando `isBlockInitCheckout=true`.
- **Mensagem de bloqueio configurável no Contentful:** criar campo dedicado no content type em vez de hardcode.
- **Régua no minicart:** avaliar demanda e implementar se necessário, ampliando `PromotionProgressCardInsertionPage`.
- **Deprecar flag legada:** remover ou migrar `show_promotion_progress_on_cart` nos arquivos de configuração de clientes.

## Extensões de plataforma

- **Generalizar HintUp:** abstrair provider HintUp como extensão genérica disponível para outros clientes além de Biostevi.
- **Cobertura de testes:** ampliar testes de integração ponta a ponta por engine e contexto de UI.
- **Telemetria/analytics:** implementar eventos de analytics específicos da régua (ex.: `promotion_bar_viewed`, `checkout_blocked_by_min_value`, `goal_reached`).

---

# 13. Critérios de avaliação para agente de IA

## Pré-requisitos

- `promotion_progress_card_enabled=true` (ou `pdp_enabled`/`snackbar_enabled`) na configuração do cliente.
- Meta (`first_value` ou `max_value`) definida e maior que zero.
- Se CMS: campanha `progressive-promotional-campaign` com `enabled=true` e `firstValue>0` publicada.
- Se bloqueio: `promotion_progress_card_is_block_init_checkout=true` e mensagem de erro configurada.
- Se Shopify: verificar que CMS de campanha não está disponível (retorna `null`) e que dados virão apenas de Remote Config.
- Identificar engine do cliente para determinar se `getShippingBarData` retornará dados (apenas Magento).

## Checks funcionais

- `PromotionProgressCard` exibido no carrinho com valor de progresso atualizado a cada mudança do orderForm.
- Valor atual (`currentValue`) calculado corretamente conforme `typeValue` configurado.
- Textos de prefixo, sufixo e sucesso renderizados conforme configuração.
- Segunda etapa exibida corretamente quando `second_value > 0`.
- Bloqueio de checkout acionado quando `isBlockInitCheckout=true` e `currentValue < firstValue`.
- Mensagem de erro exibida corretamente no bloqueio.
- Campanha CMS sobrescreve Remote Config quando carregada (verificar janela assíncrona).
- No Magento: `getShippingBarData` retorna payload e sobrescreve config do card do carrinho.

## Validação por contexto

| Contexto | Check |
|---|---|
| Cart | Card exibido; progresso atualizado; bloqueio funcional; dados Magento sobrescrevem config (se Magento) |
| PDP | Card exibido com config independente (`pdp_first_value`); sem dados dinâmicos de engine |
| Snackbar | Snackbar disparado nos triggers corretos; config independente (`snackbar_first_value`) |
| Checkout | Bloqueio lógico ativo; sem UI visual de régua |
| Minicart | Sem suporte — verificar se cliente tem expectativa de exibição nesse contexto |

## Sinais de problema

- Régua não exibida com flag ativa: verificar se `first_value > 0` e se CMS retornou campanha válida.
- Mensagem desatualizada na régua: possível janela assíncrona CMS ainda não carregado; verificar precedência.
- Bloqueio de checkout sem mensagem: verificar `message_error_is_block_init_checkout`.
- Card desabilitado no Magento com config ativa: verificar `status` retornado por `getShippingBarData`.
- Shopify sem campanha CMS: comportamento esperado — Shopify CMS retorna `null` (TODO).
- Flag `show_promotion_progress_on_cart=true` sem efeito: flag legada sem uso no runtime atual.
- PDP/snackbar com dados desatualizados: sem `CheckoutService` nesses contextos; dados estáticos de Remote Config/CMS.

---

# Referências técnicas

## Arquivos de código referenciados

| Arquivo | Responsabilidade |
|---|---|
| `app/lib/initializers/app_initializer.dart` | Ordem de init, fetch CMS assíncrono, registro de provider |
| `app/lib/initializers/remote_config_values_initializer.dart` | Mapeamento de todas as chaves `promotion_progress_*` (line 1851) |
| `app/lib/modules/config/models/promotion_progress_card_config.dart` | Contrato unificado da régua (`PromotionProgressCardConfig`) |
| `app/lib/modules/cart/pages/promotion/promotion_progress_card.dart` | UI, animação, fallback, integração `getShippingBarData` (line 114) |
| `app/lib/modules/cart/widgets/cart_list_view_widget.dart` | Inserção do card no carrinho com `CheckoutService` |
| `app/lib/modules/design_system/components/product_edit/product_edit.dart` | Inserção do card na PDP |
| `app/lib/modules/cart/widgets/promotion_snackbar/promotion_progress_snackbar_mixin.dart` | Snackbar reativo |
| `app/lib/modules/cart/controller/base_cart_controller.dart` | Resolução dinâmica de config e cálculo de `currentValue` (line 1128) |
| `app/lib/util/cart_utils.dart` | Bloqueio `onCloseOrder` por valor mínimo |
| `app/lib/adapters/cms_adapter/model/shipping_progress_bar/shipping_progress_bar_campaign_adapter.dart` | Adapter de campanha CMS (`ShippingProgressBarCampaignAdapter`) |
| `app/lib/adapters/modular_adapter/services/checkout_service.dart` | Contrato `getShippingBarData()` |
| `app/lib/modules/contentful_api/content_delivery.dart` | Campanha progressiva via Contentful (line 1581) |
| `app/lib/modules/vtex_cms_api/vtex_content_delivery.dart` | Campanha progressiva via VTEX CMS (line 613) |
| `app/lib/modules/shopify_cms/shopify_content_delivery.dart` | Retorna `null` com TODO |
| `app/lib/modules/magento_api/services/magento_shipping_bar_service.dart` | GraphQL `getShippingBar` — fonte dinâmica Magento |
| `app/lib/modules/vtex_api/repositories/checkout/checkout.dart` | `getShippingBarData` retorna `null` |
| `app/lib/modules/promotion_progress/hint_up/hint_up_provider.dart` | Provider HintUp (line 34) |
| `app/lib/custom_modules/biostevi/hint_up/domain/services/biostevi_hint_up_service.dart` | Implementação HintUp para Biostevi (line 23) |
| `customizations/default_remote.json` | Flag legada `show_promotion_progress_on_cart` |
| `customizations/gang/args_remote.json` | Evidência de card + pdp + snackbar ativos |
| `customizations/osklen/args_remote.json` | Flag atual + legada ativas |
| `customizations/savegnago/args_remote.json` | Card + bloqueio de checkout ativos |

## Terminologia e enums

| Termo | Tipo | Valores |
|---|---|---|
| `TotalValueType` | enum | `subTotalValue`, `totalValue`, `priceItems` |
| `PromotionProgressCardInsertionPage` | enum | `cart`, `pdp`, `snackbar` (checkout e minicart ausentes) |
| `promotion_progress_card_mode` | string enum | `contrast`, `highlight` |
| Content type CMS | string | `progressive-promotional-campaign` |

## Queries e APIs externas

| Serviço | Query / Endpoint | Descrição |
|---|---|---|
| Magento GraphQL | `getShippingBar` | Retorna `achieve_message`, `below_goal_message`, `first_message`, `goal`, `status` |
| Contentful | `getShippingProgressBarCampaign` | Busca campanha `progressive-promotional-campaign` (`limit:1`) |
| VTEX CMS | `shipping_progress_bar_extension` | Busca campanha via `sections` |
| Shopify CMS | `getShippingProgressBarCampaign` | Não implementado — retorna `null` (TODO) |
