---
title: Salesforce Personalization Integration
feature: salesforce_personalization_integration
module: integration
engine: agnostic
analysis_type: consolidated_analysis
analysis_date: 2026-04-13
confidence: medium
status: valid
source: multi_document_merge
suggested_question: Como funciona a integração com Salesforce Personalization no SaaS?
display_priority: 9
---

# 1. Resumo executivo

## Visão consolidada da integração

A integração com Salesforce Marketing Cloud Personalization (anteriormente Evergage) no SaaS é composta por dois caminhos distintos e complementares:

1. **SDK client-side** via pacote `salesforce_personalization`: responsável por tracking comportamental e identidade do usuário, com inicialização no startup do app e envio de eventos de jornada de compra para o Personalization.

2. **API customizada** (implementada no módulo Granado): responsável por buscar campanhas de vitrine personalizadas e enviar eventos de campanha, compra e cadastro de forma server-side/customizada, via endpoints HTTP dedicados.

A integração é transversal ao engine de e-commerce do cliente — não está exclusivamente acoplada ao Salesforce Commerce Cloud. O único cliente com Personalization ativo no repositório é Granado, cujo engine é Magento.

O objetivo de negócio é personalizar vitrines/recomendações e enriquecer eventos de jornada de compra. O encaixe na plataforma se dá pelo `analytics service` + módulo de showcases + custom module por cliente.

## Principais capacidades

- Tracking via SDK: inicialização, definição de userId, eventos principais de jornada e cart/purchase implementados e plugados no `AnalyticsManager`.
- Campanhas de vitrine via custom API: fluxo de request/response e render implementado (condicionado a `PersonalizationService` registrado).
- Eventos de impressão e clique de campanha disparados no carregamento e clique.
- Estratégia padrão (`DefaultPersonalizationAnalyticsStrategy`) usando SDK diretamente.
- Estratégia customizada Granado (`GranadoPersonalizationAnalyticsStrategy`) usando API custom para registration e purchase.

## Principais limitações

- Apenas 1 cliente ativo em 174 (~0,57% de adoção): Granado.
- Campanhas de vitrine retornam `null` sem binding ativo de `PersonalizationService`.
- Tratamento de erro limitado: `catch` genérico retornando `null` em datasource de campanha; sem telemetria de falha robusta.
- Eventos de impressão/clique com `try/catch` silencioso.
- Risco crítico de segurança: credenciais sensíveis em arquivo versionado (`customizations/granado/args_remote.json`).
- Documentação de integração genérica ausente (`integrations/personalization/README.md`).
- Acoplamento implícito: showcase depende de `Get.isRegistered<PersonalizationService>` sem fallback observável.
- Cobertura de testes focada em modelos Granado; lacuna para fluxo end-to-end.

---

# 2. Visão geral da integração

## O que a integração cobre

- Inicialização do Personalization SDK no startup do app.
- Identificação do usuário (userId via CPF/email ou anonymousId).
- Tracking comportamental: eventos de navegação, jornada de compra, carrinho, wishlist, busca.
- Campanhas de vitrine personalizadas (via custom API, implementado para Granado).
- Eventos de campanha: impressão e clique.
- Fluxo de cadastro e compra server-side customizado (estratégia Granado).

## Produtos/serviços Salesforce envolvidos

- **Salesforce Marketing Cloud Personalization (Evergage):** plataforma de personalização comportamental e campanhas de vitrine.
- SDK client-side via pacote `salesforce_personalization` (Flutter).
- Endpoints HTTP customizados Evergage (API custom Granado): campanhas, interações, dados de pedido.

## Objetivo dentro do SaaS

- Personalizar vitrines/recomendações com base em comportamento do usuário.
- Enriquecer eventos de jornada de compra para ativação de campanhas contextuais.
- Habilitar showcases dinâmicos com conteúdo proveniente do Personalization como fonte.

---

# 3. Arquitetura e funcionamento

## Componentes principais

| Componente | Arquivo | Responsabilidade |
|---|---|---|
| PersonalizationAnalytics | `personalization_analytics.dart` | Eventos de app/e-commerce para SDK |
| DefaultPersonalizationAnalyticsStrategy | `default_personalization_analytics_strategy.dart` | Estratégia padrão: usa SDK diretamente |
| GranadoPersonalizationAnalyticsStrategy | `granado_personalization_analytics_strategy.dart` | Estratégia Granado: usa API custom para registration e purchase |
| PersonalizationCampaignGenericShowcaseDataSource | `personalization_campaign_generic_showcase_datasource.dart` | Datasource de campanhas de vitrine via API custom |
| PersonalizationService | `personalization_service.dart` | Serviço de integração com API Granado/Evergage |
| PersonalizationDataSource | `personalization_datasource.dart` | Camada de acesso a dados da API custom |
| Inicialização | `app_initializer.dart` | Inicialização do SDK no startup |

