**Escopo e base usada**
Mapeamento feito no codebase local (módulos `cashback`, `rewards`, `offers`, `fidelity_program`, `currency_balance`, `custom_modules`, `initializers`, `customizations/*`) + benchmark público BonifiQ.

Evidências principais no repo:
- [cashback_config_keys.dart](/Users/franciellestival/Desktop/merx/app/lib/modules/cashback/config/cashback_config_keys.dart)
- [cashback_config_registrations.dart](/Users/franciellestival/Desktop/merx/app/lib/modules/cashback/config/cashback_config_registrations.dart)
- [cashback_bindings.dart](/Users/franciellestival/Desktop/merx/app/lib/modules/cashback/cashback_bindings.dart)
- [payment_choose_page.dart](/Users/franciellestival/Desktop/merx/app/lib/modules/cart/pages/cart_payment/payment_choose_page.dart)
- [card_product_cashback_section.dart](/Users/franciellestival/Desktop/merx/app/lib/modules/design_system/components/card_product/widgets/card_product_cashback_section.dart)
- [pdp_page.dart](/Users/franciellestival/Desktop/merx/app/lib/modules/pdp/pages/pdp_page.dart)
- [cashback_transactions_page.dart](/Users/franciellestival/Desktop/merx/app/lib/modules/cashback/presentation/page/cashback_transactions_page.dart)
- [offers_config.dart](/Users/franciellestival/Desktop/merx/app/lib/modules/config/custom_components/offers_config.dart)
- [rewards_initializer.dart](/Users/franciellestival/Desktop/merx/app/lib/initializers/rewards_initializer.dart)
- [config.dart](/Users/franciellestival/Desktop/merx/app/lib/modules/fidelity_program/config.dart)

---

## Etapa 1 + 2 — Mapeamento interno Kobe (cashback/loyalty/saldo/créditos)

