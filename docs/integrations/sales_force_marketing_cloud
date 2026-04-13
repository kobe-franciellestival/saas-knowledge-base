---
title: Salesforce Marketing Cloud Integration
feature: salesforce_marketing_cloud_integration
module: integration
engine: agnostic
analysis_type: consolidated_analysis
analysis_date: 2026-04-13
confidence: medium
status: valid
source: multi_document_merge
suggested_question: Como funciona a integração com Salesforce Marketing Cloud no SaaS?
display_priority: 10
---

# 1. Resumo executivo

## Visão consolidada da integração

A integração com Salesforce Marketing Cloud (SFMC) no SaaS Kobe é uma camada estrutural do core do app mobile, não um plugin isolado. Ela cobre cinco domínios funcionais: identidade e consentimento, tracking e eventos de jornada, push e deeplink, vitrines e banners com SFMC como fonte, e geolocalização para campanhas contextuais.

A arquitetura está dividida em duas camadas complementares: uma camada Flutter compartilhada no SaaS (inicialização da SDK, bridge, operações, leitura de estado) e uma camada nativa por plataforma (Android/iOS), gerada por configuração de cliente via `merx_cli`. Há evidência de uso real em pelo menos 8 clientes com SFMC habilitado no repositório atual.

A decisão arquitetural de implementar no core — e não via plugin — foi tomada porque a integração envolve capacidades transversais já existentes (tracking, identidade, push, navegação, renderização de conteúdo). Essa abordagem garante governança centralizada, reduz risco de inconsistência entre jornadas e evita fragmentação funcional.

O escopo da integração cobre exclusivamente as responsabilidades do app: produção e governança de dados (tracking), identidade e consentimento, comunicação (push/deeplink), renderização de conteúdo personalizado e suporte a contexto (geolocalização). Não está no escopo: operação do CRM, definição de jornadas, backend ou site.

## Principais capacidades

- SDK SFMC Flutter integrada ao app base (sfmc 8.2.0).
- Suporte nativo Android e iOS com handlers e method channel.
- Inicialização condicional por flag `hasSalesforceMarketingCloud`.
- Registro de device token (FCM no Android, APNs no iOS).
- Associação device ↔ usuário via contactKey.
- Atributos de usuário (cobertura parcial, com caso mais completo no fluxo BRF).
- Recebimento e abertura de push com roteamento para telas internas.
- Roteamento genérico de campanhas reaproveitando pipeline de notificações existente.
- Analytics básico de parâmetros de campanha de push no app.
- Governança por flags/Remote Config por cliente.
- Configuração e geração de camada nativa por cliente via `merx_cli`.

## Principais limitações

- Eventos comportamentais do app para SFMC não implementados (login, PDP, add-to-cart, compra, abandono de carrinho, pós-compra).
- Personalização in-app via Marketing Cloud inexistente (vitrines e banners dinâmicos não consumem payload do SFMC).
- Geolocalização/geofence habilitada na SDK mas sem fluxo funcional completo no app.
- Deep link via push exige `destination` + `destinationId` no payload, criando limitação para campanhas sem esse formato.
- Cobertura de atributos além de contactKey é limitada fora do fluxo BRF.
- Documentação interna da integração (`README.md`) está vazia.
- Nível de prontidão: intermediário para push/identificação; baixo para eventos avançados, jornadas comportamentais e personalização in-app.

---

# 2. Visão geral da integração

## O que a integração cobre

- Identidade do usuário no app (anônimo vs logado, vínculo device ↔ usuário via contactKey).
- Governança de consentimento (opt-in de push, tracking, localização).
- Tracking de interações com push (parâmetros de campanha, campaign_details).
- Recebimento, abertura e roteamento de push notifications.
- Deep links de campanhas para telas internas do app.
- Suporte à habilitação de geolocalização/geofence via SDK (sem fluxo funcional completo).
- Configuração de SFMC como fonte de vitrines e banners via CMS (épico planejado, não implementado).

## Produtos/serviços Salesforce envolvidos

- **Salesforce Marketing Cloud (SFMC) / Marketing Cloud Engagement:** plataforma central de campanhas, push e jornadas.
- **MobilePush (SFMC):** canal de push notifications para app mobile.
- **Journey Builder:** motor de jornadas (operado externamente, fora do escopo do app).
- **Data Cloud:** destino de eventos (a confirmar por cliente).
- **Marketing Cloud Personalization (Evergage):** módulo de personalização — tratado em documento separado (ver `salesforce_personalization_consolidated.md`).

A distinção entre Marketing Cloud Engagement e Advanced Edition é um ponto aberto: o cliente usa Marketing Cloud Engagement ou Advanced Edition? Isso determina se apenas MobilePush é necessário ou se o Personalization SDK também é exigido.

## Objetivo dentro do SaaS