## Fluxo de dados

### Fluxo de tracking (SDK)

```
Usuário interage com app
  → PersonalizationAnalytics captura evento
  → AnalyticsManager roteia para PersonalizationAnalyticsStrategy
  → Strategy envia evento para SDK (Personalization/Evergage)
  → SDK encaminha para Salesforce Personalization
```

### Fluxo de campanhas de vitrine (API custom)

```
App carrega showcase do tipo personalization-campaign
  → PersonalizationCampaignGenericShowcaseDataSource realiza request à API custom
  → PersonalizationService / PersonalizationDataSource executam chamada HTTP
  → Resposta convertida para ShowcaseContent
  → Showcase renderiza produtos personalizados
  → Evento de impressão disparado (com try/catch silencioso)
  → Clique em produto dispara evento de interação (stat=Click)
```

### Fluxo de cadastro/compra (estratégia Granado)

```
Evento de registration ou purchase ocorre no app
  → GranadoPersonalizationAnalyticsStrategy intercepta
  → Executa chamada à API custom de order data (endpoint Granado)
  → Dados enviados server-side para Personalization
```

## Dependências

**Internas:**
- `AnalyticsManager`: roteamento de eventos para strategies.
- `AuthUserStorage`: fornece userId (CPF/email ou anonymousId).
- `BaseCartController`: itens de carrinho para eventos de carrinho.
- Módulo de Showcases: integração com `PersonalizationCampaignGenericShowcaseDataSource`.

**Externas:**
- Salesforce Marketing Cloud Personalization (Evergage) via SDK Flutter.
- Endpoints HTTP customizados Granado/Evergage: campanhas, interações, dados de pedido.

## Inputs e outputs

**Inputs:**
- `personalization_enabled` (bool)
- `personalization_account` (string)
- `personalization_dataset` (string)
- `campaignIds` (lista de IDs de campanha)
- Usuário: CPF/email ou `anonymousId`
- Itens de carrinho / produto visualizado

**Outputs:**
- Eventos de tracking enviados ao Personalization/Evergage.
- Resposta de campanhas convertida para `ShowcaseContent`.

---

# 4. Comportamento por engine

## Salesforce Commerce (SFCC)

A integração de Personalization não está exclusivamente acoplada ao engine Salesforce. O único caso implementado (Granado) usa engine Magento. Há lógica específica para Salesforce em `last-seen-products` dentro de showcase/navigation (`home_showcase.dart`), mas isso não é o núcleo da integração de Marketing Cloud Personalization.

## Magento (Granado)

Engine do único cliente ativo com Personalization habilitado. Usa tanto SDK client-side quanto API custom (campanhas, interações, purchase data). Parâmetros dedicados definidos em `customizations/granado/args_remote.json`.

## Outras engines

Não há evidência de uso de Personalization em outras engines no repositório atual.

## Diferenças relevantes por engine/cliente

A integração é agnóstica ao engine de commerce. A diferença relevante é por cliente e por configuração:
- **Estratégia padrão:** usa SDK diretamente (sem custom API).
- **Estratégia Granado:** usa API custom para registration e purchase, além do SDK.
- Seleção de estratégia via `Environment.saasClientId == 'granado'` e registro de service.

---

# 5. Features suportadas

## Tracking via SDK (PersonalizationAnalytics)

Implementado e plugado no `AnalyticsManager`. Cobre eventos principais de jornada:

| Evento | Implementado |
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

## Campanhas de vitrine (API custom)

- Request/response de campanha implementado.
- Conversão para `ShowcaseContent` implementada.
- Dependente de `PersonalizationService` registrado no startup.
- Sem binding ativo: showcase retorna `null`.

## Eventos de campanha

- Impressão de campanha: disparado no carregamento (`try/catch` silencioso).
- Clique em produto: dispara interação com `stat=Click`.

## Identificação do usuário

- Login define `userId` (CPF/email ou `anonymousId`).
- Logout limpa `userId`.
- `PersonalizationApi.start` executado na inicialização.

## Fluxo de cadastro/compra server-side (Granado)

- Estratégia customizada para registration e purchase via API custom.
- Condicionado ao cliente Granado e ao módulo custom API habilitado.

