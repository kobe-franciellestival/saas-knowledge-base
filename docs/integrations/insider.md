---
title: Insider Integration
feature: insider_integration
module: analytics_service, notifications, cms, deep_link, favorite, unavailable_product
engine: agnostic
analysis_type: consolidated_analysis
analysis_date: 2026-04-13
confidence: high
status: valid
source: multi_document_merge
suggested_question: Como funciona a integração com Insider no SaaS?
display_priority: 9
---

# 1. Resumo executivo

## Visão consolidada da integração

A integração com Insider no SaaS Kobe existe como provider dentro do `AnalyticsManager`, ativado quando `insider_partner_name` está configurado. A arquitetura é baseada em SDK Flutter (`flutter_insider 4.0.7+nh`) com camadas nativas iOS e Android, e cobre quatro frentes principais: SDK/push, analytics de e-commerce, geofence opcional e personalização limitada de banners via App Content Optimizer.

A Insider não é tratada como plataforma isolada: ela entra como um `AnalyticsService` genérico dentro do `AnalyticsManager`, recebendo eventos da camada de analytics do app junto com outros providers. O CMS (Contentful, VTEX CMS ou Shopify CMS) permanece como fonte de composição de páginas; a Insider só substitui conteúdo de banner quando existe `dynamicPersonalizationVariable` configurado.

Há evidência de uso real em 15 clientes com `INSIDER_PARTNER_NAME` preenchido no repositório.

## Principais capacidades

- SDK Flutter integrado ao app base com inicialização assíncrona no `AnalyticsManager`.
- SDK nativo iOS (`InsiderMobile`) e Android com suporte a rich push iOS (`InsiderMobileAdvancedNotification`).
- Eventos de e-commerce: add/remove cart, view cart, cart cleared, purchase, PDP, PLP, cadastro, wishlist, busca.
- Atributos de usuário: email, CPF (condicional), user ID, telefone, nome, aniversário, locale.
- Push Insider reconhecido por `source == "Insider"` e delegado ao SDK com callback de abertura.
- Deep links de campanhas roteáveis para destinos internos: Home, PDP, PLP, carrinho, favoritos, busca, landing page, webview.
- App Content Optimizer: troca dinâmica de imagem e destino de banners via `dynamicPersonalizationVariable`.
- Geofence: `startTrackingGeofence()` com controle por flag `insider_geolocation_is_enabled`.
- Configuração por cliente via `dart-define` e Remote Config.

## Principais limitações

- App Content Optimizer limitado a banners; não altera vitrines, ordem de componentes ou ordenação de produtos.
- Não há formulário genérico de lead collection controlado pela Insider.
- Token FCM não confirmado como enviado explicitamente à Insider via Dart; depende de SDK/nativo.
- Back in stock/avise-me existem no app fora da Insider (VTEX, AMP Shopify); nenhum evento específico é enviado à Insider nesses fluxos.
- Remoção de favoritos enviada como evento custom genérico (`logCustomEvent`), não como método dedicado Insider.
- Omnichannel de wishlist depende do provider de favoritos configurado (local não é omnichannel).
- Geofence exige permissão "always" no Android e cuidado com política de loja, privacidade, LGPD e bateria.
- Templates de push dinâmicos (renderização de experiência remota arbitrária) não implementados.
- Nível de maturidade: intermediário — bom para analytics básico, push e banners; insuficiente para personalização ampla de Home, ordenação de produtos, lead collection, wishlist omnichannel completa e back in stock via Insider.

---

# 2. Visão geral da integração

## O que a integração cobre

- Inicialização do SDK Insider no startup do app via `AnalyticsManager`.
- Coleta de eventos comportamentais de e-commerce e atributos de usuário.
- Recebimento e roteamento de push notifications originadas pela Insider.
- Deep links de campanhas para telas internas do app.
- Personalização dinâmica de banners via App Content Optimizer.
- Geolocalização e geofence para campanhas contextuais (opcional por cliente).

## Produtos/serviços Insider envolvidos

- **Insider SDK (MobilePush / Analytics):** coleta de eventos, atributos, push e geofence.
- **App Content Optimizer:** personalização dinâmica de banners via `getContentStringWithName`.
- **Geofence:** `startTrackingGeofence()` para campanhas por proximidade.
- **Rich Push iOS:** `InsiderMobileAdvancedNotification` (Notification Service/Content extension).

## Objetivo dentro do SaaS

