---
title: Cashback, Loyalty e Rewards Integration
feature: cashback_loyalty_rewards
module: cashback, rewards, offers, fidelity_program, currency_balance
engine: agnostic
analysis_type: consolidated_analysis
analysis_date: 2026-04-13
confidence: high
status: valid
source: multi_document_merge
suggested_question: Como funciona a integração de cashback, loyalty e rewards no SaaS?
display_priority: 10
---

# 1. Resumo executivo

## Visão consolidada da integração

O SaaS Kobe possui uma camada estruturada e madura de cashback, loyalty e rewards distribuída em múltiplos módulos: `cashback`, `rewards`, `offers`, `fidelity_program`, `currency_balance` e customizações por cliente. A cobertura funcional é ampla (~35 macro-funcionalidades identificadas), com suporte a múltiplos providers externos, configuração granular por Remote Config e variações por cliente via `customizations/*`.

Os três domínios funcionam de forma parcialmente integrada, mas ainda fragmentada entre si:
- **Cashback:** camada mais madura, com múltiplos providers, regras de uso, exibição em PLP/PDP/carrinho/checkout, extrato e perfil.
- **Rewards/Bonifiq:** pontos, metas, recompensas, tiers e cashback de recompensa, operados via `RewardsApi.bonifiq`.
- **Offers/Loyalty:** programas de fidelidade, clube de vantagens, catálogo de recompensas, QR redeem, integrado a providers como Propz, Zanthus e Mercafacil.

A avaliação geral de maturidade é **7.5/10** para cashback e intermediária para rewards/loyalty, com oportunidades relevantes de upsell e productização na base atual.

## Principais capacidades

- Cashback habilitável por toggle com suporte a 7+ providers externos.
- Exibição de cashback em PLP (card/tag), PDP, carrinho e checkout.
- Múltiplos cashbacks no mesmo checkout (`multiple_cashback_config`).
- Regras de uso configuráveis: restrição por seller, mínimo de carrinho, bloqueio com cupom.
- Extrato/histórico de transações com paginação e detalhe.
- Programa de pontos completo via Rewards/Bonifiq: metas, tiers VIP, recompensas, indique e ganhe.
- Programa de fidelidade com cadastro/login, landing page, opt-ins e seleção de loja (Propz).
- Currency balance (moedas/saldo) em perfil, carrinho e checkout.
- Integração com CRM/notificações via Propz e Dito.
- Analytics de cashback no checkout e carrinho.

## Principais limitações

- Fragmentação entre os módulos: cashback, rewards e loyalty offers operam em silos; não há carteira única unificada.
- Providers legados com implementação parcial ou stub: Kiskadi (parcial), EyeMobile (legado com TODOs e hardcodes).
- Provider VTEX não suporta extrato de transações.
- Múltiplos cashbacks (`multiple_cashback_config`) com adoção parcial.
- Validação login/SMS apenas para tipos específicos de provider.
- Regras de uso configuradas via strings (sem motor visual de regras no backoffice).
- Analytics de cashback/loyalty com cobertura parcial por fluxo.
- Orquestração CRM nativa ainda fragmentada (Dito/Propz por cliente/provider).
- Referral/Indique e Ganhe sem produto universal SaaS-first.
- Omnichannel (loja física + online) sem reconciliação padronizada no core.

---

# 2. Visão geral da integração

## O que a integração cobre

- **Cashback:** exibição de valor de cashback em PLP, PDP, carrinho e checkout; aplicação e remoção de saldo; extrato; perfil; home; FAQ; regras de uso e restrições.
- **Rewards:** pontos, metas, recompensas, tiers VIP, cashback de recompensa, histórico, autorização e aplicação/remoção no checkout.
- **Offers/Loyalty:** resumo de cashback, histórico de compras, QR redeem, catálogo de recompensas, extrato de loyalty, clube de vantagens.
- **Fidelity Program:** cadastro/login de fidelidade, landing page, opt-ins, seleção de loja.
- **Currency Balance:** exibição e uso de "moedas/saldo" em perfil, carrinho e checkout.
- **CRM/Notificações:** integração com Propz (notification service) e Dito CRM (settings/event hooks).

## Providers/serviços externos envolvidos