---

# 6. Status de implementação

| Feature | Status | Observação |
|---|---|---|
| Tracking via SDK (PersonalizationAnalytics) | Total | Inicialização, userId, eventos principais de jornada e cart/purchase implementados e plugados no AnalyticsManager |
| Campanhas de vitrine (personalization-campaign) | Parcial | Fluxo de request/response e render implementado, mas depende de PersonalizationService registrado; sem binding ativo retorna null |
| Eventos de impressão/clique de campanha | Parcial | Disparados no carregamento e clique, porém com try/catch silencioso e sem telemetria de falha robusta |
| Fluxo de cadastro/compra server-side (Granado) | Parcial | Estratégia específica existe, mas condicionada ao cliente e ao módulo custom API habilitado |
| Telemetria de erro estruturada | Não implementado | Catch genérico retornando null; sem visibilidade operacional de falhas |
| Testes end-to-end (datasource/showcase/analytics) | Não implementado | Cobertura focada em modelos Granado; lacuna no fluxo completo |
| Documentação genérica da integração | Não implementado | integrations/personalization/README.md ausente |
| Governança de credenciais | Gap crítico | Credenciais sensíveis em arquivo versionado (args_remote.json) |
| Provider abstraction para campaign datasource | Não implementado | Único provider efetivo hoje é Granado |

---

# 7. Uso atual (clientes)

## Clientes com Personalization habilitado

- **Granado:** único cliente com `PERSONALIZATION_ENABLED=true` no repositório atual (`customizations/granado/args.properties`).
- Adoção em repositório: 1 de 174 clientes (~0,57%).

## Padrões de uso identificados

- Granado usa SDK client-side + API custom (campanhas, interações, purchase data).
- Parâmetros dedicados definidos em `customizations/granado/args_remote.json`.
- Engine do cliente: Magento (não Salesforce Commerce Cloud).

## Variações por cliente

- Estratégia padrão (`DefaultPersonalizationAnalyticsStrategy`): SDK direta, sem custom API.
- Estratégia Granado (`GranadoPersonalizationAnalyticsStrategy`): SDK + API custom para registration e purchase.
- Seleção de estratégia: `Environment.saasClientId == 'granado'` e `PersonalizationService` registrado.

---

# 8. Parâmetros e configuração

## Obrigatórios para SDK

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `personalization_enabled` | bool | Habilita integração de Personalization |
| `personalization_account` | string | Conta no Personalization/Evergage |
| `personalization_dataset` | string | Dataset de dados no Personalization |

## Obrigatórios para custom API (Granado)

| Parâmetro | Descrição |
|---|---|
| `enable_personalization_custom_api` | Habilita uso da API custom |
| `personalization_custom_base_url` | URL base da API custom |
| `personalization_custom_authenticated_endpoint` | Endpoint para usuários autenticados |
| `personalization_custom_anonymous_endpoint` | Endpoint para usuários anônimos |
| `personalization_custom_order_data_base_url` | URL base para dados de pedido |
| `personalization_custom_order_data_endpoint` | Endpoint de dados de pedido |

## Opcionais com impacto

| Parâmetro | Impacto |
|---|---|
| `personalization_custom_username` | Habilita Basic Auth na API custom |
| `personalization_custom_password` | Habilita Basic Auth na API custom |
| `*_channel` | Altera payload sem mudar arquitetura |
| `*_interaction_name` | Altera payload sem mudar arquitetura |

## Escopo de configuração

- Predominantemente por cliente: `args.properties` e remote config do assinante (`args_remote.json`).
- Não é configuração global do SaaS.

## Inputs de contexto (por chamada)

- `campaignIds`: lista de IDs de campanha.
- Usuário: CPF/email (autenticado) ou `anonymousId` (anônimo).
- Itens de carrinho / produto visualizado.

---

# 9. Providers e strategies

## Strategies de analytics

| Strategy | Classe | Quando usado |
|---|---|---|
| Padrão | `DefaultPersonalizationAnalyticsStrategy` | Quando cliente não é Granado ou sem custom API registrada |
| Granado | `GranadoPersonalizationAnalyticsStrategy` | Quando `Environment.saasClientId == 'granado'` e PersonalizationService registrado |

## Provider de vitrine

- `PersonalizationCampaignGenericShowcaseDataSource` via `PersonalizationService`.
- Não há múltiplos providers no factory atual; Granado é o único provider efetivo.

## Seleção de estratégia