- Habilitar campanhas de push personalizadas (carrinho abandonado, price drop, back in stock) via Insider.
- Enriquecer o perfil do usuário na Insider com dados comportamentais e cadastrais para segmentação.
- Personalizar banners de Home e Landing Pages sem rebuild do app.
- Suportar geofence para campanhas por proximidade em clientes com permissão habilitada.

---

# 3. Arquitetura e funcionamento

## Componentes principais

| Componente | Arquivo | Responsabilidade |
|---|---|---|
| Provider Insider no AnalyticsManager | `app_initializer.dart (line 1098)` | Criação e registro do `InsiderAnalyticsService` |
| Inicialização assíncrona | `analytics_manager.dart (line 56)` | Init assíncrono do provider Insider |
| InsiderAnalyticsService | `insider_analytics.dart (line 36)` | Inicialização SDK, eventos, atributos, geofence |
| Modelo de configuração | `insider_config_model.dart (line 10)` | Habilitação por `partnerName.isNotEmpty` |
| Chaves de configuração | `analytics_service_config_keys.dart (line 69)` | `partnerName`, `group`, `customIdentifiers`, `sendUserId`, `geolocation`, UTMs, base URL |
| Push Insider | `notification_service.dart (line 846)` | Reconhecimento por `source == "Insider"`, delegação ao SDK |
| Callback de abertura push | `insider_analytics.dart (line 36)` | `NOTIFICATION_OPEN` → AppRoutes / deep link interno |
| App Content Optimizer | `enriching_content_delivery.dart (line 94, 34)` | `_applyPersonalizationToComponents()` em Home e LP |
| Banner personalizável | `banner_card_adapter.dart (line 48)` | `dynamicPersonalizationVariable` → `getContentStringWithName` |
| CMS interface | `app_cms_interface.dart (line 14)` | Entrega de conteúdo enriquecido com personalização Insider |
| Deep link router | `deep_link_router.dart (line 184)` | Roteamento de destinos de push/deeplink |
| App routes | `app_routes.dart (line 1320)` | Destinos internos suportados |
| SDK Flutter | `pubspec.yaml (line 141)` | `flutter_insider 4.0.7+nh` |
| SDK iOS | `ios/Podfile (line 44)` | `InsiderMobile` |
| Rich push iOS | `integrations/insider/ios/Podfile (line 6)` | `InsiderMobileAdvancedNotification` |
| Proguard Android | `integrations/insider/android/app/proguard-rules.pro (line 3)` | Regras de ofuscação Android |

## Fluxo de dados — Analytics

```
Usuário executa ação no app
  → AnalyticsManager distribuiu evento para providers ativos
  → InsiderAnalyticsService recebe evento
  → FlutterInsider SDK encaminha para Insider
  → Insider processa e enriquece perfil do usuário
```

## Fluxo de dados — Push e deep link

```
Push FCM com source=Insider recebido
  → NotificationService reconhece source=="Insider"
  → Delega para FlutterInsider SDK
  → SDK dispara callback NOTIFICATION_OPEN
  → AppRoutes / DeepLinkRouter roteiam para tela correta
  → Destino: Home, PDP, PLP, carrinho, favoritos, LP, busca, webview
```

## Fluxo de dados — App Content Optimizer

```
CMS Home/LP carregada via fetchHomes() / fetchLandingPageComponents()
  → EnrichingContentDelivery aplica _applyPersonalizationToComponents()
  → Banners com dynamicPersonalizationVariable:
      → getContentStringWithName(variavel, ContentOptimizerDataType.CONTENT)
      → Insider retorna variante personalizada (imagem/destino)
  → Banner renderizado com conteúdo personalizado
  → logViewPromotion / logSelectPromotion disparados como eventos de impressão/clique
```

## Dependências externas e internas

**Externas:**
- Insider SDK (`flutter_insider 4.0.7+nh`).
- InsiderMobile iOS (`InsiderMobileAdvancedNotification` para rich push).
- Firebase Cloud Messaging (FCM) — identificação e entrega de push Android.
- Apple Push Notification service (APNs) — push iOS.
- Insider platform (endpoints de eventos, campanhas, Content Optimizer).

**Internas:**
- `AnalyticsManager`: roteamento de eventos para providers.
- `AuthUserStorage`: fornece email, CPF, user ID para atributos Insider.
- CMS (Contentful, VTEX CMS, Shopify CMS): fonte de composição de páginas; Insider complementa via Content Optimizer.
- `NotificationService`: recebimento e delegação de push.
- `DeepLinkRouter` / `AppRoutes`: roteamento de destinos de campanhas.
- Remote Config: flags de habilitação e configuração por cliente.

