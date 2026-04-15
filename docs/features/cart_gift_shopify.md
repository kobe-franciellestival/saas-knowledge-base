---
title: Brinde no Carrinho (Gift in Cart)
feature: cart_gift
module: cart, checkout
engine: shopify
analysis_type: consolidated_analysis
analysis_date: 2026-04-13
confidence: high
status: valid
source: multi_document_merge
suggested_question: Como funciona o brinde no carrinho no SaaS para engine Shopify?
display_priority: 9
---

# 1. Resumo executivo

## Visão consolidada da integração

A feature de brinde no carrinho (`CartGiftList`, `show_gift_list_on_cart`) existe no SaaS como capacidade estrutural do módulo de carrinho e checkout, com implementação completa para engine VTEX e **não implementada** para engine Shopify.

O app possui a camada de UI (`CartGiftList`), os parâmetros de Remote Config e a orquestração via `OrderFormManager`, mas o provider Shopify — tanto no `ShopifyCartController` quanto no `ShopifyCheckoutService` — retorna `UnimplementedError` em todos os métodos críticos de brinde. Adicionalmente, o `OrderFormAdapter` do Shopify não popula `selectableGifts`, tornando a feature estruturalmente inoperante para clientes dessa engine.

O checkout Shopify é renderizado em webview, o que cria uma dissociação fundamental: o item grátis pode aparecer no resumo do pedido se já tiver sido inserido pelo backend Shopify (via promoção ou plugin de tema), mas esse fluxo não é controlado pelo módulo nativo de brinde do app.

O contexto de análise é o lead **Vizzela** (engine Shopify), para o qual o brinde acima de R$ 180 é requisito crítico. No estado atual do código, esse requisito não pode ser atendido sem implementação do provider Shopify.

## Principais capacidades

- UI de brinde no carrinho (`CartGiftList`) presente e configurável via Remote Config.
- Inserção condicional da lista de brindes no carrinho via flag `show_gift_list_on_cart`.
- Orquestração completa via `OrderFormManager` (addGift/removeGift/updateGifts).
- Fluxo de brinde totalmente implementado para engine VTEX (controller + repository + checkout).
- Suporte a múltiplos parâmetros de configuração de UX (mensagem, obrigatoriedade, página de frete).
- 6 de 9 clientes Shopify no repositório com `show_gift_list_on_cart=true` (66,7% de adoção configurada).

## Principais limitações

- Provider Shopify com `UnimplementedError` em todos os métodos de brinde: `addGift`, `removeGift`, `updateGifts`, `setGiftWrappingOnCartItem`, `giftSelected`, `getGiftSelectedQuantity`.
- `selectableGifts` não populado no `OrderFormAdapter` do Shopify.
- Checkout Shopify em webview: app não controla a renderização do resumo do pedido.
- Flags ativas (`show_gift_list_on_cart=true`) não implicam feature funcional para Shopify — risco de falsa percepção de suporte.
- Plugins de tema Shopify (ex.: `discount-on-cart-pro`, `free-gift-cart-upsell-pro`) não cobrem app headless nativamente.
- Regra de negócio "brinde acima de R$ 180" não implementada no código atual para Shopify.

---

# 2. Visão geral da integração

## O que a integração cobre

- **Carrinho nativo do app:** seleção e visualização do brinde pelo usuário antes do checkout.
- **Checkout:** reflexo do brinde selecionado no resumo final do pedido.
- **Regra promocional:** elegibilidade condicional ao subtotal do carrinho (ex.: brinde acima de R$ 180) com remoção automática ao perder elegibilidade.
- **Configuração:** controle de UX via Remote Config (exibição, obrigatoriedade, mensagem, página de frete).

## O que não está no escopo da feature no app

- Operação de campanhas ou regras promocionais no backend do Shopify.
- Gestão de SKUs/variantes de brinde no painel Shopify.
- Customização da renderização do checkout webview Shopify.

## Objetivo dentro do SaaS

- Aumentar conversão e ticket médio via incentivo promocional de brinde.
- Reforçar diferencial comercial para clientes com programas de brinde (ex.: Vizzela).
- Oferecer experiência nativa no app (seleção dentro do carrinho) em vez de depender de plugin de tema voltado ao storefront web.