| Provider | Domínio | Status |
|---|---|---|
| VTEX | Cashback via gift card/orderForm | Produção (sem extrato) |
| OpenCashback | Cashback: saldo, extrato, detalhe, simulação | Produção |
| Sellbie | Cashback: saldo, extrato, uso/remoção | Produção (parcial) |
| CRM Bonus | Cashback: fluxo SMS + aplicação de bônus | Produção |
| ComprouVoltou | Cashback: saldo, extrato, futuras, ativar/desativar | Produção |
| Kiskadi (genérico) | Cashback: majoritariamente stub | Parcial/legado |
| Kiskadi Piccadilly | Cashback: integração específica por cupom | Parcial (cliente específico) |
| EyeMobile | Cashback: implementação antiga com TODOs | Legado/parcial |
| Bonifiq | Rewards: auth, customer, history, goals, tiers, rewards, cashback apply/remove | Produção |
| Propz | Offers/loyalty/fidelity: cashback, catálogo, QR redeem, notification | Produção |
| Zanthus | Offers loyalty: cashback e loyalty | Produção |
| Mercafacil | Offers: loyalty | Produção |
| Dito CRM | CRM: settings, event hooks | Produção (parcial) |

## Objetivo dentro do SaaS

- Habilitar programas de fidelização (cashback, pontos, tiers) como diferencial competitivo para clientes do SaaS.
- Permitir configuração granular por cliente via Remote Config sem desenvolvimento adicional.
- Suportar múltiplos providers externos com contrato único interno.
- Servir como base para productização de planos de loyalty (`Cashback Basic`, `Cashback Pro`, `Loyalty 360`).

---

# 3. Arquitetura e funcionamento

## Componentes principais

| Módulo | Arquivos principais | Responsabilidade |
|---|---|---|
| Cashback Core | `cashback_config_keys.dart`, `cashback_config_registrations.dart`, `cashback_bindings.dart` | Toggle, tipo, provider, configuração global |
| Cashback UI | `card_product_cashback_section.dart`, `pdp_page.dart`, `payment_choose_page.dart`, `cashback_transactions_page.dart` | Exibição em PLP, PDP, checkout, extrato |
| Offers/Loyalty | `offers_config.dart` | Configuração de modo offers e loyalty |
| Rewards | `rewards_initializer.dart` | Inicialização do módulo rewards/Bonifiq |
| Fidelity Program | `fidelity_program/config.dart` | Configuração do programa de fidelidade |
| Customizações | `customizations/*` | Variações por cliente (Atacadão, FastShop, Compra Certa, etc.) |

## Fluxo de dados — Cashback

```
Config (RC/env): cashback_enabled, cashback_type, cashback_provider
  → CashbackBindings registra datasource do provider selecionado
  → PLP: card exibe valor de cashback (lazy load, tag flags)
  → PDP: exibe cashback abaixo/à direita do preço
  → Carrinho: bloco "cashback acumulado no pedido"
  → Checkout (pagamento):
      → Consulta saldo (remaining/maximum/balanceRemaining)
      → Aplica limiter e permite escolher valor
      → Validação login/SMS (providers específicos)
      → Aplica ou remove cashback
      → Bloqueio com cupom (se configurado)
  → Extrato: tela de transações com paginação e detalhe
  → Perfil: card/atalho + widget de saldo
  → Home: card de cashback no topo (se configurado)
```

## Fluxo de dados — Rewards (Bonifiq)

```
Config: rewards_configs, RewardsApi.bonifiq
  → RewardsInitializer carrega módulo
  → Auth/customer via Bonifiq API
  → Exibição de pontos, metas, tiers e recompensas na área Rewards
  → Goals/tiers/rewards consultados via Bonifiq API
  → Cashback de recompensa: apply/remove no checkout
  → Histórico via Bonifiq history API
```

## Fluxo de dados — Offers/Loyalty

```
Config: offers_cashback_mode, regras, QR mode
  → Provider selecionado (Propz/Zanthus/Mercafacil)
  → Resumo de cashback + histórico de compras
  → QR redeem (quando habilitado)
  → Catálogo de recompensas loyalty
  → Extrato de loyalty/statement
  → Fidelity Program: cadastro/login + opt-ins + seleção de loja
```

## Dependências externas e internas

**Externas:**
- APIs dos providers: OpenCashback, Sellbie, CRM Bonus, ComprouVoltou, Kiskadi, EyeMobile, VTEX (orderForm/gift card), Bonifiq, Propz, Zanthus, Mercafacil, Dito CRM.

**Internas:**
- Remote Config: flags e parâmetros de configuração por cliente.
- `AnalyticsManager`: eventos de cashback/loyalty no checkout e carrinho.
- Módulo de carrinho e checkout: integração de aplicação/remoção de saldo.
- `AuthUserStorage`: identidade do usuário para consultas de saldo.
- Customizações por cliente em `customizations/*`.

## Papel de middleware

