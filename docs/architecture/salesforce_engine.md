---
title: Salesforce Integration
feature: salesforce_integration
module: integration
engine: agnostic
analysis_type: consolidated_analysis
analysis_date: 2026-04-13
confidence: medium
status: valid
source: multi_document_merge
suggested_question: Como funciona a integração com Salesforce no SaaS?
display_priority: 10
---

# 1. Resumo executivo

## Visão consolidada da integração

A integração com Salesforce no SaaS abrange três dimensões distintas e complementares:

1. **Camada de e-commerce mobile (SFCC + Contentful + BFF):** Arquitetura que posiciona o Salesforce Commerce Cloud (SFCC) como sistema de registro para domínio transacional (catálogo, preço, estoque, promoções, cliente, carrinho, checkout, pedidos), o Contentful como sistema de governança editorial (banners, landing pages, navegação, blocos de conteúdo) e um BFF (Backend for Frontend) como camada de orquestração mobile, cache, contrato único e fallback.

2. **Camada de busca e merchandising SaaS (Search Platform):** O SaaS atua como plataforma de busca/merchandising sobre o catálogo do SFCC. O SFCC permanece como sistema de registro; o SaaS é responsável por indexação, ranking, personalização, faceting, analytics e aprendizado de relevância.

3. **Integração OMS (Order Management System):** Análise do case Cacau Show como referência de integração OMS via CustomApi no app Salesforce. Identifica o que é reutilizável, o que está fortemente acoplado ao cliente Cacau Show e o que precisa ser refatorado para suportar novos clientes OMS.

## Principais capacidades

- Suporte a jornada completa de e-commerce mobile: splash/bootstrap, home, menu, vitrines, busca, PLP, PDP, carrinho, checkout, login, cadastro, minha conta, meus pedidos, wishlist, push/deeplink, CMS pages, configurações remotas.
- Indexação e serving de busca com baixa latência (meta P95 < 150ms), com suporte a autocomplete, tolerância a erros de digitação, ranking semântico, faceting e merchandising.
- Capacidade de segmentação e personalização baseada em grupos de clientes, geolocalização, dispositivo e coorte.
- Estrutura de CustomApi no app Salesforce com ponto de extensão OMS para integração com sistemas de gestão de pedidos de terceiros.

## Principais limitações

- A integração OMS existente está fortemente acoplada ao cliente Cacau Show, com hardcodes de identificadores, mapeamentos de status específicos e ausência de abstração multi-provider OMS.
- Vários métodos críticos do repositório Salesforce estão marcados como `Unimplemented` (pickup shipping, logística, detalhes de pedido, cancelamento).
- A ausência de BFF aumenta o acoplamento do app com múltiplas APIs e compromete performance no cold start.
- Contratos híbridos entre Contentful e SFCC (Contentful referenciando SKUs/categorias) podem quebrar sem validação automática de consistência.
- Checkout, pagamento e antifraude representam o ponto de maior risco técnico e de prazo.

---

# 2. Visão geral da integração

## O que a integração cobre

A integração cobre três camadas funcionais:

**Camada transacional (SFCC):**
- Catálogo de produtos: produtos master, variantes, associações de categorias, flags de busca/online.
- Preços: price books, preços de lista e promoção, preços por moeda.
- Estoque: níveis de estoque/ATS por lista de inventário/site.
- Sessão do shopper, autenticação, carrinho, checkout e pedidos.
- Grupos de clientes para segmentação/personalização (opcional).

**Camada editorial (Contentful):**
- Hero banners, cards, storytelling, slots editoriais.
- Estrutura de menus com itens editoriais.
- Blocos de conteúdo por página, landing pages, CMS pages.
- Textos de UX (ajuda, políticas, termos, empty states, mensagens de manutenção).

**Camada de busca/merchandising (SaaS Search Platform):**
- Indexação e serving de busca sobre o catálogo do SFCC.
- Regras de pin/bury, produtos promovidos, regras de campanha.
- Ranking personalizado por segmento, analytics, AI/ML para aprendizado de relevância.

## Produtos/serviços Salesforce envolvidos

- **Salesforce Commerce Cloud (SFCC):** sistema central de registro transacional.
- **OCAPI (Open Commerce API):** Data API para recuperação em lote/leitura administrativa.
- **SCAPI (Shopper Commerce API):** APIs de storefront para dados em contexto de cliente.
- **Salesforce Account Manager:** provedor de OAuth para autenticação OCAPI/SCAPI.
- **Cartridges customizados:** endpoints específicos do tenant não expostos nas APIs padrão, utilizados para exports customizados.
- **Business Manager (BM):** console de configuração do SFCC onde permissões de OCAPI são habilitadas.

## Objetivo dentro do SaaS

O SaaS atua em duas frentes complementares:
1. Como **plataforma de app mobile** que consome dados do SFCC e do Contentful, orquestrando a jornada completa do consumidor final.
2. Como **plataforma de busca e merchandising** que indexa o catálogo do SFCC e entrega resultados ranqueados/facetados ao storefront, desacoplando a experiência de busca do motor nativo do SFCC.