---

# 4. Comportamento por engine

## VTEX

- Analytics e push funcionam independentemente do engine.
- Back in stock: fluxo `AviseMe.aspx` próprio do VTEX; sem envio de evento à Insider.
- Wishlist: provider VTEX com sincronização backend; `add_to_wishlist` enviado à Insider.

## Shopify

- Analytics e push funcionam independentemente do engine.
- Back in stock: plugin `amp_back_in_stock`; sem envio de evento à Insider.
- Wishlist: provider local ou customizado; omnichannel não garantido.

## Magento

- Analytics e push funcionam independentemente do engine.
- Wishlist: provider Magento com sincronização backend; `add_to_wishlist` enviado à Insider.
- Back in stock: `NotifyMeService` próprio; sem envio à Insider.

## Wake Commerce

- Analytics e push funcionam independentemente do engine.
- Wishlist: provider `wakeCommerce` com sincronização backend.
- Back in stock: `NotifyMeService`; sem envio à Insider.

## Diferenças relevantes por engine

A integração Insider é agnóstica ao engine de commerce para os domínios de analytics, push e Content Optimizer. A diferença relevante por engine está nos provedores de wishlist (local vs backend) e nos fluxos de back in stock — nenhum deles envia eventos à Insider atualmente.

---

# 5. Features suportadas

## SDK e inicialização

- `FlutterInsider.Instance.init(partner, group, callback)` no startup via `AnalyticsManager`.
- Inicialização condicional: `partnerName.isNotEmpty` em `insider_config_model.dart`.
- Callback de init trata push open e in-app/temp store action.

## Eventos de e-commerce

| Evento Insider | Quando disparado | Dados enviados |
|---|---|---|
| `itemAddedToCart` | Add to cart | Produto Insider: id, nome, categorias, imagem, preço, moeda, quantidade |
| `itemRemovedFromCart` | Remove from cart | `item.itemId` |
| `visitCartPage` | View cart | Lista de produtos |
| `cartCleared` | Carrinho limpo | Sem lista detalhada |
| `itemPurchased` | Compra concluída | `orderId` + produto Insider por item |
| `visitProductDetailPage` | PDP | Produto Insider + referrer |
| `visitListingPage` | Categoria (PLP) | Nome da categoria |
| `signUpConfirmation` | Cadastro | Sem payload extra |
| `add_to_wishlist` | Favoritar produto | `product_id`, `product_name`, `category`, `items_count`, `currency` |
| `remove_from_wishlist` | Remover favorito | Via `logCustomEvent` genérico (não método dedicado Insider) |
| `last_search_query` | Busca | `searchTerm` |
| `logViewPromotion` / `logSelectPromotion` | Impressão/clique em banner personalizado | `insiderVariable`, promotion metadata |

## Atributos de usuário

| Atributo | Quando enviado | Observação |
|---|---|---|
| Email | Login/register | Identificador principal |
| CPF | Login/register | Apenas se `customIdentifiers` contém `cpf` |
| User ID | Login | Pode ser MD5 do CPF |
| Telefone | Login/register | Normalizado com `+55` |
| Nome / sobrenome | Login/register/begin checkout | — |
| Idade / aniversário | Login/register | Birthday direto no register |
| Locale | Login/register | Fixo `pt_BR` |
| `birthday_month` | Se `Environment.enablePrimeUserConfig` | Custom attribute |
| Location opt-in | Permissão nativa de geofence | `setLocationOptin(true)` |

## Push notifications

- Reconhecimento de push Insider por `source == "Insider"`.
- Delegação para FlutterInsider SDK.
- Callback `NOTIFICATION_OPEN` integrado ao `NotificationService`.
- Suporte a rich push iOS via `InsiderMobileAdvancedNotification`.
- `registerWithQuietPermission(true)` e handlers SDK (ver limitação de token FCM na seção 10).

## Deep links de campanhas

| Destino | Suportado via push | Observação |
|---|---|---|
| Home | Sim | Via navbar/home |
| Carrinho | Sim | Cart reminder viável |
| Favoritos | Sim | Wishlist |
| PDP produto | Sim | `product` / `product-sku` |
| PLP por categoria | Sim | — |
| PLP por coleção | Sim | — |
| Busca | Sim | — |
| Landing page | Sim | Por ID de LP |
| Webview | Sim | URL externa ou in-app |
| Template remoto arbitrário | Não confirmado | Exige nova camada de renderização |