Não há BFF ou middleware centralizado para cashback/loyalty. A comunicação é direta entre app e APIs dos providers, orquestrada por bindings e datasources internos. A camada de `cashback_bindings.dart` seleciona o datasource correto com base no `cashback_provider` configurado.

---

# 4. Comportamento por engine

## VTEX

- Cashback via gift card/orderForm.
- Extrato de transações **não suportado** neste provider.
- Compra Certa (cliente específico): datasource VTEX customizado + wallet custom (`vtexCompraCerta`).
- FastShop (cliente específico): restrição por seller customizada + fluxo específico no checkout.

## Salesforce Commerce (SFCC)

Não há menção a provider de cashback específico para engine SFCC nas documentações fornecidas. A camada de cashback é tratada como agnóstica ao engine de commerce, com o provider sendo selecionado independentemente.

## Outras engines (Magento, etc.)

A integração de cashback/loyalty é agnóstica ao engine de commerce. A seleção de provider ocorre via `cashback_type`/`cashback_provider` no Remote Config, independentemente do engine.

## Diferenças relevantes por provider

| Provider | Extrato | Saldo | Simulação de pedido | Fluxo SMS | Multistore |
|---|---|---|---|---|---|
| VTEX | Não suportado | Sim | Não | Não | Não |
| OpenCashback | Sim | Sim | Sim | Não | Não |
| Sellbie | Sim (parcial) | Sim | Não | Não | Não |
| CRM Bonus | Não | Sim | Não | Sim | Não |
| ComprouVoltou | Sim | Sim | Não | Não | Não |
| Kiskadi | Stub/parcial | Parcial | Não | Não | Não |
| EyeMobile | Legado | Legado | Não | Não | Não |
| Bonifiq | Sim (rewards history) | Sim (pontos/tiers) | Não | Não | Não |
| Propz | Sim | Sim | Não | Não | Sim (fidelity) |

---

# 5. Features suportadas

## Cashback Core

- Toggle de habilitação: `cashback_enabled`.
- Tipo de cashback: `cashback_type`.
- Provider de cashback: `cashback_provider`.
- Suporte a múltiplos cashbacks no mesmo checkout: `multiple_cashback_config`.

## Exibição em jornada de compra

- **PLP/Card:** exibição de valor de cashback no card de produto + tag + lazy load. Configurável por `cashback_show_amount_on_card_product` e flags de tag.
- **PDP:** exibição de cashback abaixo ou à direita do preço. Configurável por `cashback_show_amount_on_pdp` e posição no `PdpConfig`.
- **Carrinho:** bloco "cashback acumulado no pedido". Configurável por `cashback_show_total_order_on_cart`.
- **Checkout (single):** expandable para consultar saldo, aplicar e remover cashback. Configurável por texto, estilo e regras.
- **Checkout (multi-provider):** múltiplos cashbacks exibidos e aplicáveis no mesmo checkout.

## Regras de uso

- Restrição por seller (ex: FastShop).
- Mínimo de valor de carrinho para uso.
- Bloqueio simultâneo com cupom: `cashback_has_coupon_blocker_active`.
- Parsing de regras via string (sem motor visual, atualmente).
- `cashback_restrictions`: configuração de restrições gerais.

## Lógica de valor aplicável

- `remaining`, `maximum`, `balanceRemaining`.
- Limiter de valor máximo aplicável.
- Interface para o usuário escolher o valor a aplicar.

## Validação de segurança

- Fluxo de validação por login antes de usar saldo.
- Fluxo de validação por SMS (providers específicos: CRM Bonus e similares).

## Extrato e histórico

- Tela de transações com paginação.
- Detalhe de transação.
- Configurável por `cashback_enable_statement` e títulos customizáveis.
- Suportado por: OpenCashback, Sellbie, ComprouVoltou e outros.

## FAQ e regras

- CTA para FAQ via WebView ou rota interna.
- Configurável por `cashback_show_faq_cta` e `cashback_faq_webview_url`.

## Perfil

- Card/atalho de cashback em Minha Conta: `show_cashback_in_profile`, logo, landing id.
- Widget de saldo no perfil: `cashback_show_balance_in_profile`.

## Home

- Card de cashback no topo da home: `show_cashback_card_in_home_screen`.

## Landing/WebView

- Redirecionamento para landing/CMS/WebView via configuração: `cashback_landing_page_id` e URLs.

## Rewards (Bonifiq)

- Pontos, metas (goals), recompensas (rewards), tiers VIP.
- Cashback de recompensa com aplicação e remoção no checkout.
- Histórico de rewards.
- Autorização e vínculo de customer via Bonifiq API.
- Indique e ganhe (referral): presente no `goal data source` Bonifiq (UX não universal no SaaS).