---

# 3. Arquitetura e funcionamento

## Componentes principais

### Arquitetura mobile (SFCC + Contentful + BFF)

| Componente | Responsabilidade |
|---|---|
| SFCC | Verdade transacional: catálogo, preço, estoque, promoções, cliente, carrinho, checkout, pedidos |
| Contentful | Verdade editorial: composição de páginas, banners, slots, landing pages, navegação |
| BFF (Backend for Frontend) | Orquestração mobile, cache, contrato único, segurança, fallback |
| App mobile | Renderização de componentes, consumo de APIs via BFF |
| OMS externo | Integração via CustomApi: merchants, estoque por loja, endereços, status de pedido |

### Arquitetura da plataforma de busca SaaS

| Componente | Responsabilidade |
|---|---|
| Conectores SFCC (OCAPI/SCAPI + jobs) | Extração de dados do SFCC |
| Pipeline de ingestão | Normalização de entidades SFCC (produtos, variantes, categorias, disponibilidade, preços, localidade) |
| Serviço de indexação | Construção de índices pesquisáveis por site/localidade/moeda |
| Query API | Endpoint de busca de baixa latência consumido pelo storefront |
| Pipeline de eventos/feedback | Coleta de eventos de clique/conversão para analytics e sinais de ranking |

## Fluxo de dados

### Fluxo mobile (app)

```
App → BFF → SFCC (dados transacionais)
App → BFF → Contentful (dados editoriais)
BFF → App (payload consolidado)
App → BFF → OMS (integração logística, via CustomApi)
```

### Fluxo da plataforma de busca

```
SFCC → (pull via API / push via jobs) → Pipeline de ingestão
→ Transformação/enriquecimento → Publicação no índice de serving
Storefront → Query API SaaS → Resultados ranqueados/facets
Storefront → Eventos (clique/conversão) → Pipeline de feedback → Analytics/ML
```

## Dependências externas e internas

**Externas:**
- SFCC: catálogo, preço, estoque, sessão, pedidos.
- Contentful: conteúdo editorial e assets.
- OMS de terceiros: logística, merchants, status de pedido (via CustomApi).
- Gateways de pagamento e sistemas de antifraude.
- Provedores de push (Firebase/APNs/CRM).
- IdP corporativo (opcional, para SSO/social login).
- Transportadoras/rastreamento (opcional, para Meus Pedidos).

**Internas:**
- BFF: orquestração e contrato único mobile.
- Serviço de feature flags/remote config (pode ser corporativo ou próprio).
- Motor de indexação SaaS.

## Como o app se comunica com Salesforce

- Via OCAPI ou SCAPI (a depender da configuração do cliente), com OAuth via Account Manager (client credentials).
- Configurações de OCAPI no Business Manager devem habilitar explicitamente clientes, recursos e métodos.
- Endpoints customizados via cartridges para exports específicos do tenant.
- Transferência via SFTP para exports em lote (opcional).

## Papel do middleware (BFF)

O BFF é fortemente recomendado como camada intermediária para:
- Consolidar payload único de bootstrap (evitando múltiplas chamadas no cold start).
- Aplicar cache e estratégias de fallback.
- Garantir contrato único entre app e backends.
- Centralizar segurança (armazenamento e validação de tokens).
- Orquestrar chamadas paralelas a SFCC e Contentful.

A ausência do BFF aumenta o acoplamento do app com múltiplas APIs e pode comprometer a performance e a manutenibilidade.

---

# 4. Comportamento por engine

## Salesforce Commerce (SFCC)

Engine principal. Responsável por todo o domínio transacional. A integração cobre:
- Autenticação via OCAPI/SCAPI com OAuth.
- Catálogo, preços (price books), estoque (listas de inventário).
- Sessão do shopper, carrinho (basket), checkout, pedidos.
- Grupos de clientes para segmentação.
- Suporte a multi-site, multi-moeda e multi-localidade (configurável).

Variações relevantes por padrão de API:
- **OCAPI:** mais antigo, amplamente suportado, usado para Data API (lote/admin) e Shop API (storefront). Requer configuração explícita de permissões no BM.
- **SCAPI (Shopper Commerce API):** mais moderno, recomendado para novos projetos. A decisão entre OCAPI, SCAPI ou camada própria é um ponto de definição crítico por cliente.
- **Cartridges customizados:** utilizados para endpoints específicos do tenant não expostos nas APIs padrão.

## Shopify

Não há menção a integração com Shopify nas documentações fornecidas. Não é possível descrever comportamento por esta engine com base nas fontes disponíveis.

## VTEX

Não há menção a integração com VTEX nas documentações fornecidas. Não é possível descrever comportamento por esta engine com base nas fontes disponíveis.

## Diferenças relevantes por configuração SFCC