## App Content Optimizer

- Troca dinâmica de imagem e destino de banners via `dynamicPersonalizationVariable`.
- `getContentStringWithName(variavel, ContentOptimizerDataType.CONTENT)` em `CMSBannerCarouselAdapter` e `CBannerGridAdapter`.
- Aplicado em Home (`fetchHomes()`) e Landing Pages (`fetchLandingPageComponents()`).
- Eventos de impressão (`logViewPromotion`) e clique (`logSelectPromotion`) disparados automaticamente.
- Limitação: apenas banners; não altera vitrines, ordenação de componentes ou produtos.

## Geofence

- `startTrackingGeofence()` após serviço de localização, permissão e precisão adequadas.
- Controlado por flag `insider_geolocation_is_enabled` (default `false`).
- Clientes com geofence habilitado: Americanas, Drogasmil, Farmalife, Farmácias São João, Tamoio.

## Wishlist (parcial)

- Evento `add_to_wishlist` enviado via `analyticsManager.logAddToWishlist`.
- Evento `remove_from_wishlist` enviado como `logCustomEvent` genérico.
- Omnichannel dependente do provider de favoritos (local, VTEX, Wake, Magento, Zaffari).
- Sem contrato Insider completo para price drop baseado em wishlist.

## Back in stock / Avise-me (parcial, fora da Insider)

- Três caminhos no app: `NotifyMeService` (genérico), fluxo VTEX `AviseMe.aspx`, plugin `amp_back_in_stock` (Shopify/AMP).
- Nenhum deles envia evento específico à Insider.
- Campanhas de back in stock via Insider exigem novo evento/atributo ou integração backend/middleware.

---

# 6. Status de implementação

| Frente | Funcionalidade | Status | O que existe | O que falta |
|---|---|---|---|---|
| App Content Optimizer | Home dinâmica | Parcial | Troca de imagem/destino de banners com `dynamicPersonalizationVariable` | Vitrines, ordem de componentes, ordenação de produtos |
| App Content Optimizer | LP dinâmica | Parcial | LP CMS aceita componentes e banners personalizados via enriquecedor | Formulário/lead collection Insider genérico |
| Push | Carrinho abandonado | Parcial | Eventos add/remove/view cart/purchase + push Insider reconhecido e roteado | Confirmação de token FCM enviado à Insider via Dart |
| Push | Rich push iOS | Parcial | `InsiderMobileAdvancedNotification` configurado | Validação por build real; depende de extensões iOS adequadas |
| Push | Geofence | Parcial | `startTrackingGeofence()`, permissões, opt-in, flag de controle | Permissões nativas adequadas por cliente; risco privacidade/bateria |
| Push | Templates dinâmicos | Parcial | Payload custom `destination`, `destinationId`, `link`, destinos existentes | Renderização de experiência remota arbitrária |
| Analytics | Eventos de e-commerce | Total | Todos os eventos principais de jornada implementados e plugados no AnalyticsManager | — |
| Analytics | Atributos de usuário | Total | Email, CPF (condicional), user ID, telefone, nome, aniversário, locale | — |
| Omnichannel | Wishlist | Parcial | `add_to_wishlist` implementado; `remove_from_wishlist` como evento genérico | Contrato completo de remoção; price drop via Insider; omnichannel depende de provider |
| Omnichannel | Back in stock | Parcial | Fluxos no app fora da Insider (NotifyMeService, VTEX, AMP) | Evento específico para Insider; integração backend/middleware |
| Lead collection | Formulário Insider | Não implementado | — | Componente novo, contrato de payload/formulário, integração de envio |
| Geolocation | Opt-in e rastreio | Parcial | SDK configurado, flag de controle, `setLocationOptin(true)` | Validação por cliente (permissões, política App Store/Google Play, LGPD) |

---

# 7. Uso atual (clientes)

## Clientes com INSIDER_PARTNER_NAME preenchido (15)

Evidência de `INSIDER_PARTNER_NAME` configurado em `customizations/*/args.properties`:
- americanas
- fast_shop
- farmacias_sao_joao
- patbo
- vivara
- vix
- (e outros — total 15 clientes no repositório)

## Clientes com geofence habilitado

`INSIDER_GEOLOCATION_ENABLED=true`:
- Americanas
- Drogasmil
- Farmalife
- Farmácias São João
- Tamoio