## Offers/Loyalty

- Resumo de cashback.
- Histórico de compras.
- QR redeem: `offers_cashback_mode` com QR mode.
- Catálogo de recompensas loyalty.
- Extrato de loyalty/statement.
- Providers: Propz, Zanthus, Mercafacil.

## Fidelity Program

- Cadastro e login de fidelidade.
- Landing page de fidelidade.
- Opt-ins de comunicação.
- Seleção de loja.
- Provider: Propz.
- Configurável por `fidelity_program_config`.

## Currency Balance

- Exibição de "moedas/saldo" em perfil, carrinho e checkout.
- Configurável por `currency_balance_*`.
- Datasource de loyalty customizado.

## CRM e Notificações

- Propz notification service: push e comunicação loyalty.
- Dito CRM: settings e event hooks.
- Configurável por `enable_propz_integration` e `dito_crm`.

## Analytics de Cashback

- `checkout_apply_cashback`: evento de aplicação no checkout.
- `checkout_remove_cashback`: evento de remoção no checkout.
- `cart_cashback_balance`: evento de saldo no carrinho.
- Cobertura parcial por fluxo via `AnalyticsManager`.

## Customizações por cliente

- **Atacadão:** onboarding próprio, ativação, subhome de benefícios, transações custom (ComprouVoltou).
- **FastShop:** restrição por seller + fluxo específico no checkout (VTEX).
- **Compra Certa:** datasource VTEX customizado + wallet custom (`vtexCompraCerta`).
- **Piccadilly:** integração Kiskadi específica por cupom.

---

# 6. Status de implementação

| Módulo / Funcionalidade | Status | Observação |
|---|---|---|
| Cashback Core (toggle/tipo/provider) | Total | Ampla presença em `customizations`; em produção |
| Cashback PLP/Card | Total | Em produção |
| Cashback PDP | Total | Em produção |
| Cashback Carrinho | Total | Em produção |
| Cashback Checkout (single) | Total | Em produção |
| Cashback Checkout (multi-provider) | Parcial | Adoção parcial por clientes |
| Regras de uso (seller, mínimo, FastShop) | Total | Em produção |
| Bloqueio com cupom | Total | Em produção |
| Valor aplicável (limiter/escolha) | Total | Em produção |
| Validação login/SMS | Parcial | Apenas para tipos específicos de provider |
| Extrato/Histórico | Total | Em produção; VTEX não suporta |
| FAQ/Regras | Total | Em produção |
| Perfil (entrada cashback) | Total | Em produção |
| Perfil (saldo) | Parcial | Parcial por cliente |
| Home (card cashback) | Parcial | Parcial por cliente |
| Landing/WebView | Total | Em produção |
| Provider VTEX | Total | Produção; extrato não suportado |
| Provider OpenCashback | Total | Em produção |
| Provider Sellbie | Parcial | Em produção (parcial) |
| Provider CRM Bonus | Total | Em produção |
| Provider ComprouVoltou | Total | Em produção |
| Provider Kiskadi (genérico) | Parcial | Majoritariamente stub; parcial/legado |
| Provider Kiskadi Piccadilly | Parcial | Cliente específico; métodos faltantes |
| Provider EyeMobile | Legado | Implementação antiga com TODOs/hardcodes |
| Atacadão custom | Total | Custom em produção |
| FastShop custom | Total | Custom em produção |
| Compra Certa custom | Parcial | Custom em produção (parcial) |
| Rewards (pontos/metas/tiers/recompensas) | Total | Em produção para clientes com RewardsApi.bonifiq |
| Bonifiq: auth/customer/history/goals/tiers | Total | Em produção |
| Bonifiq: cashback apply/remove no checkout | Total | Em produção |
| Referral/Indique e Ganhe | Parcial | Presente no goal data source Bonifiq; UX não universal |
| Offers: resumo cashback e histórico | Total | Em produção |
| Offers: QR redeem | Total | Em produção |
| Offers: catálogo loyalty e statement | Total | Em produção |
| Fidelity Program | Total | Em produção (clientes específicos) |
| Currency Balance | Parcial | Parcial por cliente |
| CRM/Notificações (Propz/Dito) | Parcial | Em produção (parcial por cliente) |
| Analytics cashback (checkout/carrinho) | Parcial | Cobertura parcial por fluxo |
| Motor visual de regras (backoffice) | Não implementado | Regras via strings atualmente |
| Carteira única unificada (cashback+rewards+loyalty) | Não implementado | Módulos operam em silos |
| Omnichannel wallet (online + loja física) | Parcial | Offers e loyalty no caixa existem; reconciliação não padronizada no core |
| Produto Referral SaaS-first | Não implementado | Dependente de provider específico hoje |