- **Multi-site:** índices separados por site + localidade (+ moeda) são a abordagem recomendada. Analyzers, sinônimos e stemming específicos por localidade.
- **Price books:** cobertura consistente de preço por site é requisito para indexação correta.
- **Flags online/searchable:** devem estar corretamente configuradas no SFCC para que produtos apareçam nos índices.

---

# 5. Features suportadas

## Autenticação e identidade

- Login via SFCC nativo ou IdP externo (SSO/social login — configurável por cliente).
- Token refresh/revoke.
- Reset de senha (start/confirm).
- Cadastro de novo cliente com consentimentos (LGPD/marketing).
- Verificação de e-mail/telefone (se aplicável).
- Atualização de consentimentos.

## Bootstrap e configuração remota

- Carregamento de configuração mínima de inicialização: versão mínima do app, feature flags, país/moeda, catálogo/pricebook ativo, tenant/site, idioma, links críticos, timeout/retry.
- Sessão shopper (init/refresh token).
- Configurações editoriais globais (Contentful).
- Contexto comercial (site, currency, locale).
- Remote config, feature flags, mensagens globais, kill switch.

## Busca

- Busca por palavra-chave, autocomplete, tolerância a erros de digitação, ranking semântico e orientado a atributos.
- Search suggestions/autocomplete.
- Facets/refinements.
- Busca mista (produto + conteúdo CMS) — opcional, a confirmar por cliente.
- Engine externa de recomendação/busca (Einstein, Algolia etc.) — opcional, a confirmar.

## PLP (Product Listing Page)

- Listagem por categoria/coleção com filtros e ordenação.
- Facets e paginação (estratégia offset vs cursor — a definir por cliente).
- Enriquecimento visual/editorial via Contentful (header de categoria) — opcional.
- Preço, promoção, estoque por produto.

## PDP (Product Detail Page)

- Detalhe completo do produto: nome, imagens, variações, preço, promoção, estoque, SLA de entrega, atributos.
- Seleção de variante.
- Disponibilidade por SKU.
- Produtos relacionados/cross-sell.
- Blocos editoriais PDP (Contentful) — opcional.

## Carrinho

- Criação/leitura do basket.
- Adição, atualização e remoção de itens.
- Aplicação e remoção de cupons.
- Estimativa de frete.
- Recálculo de totais.
- Carrinho anônimo com merge no login — a confirmar por cliente.

## Checkout

- Sessão de checkout.
- Métodos de envio.
- Definição de endereços de entrega e cobrança.
- Métodos de pagamento/tokenização.
- Finalização do pedido (place order).
- 3DS/status de pagamento (se aplicável).
- Checkout guest — a confirmar por cliente.

## Minha Conta

- Leitura e atualização de perfil.
- Gerenciamento de livro de endereços (CRUD).
- Atualização de preferências e consentimentos.

## Meus Pedidos

- Histórico de pedidos.
- Detalhe do pedido por número.
- Rastreamento de entrega.
- Links para nota fiscal/documentos (se aplicável).

## Wishlist / Favoritos

- Criação e leitura de wishlist.
- Adição e remoção de itens.
- Mover wishlist para carrinho.
- Wishlist para usuário anônimo — a confirmar por cliente.

## Vitrines de Produtos

- Configuração de vitrine via Contentful (título, regra, coleção).
- Produtos por IDs ou por categoria/regra.
- Preço promocional e estoque em tempo real via SFCC.

## Banners e Conteúdo CMS

- Banners por slot/página.
- Blocos de conteúdo por tipo.
- Assets via CDN Contentful.
- Segmentação (regras no CMS, BFF ou ferramenta externa — a confirmar).

## Home

- Composição dinâmica via Contentful (hero, cards, storytelling, vitrines).
- Resolução de dados comerciais em tempo real via SFCC (preço/estoque dos produtos exibidos).

## Menu / Navegação

- Árvore de categorias comerciais (SFCC).
- Itens editoriais (Contentful).
- Resolução de deeplinks.

## Notificações Push / Deeplink

- Registro de device token.
- Preferências de notificação.
- Resolução de deeplink (produto/categoria/página).
- Conteúdo de destino editorial (Contentful).
- Orquestração via BFF/CRM.

## CMS Pages / Landing Pages

- Renderização de páginas editoriais dinâmicas.
- Estrutura de página, componentes, mídia, metadata.
- Resolução de referências a produto/categoria via SFCC.

## Integração OMS (Order Management System)

- Integração via CustomApi com adapter por cliente.
- Obtenção de merchants, estoque por merchant, endereço de merchant.
- Persistência e seleção de loja.
- Ajuste de endereço para pickup.
- Leitura de status de pedido vindo do OMS.
- Jornada logística: disponibilidade/SLA/janelas/modalidades de fulfillment.

## Merchandising e Personalização (Plataforma de Busca SaaS)

- Regras de pin/bury, produtos promovidos, regras de campanha, ranking por categoria.
- Ranking baseado em segmento (geolocalização, dispositivo, coorte, grupo de cliente).
- Analytics: performance de queries, taxa de zero resultados, uso de facets, CTR/CVR, atribuição de receita.
- AI/ML: learning-to-rank baseado em eventos de clique/conversão, entendimento de query (sinônimos, classificação de intenção, sugestões de rewrite).