## Padrões de uso identificados

- Uso predominante de analytics e push.
- App Content Optimizer utilizado em clientes com banners parametrizados no CMS.
- Geofence habilitado em poucos clientes; configuração não implica permissões nativas corretas.
- Nenhum cliente com back in stock ou lead collection via Insider identificado.

---

# 8. Parâmetros e configuração

## Configurações principais

| Parâmetro | Origem | Descrição |
|---|---|---|
| `insider_partner_name` (`INSIDER_PARTNER_NAME`) | dart-define / Remote Config | Nome do parceiro Insider; habilita o provider quando não vazio |
| `group` | dart-define / Remote Config | Grupo Insider para segmentação |
| `customIdentifiers` | Remote Config | Lista de identificadores customizados (ex.: `cpf`) |
| `sendUserId` | Remote Config | Envia user ID para Insider (pode ser MD5 do CPF) |
| `insider_geolocation_is_enabled` | Remote Config | Habilita geofence (default `false`) |
| UTMs | Remote Config | Parâmetros de campanha para atribuição |
| Base URL | Remote Config | URL base de integração Insider |

**Localização de chaves:** `analytics_service_config_keys.dart (line 69)`.

## Habilitação do provider

- `InsiderAnalyticsService` só entra no `AnalyticsManager` quando `partnerName.isNotEmpty`.
- Verificação em `insider_config_model.dart (line 10)`.
- Criação do provider em `app_initializer.dart (line 1098)`.

## Configuração do App Content Optimizer

- Campo `dynamicPersonalizationVariable` no modelo de banner CMS.
- Quando preenchido, substitui imagem/destino do banner pela variante Insider.
- Sem configuração adicional necessária no app além do campo no CMS e da variável configurada na plataforma Insider.

## Configuração de geofence

- Flag `insider_geolocation_is_enabled` no Remote Config.
- Requer permissões nativas adequadas (Android: `ACCESS_BACKGROUND_LOCATION`; iOS: `Always`).
- Requer plists/manifests configurados por cliente via `merx_cli` ou customização.

---

# 9. Providers e strategies

## Estratégia de integração

A Insider não é um provider isolado de marketing: ela entra como um `AnalyticsService` genérico dentro do `AnalyticsManager`. Todos os eventos de analytics do app são distribuídos para providers ativos; a Insider recebe os mesmos eventos que outros providers (Firebase, Segment, etc.), sem camada exclusiva.

## Seleção de provider

- Ativação condicional por `partnerName.isNotEmpty` — sem necessidade de flag de engine separada.
- A integração é agnóstica ao engine de commerce.

## Provider de favoritos (impacta omnichannel Insider)

| Provider | Omnichannel | `add_to_wishlist` enviado à Insider |
|---|---|---|
| `local` | Não | Sim |
| `vtex` | Sim (backend) | Sim |
| `wakeCommerce` | Sim (backend) | Sim |
| `magento` | Sim (backend) | Sim |
| `zaffari` | Sim (backend) | Sim |

## Providers de back in stock (fora da Insider)

| Provider | Integração Insider |
|---|---|
| `NotifyMeService` (genérico) | Sem evento Insider |
| VTEX `AviseMe.aspx` | Sem evento Insider |
| `amp_back_in_stock` (Shopify/AMP) | Sem evento Insider |

## Pontos de extensão

- **Novo evento:** adicionar em `InsiderAnalyticsService` e chamar via `AnalyticsManager`.
- **Novo atributo de usuário:** adicionar em `insider_analytics.dart` nos fluxos de login/register.
- **Novo destino de push:** ampliar `DeepLinkRouter` / `AppRoutes` e mapeamento de payload.
- **Lead collection:** implementar componente novo com contrato de payload e integração de envio à Insider.
- **Back in stock via Insider:** adicionar evento específico nos fluxos de `NotifyMeService` e providers de avise-me.

---

# 10. Gaps e limitações

## Gaps técnicos

