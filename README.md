# Conversão por Vendedor — Hapvida (`conversao-time`)

Site em produção: **https://conversao-time.vercel.app/**
Projeto Vercel: `conversao-time` (time *mackoy7-sys' projects*) · Repo: `mackoy7-sys/conversao_time` · Branch de produção: **`main`**

Dashboard de conversão de leads → vidas por vendedor (leads da GUP × vendas do datalake, mesmo mês).
Abas: *Visão Geral*, *Análise dos Leads*, *Estados & Cidades*, *Análise de Quartil*, *Próprio × Terceiros*,
*Tempo de Casa*, *Extrato do Vendedor*.

---

## ⛔ Regra de ouro

> **Publicar SÓ por `git push` na `main`.** O projeto é git-conectado: o push dispara o deploy
> automático (~1 min). **Não usar `vercel --prod` / `vercel deploy` de cópia local.**

Motivo, na prática: o site **não é um HTML só**. Ele precisa da pasta `data/` que está neste repo.
Um deploy por CLI de uma cópia local sem `data/` (ou desatualizada) sobe, o HTML até renderiza, e o
painel morre com:

```
Erro ao carregar os dados: HTTP 404 em conversao_vendedor_meta/_index.json
```

Já houve **5 incidentes** de deploy por cima do git só neste projeto, todos com a mesma assinatura
(deploy sem `githubCommitSha`, publicado de uma cópia local): **13/07/2026** artefato truncado
(~50 KB de 190 KB), **21/07** um loader antigo apontando para `raw.githubusercontent.com`, **25/06** e
**28/07** cópia defasada / pasta errada (o Vercel compilou como função Node e devolveu 404), e
**02/08** um stub de 793 bytes sem a pasta `data/`. O conserto é sempre o mesmo: `git pull` + push na
`main`. (O `hapvida-hub` sofreu o mesmo em 14/07 — a CLI vem truncando uploads grandes.)

Se você tem algo local que não está no repo, **`git fetch` + `git pull --rebase` primeiro** — não
publique por cima, ou o dado do dia é perdido.

---

## Arquitetura (importante antes de mexer)

Site **100% estático, sem backend, sem banco, sem chave de API**. Antes lia Supabase ao vivo; desde
29/07/2026 os dados são arquivos JSON commitados no próprio repo.

```
index.html                 <- a "casca": layout + toda a lógica JS (~196 KB). É o que está no ar.
data/                      <- os DADOS mensais + sidecars diarios sob demanda. O JS busca com fetch RELATIVO ./data/...
```

O `index.html` tem um shim de ~50 linhas chamado **`_staticClient('./data')`** (procure por
`const SB=_staticClient` perto da linha 2514). Ele imita a API do `supabase-js`
(`.from().select().eq().in().not().like().range()`), mas lê dos arquivos em `data/`. Por isso
**nenhuma chamada do código acima dele mudou** quando saímos do Supabase — as 3 linhas do
`createClient` antigo estão ali logo acima, comentadas, só como registro.

### Estrutura de `data/`

| Caminho | Conteúdo |
|---|---|
| `data/_manifest.json.gz` | **Arquivo de verificação.** Proveniência: CSV/HTML de origem, mtimes, `dt_carga`, `linhas_raw`, lista de chaves e o tamanho de cada arquivo gerado |
| `data/conversao_vendedor_raw.json.gz` | A base. Array de objetos, ~1.574 linhas × 19 colunas (`nome, area, unidade, supervisor, gestao, supdir, organizacao, faixa_tempo_casa, status, tempo_exato, mes, leads_tot, leads_prod, vidas, ttfs_dias, vidas_vinc, mes_nome, uf, dt_carga`) |
| `data/conversao_vendedor_meta/_index.json.gz` | Lista das 18 chaves de cubo. O shim lê **este primeiro** para saber quais arquivos baixar |
| `data/conversao_vendedor_meta/<CHAVE>.json.gz` | **1 arquivo por cubo** (`ORIGEM_TAG_STATS`, `ORIGEMFILA_STATS`, `TIPOFILA_STATS`, `SLA_BY_NOME`, `ORIG_BY_NOME_MES`, …) |
| `data/conversao_vendedor_raw_uf.json.gz` · `data/conversao_vendedor_meta_uf/` | Recorte geográfico (aba *Estados & Cidades*; cubo `VIDAS_BY_CIDADE`) |
| `data/daily/leads.json.gz` | Eventos diários de leads e vidas vinculadas. Só é baixado depois de aplicar o filtro `De/Até` dentro da aba *Análise dos Leads* |
| `data/daily/core.json.gz` · `data/daily/geo.json.gz` | Sidecars gerados pelo pipeline e mantidos como reserva; a casca atual não os requisita |

Três decisões que **não devem ser desfeitas sem entender o porquê**:

1. **Os arquivos são `.gz`.** Crus dariam ~31 MB por publicação; commitados todo dia isso mataria o
   histórico do repo. Em `.gz` dão ~2 MB e carregam mais rápido. O shim detecta pelos magic bytes
   `1f 8b` e descomprime no navegador com `DecompressionStream` — e se algum dia a Vercel passar a
   servir com `Content-Encoding: gzip` (o navegador descomprimindo sozinho), o shim também aceita
   JSON puro. Não precisa de configuração no Vercel.
2. **1 arquivo por chave no `meta/`, não um arquivo só.** É o que mantém o *lazy-load*: o boot baixa
   ~1 MB, e os cubos pesados (~18 MB de leads, ~12 MB de geografia) só descem quando o usuário abre
   a aba correspondente. Juntar tudo num arquivo faz o boot voltar de ~2s para ~9s.
3. **`dt_carga` é a data do dado, não a data de hoje.** A casca usa `max(dt_carga)` para proratear a
   meta de volume do mês. Carimbar "hoje" num dado de ontem infla a projeção.
4. **O boot e as abas gerais continuam mensais.** Nenhum sidecar diário entra no carregamento inicial.
   A casca só requisita `daily/leads` quando o usuário aplica datas na aba *Análise dos Leads*.

### Filtro diário exclusivo da Análise dos Leads

- O filtro global **Mês** voltou a ser o multisseletor simples, sem dropdown de dias.
- A aba *Análise dos Leads* tem seu próprio `De/Até`. Quando aplicado, ele substitui o filtro Mês
  apenas nas tabelas de Origem, Tipo, Situação, Fila e Vendedor; as demais abas continuam no mês.
- O intervalo corta **leads pela data do atendimento** e **vidas/contratos vinculados pela data de
  cadastro da venda**. Por isso Leads, Contratos, Vidas PF/PME/ADESÃO e conversões mudam juntos.
- `Mês inteiro` limpa o recorte local e devolve essas tabelas ao filtro Mês do topo.

### População do quartil

- Os filtros organizacionais — **Vendedor, Supervisor, Unidade, Gestão, Times, Tempo de Casa e
  Status** — são apenas um **holofote**: escondem/mostram linhas, mas o quartil exibido permanece o
  calculado sobre toda a população válida daquele período. Administrativos e `NÃO MAPEADO`
  continuam fora da população.

---

## Como os dados são atualizados (fluxo do David)

**Não edite `data/` na mão.** Os arquivos são gerados por um pipeline que roda fora deste repo,
em `Documents\RELATORIOS_EVENTUAIS\CONVERSAO_ANO_TODO\` (skill `/atualizar-conversao`):

```
leads GUP + vendas do datalake
   -> build_conversao_vendedor.py      (gera CSV/HTML + daily_payload/{core,leads,geo}.json.gz)
   -> exportar_json_estatico.py --uf   (converte para os .json.gz)
        --out ...\hapvida-dashboards\conversao_time
        --out ...\hapvida-dashboards\extrato-vendedor
   -> git add -A; git commit -m "Dados DD/MM/AAAA"; git push origin main   (nos DOIS repos)
```

O mesmo export alimenta o **Extrato do Vendedor** (`painelvendedores-hap.vercel.app`, repo
`SchuedaHP/painelvendedores_hap`) — a aba *Extrato do Vendedor* e os links de nome do ranking apontam
para lá. Se mexer no formato dos dados, **os dois** dashboards precisam ser atualizados juntos.

---

## Verificar se a produção está sã (30 segundos)

```powershell
# 1) a casca completa está no ar? (esperado ~196000 bytes; ~800 = stub, ~50000 = truncado)
curl.exe -s --ssl-no-revoke -o NUL -w "index: %{http_code} %{size_download}`n" https://conversao-time.vercel.app/

# 2) os dados estão no ar? (esperado 200; 404 = deploy sem a pasta data/)
curl.exe -s --ssl-no-revoke -o NUL -w "meta:  %{http_code}`n" https://conversao-time.vercel.app/data/conversao_vendedor_meta/_index.json.gz

# 3) o dado no ar é o mesmo do git?
curl.exe -sL --ssl-no-revoke https://raw.githubusercontent.com/mackoy7-sys/conversao_time/main/data/_manifest.json.gz -o m_git.gz
curl.exe -s  --ssl-no-revoke https://conversao-time.vercel.app/data/_manifest.json.gz -o m_prod.gz
python -c "import gzip,json;[print(f, json.load(gzip.open(f))['dt_carga'], json.load(gzip.open(f))['linhas_raw']) for f in ('m_git.gz','m_prod.gz')]"
```

> ⚠️ O `--ssl-no-revoke` é necessário **na rede Hapvida**: sem ele o `curl.exe` devolve
> `000` / tamanho `0` (o schannel não consegue checar a lista de revogação de certificados — parece
> site fora do ar, mas é a rede). Alternativa em PowerShell puro:
> `(Invoke-WebRequest 'https://conversao-time.vercel.app/' -UseBasicParsing).RawContentLength`.

Os dois `_manifest` devem trazer o **mesmo `dt_carga` e o mesmo `linhas_raw`**. Se o de produção
estiver mais antigo, algum deploy subiu por cima do git → `git pull` + push.

Para ler qualquer arquivo de dados localmente:

```powershell
python -c "import gzip,json;print(json.load(gzip.open('data/_manifest.json.gz')))"
```

(o GitHub não pré-visualiza `.gz` — tem que baixar e descomprimir)

---

## Sintomas conhecidos → conserto

| Sintoma no navegador | Causa | Conserto |
|---|---|---|
| `HTTP 404 em conversao_vendedor_meta/_index.json` | Deploy sem a pasta `data/` (CLI de cópia local) | `git pull` + push na `main` |
| Página em branco, só o cabeçalho, sem banner de erro | Artefato publicado **truncado** (upload parcial) | push trivial (`git commit --allow-empty`) → rebuild |
| `<função> is not defined` no console | Idem, truncado no meio | idem |
| Trava em "Carregando..." | Deploy antigo com um loader que busca `raw.githubusercontent.com` (**bloqueado** na rede Hapvida) | idem |
| `navegador sem suporte a gzip (DecompressionStream)` | Navegador muito antigo | usar Chrome/Edge atual |
| Números do dia anterior | Só faltou publicar o dado do dia | rodar o pipeline + push |

Em todos os casos: **o repo é a fonte da verdade**. Se o git está íntegro e o site não, o problema é
o deploy — um push resolve, não precisa mexer em código.

---

## Se você for mexer no HTML

- Edite **`index.html`** direto (não há build, não há `npm install`, não há framework). Abrir o
  arquivo local no navegador funciona: ele lê `./data/` da própria pasta.
- **Não mexa no bloco `_staticClient`** a menos que a intenção seja trocar a fonte de dados. Ele é o
  contrato entre a casca e a pasta `data/`.
- Dependência externa: só **Chart.js 4.5.1 via CDN jsdelivr** (`<script>` na linha 8). Sem isso os
  gráficos não desenham.
- Depois de editar: `git add index.html; git commit; git push origin main`. Se o push for rejeitado,
  `git fetch origin` + `git rebase origin/main` e push de novo (o commit de dados do dia costuma
  chegar no meio).
- Teste local antes: abra `index.html`, confira as 7 abas e o console limpo.

### Outros arquivos do repo

| Arquivo | O que é |
|---|---|
| `index.html` | **Produção.** É o que a Vercel serve na raiz |
| `data/` | Os dados (ver acima) |
| `dashboard_quartil_fila.html` | Página auxiliar antiga ("Quartil por Fila"), **autônoma e com dados embutidos**, não linkada pelo `index.html`. Não é atualizada pelo pipeline |
| `index.html.bak_*` | Backups de versões anteriores da casca — **ignorados pelo git** (`.gitignore`) |
| `index.html.pendente_quartil_2407` | Experimento histórico de 24/07/2026 que serviu de referência para o comportamento de holofote. Não é usado pelo site |

---

## Contato

Pipeline de dados e publicação: **David Schueda** (david.schueda@hapvida.com.br).
Antes de publicar qualquer coisa por outro canal que não `git push`, alinhar — é de onde vêm todos os
incidentes de produção deste projeto.