---

# 6. Status de implementação

| Feature / Módulo | Status | Observação |
|---|---|---|
| Bootstrap / splash | Parcial | Depende de BFF e definição de remote config |
| Home | Parcial | Depende de contrato Contentful + SFCC batch |
| Menu / navegação | Parcial | Depende de definição de source of truth da ordem final |
| Vitrines de produtos | Parcial | Depende de cache e batch |
| Banners e conteúdo CMS | Total | Contentful como fonte única; menor dependência SFCC |
| Busca (produto) | Total | SFCC como fonte principal; tuning de relevância necessário |
| Busca mista (produto + conteúdo) | Não implementado | Requisito a confirmar por cliente |
| PLP | Total | Alta confiança |
| PDP | Total | Alta confiança; editorial opcional via Contentful |
| Carrinho | Total | Alta confiança |
| Checkout | Parcial | Gateways, antifraude, 3DS a definir; maior risco técnico |
| Login | Parcial | Depende de Auth SFCC nativo vs IdP corporativo |
| Cadastro | Parcial | Depende de duplo opt-in e requisitos LGPD |
| Minha Conta | Total | Alta confiança |
| Meus Pedidos | Parcial | Depende de OMS/transportadora e contrato unificado |
| Wishlist | Parcial | Cenário anônimo pode exigir microserviço adicional |
| Push / Deeplink | Parcial | Depende de plataforma CRM/push e resolver central |
| CMS Pages / Landing Pages | Parcial | Depende de renderer genérico e design system |
| Remote Config / Feature Flags | Parcial | Depende de serviço corporativo ou próprio |
| Integração OMS (genérica) | Não implementado | Existente apenas para Cacau Show; novo adapter necessário |
| Pickup shipping (SFCC) | Não implementado | `Unimplemented` no repositório atual |
| getPickupPoints / getPickupPointById | Não implementado | `Unimplemented` em `sf_logistics_repository.dart` |
| getOrderDetail, timeline, cancel (SFCC) | Não implementado | `Unimplemented` em `sf_order_repository.dart` |
| Cart simulation (SFCC) | Não implementado | `Unimplemented` em `sf_checkout_repository.dart` |
| Personalização / Ranking ML | Parcial | Depende de volume de eventos e consentimento |

---

# 7. Uso atual (clientes)

## Clientes que usam Salesforce

### Cacau Show

- Cliente de referência para integração OMS via CustomApi no app Salesforce.
- Integração inclui: obtenção de merchants, estoque por merchant, endereço de merchant, seleção e persistência de loja, ajuste de endereço para pickup, mapeamento de status de pedido específico do cliente.
- A integração está fortemente acoplada ao cliente, com hardcodes explícitos.

## Padrões de uso identificados

- Arquitetura SFCC + Contentful como padrão de mercado para e-commerce mobile.
- Uso de CustomApi como ponto de extensão OMS, com adapter por cliente.
- BFF recomendado mas não obrigatório na implementação atual.

## Variações por cliente

- **Padrão de API SFCC:** OCAPI, SCAPI ou camada própria — variável por cliente e por legado.
- **OMS:** adapter específico por cliente (único case implementado: Cacau Show).
- **Autenticação:** SFCC nativo ou IdP corporativo — a definir por cliente.
- **Modalidades logísticas:** pickup vs delivery — variável por cliente.
- **Gateways de pagamento/antifraude:** variável por cliente.

---

# 8. Parâmetros e configuração

## Remote Config

Parâmetros controlados sem nova publicação nas lojas:
- Flags de módulo (toggles on/off por feature).
- Thresholds e limites operacionais.
- Mensagens de manutenção.
- Versão mínima do app.
- Kill switch (opcional).

## Configurações do SFCC (Business Manager)

- Permissões de cliente OCAPI: clientes, recursos e métodos habilitados explicitamente.
- Associações corretas de site/catálogo.
- Flags online/searchable por produto.
- Price books por site.
- Listas de inventário por site.
- Jobs de export full + delta configurados.

## Configurações da plataforma de busca SaaS (por cliente)

- Mapeamento de atributos SFCC para campos do índice.
- Campos pesquisáveis e definição de facets.
- Regras de boost/bury, sinônimos.
- Política de out-of-stock (ocultar, rebaixar, exibir com badge backorder).
- Estratégia de exibição de variantes.
- Estratégia de localidade/moeda (índices separados por site + localidade recomendado).
- Taxonomia de eventos e janelas de atribuição.
- Feature toggles: reconciliação de inventário em tempo real (on/off), ranking AI (on/off por localidade/site), ranking personalizado (condicionado a consentimento).

## CMS (Contentful)

- Modelo de conteúdo padronizado (obrigatório para rollout).
- Segmentação editorial (regras no CMS, BFF ou ferramenta externa — a definir).
- Preview em ambiente de homologação (a confirmar).
- Localização/idioma por país.