| Gap | Localização | Impacto |
|---|---|---|
| Token FCM não confirmado enviado à Insider via Dart | `insider_analytics.dart` | Risco de push Insider não funcionar sem validação em build real |
| `remove_from_wishlist` como `logCustomEvent` genérico | `favorite_controller.dart (line 250)` | Contrato Insider incompleto para price drop e estado de wishlist |
| Back in stock sem evento Insider | `unavailable_product_cubit.dart`, `pdp_page.dart` | Campanhas de back in stock via Insider indisponíveis |
| App Content Optimizer limitado a banners | `enriching_content_delivery.dart` | Sem personalização de vitrines, ordenação de componentes ou produtos |
| Lead collection não implementado | — | Formulários Insider e coleta de leads indisponíveis |
| Templates de push dinâmicos não renderizados | `notification_service.dart` | Experiências remotas arbitrárias via payload não suportadas |
| Shopify CMS sem Content Optimizer | — | Clientes Shopify sem personalização de banners via Insider (dependem de CMS) |

## Limitações de arquitetura

- O CMS permanece como fonte de verdade de composição de páginas; a Insider complementa via Content Optimizer apenas nos banners com `dynamicPersonalizationVariable`. Não há integração Insider-CMS para ordenar produtos, manipular vitrines ou reordenar Home inteira.
- Omnichannel de wishlist depende do provider de favoritos configurado; provider `local` não é omnichannel.
- Back in stock fora da Insider — os três caminhos existentes (`NotifyMeService`, VTEX `AviseMe.aspx`, `amp_back_in_stock`) não notificam a Insider, impossibilitando campanhas de reestoque via plataforma.

## Riscos identificados

- **Token FCM/Insider:** não confirmado em build real; push pode não funcionar para clientes novos sem validação.
- **Geofence:** exige permissão "always" (Android) com riscos de política de loja, LGPD, bateria e comportamento em background. Não é apenas configuração — requer validação por cliente.
- **LGPD:** há envio de identificadores sensíveis (CPF, telefone) à Insider; uso deve estar coberto por consentimento e base legal documentada.
- **Dados de produto limitados:** payload de produto enviado à Insider cobre e-commerce básico, mas é insuficiente para price drop/back in stock sem catálogo e estoque integrados.
- **Omnichannel não assumir:** quando wishlist é local, a Insider não tem visibilidade cross-device/cross-channel.
- **Geofence ativo sem permissões nativas:** flag `insider_geolocation_is_enabled=true` sem permissões adequadas não funciona e pode gerar erros silenciosos.

---

# 11. Impacto e riscos

## Impacto de negócio

- Push de carrinho abandonado é a funcionalidade de maior impacto de receita; a base está implementada, mas depende de validação de token.
- App Content Optimizer para banners habilita personalização sem rebuild; valor imediato para clientes com CMS parametrizado.
- Geofence e lead collection têm alto valor potencial mas exigem desenvolvimento ou validação antes de serem posicionados como capacidades prontas.
- Back in stock e price drop via Insider não podem ser entregues no estado atual sem desenvolvimento adicional.

## Riscos técnicos

- **Token FCM não validado:** push Insider pode falhar silenciosamente em produção sem validação em build real por cliente.
- **Geofence sem permissões:** configuração ativa sem permissões nativas resulta em falha silenciosa ou erros de runtime.
- **Dados insuficientes para price drop:** sem catálogo/estoque integrado na Insider, campanhas de price drop e back in stock não têm base de dados adequada.
- **`remove_from_wishlist` genérico:** estado de wishlist na Insider pode ficar inconsistente para campanhas de reengajamento.

## Avaliação de maturidade por frente

| Frente | Maturidade |
|---|---|
| Analytics de e-commerce | Alta |
| Push / deep link | Intermediária (validação de token pendente) |
| App Content Optimizer (banners) | Intermediária |
| Geofence | Baixa (configuração parcial; risco de runtime) |
| Wishlist omnichannel | Baixa |
| Back in stock / price drop via Insider | Inexistente |
| Lead collection | Inexistente |
| Templates dinâmicos de push | Inexistente |

---

# 12. Requisitos para completar

## Validações urgentes (sem desenvolvimento)

- **Token FCM/Insider:** validar em build real que o device token está sendo registrado corretamente na Insider para clientes Android e iOS.
- **Push E2E:** testar fluxo completo de campanha Insider → push → abertura → deep link para todos os destinos suportados.
- **Geofence por cliente:** para cada cliente com `INSIDER_GEOLOCATION_ENABLED=true`, validar permissões nativas, manifests/plists e comportamento em background.

## Implementações necessárias