| Módulo | Funcionalidade | Onde aparece | Cliente usa hoje? | Configurável? | Provider | Status |
|---|---|---|---|---|---|---|
| Cashback Core | Toggle + tipo + provider (`cashback_enabled`, `cashback_type`, `cashback_provider`) | Global | Sim (ampla presença em `customizations`) | Sim (RC + env) | vtex/openCashback/crmBonus/sellbie/comprouVoltou/kiskadi/eyeMobile | Em produção |
| Cashback PLP/Card | Exibição de valor de cashback no card + tag + lazy load | Lista de produtos | Sim | Sim (`cashback_show_amount_on_card_product`, tag flags) | Principalmente VTEX/OpenCashback | Em produção |
| Cashback PDP | Exibição cashback na PDP (abaixo/à direita do preço) | PDP | Sim | Sim (`cashback_show_amount_on_pdp`, posição no PdpConfig) | Idem | Em produção |
| Cashback Cart | Bloco “cashback acumulado no pedido” | Carrinho | Sim | Sim (`cashback_show_total_order_on_cart`) | VTEX/OpenCashback | Em produção |
| Cashback Checkout (single) | Expandable para consultar saldo/aplicar/remover | Pagamento checkout | Sim | Sim (texto, estilo, regras) | Todos os tipos suportados | Em produção |
| Cashback Checkout (multi-provider) | Vários cashbacks no mesmo checkout (`multiple_cashback_config`) | Pagamento checkout | Parcial (alguns clientes) | Sim (por provider) | Multi | Em produção (parcial adoção) |
| Regras de uso | Restrição por seller, mínimo de carrinho, FastShop; parsing por string | Checkout/uso | Sim | Sim (`cashback_restrictions`) | Engine interna | Em produção |
| Bloqueio com cupom | Impede uso simultâneo em cenários configurados | Checkout | Sim | Sim (`cashback_has_coupon_blocker_active`) | Engine interna | Em produção |
| Valor aplicável | `remaining/maximum/balanceRemaining` + limiter + escolher valor | Checkout | Sim | Sim | Engine interna | Em produção |
| Validação login/SMS | Fluxo de segurança antes de usar saldo | Checkout | Parcial (tipos específicos) | Parcial (por provider/tipo) | crmBonus etc | Em produção |
| Extrato/Histórico | Tela de transações com paginação + detalhe de transação | Carteira/benefícios | Sim | Sim (`cashback_enable_statement`, títulos) | OpenCashback/Sellbie/ComprouVoltou etc | Em produção |
| FAQ/Regras | CTA para FAQ por WebView ou rota FAQ interna | Extrato/gadget | Sim | Sim (`cashback_show_faq_cta`, `cashback_faq_webview_url`) | N/A | Em produção |
| Perfil (entrada cashback) | Card/atalho de cashback em Minha Conta | Perfil | Sim | Sim (`show_cashback_in_profile`, logo, landing id) | N/A | Em produção |
| Perfil (saldo) | Widget de saldo no perfil | Perfil | Parcial por cliente | Sim (`cashback_show_balance_in_profile`) | N/A | Em produção |
| Home | Card de cashback no topo home | Home | Parcial por cliente | Sim (`show_cashback_card_in_home_screen`) | N/A | Em produção |
| Landing/WebView | Redireciono para landing/cms/webview via config | Perfil/CTA | Sim | Sim (`cashback_landing_page_id`, urls) | N/A | Em produção |
| Atacadão custom | Onboarding, ativação, subhome própria, transações custom | Jornada benefícios | Sim (cliente específico) | Parcial | comprouVoltou | Custom em produção |
| FastShop custom | Restrição seller custom + fluxo específico | Checkout | Sim (cliente específico) | Parcial | vtex | Custom em produção |
| Compra Certa custom | Data source VTEX + wallet custom | Checkout/extrato | Sim (cliente específico) | Parcial | vtexCompraCerta | Custom em produção |
| Provider VTEX | Uso/remoção via gift card/orderForm; extrato não suportado | Checkout/cart | Sim | Sim | VTEX | Produção com limitações |
| Provider OpenCashback | Saldo, extrato, detalhe, simulação de pedido | Checkout/wallet | Sim | Sim | OpenCashback | Em produção |
| Provider Sellbie | Saldo, extrato, uso/remoção; alguns comportamentos restritos | Checkout/wallet | Sim (alguns clientes) | Sim | Sellbie | Em produção (parcial) |
| Provider CRM Bonus | Fluxo SMS + aplicação de bônus | Checkout | Sim (alguns clientes) | Sim | CRM Bonus | Em produção |
| Provider ComprouVoltou | Saldo, extrato, futuras, ativar/desativar cashback | Wallet/checkout | Sim (cliente específico) | Sim | ComprouVoltou | Em produção |
| Provider Kiskadi (genérico) | Implementação majoritariamente stub | N/A | Não evidenciado como uso massivo | Sim | Kiskadi | Parcial/legado |
| Provider Kiskadi Piccadilly | Integração específica por cupom + métodos faltantes | Cliente específico | Sim (Piccadilly) | Parcial | Kiskadi | Parcial |
| Provider EyeMobile | Implementação antiga com TODO/hardcoded note | N/A | Baixa evidência de uso atual | Baixa | EyeMobile | Legado/parcial |
| Rewards module | Pontos, metas, recompensas, tiers, cashback de recompensa | Área Rewards | Sim (quando `RewardsApi.bonifiq`) | Sim (`rewards_configs`) | Bonifiq | Em produção |
| Bonifiq custom integration | Auth/customer/history/goals/tiers/rewards/cashback apply-remove | Rewards + checkout | Sim (clientes com rewards bonifiq) | Sim | Bonifiq | Em produção |
| Offers loyalty/cashback | Resumo cashback, histórico compras, QR redeem, loyalty catalog/statement | Offers/club | Sim | Sim (`offers_cashback_mode`, regras, QR mode) | Propz/Zanthus/Mercafacil | Em produção |
| Fidelity Program | Cadastro/login fidelidade, LP, opt-ins, store selection | Perfil/LP/PLP | Sim (clientes específicos) | Sim (`fidelity_program_config`) | Propz | Em produção |
| Currency Balance | “Moedas/saldo” em perfil/carrinho/checkout | Perfil/cart/checkout | Parcial por cliente | Sim (`currency_balance_*`) | custom loyalty datasource | Em produção |
| CRM/Notificações | Propz notification service + Dito CRM settings/event hooks | Push/CRM | Parcial por cliente | Sim (`enable_propz_integration`, `dito_crm`) | Propz/Dito | Em produção (parcial) |
| Analytics Cashback | `checkout_apply_cashback`, `checkout_remove_cashback`, `cart_cashback_balance`, etc. | Cart/checkout/extrato | Sim | Parcial (por fluxo) | AnalyticsManager | Em produção |

Observações de status técnico:
- **Parcial/legado explícito** em `kiskadi_*`, `eyemobile_*`, e partes de `vtex_cashback_data_source` (ex.: transações não suportadas).
- Há **feature flag escondida/menos óbvia** de múltiplos cashbacks (`multiple_cashback_config`) e diversos toggles finos de UX.

---

## Etapa 3 e 4 — Kobe vs BonifiQ (benchmark estratégico)

Fontes públicas BonifiQ usadas:
- https://www.bonifiq.com.br/contato/
- https://suporte2.bonifiq.com.br/support/solutions

