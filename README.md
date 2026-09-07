# Dashboard — Dr. Anderson Rodrigues (Meta Ads + Google Ads)

Painel de performance de mídia paga do **Dr. Anderson Rodrigues**, no padrão visual da
Agência B16. Mesma base do dashboard do Matheus Godinho, com uma aba a mais para o
Google Ads.

🔗 https://suporteb16-collab.github.io/dashboard-dr-anderson/

---

## As duas abas

**Google Ads abre primeiro** — é a frente principal do cliente hoje.

| Aba | Fonte | Janela de dado |
|---|---|---|
| **Google Ads** | `public.dash_google_ads`, filtrada em `cliente=Dr. Anderson Rodrigues` | 03/09/2026 → 07/09/2026 |
| **Meta Ads** | `public.dash_dranderson_midia` → `"trafego-pago".meta_ads_drandersonr` | 01/01/2026 → 03/08/2026 |

As duas contas veicularam em períodos **diferentes e sem sobreposição**, por isso cada
aba resolve o filtro de período contra as próprias datas e o subtítulo de cada aba mostra
o intervalo real que ela cobre. O período padrão é o **mês atual** — no Google isso traz
os 5 dias existentes; no Meta, que parou em agosto, a aba abre com "sem dado no período"
e um aviso apontando o botão **Tudo**, em vez de parecer quebrada.

---

## O resultado desta conta é conversa, não venda

Este é o ponto que diferencia o painel do Matheus. A conta de Meta do Dr. Anderson
**não roda pixel de compra**: as campanhas são de tráfego para o perfil, engajamento,
seguidores e WhatsApp. `fb_pixel_purchase`, `initiate_checkout` e `landing_page_view`
são **0 em todas as 1.581 linhas** — não é falha de ingestão, é o objetivo da campanha.

O evento que realmente mede resultado aqui é
`onsite_conversion.messaging_conversation_started_7d` (conversa iniciada no
WhatsApp/Direct). No total do histórico: **140 conversas, 89 respostas ao primeiro
contato e R$ 74,75 de custo por conversa**.

`actions.lead` também foi testado e é **0** — não há formulário de lead nesta conta, então
não adianta procurar lead por ali.

Por isso o funil desta aba é `Investimento → Impressões → Cliques → Conversas iniciadas`,
e não o funil de e-commerce do dashboard do Matheus.

### O trigger do Stract precisou ser corrigido

A extração `Meta Ads - Dr. Anderson Rodrigues` (id `1788608330`) tinha sido criada
copiando os campos da conta do Matheus, que é e-commerce — ou seja, puxava só eventos de
pixel de compra, todos zerados nesta conta. Foram acrescentados três campos:

- `actions.onsite_conversion.messaging_conversation_started_7d` → `conversas`
- `actions.onsite_conversion.messaging_first_reply` → `respostas`
- `actions.post_engagement` → `engajamentos`

As colunas correspondentes foram criadas na tabela e **carregadas com o histórico
completo** (01/01 a 03/08) a partir da API do Meta, batendo 1.581/1.581 linhas. As
próximas cargas do Stract passam a alimentá-las sozinhas.

---

## Fonte de dados

Tudo vem do **Supabase** (projeto `Data&Revenue`), por PostgREST, com a chave publishable.

### Por que views e não as tabelas direto

O schema `trafego-pago` **não é publicado no PostgREST** (só `public` e `graphql_public`).
A view em `public` é a ponte, com `security_definer` — é o que permite o anon ler a view
sem ter `select` na tabela do schema não publicado. Aparece no linter do Supabase como
`security_definer_view` (nível ERROR) e **é intencional**, igual às views do Maestro e do
Google Ads. Nenhuma das duas tabelas tem dado pessoal, então as views são projeções
diretas, sem mascaramento.

### Duas armadilhas herdadas do dashboard de Google Ads