- **`remove_from_wishlist` dedicado:** substituir `logCustomEvent` genérico por método dedicado Insider para remoção de favoritos.
- **Evento de back in stock/avise-me:** adicionar evento específico nos fluxos de `NotifyMeService`, VTEX `AviseMe.aspx` e `amp_back_in_stock` para campanhas de reestoque via Insider.
- **Lead collection:** implementar componente de formulário, contrato de payload e integração de envio à plataforma Insider.
- **Templates dinâmicos de push:** avaliar e implementar renderizador de experiência remota via payload (mapeamento para schema de componentes ou LP CMS/Insider por ID).

## Evoluções de personalização

- **App Content Optimizer para vitrines:** ampliar `_applyPersonalizationToComponents()` para suportar vitrines e ordenação de produtos além de banners.
- **Ordenação de componentes da Home:** avaliar integração Insider-CMS para reordenar componentes de Home dinamicamente.
- **Price drop via Insider:** adicionar catálogo/estoque integrado e contrato de produto mais completo no payload enviado à Insider.

## Governança e conformidade

- **LGPD:** documentar base legal para envio de CPF e telefone à Insider. Garantir que consentimento esteja coberto antes de habilitar para novos clientes.
- **Política de lojas para geofence:** validar conformidade com App Store e Google Play antes de habilitar geofence para novos clientes.

---

# 13. Critérios de avaliação para agente de IA

## Pré-requisitos

- `INSIDER_PARTNER_NAME` não vazio na configuração do cliente.
- `group` definido.
- Para geofence: `insider_geolocation_is_enabled=true` + permissões nativas validadas.
- Para Content Optimizer: banners com `dynamicPersonalizationVariable` configurado no CMS e variável correspondente criada na plataforma Insider.
- Para CPF: `customIdentifiers` contém `cpf` na configuração.

## Checks funcionais

- `InsiderAnalyticsService` inicializado no `AnalyticsManager` (verificar `partnerName.isNotEmpty`).
- Login define atributos de usuário (email, user ID, telefone, nome) na Insider.
- Eventos de e-commerce disparados: `itemAddedToCart`, `itemRemovedFromCart`, `visitCartPage`, `itemPurchased`, `visitProductDetailPage`.
- Push Insider recebido, reconhecido por `source == "Insider"` e roteado para destino correto.
- Banners com `dynamicPersonalizationVariable` exibem variante personalizada.
- `logViewPromotion` e `logSelectPromotion` disparados em impressão e clique de banners personalizados.

## Validação E2E por funcionalidade

| Funcionalidade | Cenário | Comportamento esperado |
|---|---|---|
| Carrinho abandonado | Adicionar produto + fechar app + receber push | Push Insider exibido; abertura roteia para carrinho |
| App Content Optimizer | Banner com `dynamicPersonalizationVariable` | Variante Insider exibida (não imagem padrão CMS) |
| Geofence | Entrar em área de geofence configurada | Push/trigger disparado pela Insider |
| Wishlist | Favoritar produto | Evento `add_to_wishlist` registrado na Insider com dados corretos |
| Deep link | Push com `destination=product` + `destinationId` | App abre PDP do produto correto |
| Compra | Finalizar pedido | `itemPurchased` com `orderId` e itens registrado na Insider |

## Sinais de problema

- Provider Insider não inicializado: verificar `partnerName.isNotEmpty` e `app_initializer.dart (line 1098)`.
- Eventos ausentes na Insider após ações no app: verificar se `AnalyticsManager` está distribuindo para `InsiderAnalyticsService` (verificar init assíncrono em `analytics_manager.dart (line 56)`).
- Push Insider não roteando para tela correta: verificar `destination` + `destinationId` no payload e cobertura no `DeepLinkRouter`.
- Banners não personalizados com `dynamicPersonalizationVariable` configurado: verificar se variável existe na plataforma Insider e se `getContentStringWithName` está sendo chamado.
- Geofence não disparando: verificar permissões nativas, flag `insider_geolocation_is_enabled` e se `startTrackingGeofence()` foi chamado após opt-in.
- Atributos de usuário ausentes na Insider: verificar fluxo de login/register e campos `customIdentifiers` na configuração.
- Token FCM não registrado na Insider: validar em build real; `registerWithQuietPermission(true)` e handlers SDK podem não ser suficientes sem chamada explícita.

---

# Referências técnicas

## Arquivos de código referenciados