## Configurações OMS (CustomApi)

- Chaves de configuração OMS em Salesforce: `*_OMS_CS` (exemplo: `salesforce_api_config_keys.dart:22`).
- Injeção no startup via `app_initializer.dart:348`.
- Binding de CustomApi via DI: `custom_api_bindings.dart`.

## Flags relevantes

- `client == 'cacau_show'`: condição atual para instanciar OMS custom (hardcode a ser removido em refatoração).
- Reconciliação de inventário em tempo real: feature toggle por site/localidade.
- Ranking com AI: feature toggle por site/localidade.
- Ranking personalizado: condicionado a consentimento de dados.

---

# 9. Providers e strategies

## Estratégias de integração

### Estratégia 1: Pull via API (OCAPI/SCAPI)

- Recuperação de dados do SFCC via chamadas de API (Data API para lote/admin, Shop API para storefront).
- Indicado para dados de catálogo, preços e estoque com cadência de atualização controlada.
- Requer configuração de permissões no Business Manager.

### Estratégia 2: Push via jobs/feeds

- Jobs agendados no SFCC exportam dados (full + delta) para consumo pela plataforma SaaS.
- Indicado para carga inicial e atualizações periódicas de catálogo.
- Transferência opcional via SFTP para exports em lote.

### Estratégia 3: Streaming / quase tempo real

- Eventos de mudança, hooks customizados ou bridge de mensagens para updates de alta volatilidade (estoque/preço).
- Indicado para dados que mudam com alta frequência e impactam diretamente a experiência de busca.

### Estratégia 4: Fallback de consulta em tempo real

- Para campos de alta volatilidade (estoque/preço), consulta direta ao SFCC como fallback quando os dados do índice podem estar desatualizados.
- Aumenta latência mas garante acurácia em picos.

### Estratégia 5: Adapter OMS via CustomApi

- Implementação de adapter específico por cliente OMS.
- Contrato genérico de CustomApi com implementações plug-in por cliente.
- Recomendado: abstrair domínio OMS genérico e remover tipos Cacau Show do contrato base.

## Providers utilizados

| Domínio | Provider atual | Observação |
|---|---|---|
| Commerce transacional | SFCC (OCAPI/SCAPI) | Principal |
| Conteúdo editorial | Contentful | Principal |
| OMS | CustomApi (único adapter: CacauShowApi) | Fortemente acoplado ao cliente |
| Autenticação | SFCC nativo ou IdP externo | Variável por cliente |
| Push notifications | Firebase/APNs/CRM | Variável por cliente |
| Busca/merchandising | SaaS Search Platform (índice próprio) | Sobre catálogo SFCC |
| Pagamentos/antifraude | Gateways externos | Variável por cliente |

## Variações por cenário

- **Multi-site:** índices separados por site + localidade. Regras de sortimento regionais.
- **Multi-moeda:** price books por moeda, pré-computados no índice.
- **Autenticação SSO:** IdP corporativo com OAuth, em vez de SFCC nativo.
- **Logística pickup:** jornada específica com seleção de merchant e endereço de pickup.

---

# 10. Gaps e limitações

## Limitações técnicas

- **Integração OMS fortemente acoplada ao Cacau Show:** `CustomApi` exporta tipos Cacau Show no contrato base; binding Salesforce só instancia OMS custom quando `client == 'cacau_show'`; apenas uma implementação de CustomApi no app (`cacau_show_api.dart`); hardcodes de merchant id (fixo `6045`), chaves locais (`cacau_show_*`) e `storeName: 'Cacau Show'`.
- **Métodos Unimplemented no repositório Salesforce:**
  - `pickupShipping*`, simulações, lista pickup, cart simulation: `sf_checkout_repository.dart:315`, `:337`, `:343`, `:683`.
  - `getOrderDetail`, timeline, cancel: `sf_order_repository.dart:21`, `:93`.
  - `getPickupPoints`/`getPickupPointById`: `sf_logistics_repository.dart:13`.
- **Risco de race condition:** patch de pickup em pedidos usa `forEach(async)` sem `await`: `sf_order_repository.dart:63`.
- **Ausência de BFF:** sem BFF, o app pode precisar de múltiplas chamadas no cold start, aumentando tempo de inicialização e acoplamento.
- **Ausência de endpoint batch (preço/estoque):** compromete performance de Home, Vitrines, PLP e PDP.
- **Sem contrato único de componentes CMS:** esforço de renderer cresce sem design system/contrato padronizado.

## Limitações de API

- **Rate limits do OCAPI:** throttling e erros de escopo/permissão podem impactar ingestão.
- **Jobs sobrepostos ou demorados no SFCC:** geram snapshots desatualizados.
- **Diferenças de timing entre staging e produção:** impactam testes de paridade.
- **Ausência de endpoint batch nativo para preço/estoque:** múltiplas chamadas por produto.

## Limitações de arquitetura