1. **`metrics_cost_micros` não está em micros.** O nome sugere valor bruto (dividir por
   1.000.000), mas o Stract já converte para reais antes de gravar. **Não dividir de
   novo** — a view expõe a coluna como `investimento` sem tocar no valor.
2. **Paginação.** `dash_google_ads` tem ~11 mil linhas somando todos os clientes e o
   PostgREST corta em 1.000 por página. A carga usa `sbPaginado()`, que pagina com o
   header `Range` até a resposta vir menor que a página.

### O nome do cliente muda entre as duas bases

No Google Ads é `Dr. Anderson Rodrigues` (com ponto); no Meta, a conta é
`Dr Anderson Rodrigues` (sem ponto). O filtro do PostgREST usa a grafia **com ponto** —
trocar isso faz a aba de Google voltar vazia.

---

## Como os números são calculados

- **CPM** = investimento ÷ impressões × 1.000. **CPC** = investimento ÷ cliques.
  **Custo por conversa/conversão** = investimento ÷ quantidade. Todos retornam `—` (não
  `R$ 0,00`) quando a base é zero — `R$ 0,00` se leria como "saiu de graça".
- **Comparação "vs período anterior"** usa uma janela de mesma duração imediatamente
  anterior à selecionada, recalculada no cliente a cada filtro.
- **Conversões do Google são fracionárias** (ex.: `115,94`), por causa do modelo de
  atribuição — o painel mostra com uma casa decimal em vez de arredondar. Nesta conta,
  hoje, a conversão é inteira (`1`) porque só houve uma.
- **Custo por conversão é recalculado** (investimento ÷ conversões) em vez de usar
  `metrics.cost_per_conversion` da API, senão não acompanharia o período filtrado. Para
  uma janela que cobre a conversão inteira, os dois batem — conferido em 03/09: o Google
  reporta **R$ 134,52** e o painel chega ao mesmo valor.

### Conferência da conversão (07/09/2026)

Batido campo a campo contra a API do Google Ads:

| Dia | Investimento | Impressões | Cliques | Conversões | Custo/conv. |
|---|---|---|---|---|---|
| 03/09 | R$ 134,52 | 1.493 | 54 | **1** | **R$ 134,52** |
| 04/09 a 07/09 | — | — | — | 0 | — |

**Total do período: 1 conversão.** No recorte inteiro (03→07/09, R$ 263,74) o painel
mostra R$ 263,74 por conversão, que é o custo real de aquisição considerando todo o
investimento da janela — diferente dos R$ 134,52 que o Google atribui ao dia isolado.
As duas leituras estão certas, só respondem a perguntas diferentes.

⚠️ **O dia corrente fica defasado.** O banco guarda o snapshot da última carga do Stract;
a campanha continua rodando depois disso. Em 07/09 o banco tinha R$ 24,38 / 632 impressões
/ 12 cliques, enquanto a API já marcava R$ 28,41 / 677 / 14. Dias fechados batem exato —
só o último dia se move até a carga seguinte.
- **Funil:** a pílula de "% da etapa anterior" só aparece de Impressões em diante. Entre
  Investimento → Impressões são unidades diferentes (R$ vs. contagem) e a taxa não teria
  leitura válida.

---

## Stack e design

HTML/CSS/JS puro, Chart.js 4, sem build. Paleta **clássica B16**: `#f4f4f2` / `#d4a800` /
`#111`, Bebas Neue + DM Sans, tema claro e escuro. 2 slots categóricos (amarelo =
investimento, azul = etapa final). O amarelo da marca fica abaixo de 3:1 no tema claro,
então **toda barra leva rótulo direto** — a leitura nunca depende só da cor. Funil em
trapézio (CSS puro), com a magnitude na largura, nunca na cor.

---

## Arquivos

| | |
|---|---|
| `index.html` | o dashboard |

---

## Deploy

Repositório `suporteb16-collab/dashboard-dr-anderson`, branch `main`, GitHub Pages.
`git push origin main` e o Pages republica em ~20s.

**Agência B16** — Henrique Cardoso, Business Intelligence · 07/09/2026.