---

# 7. Uso atual (clientes)

## Padrões de uso identificados

- Cashback habilitado em ampla base de clientes com presença em `customizations` (alto nível de adoção via RC).
- Maturidade de uso desigual: muitos clientes têm `cashback_type` configurado, mas UX parcial (sem home/profile/extrato ativados).
- Rewards/Bonifiq usado por clientes específicos com `RewardsApi.bonifiq`.
- Offers/loyalty ativo em clientes com Propz, Zanthus e Mercafacil.
- Fidelity Program ativo em clientes específicos com Propz.
- Currency Balance ativo parcialmente por cliente.
- Dito CRM e Propz notification ativados parcialmente por cliente.

## Variações por cliente (customizações identificadas)

| Cliente | Customização | Provider | Status |
|---|---|---|---|
| Atacadão | Onboarding, ativação, subhome benefícios, transações custom | ComprouVoltou | Custom em produção |
| FastShop | Restrição seller custom + fluxo específico checkout | VTEX | Custom em produção |
| Compra Certa | Datasource VTEX custom + wallet custom | vtexCompraCerta | Parcial em produção |
| Piccadilly | Integração Kiskadi específica por cupom | Kiskadi | Parcial |

## Oportunidades de upsell na base atual

- Clientes com `cashback_type` configurado mas UX parcial (sem home/profile/extrato): upsell de ativação completa.
- Clientes com `offers_cashback_mode` ativo sem rewards: upsell de pontos + tiers.
- Clientes com Dito/Propz configurados sem jornadas automatizadas: upsell de réguas de recompra.

---

# 8. Parâmetros e configuração

## Parâmetros de Cashback Core

| Parâmetro | Tipo | Descrição |
|---|---|---|
| `cashback_enabled` | bool | Toggle global de habilitação |
| `cashback_type` | string | Tipo de cashback (define comportamento) |
| `cashback_provider` | string | Provider externo selecionado |
| `multiple_cashback_config` | config | Configuração de múltiplos cashbacks no checkout |
| `cashback_restrictions` | string/config | Regras de restrição (seller, mínimo, etc.) |
| `cashback_has_coupon_blocker_active` | bool | Bloqueia uso simultâneo com cupom |

## Parâmetros de exibição

| Parâmetro | Tela | Descrição |
|---|---|---|
| `cashback_show_amount_on_card_product` | PLP | Exibe valor no card de produto |
| `cashback_show_amount_on_pdp` | PDP | Exibe valor na PDP |
| `cashback_show_total_order_on_cart` | Carrinho | Exibe total acumulado no carrinho |

## Parâmetros de extrato e FAQ

| Parâmetro | Descrição |
|---|---|
| `cashback_enable_statement` | Habilita tela de extrato |
| `cashback_show_faq_cta` | Exibe CTA de FAQ |
| `cashback_faq_webview_url` | URL do FAQ via WebView |

## Parâmetros de perfil e home

| Parâmetro | Descrição |
|---|---|
| `show_cashback_in_profile` | Exibe card/atalho de cashback no perfil |
| `cashback_show_balance_in_profile` | Exibe widget de saldo no perfil |
| `show_cashback_card_in_home_screen` | Exibe card de cashback na home |
| `cashback_landing_page_id` | ID da landing page de cashback |

## Parâmetros de Rewards

| Parâmetro | Descrição |
|---|---|
| `rewards_configs` | Configuração geral do módulo rewards |
| `RewardsApi.bonifiq` | Habilita integração Bonifiq no módulo rewards |

## Parâmetros de Offers/Loyalty

| Parâmetro | Descrição |
|---|---|
| `offers_cashback_mode` | Modo de cashback no módulo offers |
| Regras de QR mode | Configuração de resgate por QR code |

## Parâmetros de Fidelity Program

| Parâmetro | Descrição |
|---|---|
| `fidelity_program_config` | Configuração geral do programa de fidelidade |

## Parâmetros de Currency Balance

| Parâmetro | Descrição |
|---|---|
| `currency_balance_*` | Configuração de exibição e uso de moedas/saldo |

## Parâmetros de CRM/Notificações

| Parâmetro | Descrição |
|---|---|
| `enable_propz_integration` | Habilita integração Propz |
| `dito_crm` | Configuração de settings e event hooks do Dito CRM |

---

# 9. Providers e strategies

## Estratégias de seleção de provider