---

# 3. Arquitetura e funcionamento

## Componentes principais

| Componente | Arquivo | Responsabilidade |
|---|---|---|
| UI de brinde no carrinho | `cart_gift_list.dart (line 27, 95)` | Renderização da lista de brindes selecionáveis |
| Inserção condicional no carrinho | `cart_list_view_widget.dart (line 133, 355)` | Exibe `CartGiftList` com base na flag `show_gift_list_on_cart` |
| Orquestração no manager | `order_form_manager.dart (line 391, 1075)` | Lógica de addGift/removeGift/updateGifts |
| Controller VTEX (referência) | `cart_controller.dart (line 1969, 2295)` | Implementação completa de brinde para VTEX |
| Checkout VTEX (referência) | `checkout.dart (line 1701, 1747)` | Refluxo de brinde no checkout para VTEX |
| Controller Shopify | `shopify_cart_controller.dart (line 220, 286, 325)` | UnimplementedError em todos os métodos de brinde |
| Checkout Service Shopify | `shopify_checkout_service.dart (line 176, 468, 588)` | UnimplementedError em todos os métodos de brinde |
| OrderFormAdapter Shopify | `shopify_checkout_service.dart (line 697)` | Popula items e checkoutUrl; **não popula selectableGifts** |
| Checkout webview Shopify | `shopify_webview_checkout_page.dart (line 47)` | Renderização do checkout Shopify em webview |
| Inicializador de controller | `cart_controller_initializer.dart (line 58, 61)` | Seleção de controller por engine |
| Utilitários de carrinho | `cart_utils.dart (line 402)` | Lógica auxiliar de carrinho e abertura de webview |
| Roteamento | `app_routes.dart (line 541)` | Rota para webview de checkout Shopify |
| Remote Config | `remote_config_values_initializer.dart (line 799, 827, 1479)` | Carga de flags de brinde |

## Fluxo de dados — VTEX (referência implementada)

```
Usuário abre carrinho
  → CartGiftList exibida (show_gift_list_on_cart=true)
  → selectableGifts populados no OrderFormAdapter
  → Usuário seleciona brinde
  → OrderFormManager.addGift → CartController.giftSelected
  → CheckoutRepository atualiza orderForm com brinde
  → Checkout reflete brinde no resumo do pedido
  → Ao cair abaixo do limiar: remoção automática via OrderFormManager.removeGift
```

## Fluxo de dados — Shopify (estado atual)

```
Usuário abre carrinho
  → show_gift_list_on_cart=true (flag ativa)
  → CartGiftList tenta exibir (selectableGifts NÃO populados no adapter Shopify)
  → Feature inoperante no carrinho nativo
  → Usuário prossegue para checkout
  → App abre shopify_webview_checkout_page (webview)
  → Se Shopify backend já inseriu item grátis via promoção/plugin de tema:
      → Item aparece no resumo do checkout webview (controlado pelo Shopify, não pelo app)
  → Brinde não foi selecionado/gerenciado pelo módulo nativo do app
```

## Fluxo esperado — Shopify (após implementação)

```
Usuário abre carrinho
  → CartGiftList exibida (show_gift_list_on_cart=true)
  → selectableGifts populados via ShopifyCheckoutService ou equivalente Shopify-first
  → Usuário seleciona brinde
  → ShopifyCartController.giftSelected → ShopifyCheckoutService.addGift
  → Carrinho Shopify atualizado via Storefront API
  → App sincroniza carrinho nativo com carrinho Shopify antes de abrir webview
  → Checkout webview reflete brinde selecionado no resumo do pedido
  → Ao cair abaixo de R$ 180: remoção automática via ShopifyCartController.updateGifts
```

## Dependências externas e internas

**Externas:**
- Shopify Storefront API / Checkout API: para manipulação do carrinho e leitura de promoções.
- Backend Shopify ou middleware próprio: fonte única de regra de elegibilidade de brinde.
- Shopify Functions (opcional): validação/aplicação de regra no checkout Shopify de forma confiável.
- Plugins de tema Shopify (limitação): atuam apenas no storefront web; não cobrem app headless.