- **Contrato híbrido Contentful → SFCC:** referências a SKUs/categorias podem quebrar sem validação automática de consistência.
- **Status inconsistente de pedido:** sem contrato unificado entre SFCC e OMS, status pode divergir.
- **Conflito de fonte de verdade de perfil:** entre SFCC e CRM (se integrado).
- **Deeplink sem resolver central:** links inválidos após mudanças de slug/estrutura.
- **Governança fraca de flags:** remote config sem governança pode causar comportamento inconsistente por versão do app.
- **Índice desatualizado em falha do SFCC:** sem estratégia de fallback, busca pode servir dados obsoletos.

## Dependências externas

- Definição do padrão de API SFCC (SCAPI/OCAPI/custom) — crítica e variável por cliente.
- Contratos de pagamento/antifraude — não padronizados.
- Plataforma de CRM/push — variável por cliente.
- Serviço corporativo de feature flags — pode ou não existir.
- OMS de terceiros — adapter a ser desenvolvido por cliente.

---

# 11. Impacto e riscos

## Riscos técnicos

1. **Ausência de BFF:** alto acoplamento do app com múltiplas APIs e baixa performance no cold start.
2. **Contratos híbridos Contentful → SFCC:** referências quebradas sem validação automática.
3. **Checkout/pagamento/antifraude:** ponto mais crítico de prazo e estabilidade; maior risco do projeto.
4. **Divergência de status de pedido:** múltiplos sistemas (SFCC + OMS) sem contrato unificado.
5. **Estratégia de autenticação/token mal definida:** retrabalho em Login/Cadastro/Minha Conta.
6. **Busca sem tuning de relevância e facets:** impacto direto na conversão.
7. **Governança fraca de componentes CMS:** aumento de esforço de renderer no app.
8. **Ausência de endpoint batch (preço/estoque):** compromete performance de Home, Vitrines, PLP e PDP.
9. **Deeplink sem resolver central:** links inválidos após mudanças de slug/estrutura.
10. **Remote config sem governança:** comportamento inconsistente por versão do app.
11. **Race condition no patch de pickup:** `forEach(async)` sem `await` em `sf_order_repository.dart:63`.
12. **Rate limits e throttling OCAPI:** impacto em ingestão e sincronização.
13. **Sinônimos/boosts agressivos:** baixa precisão de busca.
14. **Baixo volume de eventos em locais com pouco tráfego:** prejudica aprendizado ML.

## Riscos de negócio

- Vender integração OMS como "plug and play do case Cacau Show" sem esclarecer necessidade de customização nova.
- Divergência de preço/estoque em picos de tráfego se cadência de delta for lenta.
- Produto online no SFCC sem atributos obrigatórios no índice SaaS (invisível na busca).
- Dependência jurídica em consentimento/versionamento de termos (LGPD).
- Segurança mobile: armazenamento de token, revogação, sessão concorrente.
- Tokenização PCI para mobile (a confirmar se já disponível).

## Pontos críticos da integração

- **Checkout e pagamentos:** maior risco técnico e de prazo.
- **Integração OMS:** requer nova customização, não é reaproveitamento direto.
- **Definição do padrão de API SFCC:** SCAPI vs OCAPI vs camada própria — impacta toda a integração.
- **Presença ou ausência de BFF:** impacta arquitetura e performance global.

---

# 12. Requisitos para completar

## O que falta para cobertura total

### Frente 1: Entrega do cliente (novo caso OMS)

- Implementar novo adapter OMS (provider real por contrato do cliente), em vez de reaproveitar diretamente `CacauShowApi`.
- Refatorar `CustomApi` para remover tipos Cacau Show do contrato base e separar domínio OMS genérico.
- Implementar jornada logística Salesforce B2C completa:
  - Disponibilidade/SLA/janelas/modalidades de fulfillment em checkout.
  - Seleção e persistência robusta da opção logística no pedido.
  - Leitura e exibição de status de pedido vindo do OMS.
- Fechar `Unimplemented` críticos:
  - `pickupShipping*`, simulações, lista pickup, cart simulation (`sf_checkout_repository.dart`).
  - `getOrderDetail`, timeline, cancel (`sf_order_repository.dart`).
  - `getPickupPoints`/`getPickupPointById` (`sf_logistics_repository.dart`).
- Corrigir race condition: `forEach(async)` sem `await` em `sf_order_repository.dart:63`.

### Frente 2: Hardening de core

- Abstração multi-provider OMS (remover acoplamentos Cacau Show).
- Implementar BFF como camada de orquestração mobile (se não existir).
- Implementar endpoint batch para preço/estoque (Home, Vitrines, PLP, PDP).
- Definir resolver central de deeplinks.
- Implementar validação automática de referências Contentful → SFCC.
- Implementar governança de feature flags/remote config.
- Padronizar contrato de componentes CMS (design system/renderer genérico).

### Plataforma de busca SaaS