Transformar o app em um canal confiável de:
- Captura de dados comportamentais e de identidade para alimentar jornadas no CRM.
- Ativação de campanhas recebidas do SFMC (push, deeplink, conteúdo personalizado).
- Renderização de conteúdo dinâmico proveniente do SFMC (vitrines, banners — planejado).

---

# 3. Arquitetura e funcionamento

## Componentes principais

| Componente | Localização | Responsabilidade |
|---|---|---|
| SDK Flutter SFMC | `pubspec.yaml (line 149)`, `centered_dependencies.dart (line 103)` | Dependência sfmc 8.2.0 no app base |
| Módulo Flutter SFMC | `salesforce_marketing_cloud.dart` | Inicialização, bridge, operações, leitura de estado |
| Inicialização no startup | `app_initializer.dart (line 325)` | Inicializa condicionalmente por flag |
| SDK nativa Android | `app/build.gradle.kts (line 116)`, `MarketingCloudHandler.kt` | Dependência Maven, handler nativo |
| SDK nativa iOS | `AppDelegate.swift (line 27)`, `MarketingCloudHandler.swift` | Handler nativo iOS |
| Bridge Flutter ↔ nativo | `salesforce_marketing_cloud.dart (line 106)`, `MarketingCloudMethodChannel.kt`, `MarketingCloudMethodChannel.swift` | Canal push redirect para Flutter |
| Configuração por cliente | `salesforce_marketing_cloud.dart (line 8)` | Injeta arquivos nativos e credenciais por cliente via `merx_cli` |
| Associação contactKey | `salesforce_marketing_cloud.dart (line 88)`, `auth_update_marketing_cloud.dart (line 14)` | Vínculo device ↔ usuário |
| Atributos de usuário | `salesforce_marketing_cloud.dart (line 74)`, `brf_channel_controller.dart (line 214)` | Atributos enviados ao SFMC |
| Roteador de notificações | `app_routes.dart (line 1211, 1276)` | Roteamento genérico de push para telas internas |
| Fetchers de estado/dados | `marketing_cloud_data_fetcher.dart`, `marketing_cloud_state_fetcher.dart` | Busca token, deviceId, tags, atributos e flags |
| Operações SFMC | `marketing_cloud_operations.dart` | Tags, analytics enable, operações diversas |
| Governança/flags | `credentials.dart (line 835, 1528, 1530)`, `remote_config_values_initializer.dart (line 1380)` | Flag principal por build/env; hash e salt por Remote Config |

## Fluxo de dados

```
Usuário interage com app
  → Identificação do usuário (contactKey atribuído/limpo em login/logout)
  → Captura de evento (tracking analítico interno; eventos comportamentais para SFMC: NÃO implementado)
  → Envio para CRM (somente identificação e atributos; eventos comportamentais: gap)
  → CRM processa jornada e dispara campanha
  → Push / Deeplink recebido no device
  → App abre (via method channel Flutter ↔ nativo)
  → Valida contexto (destination + destinationId no payload)
  → Roteia para tela (usando roteador genérico de notificações)
  → Renderiza conteúdo (ou fallback se destino inválido)
```

## Fluxo de push (detalhado)

**Android:**
- FCM recebe push → `NotificationMessagingService.kt` repassa para `MarketingCloudHandler`.
- Token FCM registrado via `setPushToken` em `MarketingCloudHandler.kt (line 92)`.
- Analytics nativo: `NotificationManager.redirectIntentForAnalytics(...)` em `MarketingCloudHandler.kt (line 69)`.

**iOS:**
- APNs push → `AppDelegate.swift (line 57)` e `MarketingCloudHandler.swift (line 60)` repassam para SDK.
- Token APNs registrado via `setDeviceToken` em `AppDelegate.swift (line 112)` e `MarketingCloudHandler.swift (line 54)`.

**Flutter (ambas as plataformas):**
- Clique/abertura de push → callback nativo → method channel `salesforce_marketing_cloud_push_redirect_channel`.
- Flutter recebe `redirectFromPush` e invoca roteador genérico de notificações (`app_routes.dart (line 1211)`).
- Roteador extrai `destination` + `destinationId` e navega para tela correspondente.
- Fallback para rota padrão quando destino é inválido (`app_routes.dart (line 1603)`).
- Prevenção de clique duplicado implementada (`app_routes.dart (line 1193)`).

## Inicialização do SDK Android (MarketingCloudHandler.kt)

```
setAnalyticsEnabled
setInboxEnabled
setGeofencingEnabled
setPiAnalyticsEnabled
setProximityEnabled
setDelayRegistrationUntilContactKeyIsSet
```

## Inicialização do SDK iOS (MarketingCloudHandler.swift)

```
setLocationEnabled(true)
startWatchingLocation()
```

## Dependências externas e internas