| Arquivo | Linha(s) | Responsabilidade |
|---|---|---|
| `app/lib/modules/analytics_service/services/insider_analytics.dart` | 36, 85, 102, 105–110, 145, 206, 213, 250, 282, 310, 317, 321, 327, 342, 417, 436, 450, 463 | Inicialização SDK, eventos, atributos, geofence |
| `app/lib/modules/analytics_service/config/models/insider_config_model.dart` | 10 | Habilitação por `partnerName.isNotEmpty` |
| `app/lib/modules/analytics_service/config/analytics_service_config_keys.dart` | 69 | Chaves de configuração Insider |
| `app/lib/initializers/app_initializer.dart` | 1098 | Criação e registro do `InsiderAnalyticsService` |
| `app/lib/modules/analytics_service/analytics_manager.dart` | 56 | Init assíncrono do provider |
| `app/lib/modules/notifications/notification_service.dart` | 846 | Reconhecimento e delegação de push Insider |
| `app/lib/cms/enriching_content_delivery.dart` | 34, 94, 119 | App Content Optimizer em Home e LP |
| `app/lib/adapters/cms_adapter/model/banner_card_adapter.dart` | 48 | `dynamicPersonalizationVariable` → `getContentStringWithName` |
| `app/lib/cms/app_cms_interface.dart` | 14 | CMS enriquecido com personalização Insider |
| `app/lib/routes/app_routes.dart` | 1320, 1379, 1400, 1454, 1463, 1466, 1501 | Destinos internos de deep link e push |
| `app/lib/modules/deep_link/presentation/deep_link_router.dart` | 184, 316, 331, 353 | Roteamento de deep links |
| `app/lib/modules/deep_link/data/native_deep_link/native_deep_link.dart` | 31 | Deep link nativo |
| `app/lib/modules/favorite/controller/favorite_controller.dart` | 250 | `remove_from_wishlist` como `logCustomEvent` genérico |
| `app/lib/initializers/favorite_initializer.dart` | 17 | Providers de favoritos (local, vtex, wakeCommerce, zaffari, magento) |
| `app/lib/modules/unavailable_product/presentation/unavailable_product/cubit/unavailable_product_cubit.dart` | 106 | Avise-me / back in stock |
| `app/lib/modules/unavailable_product/data/repositories/unavailable_product_repository.dart` | 31 | Repositório de avise-me |
| `app/lib/modules/pdp/pages/pdp_page.dart` | 2926 | AMP Back In Stock (Shopify) |
| `app/lib/modules/landing_page/landing_page.dart` | 11 | Landing Pages com Content Optimizer |
| `app/lib/modules/landing_page/controller/lp_controller.dart` | 68 | Controller de LP |
| `app/lib/initializers/cms_interface_initializer.dart` | 20 | Inicialização do CMS source |
| `app/pubspec.yaml` | 141 | `flutter_insider 4.0.7+nh` |
| `app/ios/Podfile` | 44 | `InsiderMobile` (SDK iOS) |
| `integrations/insider/ios/Podfile` | 6 | `InsiderMobileAdvancedNotification` (rich push iOS) |
| `integrations/insider/android/app/proguard-rules.pro` | 3 | Regras de ofuscação Android |
| `customizations/*/args.properties` | — | `INSIDER_PARTNER_NAME` e `INSIDER_GEOLOCATION_ENABLED` por cliente |

## SDK e dependências

| Dependência | Versão / Identificador | Plataforma |
|---|---|---|
| `flutter_insider` | `4.0.7+nh` | Flutter |
| `InsiderMobile` | — | iOS (CocoaPods) |
| `InsiderMobileAdvancedNotification` | — | iOS (Notification Service/Content extension) |
| Firebase Cloud Messaging (FCM) | — | Android |
| Apple Push Notification service (APNs) | — | iOS |

## Eventos Insider mapeados

| Evento | Tipo | Implementado |
|---|---|---|
| `itemAddedToCart` | SDK nativo | Sim |
| `itemRemovedFromCart` | SDK nativo | Sim |
| `visitCartPage` | SDK nativo | Sim |
| `cartCleared` | SDK nativo | Sim |
| `itemPurchased` | SDK nativo | Sim |
| `visitProductDetailPage` | SDK nativo | Sim |
| `visitListingPage` | SDK nativo | Sim |
| `signUpConfirmation` | SDK nativo | Sim |
| `add_to_wishlist` | SDK nativo | Sim |
| `remove_from_wishlist` | `logCustomEvent` genérico | Parcial |
| `last_search_query` | `logCustomEvent` | Sim |
| `logViewPromotion` / `logSelectPromotion` | SDK nativo | Sim (Content Optimizer) |
| Back in stock / avise-me | — | Não implementado |