- Configuração de permissões OCAPI no Business Manager do cliente.
- Carga inicial completa (catálogo + categorias + atributos + preços + inventário).
- Relatório de validação e mapeamento de atributos.
- Publicação do índice e testes de paridade com comportamento esperado do storefront SFCC.
- Configuração de sinônimos, stopwords, regras de boost.
- Definição de política de out-of-stock e estratégia de variantes.
- Jobs de export full + delta no SFCC.
- Estratégia de fallback para falha de API/SFCC.

## Melhorias necessárias

- Definição de SLAs de performance mobile e limites de payload por tela.
- Estratégia de paginação mobile (offset vs cursor).
- Estratégia de autenticação (SFCC nativo vs IdP corporativo).
- Definição de source of truth para status de pedido e tracking (SFCC, OMS ou transportadora).
- Modelo de conteúdo padronizado no Contentful.
- Estratégia de rollout gradual por percentual/coorte para feature flags.

## Evoluções possíveis

- Busca mista (produto + conteúdo CMS).
- Engine externa de recomendação (Einstein, Algolia etc.).
- Ranking personalizado com consentimento e dados de comportamento.
- Suporte a multi-país/multi-moeda/multi-site no go-live.
- Duplo opt-in no cadastro.
- Wishlist para usuário anônimo.
- Integração com CRM para preferências de conta.

---

# 13. Critérios de avaliação para agente de IA

## Pré-requisitos (checks pré-onboarding)

- Credenciais OCAPI/SCAPI válidas, escopos e métodos habilitados, endpoints acessíveis.
- Matriz de site/catálogo/localidade/moeda documentada e completa.
- Amostra de dados validada contra schema e campos obrigatórios.
- Configuração correta de OCAPI no Business Manager (clientes, recursos, métodos).
- Jobs de export full + delta configurados e testados no SFCC.
- Flags online/searchable corretos por produto e por site.
- Price books e listas de inventário consistentes por site.
- Definição clara de ownership de atributos customizados e nomenclatura.
- Comportamento definido para produtos indisponíveis e promoções futuras.
- Padrão de API SFCC definido (OCAPI, SCAPI ou camada própria).
- Definição de autenticação (SFCC nativo ou IdP corporativo).
- Definição de gateways de pagamento e antifraude.

## Padrões obrigatórios de qualidade de dados

- IDs únicos de produto.
- Relacionamento de variantes consistente.
- Categorias normalizadas.
- Títulos, descritivos e marca completos.
- Atributos facetáveis essenciais presentes.
- Cobertura consistente de preço/inventário para todos os produtos online por site.

## Checks funcionais

- Busca retorna resultados com preço/estoque corretos por site/moeda.
- Facets alinhados à árvore de categorias do SFCC.
- Autocomplete e tolerância a erros de digitação funcionando.
- PDP exibe variantes, preço e disponibilidade corretos.
- Carrinho aceita adição, atualização e remoção de itens; cupons aplicados corretamente.
- Checkout completa fluxo de endereço, frete, pagamento e pedido.
- Login e cadastro funcionando com consentimentos.
- Meus Pedidos exibe histórico e status corretos.
- Deeplinks resolvem para telas/produtos/categorias corretos.
- Remote config atualiza flags sem nova publicação.

## Sinais de problema

- Produto online no SFCC ausente nos resultados de busca SaaS (checar flags online/searchable e cobertura de índice).
- Desalinhamento variante-pai causando duplicação ou ausência de resultados.
- Divergência de preço/estoque entre PLP e PDP (verificar TTL e fonte dos dados).
- Status de pedido inconsistente (verificar contrato entre SFCC e OMS).
- Taxa de zero resultados elevada (verificar sinônimos, tuning e cobertura de catálogo).
- Latência P95 acima de 150ms na Query API (verificar cache CDN/edge e configuração do índice).
- Erro de permissão OCAPI (verificar configurações no Business Manager).
- Rate limiting do OCAPI durante ingestão (verificar cadência de jobs e backoff).
- Deeplink inválido após mudança de slug (verificar resolver central).
- Comportamento inconsistente do app por versão (verificar governança de feature flags).
- Race condition em operações de pickup (checar `forEach(async)` sem `await`).

---

# Referências técnicas

## Arquivos de código referenciados

| Arquivo | Referência |
|---|---|
| `custom_api_bindings.dart` | Binding de CustomApi via DI; condição `client == 'cacau_show'` para instanciar OMS |
| `cacau_show_api.dart` | Única implementação de CustomApi OMS: token, endereço, merchants, estoque |
| `cacau_show_controller.dart` | Controller OMS Cacau Show; hardcodes: merchant id `6045`, chaves `cacau_show_*`, `storeName: 'Cacau Show'` |
| `dynamic_sales_channel_binding.dart` | Jornada dinâmica Salesforce ligada ao CacauShowController |
| `sf_order_repository.dart` | Pedidos Salesforce; ajuste de endereço pickup; métodos `Unimplemented`: getOrderDetail, timeline, cancel |
| `sf_order_model.dart` | Mapeamentos de status de pedido Cacau Show |
| `sf_checkout_repository.dart` | Checkout Salesforce; seleção de envio e shipment; métodos `Unimplemented`: pickupShipping*, simulações, lista pickup, cart simulation |
| `sf_logistics_repository.dart` | Logística Salesforce; métodos `Unimplemented`: getPickupPoints, getPickupPointById |
| `salesforce_api_config_keys.dart` | Chaves de configuração OMS em Salesforce (`*_OMS_CS`) |
| `app_initializer.dart` | Injeção de configuração OMS no startup |
| `moved_custom_api.dart` | Ponto de extensão CustomApi e pipeline de DI |
| `custom_api.dart` | Exporta tipos Cacau Show no contrato base (a refatorar) |