**Externas:**
- Salesforce Marketing Cloud (endpoints de push, jornadas, campanhas).
- Firebase Cloud Messaging (FCM) — Android.
- Apple Push Notification service (APNs) — iOS.

**Internas:**
- `AnalyticsManager` do app.
- Camada de autenticação (login/logout para atualização de contactKey).
- Remote Config (governança de flags, hash e salt).
- `merx_cli` (geração de configuração nativa por cliente).

## Papel do middleware

Não há BFF ou middleware específico para SFMC na arquitetura atual. A comunicação é direta entre app (SDK/bridge) e SFMC. O `merx_cli` atua como ferramenta de geração de configuração nativa por cliente, não como middleware de runtime.

---

# 4. Comportamento por engine

## Salesforce Commerce (SFCC)

A integração SFMC não está acoplada ao engine SFCC. A camada de Marketing Cloud é transversal ao engine de e-commerce do cliente.

## Outras engines (Magento, etc.)

A integração SFMC funciona independentemente do engine de commerce. Há evidência de clientes com engine Magento (ex: Granado) utilizando SFMC. A distinção relevante não é por engine de commerce, mas por configuração de cliente via `merx_cli` e por flags habilitadas.

## Diferenças relevantes por cliente

- **BRF (`central_brf`):** fluxo mais completo de atualização de contactKey e atributos (email, contactId, contactkey_sf, cpf, mobilePhone) em `brf_channel_controller.dart (line 214)`.
- **Demais clientes:** cobertura limitada a contactKey; atributos adicionais têm pouca evidência fora do fluxo BRF.
- **Granado:** usa SFMC + Personalization SDK (tratado no documento de Personalization).

## Clientes com SFMC habilitado (HAS_SALESFORCE_MARKETING_CLOUD=true)

8 clientes com os 4 campos obrigatórios preenchidos (`SFMC_PUSH_ACCESS_TOKEN`, `SFMC_PUSH_APP_ID`, `SFMC_PUSH_SENDER_ID`, `SFMC_PUSH_SERVER_URL`):
- atacadao
- brastemp
- cacau_show
- central_brf
- compra_certa
- comper
- fort_atacadista
- granado

---

# 5. Features suportadas

## Identidade e consentimento

- Identificação de usuário (anônimo vs logado).
- Vínculo usuário ↔ dispositivo via contactKey.
- Atualização de identidade em login/logout.
- Sanitização do identificador e hash opcional com salt (governado por Remote Config).
- Governança de opt-in de push.
- Disponibilização de estado de consentimento para o CRM.

## Push e navegação

- Recebimento de push Android (FCM) e iOS (APNs).
- Registro de device token (FCM e APNs).
- Associação device ↔ contactKey com atraso de registro até contactKey existir.
- Abertura de app via push com callback nativo para Flutter.
- Roteamento para telas internas: Home, PLP, PDP, Carrinho, e outros destinos cobertos pelo roteador genérico.
- Destinos suportados pelo roteador: `category`, `collection`, `sku-list`, `product`, `product-sku`, `search`, `cart`, `contentpage`, `route`, `landing-page`, `webview`, `offer_*`, apppage específicos.
- Fallback para rota padrão quando destino é inválido.
- Prevenção de navegação duplicada por clique em push.

## Tracking analítico de push

- Detecção de push SFMC por chaves como `sfmc_journey_id`.
- Registro de `campaign_details` no analytics manager interno do app.
- Analytics nativo da SDK via `NotificationManager.redirectIntentForAnalytics(...)` (Android).

## Atributos de usuário

- Envio de atributos ao SFMC (cobertura parcial; mais completa no fluxo BRF).
- Tags SFMC: API existe em `marketing_cloud_operations.dart (line 29)`, sem uso funcional encontrado.

## Leitura de estado da SDK

- Busca de token, deviceId, tags, atributos e flags via `marketing_cloud_data_fetcher.dart` e `marketing_cloud_state_fetcher.dart`.

## Geolocalização (parcial/experimental)

- Android: `setGeofencingEnabled(true)`, `setProximityEnabled(true)` em `MarketingCloudHandler.kt (line 45)`.
- iOS: `setLocationEnabled(true)`, `startWatchingLocation()` em `MarketingCloudHandler.swift (line 22, 49)`.
- Permissões de localização no `AndroidManifest.xml (line 36)`.
- Sem fluxo funcional completo de permissão dedicada, geofences operacionais ou analytics de proximidade no app.

## Governança e configuração

- Flag principal por build/env: `HAS_SALESFORCE_MARKETING_CLOUD` em `credentials.dart (line 1528)`.
- Configurações de hash e salt por build/env e Remote Config.
- Automação por cliente via `merx_cli`: copia arquivos nativos, faz merge em manifest/gradle/AppDelegate/MainActivity e injeta credenciais.

---

# 6. Status de implementação

## Eventos padrão para Salesforce Personalization