A seleção do provider de cashback é feita via `cashback_provider` no Remote Config, com `cashback_bindings.dart` registrando o datasource correspondente. Cada provider implementa o contrato interno de cashback (saldo, aplicar, remover, extrato, simulação).

## Providers de Cashback

| Provider | Estratégia de integração | Capacidades |
|---|---|---|
| VTEX | Gift card / orderForm | Saldo, aplicar, remover; sem extrato |
| OpenCashback | API direta | Saldo, extrato, detalhe, simulação de pedido |
| Sellbie | API direta | Saldo, extrato, uso/remoção (comportamentos restritos) |
| CRM Bonus | API + fluxo SMS | Saldo, aplicação de bônus, validação SMS |
| ComprouVoltou | API direta | Saldo, extrato, futuras, ativar/desativar |
| Kiskadi (genérico) | Stub/parcial | Implementação majoritariamente incompleta |
| Kiskadi Piccadilly | API + cupom específico | Integração por cupom; métodos faltantes |
| EyeMobile | API legada | Implementação antiga com TODOs/hardcodes; baixa confiança |

## Providers de Rewards

| Provider | Estratégia | Capacidades |
|---|---|---|
| Bonifiq | API dedicada | Auth, customer, history, goals, tiers, rewards, cashback apply/remove |

## Providers de Offers/Loyalty

| Provider | Capacidades |
|---|---|
| Propz | Cashback, catálogo, QR redeem, loyalty statement, fidelity, notification |
| Zanthus | Cashback e loyalty |
| Mercafacil | Loyalty |

## Providers de CRM

| Provider | Capacidades |
|---|---|
| Propz | Notification service, push loyalty |
| Dito CRM | Settings, event hooks |

## Estratégia multi-provider (multiple_cashback_config)

- Permite múltiplos providers de cashback ativos no mesmo checkout.
- Adoção atual parcial (apenas alguns clientes).
- Configuração por provider no `multiple_cashback_config`.

---

# 10. Gaps e limitações

## Limitações técnicas

- **Provider VTEX:** extrato de transações não suportado. Clientes que usam VTEX não têm histórico de cashback disponível.
- **Provider Kiskadi (genérico):** implementação majoritariamente stub; métodos críticos não implementados.
- **Provider EyeMobile:** implementação legada com TODOs e notas de hardcode; não recomendado para novos projetos.
- **Kiskadi Piccadilly:** integração específica por cupom com métodos faltantes; não generalizado.
- **Validação login/SMS:** disponível apenas para providers específicos (CRM Bonus e similares); não universal.
- **Multiple cashback config:** adoção parcial; não há cobertura completa de todos os cenários de multi-provider.
- **Analytics de cashback:** cobertura parcial por fluxo; eventos como `view_balance`, `earn`, `error` sem taxonomia padronizada.

## Limitações de arquitetura

- **Fragmentação entre módulos:** cashback, rewards e loyalty offers operam como silos independentes. Não há carteira única unificada do usuário.
- **Regras de uso via strings:** `cashback_restrictions` é parseado por string, sem motor visual de regras no backoffice. Dependência de desenvolvimento para alterar regras.
- **Orquestração CRM fragmentada:** Dito e Propz configurados por cliente/provider sem orquestrador central de campanhas e réguas.
- **Referral/Indique e Ganhe:** presente no goal data source Bonifiq, mas sem produto universal SaaS-first; experiência não padronizada.
- **Omnichannel wallet:** offers e loyalty no caixa existem, mas reconciliação de saldo entre canal online e loja física não está padronizada no core.
- **Tiers VIP:** disponíveis via Rewards/Bonifiq, mas sem camada SaaS genérica independente de provider.
- **Gamificação/missões:** goals e rewards existem via Bonifiq, mas missões gamificadas transversais não estão no core cashback.
- **Widget loyalty plugável:** widgets de cashback e offers existem separados; não há widget único white-label com SDK web/app.

## Dependências externas

- Disponibilidade das APIs de cada provider (OpenCashback, Sellbie, CRM Bonus, ComprouVoltou, Bonifiq, Propz, Zanthus, Mercafacil, Dito).
- Contratos de API por provider podem divergir e exigir customização adicional.
- Configuração correta de Remote Config por cliente para habilitação de funcionalidades.

---

# 11. Impacto e riscos

## Riscos técnicos

