# Arquitetura — Dashboard Pri Riciardi

Referência técnica do painel. A aplicação inteira é **um único arquivo** (`Dashboard.dc.html`): template + lógica + estilos inline, renderizado pelo runtime `support.js`. Sem build, sem bundler, sem backend.

---

## 1. Estrutura do arquivo

`Dashboard.dc.html` tem três partes:

1. **`<helmet>`** — carrega os tokens do design system (`_ds/l-marques-…`), o Chart.js (CDN) e os `@font-face`/resets globais.
2. **Template** (`<x-dc>…</x-dc>`) — markup das 4 páginas (conta, conteúdos, insights, campanhas), com estilos inline e holes `{{ … }}`.
3. **Classe `Component extends DCLogic`** — todo o estado, fetch de dados e cálculo dos valores que o template consome (`renderVals()`/métodos `*Vals`/`*Info`).

---

## 2. Estado e navegação

- `page` — qual das 4 páginas está visível (`conta`, `conteudos`, `insights`, `candidaturas`).
- `campaign` — qual aba de produto está ativa na página de campanhas (`wa`, `reset`, `ciclo`, `essencia`, `seguidores`, `engajamento`).
- `candMonth` / `candFrom` / `candTo` — filtro de data da página de campanhas. O `to` default acompanha a data de atualização do relatório (`lastSyncYMD`).
- `month` / `week` — filtros das páginas de conta/conteúdos.
- `sales`, `adRows`, `demo` — dados carregados (vendas, anúncios, demografia).

---

## 3. Fontes de dados

### Instagram (Windsor.ai)
`fetchLive()` busca perfil, série diária (views, alcance, interações), saldo de seguidores e publicações. `fetchDemographics()` traz idade/gênero/cidade (cache de 7 dias).

### Facebook Ads (Windsor.ai)
`fetchAds()` busca a conta `10101511135973327` e atribui cada linha a um produto por regex no nome da campanha (`ADS_CAMPAIGN_MATCH`). Semeia desde mai/2025 em cache (`priAdsCampaigns:v1`) e depois revalida só os dias recentes.

### Vendas (Eduzz)
`fetchSales()` lê uma planilha pública por mês (`SALES_SHEETS`) via `gviz`. Cada linha vira `{ fatura, status, date, item, net, tab, utm, cupom, oferta, utmCamp, utmCont }`. O produto (`tab`) é resolvido pelo ID Eduzz (`SALES_PROD_TAB`). Histórico em cache (`priSales:v6`); mês corrente sempre ao vivo.

---

## 4. Cálculos principais (página de campanhas)

- **Intervalo de datas / meses** — derivados dos próprios dados carregados (anúncios + vendas), não de planilha externa.
- **Vendas** (`salesInfo`) — faturamento e nº de vendas pagas no range/produto/origem; ticket, ROAS (faturamento ÷ investimento), custo por venda. Donuts de origem por UTM Source, UTM Campaign e UTM Content.
- **Investimento** (`investFunnel`) — funil Alcance → Impressões → Cliques → Visualizações da página → Leads, com CPM, CPC, connect rate e custo/lead; tabelas de anúncios e conjuntos.
- **Novos Seguidores** (`followerFunnel`) — turbinamento de publicações (`SEGUIDORES`): investimento → visitas ao perfil → seguidores, com rosca de distribuição do orçamento.

---

## 5. Cache (localStorage)

| Chave | Conteúdo |
|---|---|
| `dbi_posts_cache_<handle>` | Publicações + capas embutidas |
| `priAdsCampaigns:v1` | Linhas de anúncios por produto |
| `priSales:v6` | Vendas Eduzz (histórico) |
| `igdemo_<handle>` | Demografia (7 dias) |

`pruneCaches()` roda na carga e remove versões antigas / caches de contas anteriores, evitando estouro de quota (~5 MB).

---

## 6. Limitações conhecidas

- A data final dos filtros vai até o último dia sincronizado pelos conectores/planilhas.
- A chave da Windsor.ai está embutida no cliente (painel privado).
- As capas do Instagram só persistem porque são baixadas e embutidas; URLs assinadas originais expiram.