A tabela abaixo representa o status de implementação dos eventos padrão mapeados no assessment. Esses eventos são relevantes para a camada de Personalization integrada ao SFMC. Para detalhamento, ver documento `salesforce_personalization_consolidated.md`.

| Evento SF_Personalization | Implementado |
|---|---|
| requestAppTracking | SIM |
| logCustomEvent | SIM |
| logAppOpen | SIM |
| logFirstOpenApplication | SIM |
| logPromotion | NÃO |
| logItemsList | SIM |
| logOpenItemList | SIM |
| logAddToCart | SIM |
| logAddItemsToCart | NÃO |
| cartCleared | SIM |
| logOpenNavBarItem | SIM |
| logViewItem | SIM |
| logAddToWishlist | SIM |
| logShare | SIM |
| logLogin | SIM |
| logLogout | SIM |
| logSignUp | SIM |
| registerUser | SIM |
| logUserProfile | NÃO |
| setConsent | NÃO |
| readNotification | NÃO |
| logCampaignDetails | NÃO |
| logSearch | SIM |
| logProductSelectedFromSearch | NÃO |
| logViewSearchResults | SIM |
| logViewCategory | SIM |
| logRemoveFromCart | SIM |
| logRemoveFromCartItems | NÃO |
| logBeginCheckout | SIM |
| logGiveUpCheckout | NÃO |
| logViewCart | SIM |
| logAddShippingInfo | SIM |
| logDeliveryRegion | NÃO |
| logUseGiftCard | NÃO |
| logPurchase | SIM |
| logPurchasedProductCategory | NÃO |
| logPurchasedProductId | NÃO |
| logAddPaymentInfo | SIM |
| logPaymentType | SIM |
| logTapRecommendedProduct | SIM |
| logSponsoredProduct | NÃO |
| logNewTailSponsoredProduct | NÃO |
| logSponsoredBanner | NÃO |
| logCartDisplayAddToCart | SIM |
| logViewContent | NÃO |
| logSelectContent | NÃO |
| setUserProperties | SIM |
| logSetRegion | NÃO |
| logIssue | NÃO |
| logFirstPurchase | NÃO |

## Status por frente SFMC (Marketing Cloud core)

| Frente | Status | Nível de prontidão |
|---|---|---|
| SDK Flutter SFMC no app base | Total | Reutilizável |
| Inicialização da camada Flutter SFMC | Total | Reutilizável |
| SDK nativa Android | Total | Reutilizável |
| SDK nativa iOS | Total | Reutilizável |
| Bridge Flutter ↔ nativo | Total | Reutilizável |
| Configuração por cliente via CLI | Total | Reutilizável |
| Push receive Android | Parcial | Requer ajuste |
| Push receive iOS | Parcial | Requer ajuste |
| Registro de device token | Total | Reutilizável |
| Associação device ↔ usuário (contactKey) | Total | Reutilizável |
| Atributos de usuário | Parcial | Parcial (forte no BRF; limitado nos demais) |
| Deep link via push SFMC | Parcial | Requer ajuste (exige destination + destinationId) |
| Roteamento para telas internas | Total | Reutilizável |
| Prevenção de clique duplicado em push | Total | Reutilizável |
| Tracking analítico de push SFMC no app | Parcial | Parcial (analytics interno, não direto ao SFMC) |
| Estados e leitura de dados da SDK | Total | Reutilizável |
| Tags SFMC | Parcial | Experimental (API existe, sem uso funcional) |
| Eventos comportamentais para SFMC | Não implementado | Inexistente |
| Personalização in-app via SFMC | Não implementado | Inexistente |
| Geofencing / proximidade | Parcial | Experimental (SDK habilitada, sem fluxo funcional) |
| Governança por flags/config | Parcial | Requer ajuste |
| Identidade e consentimento (épico) | Parcial | Base estrutural existente |
| Tracking e eventos de jornada (épico) | Não implementado | Gap relevante |
| Vitrines e banners com SFMC como fonte | Não implementado | Inexistente |
| Geolocalização para campanhas (épico) | Parcial | Experimental |

---

# 7. Uso atual (clientes)

## Clientes com SFMC habilitado

8 clientes com `HAS_SALESFORCE_MARKETING_CLOUD=true` e os 4 campos obrigatórios preenchidos:

| Cliente | Observação |
|---|---|
| atacadao | SFMC habilitado, `args.properties (line 325)` |
| brastemp | SFMC habilitado, `args.properties (line 317)` |
| cacau_show | SFMC habilitado, `args.properties (line 317)` |
| central_brf | SFMC habilitado + fluxo de atributos mais completo (`brf_channel_controller.dart`) |
| compra_certa | SFMC habilitado, `args.properties (line 317)` |
| comper | SFMC habilitado, `args.properties (line 317)` |
| fort_atacadista | SFMC habilitado, `args.properties (line 317)` |
| granado | SFMC habilitado + Personalization SDK (`args.properties (line 319)`) |