- **Providers legados (Kiskadi/EyeMobile):** risco de falhas silenciosas e comportamento imprevisível. Devem ser sinalizados como não recomendados para novos projetos.
- **Regras de uso via strings:** qualquer alteração de regra de negócio exige desenvolvimento; risco de regressão e erro de parsing.
- **Extrato VTEX indisponível:** clientes VTEX não têm visibilidade de histórico de cashback, impactando confiança do consumidor no programa.
- **Multiple cashback config:** cobertura incompleta de cenários pode gerar comportamento inesperado em clientes com múltiplos providers.
- **Fragmentação de analytics:** taxonomia inconsistente dificulta análise consolidada de performance do programa de cashback/loyalty.

## Riscos de negócio

- **Maturidade desigual por cliente:** muitos clientes têm cashback habilitado mas UX incompleta (sem home/profile/extrato), reduzindo a percepção de valor do programa.
- **Produtização insuficiente:** alto custo de customização por cliente sem templates e playbooks padronizados.
- **Orquestração CRM fragmentada:** dificuldade de escalar jornadas de recompra e fidelização sem orquestrador central.
- **Referral sem produto universal:** impossibilidade de oferecer "Indique e Ganhe" como feature padrão SaaS sem depender de provider externo específico.

## Avaliação de maturidade

- **Cashback:** 7.5/10 (forte, com ~35 macro-funcionalidades; gaps em regras, analytics e carteira unificada).
- **Rewards:** intermediário (produção para clientes Bonifiq; sem abstração genérica multi-provider).
- **Offers/Loyalty:** intermediário (em produção; sem padronização omnichannel e orquestração CRM centralizada).

---

# 12. Requisitos para completar

## Quick wins (baixo esforço / alto impacto)

1. **Padronizar taxonomia de eventos analytics:** criar eventos canônicos `view_balance`, `apply`, `remove`, `error`, `earn` no `AnalyticsManager` para cashback/loyalty, eliminando gaps de cobertura.
2. **Expor matriz de regras no backoffice:** substituir parsing por strings de `cashback_restrictions` por configuração visual (cupom, mínimo, expiração, acumulável) sem depender de desenvolvimento.
3. **Playbook de configuração:** ativar UX padrão de extrato/FAQ/profile/home para todos os clientes com `cashback_enabled=true`, reduzindo maturidade desigual.

## Médio prazo

1. **Cashback Rule Engine visual:** motor de regras configurável por segmento, first purchase, recorrência, progressivo e combinação com cupom.
2. **Carteira única do usuário:** unificar `cashback + rewards + loyalty offers` em uma conta/saldo único na experiência do consumidor.
3. **Produto Referral SaaS-first:** produto de "Indique e Ganhe" independente de provider específico, nativo no SaaS base.
4. **Tiers VIP genéricos:** camada SaaS de tiers/níveis não dependente exclusivamente do Bonifiq.

## Diferenciais premium

1. **Omnichannel wallet:** reconciliação padronizada de saldo entre e-commerce, loja física (PDV) e app em wallet único.
2. **Orquestrador de campanhas e réguas:** conectores nativos Dito/Propz/CRM próprio com configuração de jornadas por comportamento.
3. **Pacote enterprise:** tiers VIP + gamificação + campanhas dinâmicas por segmento.

## Hardening de providers legados

- Avaliar e deprecar EyeMobile: remover TODOs e hardcodes ou implementar completamente.
- Completar implementação do Kiskadi genérico ou documentar explicitamente como não suportado para novos clientes.
- Adicionar suporte a extrato no provider VTEX (avaliar viabilidade de API).

## Evoluções possíveis (receita incremental)

1. Plano add-on `Cashback Pro`: regras avançadas + analytics + campanhas.
2. Add-on `Loyalty Omnichannel`: PDV + e-commerce + app wallet.
3. Serviços de migração/otimização de programa legado para Kobe Loyalty Core.
4. Modelo de cobrança por capacidade: regras, campanhas, canais, MAU/volume de pedidos.

---

# 13. Critérios de avaliação para agente de IA

## Pré-requisitos

- `cashback_enabled=true` na configuração do cliente.
- `cashback_type` e `cashback_provider` definidos e compatíveis.
- APIs do provider configuradas e acessíveis (credenciais, endpoints).
- Para Rewards: `RewardsApi.bonifiq` habilitado e `rewards_configs` preenchido.
- Para Offers/Loyalty: `offers_cashback_mode` e provider configurados.
- Para Fidelity Program: `fidelity_program_config` preenchido e provider Propz ativo.
- Para Currency Balance: `currency_balance_*` configurados.

## Checks funcionais