**Internas:**
- `api_cloud_commerce`: interface de checkout service por engine.
- `OrderFormManager`: orquestração de addGift/removeGift/updateGifts.
- Remote Config: flags de UX de brinde.
- `cart_controller_initializer`: seleção de controller por engine.

---

# 4. Comportamento por engine

## VTEX

- Fluxo de brinde **totalmente implementado** de ponta a ponta.
- Controller (`cart_controller.dart`) e repository de checkout (`checkout.dart`) com todas as operações: seleção, adição, remoção, atualização e reflexo no checkout.
- `selectableGifts` populados corretamente no `OrderFormAdapter`.
- Referência arquitetural para implementação no Shopify.

## Shopify

- Brinde selecionável no carrinho nativo: **Não implementado**.
  - `ShopifyCartController`: `giftSelected`, `updateGifts`, `getGiftSelectedQuantity`, `setGiftWrappingOnCartItem` retornam `UnimplementedError`.
  - `ShopifyCheckoutService`: `addGift`, `removeGift`, `updateGifts`, `setGiftWrappingOnCartItem` retornam `UnimplementedError`.
  - `OrderFormAdapter` Shopify não popula `selectableGifts`.
- Exibição de item grátis no checkout webview: **Parcialmente implementada (indireta)**.
  - Se o backend Shopify já tiver inserido o item grátis via promoção ou plugin de tema, ele aparece no resumo do checkout webview.
  - O app não controla essa lógica; ela é renderizada pelo próprio checkout Shopify.
- Regra "brinde acima de R$ 180": **Não implementada** no código atual para Shopify.

## Outros engines

Não há menção a outros engines (ex.: Salesforce Commerce, Magento) nas documentações fornecidas para esta feature. A análise está restrita a VTEX (referência) e Shopify (escopo do lead Vizzela).

---

# 5. Features suportadas

## Features da CartGiftList (transversais ao engine)

- Exibição condicional da lista de brindes no carrinho via flag `show_gift_list_on_cart`.
- Seleção de brinde pelo usuário (dependente de implementação do provider por engine).
- Brinde obrigatório com configuração de obrigatoriedade: `gift_selection_is_mandatory`, `gift_mandatory_config`.
- Exibição de mensagem de brinde: `show_gift_message_option`.
- Exibição de opção de brinde na página de escolha de frete: `show_gift_option_on_shipping_choose_page`.
- Remoção automática ao perder elegibilidade (dependente de implementação do provider).

## Features implementadas para VTEX

- Seleção de brinde no carrinho nativo.
- Adição (`addGift`) e remoção (`removeGift`) de brinde no carrinho.
- Atualização de brindes (`updateGifts`).
- Gift wrapping por item (`setGiftWrappingOnCartItem`).
- Quantidades de brinde selecionadas (`getGiftSelectedQuantity`).
- Reflexo no checkout com remoção automática ao perder elegibilidade.

## Features implementadas para Shopify

- Exibição indireta de item grátis no checkout webview (controlada pelo backend Shopify, não pelo app).

## Features não implementadas para Shopify

- Seleção de brinde no carrinho nativo do app.
- `addGift`, `removeGift`, `updateGifts`, `setGiftWrappingOnCartItem` no `ShopifyCheckoutService`.
- `giftSelected`, `updateGifts`, `getGiftSelectedQuantity`, `setGiftWrappingOnCartItem` no `ShopifyCartController`.
- Populamento de `selectableGifts` no `OrderFormAdapter` Shopify.
- Regra de elegibilidade por subtotal (ex.: brinde acima de R$ 180) no provider Shopify.
- Sincronização entre carrinho nativo do app e carrinho Shopify antes de abrir webview.
- Remoção automática do brinde ao cair abaixo do limiar no app.

---

# 6. Status de implementação