## Padrões de uso identificados

- Uso predominante de push/identificação (contactKey, device token, recebimento de push).
- Fluxo de atributos de usuário mais rico concentrado no cliente BRF (`central_brf`).
- Granado é o único cliente com Personalization SDK ativo além do SFMC core.
- Nenhum cliente com evidência de eventos comportamentais enviados para SFMC além de identificação e analytics de push.

## Variações por cliente

- **central_brf:** atributos completos (email, contactId, contactkey_sf, cpf, mobilePhone) via `brf_channel_controller.dart`.
- **granado:** usa SFMC + Personalization custom API (tratado no documento de Personalization).
- **Demais:** cobertura limitada a contactKey e push básico.

---

# 8. Parâmetros e configuração

## Campos obrigatórios por cliente (via merx_cli)

| Parâmetro | Descrição |
|---|---|
| `HAS_SALESFORCE_MARKETING_CLOUD` | Flag principal de habilitação por build/env |
| `SFMC_PUSH_ACCESS_TOKEN` | Token de acesso para push SFMC |
| `SFMC_PUSH_APP_ID` | App ID no SFMC |
| `SFMC_PUSH_SENDER_ID` | Sender ID (FCM) |
| `SFMC_PUSH_SERVER_URL` | URL do servidor SFMC |

## Campos de governança de identidade (Remote Config e build/env)

| Parâmetro | Localização | Descrição |
|---|---|---|
| Hash contactKey | `credentials.dart (line 835)`, `remote_config_values_initializer.dart (line 1380)` | Habilita hash do identificador |
| Salt contactKey | `credentials.dart (line 1530)`, Remote Config | Salt para hash do identificador |

## Flags relevantes

- `hasSalesforceMarketingCloud`: controla inicialização condicional do módulo Flutter SFMC em `app_initializer.dart (line 325)`.
- `setAnalyticsEnabled`: habilita analytics da SDK em `marketing_cloud_operations.dart (line 61)`.
- `setInboxEnabled`, `setGeofencingEnabled`, `setPiAnalyticsEnabled`, `setProximityEnabled`: configurações de funcionalidades da SDK Android.
- `setDelayRegistrationUntilContactKeyIsSet`: atrasa registro de device até contactKey estar disponível.

## Dúvidas abertas de configuração (a fechar antes do contrato)

1. Qual será o contactKey (ID único do usuário)? Esse ID será o mesmo para MobilePush, Data Cloud e Personalization?
2. Quais consentimentos o app deve controlar: push, tracking, localização?
3. Qual é a lista final de eventos que o app deve enviar? Para cada evento, qual é o payload mínimo obrigatório? Existe um event contract já definido?
4. Todos os eventos vão para Data Cloud? Para Personalization? Há eventos que não devem ser enviados para Personalization?
5. Qual é o formato do payload de campanhas? O SFMC enviará IDs (productId, categoryId) ou URLs?
6. Quais destinos o app deve suportar: PDP, PLP, carrinho?
7. Qual o comportamento esperado quando o destino não existe ou o usuário não está logado?
8. O SFMC retornará apenas IDs de produtos ou payload completo (produto/banner)?
9. O endpoint configurado no CMS será fixo por slot ou dinâmico por usuário?
10. O app deve enviar no request: contactKey? contexto da tela?
11. Qual deve ser o fallback quando não houver resposta ou vier vazio?
12. Geolocalização: apenas enviar localização no evento ou geofence (trigger automático)?
13. O cliente usa Marketing Cloud Engagement ou Advanced Edition?
14. O app precisará usar apenas MobilePush ou também Personalization SDK?
15. Para cada jornada: qual tecnologia será usada (Journey Builder, Data Cloud, Personalization)?

---

# 9. Providers e strategies

## Estratégias de integração

### Estratégia 1: SDK SFMC direta (MobilePush)

- SDK Flutter (`sfmc 8.2.0`) com camada nativa Android/iOS.
- Cobre push, contactKey, atributos, analytics básico de campanha.
- Gerada por cliente via `merx_cli`.

### Estratégia 2: SFMC como fonte de vitrines e banners via CMS

- Épico planejado (não implementado).
- CMS define origem dos dados de vitrine/banner.
- App consome payload do SFMC, compatibilizado com contrato existente de componentes.
- Fallback padronizado quando SFMC não retornar resposta.

### Estratégia 3: Geolocalização para campanhas contextuais

- SDK configurada com `setGeofencingEnabled` e `setProximityEnabled` (Android) e `setLocationEnabled` (iOS).
- Fluxo funcional completo ainda não implementado no app.
- Escopo opcional/condicional — confirmar necessidade de geofence real ou apenas contexto de localização no evento.

## Providers por domínio