- Estratégia Granado selecionada por `Environment.saasClientId == 'granado'` + `PersonalizationService` registrado.
- Sem binding ativo: showcase retorna `null` sem erro explícito.

---

# 10. Gaps e limitações

## Gaps identificados

- **Documentação de integração genérica ausente:** `integrations/personalization/README.md` inexistente, impedindo onboarding de novos clientes sem apoio direto da engenharia.
- **Acoplamento implícito:** showcase depende de `Get.isRegistered<PersonalizationService>` sem fallback observável; falha silenciosa.
- **Tratamento de erro limitado:**
  - `catch` genérico retornando `null` em datasource de campanha.
  - `try/catch` silencioso nos eventos de impressão/clique.
  - Sem telemetria ou visibilidade operacional para falhas de campanha/interação.
- **Cobertura de testes:** focada em modelos Granado; lacuna para fluxo end-to-end de datasource/showcase/analytics manager.
- **Provider único:** apenas Granado como provider efetivo de campaign datasource; sem abstração para outros clientes.
- **Estratégia Granado fortemente acoplada ao cliente:** seleção via `Environment.saasClientId == 'granado'` impede generalização direta.

## Risco crítico de segurança

- Credenciais sensíveis (Basic Auth: `personalization_custom_username` e `personalization_custom_password`) presentes em arquivo versionado: `customizations/granado/args_remote.json`.
- Esse padrão expõe segredos no repositório, aumentando o risco de comprometimento.
- Ação necessária: mover credenciais para secret manager e remover de arquivos versionados.

## Limitações técnicas

- Sem binding ativo de `PersonalizationService`: showcase do tipo `personalization-campaign` retorna `null` sem erro explícito.
- Falhas de campanha/interação não geram alertas ou logs estruturados.
- Dependência forte de configuração correta por cliente (parâmetros obrigatórios ausentes causam falha silenciosa).

## Limitações de arquitetura

- Não há abstração de domínio genérica para campaign datasource além do provider Granado.
- A estratégia de analytics Granado é condicionada ao `saasClientId`, não a uma interface de configuração extensível.
- Sem mecanismo de fallback explícito quando Personalization não retorna resposta.

---

# 11. Impacto e riscos

## Impacto de negócio

- Afeta diretamente recomendação personalizada, CTR em vitrines e qualidade de eventos de personalização.
- Falhas silenciosas reduzem eficácia das campanhas sem alertas para o time de produto ou CRM.

## Riscos técnicos

- **Falhas silenciosas:** `catch` genérico e `try/catch` sem log estruturado tornam invisíveis as falhas de campanha e interação.
- **Dependência de configuração correta por cliente:** parâmetros ausentes ou incorretos causam falha sem diagnóstico claro.
- **Exposição de segredos em repositório:** `args_remote.json` versionado com credenciais Basic Auth — risco crítico de comprometimento de segurança.
- **Acoplamento implícito ao cliente Granado:** extensão para novos clientes exige refatoração da strategy e do provider.

## Dependências críticas

- Remote Config: parâmetros de habilitação e configuração.
- Registro de `PersonalizationService` no startup: sem ele, vitrines retornam `null`.
- Disponibilidade dos endpoints Salesforce/Evergage e endpoint de order data (Granado).

---

# 12. Requisitos para completar

## Implementação necessária

- **Telemetria de erro estruturada:** implementar logs e métricas nos fluxos de campanha/interação para visibilidade operacional de falhas.
- **Testes de integração:** cobrir fluxo end-to-end de `personalization-campaign` no pipeline de showcases (datasource → showcase → analytics manager).
- **Validação de pré-requisitos padronizada:** checar configuração e binding de `PersonalizationService` no startup com logs explícitos, em vez de falha silenciosa.
- **Fallback explícito:** implementar fallback observável quando `PersonalizationService` não está registrado ou quando API não retorna resposta.

## Integrações necessárias

- **Governança de credenciais:** retirar segredos de arquivos versionados (`args_remote.json`) e migrar para secret manager.
- **Documentação:** criar e preencher `integrations/personalization/README.md` com guia de integração genérico.

## Evolução de arquitetura

- **Provider abstraction para campaign datasource:** hoje apenas Granado é provider efetivo. Criar abstração genérica para suportar novos clientes sem acoplamento a `saasClientId`.
- **Strategy extensível:** remover dependência de `Environment.saasClientId == 'granado'` para seleção de estratégia; migrar para configuração baseada em parâmetros do cliente.

## Evoluções possíveis