| Feature | Engine | Status | Observação |
|---|---|---|---|
| Brinde selecionável no carrinho (CartGiftList) | VTEX | Total | Controller + repository completos |
| Brinde selecionável no carrinho (CartGiftList) | Shopify | Não implementado | UnimplementedError no controller e service; selectableGifts ausentes |
| Exibição de item grátis no checkout | VTEX | Total | Reflexo nativo via orderForm |
| Exibição de item grátis no checkout | Shopify | Parcial (indireta) | Aparece no webview se inserido pelo backend Shopify; app não controla |
| Regra "brinde acima de R$ 180" | VTEX | Total | Implementada no fluxo de checkout |
| Regra "brinde acima de R$ 180" | Shopify | Não implementado | Não há lógica no provider Shopify |
| selectableGifts no OrderFormAdapter | VTEX | Total | Populado corretamente |
| selectableGifts no OrderFormAdapter | Shopify | Não implementado | Adapter popula apenas items e checkoutUrl |
| Remoção automática ao perder elegibilidade | VTEX | Total | Via OrderFormManager.removeGift |
| Remoção automática ao perder elegibilidade | Shopify | Não implementado | Depende de implementação do provider |
| Sincronização carrinho app ↔ checkout webview | Shopify | Não implementado | Gap crítico para consistência de UX |
| Gift wrapping por item | VTEX | Total | setGiftWrappingOnCartItem implementado |
| Gift wrapping por item | Shopify | Não implementado | UnimplementedError |
| Flags de UI (show_gift_list_on_cart, etc.) | Ambos | Total | Flags carregadas via Remote Config |

---

# 7. Uso atual (clientes)

## Clientes Shopify com feature configurada

| Cliente | show_gift_list_on_cart | Observação |
|---|---|---|
| amaro | true | Configuração estática; feature inoperante no provider Shopify |
| care | true | Idem |
| colcci | true | Idem |
| haoma | true | Idem |
| patbo | true | Idem |
| plie | true | Idem |
| (3 outros Shopify) | false ou não configurado | — |

- **Total de clientes Shopify no repositório:** 9.
- **Com `show_gift_list_on_cart=true`:** 6 (66,7%).
- **Adoção real em produção:** não confirmada — flag ativa não comprova funcionamento real, pois o provider Shopify está incompleto.

## Vizzela (lead)

- Não há configuração Vizzela no diretório `customizations` desta base de código.
- Engine confirmada: Shopify.
- Requisito crítico: brinde no carrinho acima de R$ 180.
- Status: requisito **não atendível** no estado atual do código sem implementação do provider Shopify.

## Observação sobre evidência

Os números de adoção acima são baseados em configuração estática em código (`args.properties`). Não há telemetria de runtime nesta análise. Flags ativas com provider incompleto geram falsa percepção de suporte.

---

# 8. Parâmetros e configuração

## Parâmetros de UI (Remote Config)

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `show_gift_list_on_cart` | bool | Habilita exibição da CartGiftList no carrinho |
| `show_gift_message_option` | bool | Exibe opção de mensagem de brinde |
| `show_gift_option_on_shipping_choose_page` | bool | Exibe opção de brinde na página de escolha de frete |
| `gift_selection_is_mandatory` | bool | Torna seleção de brinde obrigatória |
| `gift_mandatory_config` | config | Configuração de obrigatoriedade de brinde |

**Localização de carga no Remote Config:**
- `remote_config_values_initializer.dart (line 799)`
- `remote_config_values_initializer.dart (line 827)`
- `remote_config_values_initializer.dart (line 1479)`

## Parâmetros obrigatórios para regra de negócio (a definir por cliente)

- Identificação do brinde: variant/SKU do produto brinde no Shopify.
- Critério de elegibilidade: subtotal mínimo (ex.: R$ 180), coleção, canal, etc.
- Política de remoção automática ao perder elegibilidade.

## Impacto das flags

Flags de UI controlam a exibição dos componentes no app, mas **não substituem** a implementação do provider/backend. Flag `show_gift_list_on_cart=true` com provider Shopify incompleto resulta em UI exibida sem dados (`selectableGifts` vazio ou nulo).

---

# 9. Providers e strategies

## Provider por engine

A plataforma seleciona controller e service por `EcommerceEngine` via `cart_controller_initializer.dart`. Cada engine deve implementar o contrato completo de brinde no `CartController` e no `CheckoutService`.

