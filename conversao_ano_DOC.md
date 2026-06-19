# Dashboard Conversão Ano Todo — Documentação

Dashboard de **conversão de leads em vidas** da operação digital Hapvida, consolidando **Jan–Jun/2026**
(ano corrente até a data da geração). Liga os **leads do GUP** às **vendas digitais** e mostra a
conversão por origem, gestão, promotor, time (próprio/terceiros) e tempo de casa.

- **Entregável publicado:** `index.html` (cópia de `dashboard_conversao_ano.html`).
- **Base auditável:** `base_conversao_ano.csv` (grão por bucket) + `resumo_mensal_conversao_ano.csv`.
- **Gerado por:** `build_conversao_ano.py` (reprocessa tudo a cada rodada).

---

## 1. Fontes de dados

| Fonte | Arquivo | Papel |
|---|---|---|
| **Leads GUP** | 6 arquivos do ano: `Leads_GUP_jan26..mai26 1.xlsb` + `Leads_Gup_jun26.csv` | universo de leads (contatos) |
| **Vendas digitais** | `oracle-reports/out/vendas_2026h1.csv` (datalake **RAWZN / VENDA_DIGITAL**) | 1 linha = 1 vida vendida |
| **De-para de promotores** | `EUGENIA/bases/BASE_PROMOTORES.csv` | login GUP → NOME/ÁREA/UNIDADE/GESTÃO/SUP/SUPDIR, **tempo de casa** (DT ADMISSÃO) e **ORGANIZAÇÃO** (Time Próprio/Terceiros) |

Telefone é a chave de cruzamento: normalizado para **11 dígitos** (`utils.phone_key11`).

---

## 2. Regras de cálculo (o "racional")

1. **Dedup de leads por telefone DENTRO do mês.** Um telefone conta **1× por mês**, mas **aparece em
   cada mês** em que houve contato (quem falou em jan e em mar conta nos dois). Mantém a conversa mais
   antiga do mês. *(Não é dedup anual — isso subcontava meses recentes.)*

2. **Conversão lead → vida.** Casa venda × lead por **telefone** e exige **`DT_CADASTRAMENTO` ≥ data do
   1º contato** do lead. A venda é creditada ao **promotor/origem do 1º contato** (o "momento antigo").

3. **Venda × Venda Acumulada** (métrica = vidas, indexada pelo **mês da venda**):
   - **Venda** = 1º contato e venda no **mesmo mês**.
   - **Venda Acumulada** = 1º contato em **mês anterior** ao da venda (lead amadureceu).
   - `Venda + Acumulada = total de vidas` do mês.

4. **Tempo de casa** = a partir da `DT ADMISSÃO` do de-para, em faixas `00–03M / 04–06M / 07–09M /
   10–12M / >12M` (e `N/D` se sem data). Conversão sobe com o tempo de casa.

---

## 3. Abas do dashboard

| Aba | Conteúdo |
|---|---|
| **Visão Geral** | KPIs, Top agentes, leads por área, conversão por supervisor; tabelas por agente, origem e tempo de casa |
| **Análise de Quartil** | distribuição de agentes em Q1–Q4 por score |
| **Extrato do Agente** | perfil individual por origem vs média do quartil |
| **Evolução Temporal** | comparação entre snapshots |
| **Venda × Acumulada** | vidas por mês da venda (barras Venda+Acumulada) + linha de leads do mês; tabelas por Gestão e Promotor |
| **Próprio × Terceiros** | comparativo por `ORGANIZAÇÃO`: KPIs, Leads×Vidas, conversão e evolução mensal por time |
| **Tempo de Casa** | conversão por faixa, nº de agentes por faixa e tabela de métricas (trazida do dashboard do time) |

**Filtros** (barra superior, multi-seleção): Nome, Área, Unidade, Supervisor, Gestão, Sup.Diretor,
**Times** (Próprio/Terceiros), Tipo, Produto, Origem, Tempo de Casa, Status + **Meses da Venda**
(corte por período, vale em todas as abas).