## Endpoints e APIs (nível lógico)

### Bootstrap / Configuração

- `endpoint de app bootstrap/config`
- `endpoint de session init/refresh token`
- `query de configurações editoriais globais (Contentful)`
- `endpoint de contexto comercial (site, currency, locale)`
- `endpoint de remote config`
- `endpoint de feature flags`
- `query de mensagens globais (Contentful)`
- `endpoint de kill switch`

### Home, Menu, Vitrines, Banners

- `query de home page by slug/id (Contentful)`
- `endpoint de product tiles by ids (SFCC)`
- `endpoint de category/product recommendations (SFCC ou motor externo)`
- `endpoint de pricing/availability em lote`
- `endpoint de category tree/navigation`
- `query de menu structure e links editoriais (Contentful)`
- `endpoint de resolve deeplink`
- `query de product showcase component (Contentful)`
- `endpoint de products by ids`
- `endpoint de products by category/rule`
- `endpoint de promotional price e inventory`
- `query de banners by slot/page (Contentful)`
- `query de content blocks by type (Contentful)`
- `endpoint de asset delivery (CDN Contentful)`

### Busca

- `endpoint de product search`
- `endpoint de search suggestions/autocomplete`
- `endpoint de search facets/refinements`
- `query opcional de content search (Contentful)`

### PLP / PDP

- `endpoint de products by category/search`
- `endpoint de facets`
- `endpoint de pagination/infinite scroll`
- `query opcional de category hero content (Contentful)`
- `endpoint de product details by id/slug`
- `endpoint de variant selection`
- `endpoint de availability by sku`
- `endpoint de related products`
- `query de editorial PDP blocks (Contentful)`

### Carrinho

- `endpoint de create/get basket`
- `endpoint de add/update/remove item`
- `endpoint de apply/remove coupon`
- `endpoint de shipping estimation`
- `endpoint de basket totals recalculation`

### Checkout

- `endpoint de checkout session`
- `endpoint de shipping methods`
- `endpoint de set shipping/billing address`
- `endpoint de payment methods/tokens`
- `endpoint de place order`
- `endpoint de 3DS/payment status`

### Login / Cadastro / Conta

- `endpoint de customer authentication`
- `endpoint de token refresh/revoke`
- `endpoint de password reset start/confirm`
- `endpoint opcional de social login/SSO`
- `endpoint de customer registration`
- `endpoint de email/phone verification`
- `endpoint de consent update`
- `endpoint de get/update customer profile`
- `endpoint de address book CRUD`
- `endpoint de preferences/consents`

### Pedidos

- `endpoint de order history`
- `endpoint de order details by orderNo`
- `endpoint de shipment tracking`
- `endpoint de invoice/document links`

### Wishlist

- `endpoint de wishlist create/get`
- `endpoint de wishlist add/remove item`
- `endpoint de wishlist to cart`

### Push / Deeplink / CMS Pages

- `endpoint de device token registration`
- `endpoint de notification preferences`
- `endpoint de deeplink resolver (produto/categoria/página)`
- `query de landing content by deeplink (Contentful)`
- `query de page by slug (Contentful)`
- `query de component tree by page (Contentful)`
- `endpoint de resolve product/category references (SFCC)`

### OMS (CustomApi)

- `customApi.getToken`
- `customApi.getMerchantAddress`
- `customApi.getMerchants`
- `customApi.getStockByMerchant`

## APIs e protocolos SFCC

- **OCAPI Data API:** recuperação em lote/leitura administrativa.
- **OCAPI Shop API:** dados em contexto de storefront.
- **SCAPI (Shopper Commerce API):** alternativa moderna ao OCAPI.
- **Account Manager:** provedor de OAuth (client credentials) para autenticação OCAPI/SCAPI.
- **Cartridges customizados:** endpoints específicos do tenant.
- **Business Manager (BM):** configuração de permissões OCAPI.
- **SFTP:** transferência de exports em lote (opcional).

## Componentes da plataforma de busca SaaS

- **Pipeline de ingestão:** normalização de entidades SFCC.
- **Serviço de indexação:** índices por site/localidade/moeda.
- **Query API:** endpoint de busca de baixa latência.
- **Pipeline de eventos/feedback:** coleta de eventos comportamentais.
- **Motor de busca externo (SaaS):** hospedado como serviço, com índices por tenant/site.
- **CDN/edge cache:** cache de queries populares; pré-aquecimento de termos principais.