| Engine | Controller | CheckoutService | Status brinde |
|---|---|---|---|
| VTEX | `cart_controller.dart` | `checkout.dart` | Implementado |
| Shopify | `shopify_cart_controller.dart` | `shopify_checkout_service.dart` | UnimplementedError |

## Estratégias possíveis para Shopify

### Estratégia 1: Theme Plugin (não recomendada para app headless)

- Plugin de tema Shopify (ex.: `discount-on-cart-pro`, `free-gift-cart-upsell-pro`) aplica brinde no storefront web.
- **Limitação crítica:** não cobre app headless nativamente. Item pode aparecer no checkout webview se já inserido pelo backend, mas seleção no carrinho nativo do app não é suportada.
- Adequada apenas para storefront web; inadequada como solução principal para app mobile.

### Estratégia 2: Backend-driven com Shopify Storefront API (recomendada)

- Regra de brinde definida no backend (middleware próprio ou Shopify Functions).
- App/middleware consulta elegibilidade e insere o item brinde via Storefront API.
- `ShopifyCheckoutService` e `ShopifyCartController` implementam as operações de brinde consumindo essa API.
- Sincronização entre carrinho nativo do app e carrinho Shopify antes de abrir webview.

### Estratégia 3: Shopify Functions (complementar)

- Validação/aplicação de regra no checkout Shopify de forma confiável e sem depender de plugin de tema.
- Garante que a regra seja aplicada também no fluxo web, eliminando divergência entre canais.
- Complementa a Estratégia 2, não a substitui para o app.

---

# 10. Gaps e limitações

## Gaps técnicos

| Gap | Localização | Impacto |
|---|---|---|
| `addGift` não implementado | `shopify_checkout_service.dart (line 176)` | Impossível adicionar brinde via app |
| `removeGift` não implementado | `shopify_checkout_service.dart (line 468)` | Impossível remover brinde via app |
| `updateGifts` não implementado | `shopify_checkout_service.dart (line 588)` | Impossível atualizar brindes via app |
| `setGiftWrappingOnCartItem` não implementado | `shopify_checkout_service.dart` | Gift wrapping indisponível |
| `giftSelected` não implementado | `shopify_cart_controller.dart (line 220)` | Seleção de brinde inoperante |
| `updateGifts` não implementado | `shopify_cart_controller.dart (line 286)` | Atualização de brindes inoperante |
| `getGiftSelectedQuantity` não implementado | `shopify_cart_controller.dart (line 325)` | Quantidade de brindes indisponível |
| `selectableGifts` não populado | `shopify_checkout_service.dart (line 697)` | CartGiftList sem dados para exibir |
| Sincronização carrinho ↔ webview | Ausente | Divergência de itens entre app e checkout |
| Remoção automática ao perder elegibilidade | Ausente no Shopify | Brinde pode permanecer indevidamente no pedido |

## Limitações de arquitetura

- **Checkout em webview:** limita a customização de UX no resumo do pedido; o app não pode controlar o que o Shopify renderiza no checkout.
- **Plugins de tema:** não são solução viável para app headless; criar dependência neles gera divergência entre canal web e app.
- **Fonte única de regra ausente:** sem backend/middleware centralizado para a regra de elegibilidade, há risco de inconsistência entre canais (app, web, PDV).

## Riscos de interpretação

- `show_gift_list_on_cart=true` em 6 clientes Shopify não significa que a feature está funcionando em produção para esses clientes.
- A exibição de item grátis no checkout webview não é evidência de que o módulo nativo de brinde do app está operacional.

---

# 11. Impacto e riscos

## Impacto de negócio

- Risco de bloquear onboarding do lead Vizzela se "brinde no carrinho no app" for requisito mandatório e não for implementado antes do go-live.
- Feature com alto valor para conversão e ticket médio; ausência degrada o diferencial comercial do cliente.

## Riscos técnicos