- Card de produto exibe valor de cashback corretamente na PLP (lazy load sem bloqueio).
- PDP exibe cashback na posição configurada (abaixo/à direita do preço).
- Carrinho exibe bloco de cashback acumulado.
- Checkout exibe saldo disponível, permite aplicar e remover cashback.
- Regras de restrição (mínimo de carrinho, seller, cupom) aplicadas corretamente.
- Extrato exibe transações com paginação e detalhe (verificar se provider suporta).
- Perfil exibe card/atalho e widget de saldo (verificar flags).
- Home exibe card de cashback (verificar flag `show_cashback_card_in_home_screen`).
- Rewards: pontos, metas, tiers e recompensas exibidos corretamente; apply/remove no checkout funcional.
- Offers: QR redeem funcional; catálogo e extrato loyalty carregados.
- Analytics: eventos `checkout_apply_cashback` e `checkout_remove_cashback` disparados.

## Sinais de problema

- Saldo não carregado no checkout: verificar credenciais e disponibilidade da API do provider.
- Cashback não exibido na PLP: verificar `cashback_show_amount_on_card_product` e provider configurado.
- Extrato vazio sem erro: verificar se provider suporta extrato (VTEX não suporta).
- Regras de restrição não aplicadas: verificar `cashback_restrictions` e parsing correto.
- Cupom e cashback aplicados simultaneamente: verificar `cashback_has_coupon_blocker_active`.
- Provider Kiskadi ou EyeMobile com comportamento imprevisível: tratar como legado, não usar em novos projetos.
- Multiple cashback sem exibição correta: verificar `multiple_cashback_config` e adoção do provider.
- Rewards não carregado: verificar `RewardsApi.bonifiq` habilitado e `PersonalizationService` equivalente registrado no startup.
- Analytics ausente após apply/remove: verificar cobertura no `AnalyticsManager`.

---

# Referências técnicas

## Arquivos de código referenciados

| Arquivo | Responsabilidade |
|---|---|
| `cashback_config_keys.dart` | Chaves de configuração do módulo cashback |
| `cashback_config_registrations.dart` | Registro de configurações de cashback |
| `cashback_bindings.dart` | Binding de provider de cashback via DI |
| `payment_choose_page.dart` | Exibição e interação de cashback no checkout |
| `card_product_cashback_section.dart` | Exibição de cashback no card de produto (PLP) |
| `pdp_page.dart` | Exibição de cashback na PDP |
| `cashback_transactions_page.dart` | Tela de extrato/histórico de transações |
| `offers_config.dart` | Configuração do módulo offers/loyalty |
| `rewards_initializer.dart` | Inicialização do módulo rewards/Bonifiq |
| `fidelity_program/config.dart` | Configuração do programa de fidelidade |
| `customizations/atacadao/*` | Customizações Atacadão (ComprouVoltou) |
| `customizations/fast_shop/*` | Customizações FastShop (VTEX, restrição seller) |
| `customizations/compra_certa/*` | Customizações Compra Certa (vtexCompraCerta) |
| `customizations/piccadilly/*` | Customizações Piccadilly (Kiskadi por cupom) |

## Módulos do repositório

| Módulo | Caminho esperado |
|---|---|
| cashback | `app/lib/modules/cashback/` |
| rewards | `app/lib/modules/rewards/` (ou similar) |
| offers | `app/lib/modules/offers/` (ou similar) |
| fidelity_program | `app/lib/modules/fidelity_program/` |
| currency_balance | `app/lib/modules/currency_balance/` (ou similar) |
| custom_modules | `app/lib/modules/custom_modules/` |
| initializers | `app/lib/initializers/` |
| customizations | `app/customizations/` |

## Providers e APIs externas

| Provider | Tipo de integração |
|---|---|
| VTEX | Gift card / orderForm API |
| OpenCashback | API REST dedicada |
| Sellbie | API REST dedicada |
| CRM Bonus | API REST + SMS validation |
| ComprouVoltou | API REST dedicada |
| Kiskadi | API REST (parcialmente implementada) |
| EyeMobile | API legada (implementação com TODOs) |
| Bonifiq | API REST dedicada (rewards/goals/tiers/cashback) |
| Propz | API REST (offers/loyalty/fidelity/notification) |
| Zanthus | API REST (offers/loyalty) |
| Mercafacil | API REST (loyalty) |
| Dito CRM | API REST (settings/event hooks) |

## Analytics events identificados

| Evento | Tela | Status |
|---|---|---|
| `checkout_apply_cashback` | Checkout | Em produção |
| `checkout_remove_cashback` | Checkout | Em produção |
| `cart_cashback_balance` | Carrinho | Em produção |
| `view_balance` | Extrato/Perfil | Não padronizado (gap) |
| `earn` | PLP/PDP/Checkout | Não padronizado (gap) |
| `error` (cashback) | Checkout | Não padronizado (gap) |