| Domínio | Provider atual | Status |
|---|---|---|
| Push notifications | SFMC MobilePush (FCM/APNs) | Implementado |
| Identidade | contactKey via SDK SFMC | Implementado |
| Atributos | SDK SFMC (`setAttribute`) | Parcial |
| Tags | SDK SFMC (`addTag`/`removeTag`) | Experimental |
| Analytics de push | SDK SFMC + AnalyticsManager interno | Parcial |
| Eventos comportamentais | SFMC (não implementado) | Gap |
| Personalização/vitrines | SFMC (não implementado) | Gap |
| Geolocalização | SDK SFMC (configurada, sem fluxo funcional) | Experimental |

---

# 10. Gaps e limitações

## Gaps identificados

| Tema | Status atual | Gap | Impacto |
|---|---|---|---|
| Jornadas via push | Parcial | Recebimento e redirect existem, mas jornada depende de payload no formato esperado; sem camada funcional mais ampla de journey orchestration | Validar payloads, métricas e cobertura operacional |
| Deep links de campanha | Parcial | Roteamento genérico reutilizável, mas exige `destination` + `destinationId`; nem todas as telas têm evidência de uso por campanha | Validar matriz de destinos e padrão de payload |
| Eventos comportamentais | Não implementado | Não há envio de eventos de navegação/commerce do app para SFMC | Gap relevante se o assessment esperar tracking comportamental via Marketing Cloud |
| Abandono de carrinho | Não implementado | Sem evento, trigger ou integração app → SFMC | Tratar como lacuna atual |
| Pós-compra | Não implementado | Sem eventos de pós-compra ou order lifecycle enviados para SFMC | Lacuna para jornadas transacionais/comportamentais |
| Personalização de vitrines | Não implementado | Sem vitrine dinâmica consumindo payload de Marketing Cloud | Não considerar como base existente |
| Personalização de banners | Não implementado | Sem banners dinâmicos ligados à Marketing Cloud | Não considerar como capacidade atual |
| Geolocalização | Parcial | SDK habilitada, mas sem fluxo funcional completo, permissão dedicada, geofences operacionais ou analytics de proximidade | Requer validação técnica |
| Documentação interna | Gap | `README.md` da integração está vazio | Risco de onboarding e governança |
| Atributos multi-cliente | Parcial | Cobertura de atributos além de contactKey limitada fora do fluxo BRF | Risco de dados incompletos no CRM |

## Limitações técnicas

- Payload de push exige `destination` + `destinationId`; campanhas sem esse formato não disparam o redirect corretamente.
- Camada nativa depende de geração por `merx_cli`; não é puramente centralizada.
- Tags SFMC existem como API mas sem uso funcional encontrado.
- Tracking analítico de push registra `campaign_details` no analytics interno do app, não diretamente no SFMC.
- Não há painel interno ou configuração no app para alterar journeys, destinos ou comportamento de campanhas sem apoio do payload/campanha externa.
- "Configurável sem desenvolvimento" limitado a flags, credenciais e Remote Config de hash/salt.

## Limitações de arquitetura

- Sem camada de reconciliação de entrega, abertura e conversão no app além do repasse para a SDK e do log analítico.
- Sem abstração de domínio mais alta para SFMC além do wrapper técnico do SDK.
- Não há bridge explícita entre as camadas de analytics próprias do app e o SFMC para eventos comportamentais.
- Destino `Conta` não aparece como destino explícito de SFMC no mapeamento de deep links.

---

# 11. Impacto e riscos

## Riscos técnicos

- **Regressão em tracking e navegação:** principal ponto de atenção ao evoluir o core com SFMC, por tratar-se de camada transversal que afeta múltiplas jornadas.
- **Fragmentação de eventos:** eventos inconsistentes inviabilizam jornadas como abandono, NPS e recompra.
- **Diferença App vs Site:** eventos do app e do site podem divergir se não houver contrato único.
- **Divergência payload vs rota:** payload de campanha incompatível com rotas do app causa navegação incorreta ou perda de contexto.
- **Dependência de autenticação no push:** comportamento esperado quando usuário não está logado ao abrir push não está definido.
- **Contrato de payload SFMC para vitrines:** falhas de fallback se payload vier vazio ou indisponível.
- **Baixa adesão do usuário à geolocalização:** permissão não garantida; app não deve bloquear por ausência de localização.
- **Dependência do OS para geolocalização:** comportamento de geofence varia por plataforma e versão.

## Riscos de negócio

- Jornadas críticas (abandono de carrinho, pós-compra, NPS, recompra) não podem ser ativadas sem eventos comportamentais.
- Personalização de vitrines e banners via Marketing Cloud não é capacidade atual — não deve ser vendida como pronta.
- Geolocalização para campanhas por proximidade não opera de forma funcional hoje.
- Cobertura de atributos limitada fora do BRF pode comprometer segmentação no CRM.