- **Falsa percepção de suporte:** flags ativas com provider incompleto podem levar time comercial a afirmar que a feature existe para Shopify sem base técnica.
- **Inconsistência de UX:** item de brinde pode aparecer no checkout Shopify (via promoção backend) mas não no carrinho do app, gerando confusão para o usuário.
- **Divergência canal web vs app:** se a regra ficar apenas em plugin de tema, o app mobile diverge da loja web em comportamento de brinde.
- **UnimplementedError em produção:** se o fluxo de brinde for acionado no app Shopify, a exceção pode causar comportamento inesperado.

## Dependências críticas

- Implementação de `ShopifyCheckoutService` e `ShopifyCartController` com métodos de brinde.
- Definição de fonte única de regra de elegibilidade no backend (Shopify Functions ou middleware próprio).
- Populamento de `selectableGifts` no `OrderFormAdapter` Shopify.

---

# 12. Requisitos para completar

## Fase 1 — Fonte única de regra (pré-requisito)

- Definir regra de brinde no backend Shopify (preferencialmente Shopify Functions + configuração central).
- Garantir que a regra seja a mesma para app e storefront web, eliminando dependência de plugin de tema para o app.
- Mapear variant/SKU do produto brinde e metadados de elegibilidade (subtotal mínimo, coleção, canal).

## Fase 2 — Integração no provider Shopify

Implementar no `ShopifyCheckoutService`:
- `addGift`
- `removeGift`
- `updateGifts`
- `setGiftWrappingOnCartItem`

Implementar no `ShopifyCartController`:
- `giftSelected`
- `updateGifts`
- `getGiftSelectedQuantity`
- `setGiftWrappingOnCartItem`

## Fase 3 — Modelo de dados

- Popular `selectableGifts` no `OrderFormAdapter` Shopify (ou criar equivalente Shopify-first).
- Mapear promoções/brindes Shopify para o contrato `selectableGifts` do `OrderFormAdapter`.
- Definir metadados de elegibilidade acessíveis via Storefront API.

## Fase 4 — Consistência carrinho ↔ checkout

- Ao atualizar carrinho nativo, sincronizar com carrinho Shopify antes de abrir o webview de checkout.
- Implementar validação e remoção automática do brinde ao cair abaixo do limiar (R$ 180 para Vizzela).
- Decidir se seleção de brinde ocorrerá no carrinho do app, no checkout webview ou em modelo híbrido.

## Fase 5 — Fallback

- Se Shopify/plugin indisponível: ocultar seletor de brinde e exibir mensagem de indisponibilidade controlada por flag.
- Garantir que `UnimplementedError` não seja exposto ao usuário final em nenhum cenário.

## Posicionamento recomendado para o lead Vizzela

- Não posicionar como "plug and play" para Shopify; a feature exige implementação do provider antes do go-live.
- Estimar duas frentes no mesmo épico: (1) implementação do provider Shopify + regra backend; (2) validação E2E com o requisito de R$ 180 da Vizzela.

---

# 13. Critérios de avaliação para agente de IA

## Pré-requisitos

- Confirmar `ECOMMERCE_ENGINE=shopify` na configuração do cliente.
- Confirmar se o requisito é apenas exibir brinde no checkout Shopify (indireta, via backend Shopify) ou selecionar/gerenciar brinde dentro do app (implementação completa necessária).
- Verificar se regra de elegibilidade está definida fora do tema Shopify (não apenas em plugin front-end).
- Confirmar variant/SKU do produto brinde mapeado no sistema.
- Verificar flags remotas de brinde: não assumir suporte funcional apenas por estarem ativas.

## Checks funcionais

- `selectableGifts` populado no `OrderFormAdapter` após inicialização do carrinho Shopify.
- CartGiftList exibida com brindes selecionáveis (não lista vazia).
- Seleção de brinde no carrinho nativo do app persiste corretamente.
- Carrinho Shopify atualizado com item brinde antes de abrir webview de checkout.
- Checkout webview exibe item grátis no resumo do pedido.

## Validação E2E obrigatória