> **Filtro "Times":** filtra por `ORGANIZAÇÃO` (coluna do de-para). Aplica-se a **todas** as abas,
> inclusive Venda×Acumulada e os gráficos mensais de Próprio×Terceiros (esses usam o mapa NOME→organização,
> pois os arrays mensais carregam só o nome do promotor).

---

## 4. Base CSV (auditável por qualquer pessoa)

`base_conversao_ano.csv` (`;`, UTF-8): **um registro por bucket** = combinação de
NOME × ÁREA × UNIDADE × SUP × GESTÃO × SUP.DIR × TIPO × PRODUTO × ORIGEM × FAIXA_TEMPO × STATUS ×
ORGANIZAÇÃO, com `leadsTot`, `leadsProd`, `contratos`, `vidas`, `VIDAS_VENDA`, `VIDAS_ACUM`.
**Somando as colunas** desse arquivo (com filtros) sai **qualquer número do dashboard**.

`resumo_mensal_conversao_ano.csv`: os números-cabeçalho por mês.

### Números atuais (base 18/06/2026)

| Mês | Leads | Vidas Venda | Vidas Acum. | Vidas Total | Conv.% |
|---|--:|--:|--:|--:|--:|
| Jan | 106.339 | 3.861 | 0 | 3.861 | 3,63 |
| Fev | 99.463 | 2.911 | 1.251 | 4.162 | 4,18 |
| Mar | 132.962 | 4.217 | 2.197 | 6.414 | 4,82 |
| Abr | 122.640 | 4.021 | 2.498 | 6.519 | 5,32 |
| Mai | 128.003 | 4.432 | 2.796 | 7.228 | 5,65 |
| **Jun** | **66.385** | **1.191** | **1.461** | **2.652** | **3,99** |
| **Total** | **655.792** | **20.633** | **10.203** | **30.836** | **4,70** |

---

## 5. Diferença vs. "Conversão do Time" (conversao-time.vercel.app)

Os dois dashboards **medem coisas diferentes** — não devem bater por construção:

| | Conversão do Time | Conversão Ano Todo (este) |
|---|---|---|
| Fonte de vendas | **levantamento MANUAL** (`...LEVANTAMENTO VENDAS MANUAL.xlsb`) | **datalake RAWZN / VENDA_DIGITAL** |
| Regra | telefone/CPF, sem janela de data | telefone **+ venda ≥ 1º contato** + dedup mensal |
| Período | snapshot de 1 mês | ano todo (Jan–Jun) |

Exemplo (junho): o time mostra ~3.070 vidas (manual); aqui são 2.652 (datalake, com regra de 1º
contato). O datalake tem 3.525 vidas em junho, mas só 2.652 amarram num lead pela regra. O extrato do
datalake também pode estar alguns dias atrás (conferir a última `DT_CADASTRAMENTO` em `vendas_2026h1.csv`).

---

## 6. Como regenerar (processo diário)

1. **Atualizar leads de junho** (mês corrente): copiar o export GUP cru mais recente
   (`david_DDMM_*.csv`) por cima de `Leads_Gup_jun26.csv` (com backup) e rodar
   `python atualiza_base_leads.py --rebuild` (reconstrói `BASE_LEADS_ANO.pkl`).
2. **Atualizar vendas:** reextrair o datalake se quiser dados mais recentes (apagar
   `oracle-reports/out/vendas_2026h1.csv` força nova extração; tem cache de 30 min).
3. **Build:** `python build_conversao_ano.py` → regenera `dashboard_conversao_ano.html`,
   `base_conversao_ano.csv`, `resumo_mensal_conversao_ano.csv` e `verificacao_conversao_ano.xlsx`.
4. **Publicar:** copiar o HTML para `index.html` do repositório e dar commit/push (deploy no Vercel).

> Detalhes operacionais e validações na skill `/conversao-ano` e no `CLAUDE.md` (ativos compartilhados).