## Estimativa de esforço (por épico)

| Funcionalidade | Esforço em horas | Complexidade |
|---|---|---|
| Identidade e consentimento | 26h | Alta |
| Tracking e eventos | 34h | Alta |
| Push e navegação | 26h | Média |
| Vitrines e banners | 26h | Média |
| Geolocalização | 26h | Alta |
| **Total** | **138h** | — |

## Cronograma estimado

| Janela estimada | Épico | Dependências | Observações |
|---|---|---|---|
| Semanas 1–2 | Consolidação da base de identidade e consentimento | Definição de identificador e regras de consentimento | Base estrutural |
| Semanas 3–4 | Padronização de tracking e eventos de jornada | Definição de eventos pelo CRM | Base de dados |
| Semanas 5–6 | Consolidação de push e navegação | Payload definido | Reaproveitamento |
| Semanas 7–8 | Vitrines e banners com SFMC | Contrato CMS e SFMC | Evolução de conteúdo |
| Semanas 9–10 | Geolocalização | Confirmação de escopo | Opcional/condicional |

---

# 12. Requisitos para completar

## O que falta para cobertura total

### Épico 1 — Identidade e consentimento (26h / Alta)

- Centralizar e padronizar vínculo entre identidade, sessão e consentimento.
- Garantir atualização de identidade em login/logout em todos os fluxos do app (não apenas BRF).
- Governança de opt-in de push e disponibilização de estado de consentimento.
- Definir identificador canônico com o CRM (contactKey único e consistente).
- Tratar troca de conta no mesmo device.
- Eliminar divergência entre CRM e app.

### Épico 2 — Tracking e eventos de jornada (34h / Alta)

- Definir event contract único com o CRM (lista mínima de eventos, payload obrigatório por evento).
- Implementar envio de eventos comportamentais para SFMC: navegação, carrinho, compra, NPS, churn.
- Vínculo usuário ↔ evento respeitando estado de autenticação.
- Garantir envio consistente para Data Cloud e/ou Personalization conforme definição do cliente.
- Eliminar fragmentação de eventos entre app e site.

### Épico 3 — Push e navegação (26h / Média)

- Padronizar contrato de payload de campanhas (destination + destinationId vs outros formatos).
- Validar e expandir matriz de destinos suportados.
- Implementar fallback de navegação para destinos inválidos e comportamento quando usuário não está logado.
- Validar métricas de delivery/open e cobertura operacional de jornadas.

### Épico 4 — Vitrines e banners com SFMC (26h / Média)

- Adicionar SFMC como fonte configurável via CMS para vitrines e banners.
- Implementar leitura de origem via CMS, consumo de conteúdo SFMC, compatibilização de payload.
- Implementar fallback padronizado quando SFMC não retornar resposta.
- Definir: endpoint fixo por slot ou dinâmico por usuário; app envia contactKey e/ou contexto de tela.

### Épico 5 — Geolocalização (26h / Alta)

- Implementar fluxo funcional de permissão de localização no app.
- Definir abordagem: apenas enviar localização no evento ou geofence (trigger automático).
- Implementar estado de disponibilidade de localização e contexto para campanhas.
- Garantir que app não bloqueie sem localização (permissão obrigatória mas não bloqueante).
- Validar com engenharia se recursos de geofence da SDK já estão operacionais.

## Melhorias necessárias

- Preencher `README.md` interno da integração SFMC (atualmente vazio).
- Ampliar cobertura de atributos de usuário além de contactKey para todos os clientes (não apenas BRF).
- Implementar telemetria/logs explícitos de falhas na integração SFMC.
- Consolidar documentação operacional de configuração por cliente.
- Validar quais payloads de campanha são usados de fato em produção.
- Validar se geolocalização habilitada na SDK é realmente usada em produção por algum cliente.

## Evoluções possíveis

- Tags SFMC com uso funcional de negócio (hoje API existe sem uso).
- Sincronização ampla de perfil para SFMC em todos os fluxos de atualização cadastral.
- Jornadas comportamentais avançadas (abandono de carrinho, pós-compra, NPS, churn).
- Banners e vitrines dinâmicos abastecidos pelo Marketing Cloud.
- Geofence e campanhas por proximidade.

---

# 13. Critérios de avaliação para agente de IA

## Pré-requisitos

- `HAS_SALESFORCE_MARKETING_CLOUD=true` na configuração do cliente.
- Campos obrigatórios preenchidos: `SFMC_PUSH_ACCESS_TOKEN`, `SFMC_PUSH_APP_ID`, `SFMC_PUSH_SENDER_ID`, `SFMC_PUSH_SERVER_URL`.
- Identificador canônico (contactKey) definido e consistente com CRM.
- Event contract definido previamente com o time de CRM.
- Payload de campanhas (push/deeplink) no formato esperado pelo app (destination + destinationId).
- Consentimentos de push, tracking e localização definidos e governados.