| Cenário | Comportamento esperado |
|---|---|
| Subtotal < R$ 180 | Sem brinde; CartGiftList não exibida ou desabilitada |
| Subtotal >= R$ 180 | Brinde disponível para seleção; item inserido no carrinho ao selecionar |
| Remover item e cair abaixo de R$ 180 | Brinde removido automaticamente do carrinho |
| Abrir checkout webview | Item brinde aparece no resumo do pedido |
| Alterar quantidades no carrinho | Elegibilidade recalculada; brinde adicionado ou removido conforme limiar |

## Sinais de problema

- `UnimplementedError` ao acionar qualquer operação de brinde no Shopify: provider incompleto.
- CartGiftList exibida com lista vazia: `selectableGifts` não populado no adapter Shopify.
- Brinde aparece no checkout webview mas não foi selecionado no carrinho do app: controle pelo backend Shopify, não pelo módulo nativo.
- Divergência de itens entre carrinho nativo do app e checkout webview: sincronização ausente.
- Flag `show_gift_list_on_cart=true` ativa mas sem seletor funcional: falsa percepção de suporte.

---

# Referências técnicas

## Arquivos de código referenciados

| Arquivo | Linha(s) | Responsabilidade |
|---|---|---|
| `cart_gift_list.dart` | 27, 95 | UI de brinde no carrinho (CartGiftList) |
| `cart_list_view_widget.dart` | 133, 355 | Inserção condicional do CartGiftList no carrinho |
| `order_form_manager.dart` | 391, 1075 | Orquestração addGift/removeGift/updateGifts |
| `shopify_cart_controller.dart` | 220, 286, 325 | UnimplementedError: giftSelected, updateGifts, getGiftSelectedQuantity |
| `shopify_checkout_service.dart` | 176, 468, 588, 697 | UnimplementedError: addGift, removeGift, updateGifts; adapter sem selectableGifts |
| `cart_controller.dart` | 1969, 2295 | Implementação de referência VTEX: seleção e atualização de brindes |
| `checkout.dart` | 1701, 1747 | Implementação de referência VTEX: brinde no checkout |
| `cart_controller_initializer.dart` | 58, 61 | Seleção de controller por engine |
| `cart_utils.dart` | 402 | Lógica auxiliar de carrinho e abertura de webview |
| `app_routes.dart` | 541 | Rota para webview de checkout Shopify |
| `shopify_webview_checkout_page.dart` | 47 | Renderização de checkout Shopify em webview |
| `remote_config_values_initializer.dart` | 799, 827, 1479 | Carga de flags de brinde via Remote Config |

## Métodos críticos com UnimplementedError (Shopify)

| Método | Localização |
|---|---|
| `addGift` | `shopify_checkout_service.dart (line 176)` |
| `removeGift` | `shopify_checkout_service.dart (line 468)` |
| `updateGifts` | `shopify_checkout_service.dart (line 588)` |
| `setGiftWrappingOnCartItem` | `shopify_checkout_service.dart` |
| `giftSelected` | `shopify_cart_controller.dart (line 220)` |
| `updateGifts` | `shopify_cart_controller.dart (line 286)` |
| `getGiftSelectedQuantity` | `shopify_cart_controller.dart (line 325)` |
| `setGiftWrappingOnCartItem` | `shopify_cart_controller.dart` |

## Parâmetros Remote Config

| Parâmetro | Arquivo | Linha |
|---|---|---|
| `show_gift_list_on_cart` | `remote_config_values_initializer.dart` | 799 |
| `show_gift_message_option` | `remote_config_values_initializer.dart` | 827 |
| `show_gift_option_on_shipping_choose_page` | `remote_config_values_initializer.dart` | — |
| `gift_selection_is_mandatory` | `remote_config_values_initializer.dart` | 1479 |
| `gift_mandatory_config` | `remote_config_values_initializer.dart` | — |

## APIs e serviços externos

| Serviço | Uso esperado |
|---|---|
| Shopify Storefront API | Manipulação de carrinho (add/remove/update itens de brinde) |
| Shopify Functions | Validação/aplicação de regra de elegibilidade no checkout |
| Shopify Checkout (webview) | Renderização do resumo do pedido com item brinde |
| Plugin de tema Shopify (não recomendado para app) | Apenas storefront web; não cobre app headless |
| `api_cloud_commerce` | Interface interna de checkout service por engine |
