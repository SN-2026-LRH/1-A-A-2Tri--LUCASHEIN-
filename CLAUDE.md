# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Visão geral

Painel de voos programados (aeroportos brasileiros) usando a API pública **SIROS/ANAC**, com histórico
via **VRA/ANAC (dados abertos)**. Sem servidor próprio: o frontend estático lê direto do **Supabase**
(Postgres + REST), e o "backend" são dois scripts Python disparados por GitHub Actions em cron.

Site publicado via GitHub Pages: https://SN-2026.github.io/06-05-2026-siros/

## Arquitetura

Fluxo de dados ponta a ponta:

```
GitHub Actions (cron) → script Python → API ANAC → normaliza/dedupe → upsert Supabase
                                                                              ↓
                                                index.html (client-side) ← Supabase REST API
```

- **`index.html`** — dashboard SPA sem build (HTML/CSS/JS puro, sem dependências). Faz fetch direto na
  REST API do Supabase usando a `SUPABASE_URL`/`SUPABASE_ANON_KEY` hardcoded no `<script>` do topo do
  arquivo (é a chave `anon`/`publishable`, pública por design — a proteção de escrita fica a cargo do
  RLS no Supabase, não do segredo). Três abas: voos do dia (tabela `voos`), histórico ANAC (tabela
  `historico_vra`) e log do pipeline (tabela `execucoes`). Os selects em cascata de estado/cidade/aeroporto
  são alimentados por `data/airports.json`.
- **`scripts/fetch_flights.py`** — busca voos do dia na API SIROS (`sas.anac.gov.br/sas/siros_api`),
  filtra pelos ICAOs em `AIRPORTS`, normaliza, deduplica e faz upsert em lotes na tabela `voos`. Roda via
  `.github/workflows/update-flights.yml` (4x/dia) ou manualmente.
- **`scripts/fetch_historico_anac.py`** — baixa o CSV mensal do VRA (dados abertos ANAC), parseia,
  filtra/normaliza e faz upsert em `historico_vra`. Roda via `.github/workflows/importar-historico.yml`
  (mensal, dia 3) ou manualmente com input `ano_mes`.
- **`data/airports.json`** — lista estática de referência (ICAO/nome/cidade/estado) usada apenas pelo
  frontend para os filtros em cascata. É o único arquivo em `data/` que ainda é lido pelo código atual.
- **`data/SB*.json`** (SBBR, SBCA, SBCT, SBGL, SBGR, SBSP) — resíduo de uma versão anterior ao Supabase,
  quando os voos eram gravados em JSON por aeroporto. Nada no código atual lê ou escreve esses arquivos;
  considerar obsoletos ao investigar bugs de dados.

## Comandos

```bash
# instalar dependências dos scripts
pip install requests supabase

# rodar localmente (precisa de SUPABASE_URL e SUPABASE_SERVICE_KEY no ambiente)
python scripts/fetch_flights.py
python scripts/fetch_historico_anac.py        # aceita ANO_MES=YYYY-MM opcional

# servir o frontend localmente
python -m http.server 8080
```

Não há build step, linter nem suíte de testes configurados neste repositório.

## Variáveis de ambiente (scripts)

- `SUPABASE_URL`, `SUPABASE_SERVICE_KEY` — obrigatórias, vêm de GitHub Secrets em produção.
- `AIRPORTS` — ICAOs separados por vírgula (GitHub Variable); default é `SBCA` isolado se ausente.
- `ANO_MES` — só em `fetch_historico_anac.py`; default é o mês anterior ao atual (BRT).

## Detalhes que importam ao mexer nos scripts

- **Chaves de upsert/conflito**: `voos` usa `(data_referencia, icao_empresa, numero_voo, icao_origem,
icao_destino, etapa)`; `historico_vra` usa `(ano_mes, icao_empresa, nr_voo, icao_origem, icao_destino,
dt_referencia)`. Os dois scripts fazem deduplicação manual em memória **antes** do upsert — necessário
  porque a API/CSV de origem pode repetir o mesmo voo dentro do mesmo lote, o que causa o erro do
  Postgres `21000: ON CONFLICT DO UPDATE command cannot affect row a second time`.
- **Formato do CSV do VRA é instável** (ver docstring no topo de `fetch_historico_anac.py`): separador
  pode ser TAB ou `;`, há linhas de metadado antes do cabeçalho real, nomes de coluna variam por mês
  (por isso o dicionário `COLS` tenta múltiplos nomes), e há dois formatos de data misturados (ISO com
  nanossegundos para os horários previstos, `DD/MM/YYYY HH:MM` para os reais).
- **Código de saída dos workflows**: erro parcial em qualquer lote faz o script sair com `sys.exit(1)`
  (falha o job no GitHub Actions para ficar visível); ausência total de dados da API sai com `0` (não é
  considerado falha, é esperado em dias sem dados publicados ainda).
- Os workflows não têm permissão de escrita no repositório (`permissions: contents: read`) — os dados
  vão direto ao Supabase, o repo guarda só código.

# CLAUDE.md — Projeto 1-A-A-2Tri--LUCASHEIN-

## Stack

- Python 3.12 (scripts de pipeline)
- Supabase (Postgres + REST API) como banco de dados
- GitHub Actions (workflows agendados via cron + workflow_dispatch)
- Frontend: HTML/CSS/JS puro (sem framework), consumindo a API REST do Supabase
- Fontes de dados externas: API SIROS/ANAC (voos do dia) e arquivos VRA/ANAC (histórico mensal)

## Arquivos principais

- `scripts/fetch_flights.py` — busca voos do dia na API SIROS e faz upsert no Supabase (tabela `voos`)
- `scripts/fetch_historico_anac.py` — importa histórico mensal do VRA/ANAC (tabela `historico_vra`)
- `index.html` — painel público que consulta o Supabase via REST API
- `.github/workflows/update-flights.yml` — roda o pipeline diário (cron 4x/dia)
- `.github/workflows/importar-historico.yml` — roda a importação histórica (mensal, dia 3)
- `sql/setup.sql` — schema do banco (tabelas, RLS, constraints)

## Variáveis de ambiente

- `SUPABASE_URL` → GitHub Secret — URL do projeto Supabase
- `SUPABASE_SERVICE_KEY` → GitHub Secret — service_role key (NUNCA vai no index.html)
- `SUPABASE_ANON_KEY` → hardcoded no index.html (chave pública, pode ficar exposta)
- `AIRPORTS` → GitHub Variable — ICAOs separados por vírgula
- `ANO_MES` → input do workflow_dispatch — formato YYYY-MM (histórico)

## Restrições

- NUNCA commitar a service_role key em nenhum arquivo — ela deve existir SOMENTE como GitHub Secret
- Sempre aplicar deduplicação (`deduplicar()`) ANTES de qualquer upsert que use `on_conflict`,
  pois a API SIROS e os arquivos VRA podem conter registros duplicados no mesmo lote
- Comentários e mensagens de log em português do Brasil
- Comparações de strings vindas do banco (ex: campo `situacao`) devem ser feitas em minúsculo
  (`.toLowerCase()` no JS / `.upper()` consistente no Python) — já causou bug de contagem zerada no painel
- Todo `sys.exit(1)` nos scripts deve ser proposital — sinaliza falha real pro GitHub Actions marcar o run como vermelho