## Checks funcionais

- App inicializa SDK SFMC condicionalmente pela flag `hasSalesforceMarketingCloud`.
- Login define contactKey; logout limpa contactKey.
- Device token registrado corretamente (FCM no Android, APNs no iOS).
- Push recebido, aberto e roteado para tela correta conforme payload.
- Fallback de navegação ativo para destinos inválidos.
- Prevenção de clique duplicado em push funcionando.
- `campaign_details` registrado no analytics manager após abertura de push SFMC.
- Atributos de usuário enviados ao SFMC após login (verificar cobertura por cliente).

## Sinais de problema

- Push recebido mas sem roteamento para tela correta (verificar `destination` + `destinationId` no payload).
- ContactKey não atualizado após login/logout (verificar `auth_update_marketing_cloud.dart`).
- Device token não registrado (verificar `NotificationMessagingService.kt` e `AppDelegate.swift`).
- Vitrines/banners esperando conteúdo SFMC retornam vazios (funcionalidade não implementada).
- Eventos comportamentais ausentes no CRM (funcionalidade não implementada — gap de épico 2).
- Geolocalização habilitada na SDK mas sem fluxo de permissão no app (verificar fluxo funcional).
- Configuração habilitada mas SDK não inicializada (verificar flag e `app_initializer.dart`).
- Fragmentação de eventos entre app e site causando jornadas inconsistentes no CRM.

---

# Referências técnicas

## Arquivos de código referenciados

| Arquivo | Responsabilidade |
|---|---|
| `pubspec.yaml (line 149)` | Dependência sfmc 8.2.0 |
| `centered_dependencies.dart (line 103)` | Export compartilhado do pacote SFMC |
| `salesforce_marketing_cloud.dart` | Módulo Flutter: inicialização, bridge, operações, leitura de estado, contactKey, atributos |
| `app_initializer.dart (line 325)` | Inicialização condicional do módulo SFMC no startup global |
| `marketing_cloud_operations.dart` | Tags, analytics enable, operações diversas da SDK |
| `marketing_cloud_data_fetcher.dart` | Busca token, deviceId, atributos e flags |
| `marketing_cloud_state_fetcher.dart` | Busca estado da SDK |
| `auth_update_marketing_cloud.dart (line 14)` | Integração com camada de autenticação para contactKey |
| `brf_channel_controller.dart (line 214)` | Fluxo BRF: atualização de contactKey e atributos completos |
| `app_routes.dart (line 1193, 1211, 1219, 1276, 1420, 1566, 1603)` | Roteamento genérico de push/deeplink, prevenção de duplicata, fallback |
| `notification_service.dart (line 362, 689, 703)` | Analytics de campanha: `sfmc_journey_id`, `campaign_details` |
| `credentials.dart (line 835, 1528, 1530)` | Flags e credenciais SFMC por build/env |
| `remote_config_values_initializer.dart (line 1380)` | Hash/salt por Remote Config |
| `MarketingCloudHandler.kt` | Handler nativo Android: inicialização SDK, token FCM, analytics nativo |
| `NotificationMessagingService.kt (line 6)` | Recebimento push Android via FCM |
| `AndroidManifest.xml (line 36, 108)` | Permissões de localização e registro do NotificationMessagingService |
| `app/build.gradle.kts (line 116)` | Dependência nativa Android |
| `build.gradle.kts (line 14)` | Repositório Maven SFMC |
| `MarketingCloudMethodChannel.kt (line 7, 16, 17)` | Method channel Android: callback de push para Flutter |
| `AppDelegate.swift (line 27, 57, 112)` | Handler iOS: push receive, token APNs |
| `MarketingCloudHandler.swift (line 15, 22, 49, 54, 60)` | Handler nativo iOS: geolocalização, token, push |
| `MarketingCloudMethodChannel.swift (line 4, 19, 22)` | Method channel iOS: callback de push para Flutter |
| `README.md (line 1)` | Documentação interna da integração — atualmente vazia |

## Endpoints e canais

| Canal / Endpoint | Descrição |
|---|---|
| `salesforce_marketing_cloud_push_redirect_channel` | Method channel Flutter para recebimento de redirect de push |
| Endpoints SFMC (MobilePush) | Push access token, app ID, sender ID, server URL — configurados por cliente |
| FCM (Firebase Cloud Messaging) | Push Android |
| APNs (Apple Push Notification service) | Push iOS |

## Produtos e serviços externos

- Salesforce Marketing Cloud (SFMC) / Marketing Cloud Engagement
- MobilePush SFMC
- Journey Builder (operado externamente)
- Data Cloud (destino de eventos — a confirmar)
- Firebase Cloud Messaging (FCM)
- Apple Push Notification service (APNs)
- `merx_cli` (ferramenta interna de geração de configuração por cliente)