- Extensão de Personalization para novos clientes além de Granado.
- Eventos ainda não implementados: `logPromotion`, `logAddItemsToCart`, `logUserProfile`, `setConsent`, `logCampaignDetails`, `logGiveUpCheckout`, `logFirstPurchase`, entre outros (ver seção 5).
- Personalização de banners e slots via Personalization como fonte.
- Suporte a múltiplos providers de campaign datasource no factory.

---

# 13. Critérios de avaliação para agente de IA

## Pré-requisitos

- `personalization_enabled=true` na configuração do cliente.
- `personalization_account` e `personalization_dataset` preenchidos.
- Se campanha customizada: `enable_personalization_custom_api=true` e todos os endpoints válidos preenchidos.
- `PersonalizationService` registrado no startup.
- Credenciais Basic Auth (se aplicável) fora de arquivos versionados e em secret manager.

## Verificações funcionais

- App executa `PersonalizationApi.start` na inicialização.
- Login define `userId`; logout limpa `userId`.
- Vitrine do tipo `personalization-campaign` retorna produtos (não `null`).
- Clique em produto dispara interação com `stat=Click`.
- Eventos de purchase e cadastro executam a estratégia esperada para o cliente (Granado: via API custom; demais: via SDK).
- Logs de erro estruturados visíveis em falhas de campanha/interação (após implementação da telemetria).

## Sinais de problema

- Vitrines vazias sem erro explícito: verificar se `PersonalizationService` está registrado (`Get.isRegistered<PersonalizationService>`).
- Ausência de eventos após login/click/purchase: verificar binding da strategy no `AnalyticsManager` e configuração de `personalization_enabled`.
- Respostas HTTP 4xx/5xx nos endpoints custom: verificar credenciais, URLs e disponibilidade dos endpoints Evergage.
- Configuração habilitada sem service registrado: verificar inicialização no startup e ordem de bindings.
- Eventos de impressão/clique ausentes mesmo com vitrine exibida: verificar `try/catch` silencioso nos datasources.
- Credenciais expostas em repositório: verificar `args_remote.json` e mitigar antes do go-live.

---

# Referências técnicas

## Arquivos de código referenciados

| Arquivo | Responsabilidade |
|---|---|
| `personalization_analytics.dart` | Eventos de app/e-commerce para SDK Personalization |
| `default_personalization_analytics_strategy.dart` | Estratégia padrão: SDK direta |
| `granado_personalization_analytics_strategy.dart` | Estratégia Granado: API custom para registration e purchase |
| `personalization_campaign_generic_showcase_datasource.dart` | Datasource de campanhas de vitrine via API custom |
| `personalization_service.dart` | Serviço/API Granado: orquestração de chamadas |
| `personalization_datasource.dart` | Camada de acesso a dados da API custom |
| `app_initializer.dart` | Inicialização do SDK no startup global |
| `home_showcase.dart` | Lógica last-seen-products com referência a Salesforce (showcase/navigation) |
| `customizations/granado/args.properties` | `PERSONALIZATION_ENABLED=true` para Granado |
| `customizations/granado/args_remote.json` | Parâmetros remotos de Personalization para Granado — contém credenciais sensíveis (risco de segurança) |
| `customizations/granado/granado_envs.dart` | Variáveis de ambiente da API custom Granado |
| `integrations/personalization/README.md` | Documentação interna — ausente (gap) |

## Endpoints e APIs (nível lógico)

| Endpoint | Descrição |
|---|---|
| `personalization_custom_base_url` + `personalization_custom_authenticated_endpoint` | Campanhas para usuário autenticado (Granado) |
| `personalization_custom_base_url` + `personalization_custom_anonymous_endpoint` | Campanhas para usuário anônimo (Granado) |
| `personalization_custom_order_data_base_url` + `personalization_custom_order_data_endpoint` | Dados de pedido server-side (Granado) |
| Endpoints Salesforce/Evergage SDK | Tracking comportamental via SDK client-side |

## Dependências internas do app

| Dependência | Papel |
|---|---|
| `AnalyticsManager` | Roteamento de eventos para strategies |
| `AuthUserStorage` | Fornece userId (CPF/email ou anonymousId) |
| `BaseCartController` | Itens de carrinho para eventos de carrinho |
| Módulo de Showcases | Integração com PersonalizationCampaignGenericShowcaseDataSource |
| Remote Config | Parâmetros de habilitação e configuração por cliente |

## Produtos e serviços externos

- Salesforce Marketing Cloud Personalization (Evergage) — SDK Flutter client-side
- Endpoints HTTP customizados Evergage (Granado) — campanhas, interações, order data