| Funcionalidade | Kobe já possui | Bonifiq possui | Gap Kobe | Complexidade estimada | Prioridade |
|---|---|---|---|---|---|
| Cashback simples e aplicação no checkout | Sim (forte) | Sim | Sem gap relevante | Baixa | Média |
| Cashback + pontos combinados | Parcial (cashback + rewards separados por módulos) | Sim (pontos↔cashback) | Unificar experiência/conta/saldo em um “motor único” | Alta | Alta |
| Programa de pontos completo | Sim (rewards/offers loyalty) | Sim | Padronizar cross-cliente e reduzir customizações | Média | Alta |
| Indique e ganhe | Parcial (Bonifiq goal data source tem refer friend; UX não universal) | Sim | Produto único “referral” no SaaS base | Média | Alta |
| VIP/Níveis | Parcial (tiers no rewards bonifiq) | Sim | Falta camada SaaS genérica (não só via Bonifiq) | Média | Alta |
| Gamificação/metas | Parcial (goals/rewards) | Sim | Missões gamificadas transversais no core cashback | Média | Média |
| Comunicação personalizada loyalty | Parcial (Dito/Propz + offers/rewards) | Sim (comunicações/régua/widget) | Orquestração CRM nativa Kobe ainda fragmentada | Alta | Alta |
| Widget loyalty plugável | Parcial (cashback widgets + offers widgets) | Sim | Widget único white-label com SDK web/app | Média | Alta |
| Omnichannel loja física + online | Parcial (offers, loyalty no caixa, integrações diversas) | Sim | Padronizar reconciliação saldo omnichannel no core | Alta | Alta |
| Campanhas segmentadas | Parcial (muito por provider/cliente) | Sim | Segmentação nativa no backoffice Kobe | Alta | Alta |
| White label forte | Sim (muito configurável por RC) | Sim | Melhorar governança e templates prontos | Média | Média |
| Programas customizados por cliente | Sim (alto nível de custom module) | Sim | Reduzir custo de custom via “productização” | Alta | Alta |

---

## Etapa 5 — Gaps estratégicos

### Quick wins (baixo esforço / alto impacto)
1. Padronizar eventos analytics cashback/loyalty com taxonomia única (`view_balance`, `apply`, `remove`, `error`, `earn`).
2. Expor no backoffice “matriz de regras” (cupom, mínimo, expiração, acumulável) sem depender de strings.
3. Ativar UX padrão de extrato/FAQ/profile/home para clientes com `cashback_enabled=true` (playbook de configuração).

### Médio prazo
1. Criar **Cashback Rule Engine** visual (segmento, first purchase, recorrência, progressivo, cupom combinável).
2. Unificar `cashback + rewards + loyalty offers` em uma carteira única do usuário.
3. Produto “Referral/Indique e Ganhe” SaaS-first (sem depender de provider específico).

### Diferenciais premium
1. Omnichannel wallet único (ecommerce + loja física + app).
2. Orquestrador de campanhas e réguas baseado em comportamento (com conectores Dito/Propz/CRM próprio).
3. Pacote enterprise: tiers VIP + gamificação + campanhas dinâmicas por segmento.

### Receita incremental possível
1. Novo plano add-on: `Cashback Pro` (regras avançadas + analytics + campanhas).
2. Add-on `Loyalty Omnichannel` (PDV + ecommerce + app wallet).
3. Serviços de migração/otimização de programa legado para Kobe Loyalty Core.

### Oportunidades comerciais imediatas
1. Base que já tem `cashback_type` mas UX parcial (sem home/profile/extrato) -> upsell de ativação completa.
2. Clientes com `offers_cashback_mode` ativo sem rewards -> upsell “points + tiers”.
3. Clientes com Dito/Propz configurados -> upsell de jornadas de recompra automatizadas.

---

## Etapa 6 — Visão executiva final

1. **Nível atual Kobe em Cashback (0 a 10): 7.5/10**
2. **Funcionalidades já existentes (macro): ~35**
3. **Gaps críticos: 10**
4. **Oportunidade de upsell na base atual: alta** (muitos tenants com chaves/flags de cashback no RC, mas maturidade desigual por cliente)
5. **Como transformar em linha de receita SaaS**
   1. Empacotar em tiers (`Cashback Basic`, `Cashback Pro`, `Loyalty 360`).
   2. Cobrar por capacidade (regras, campanhas, canais, MAU/order volume).
   3. Oferta de ativação rápida por playbook + migração de programa legado.
   4. Medir e vender por resultado (recompra, frequência, ticket, breakage, margem promocional).

Se quiser, no próximo passo eu te entrego uma versão “board-ready” em 1 página (resumo executivo + roadmap 90 dias + pricing framework).
