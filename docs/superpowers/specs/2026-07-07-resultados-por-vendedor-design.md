# Resultados por Vendedor — design

## Contexto

A faixa de "Desafio das Vendas" foi removida do topo do dashboard (commit `7f84fbd`). O espaço ficou vazio. O pedido agora é ocupar esse mesmo lugar com uma seção orientada a dados reais: o desempenho de cada vendedor no mês, comparado lado a lado.

## Objetivo

Substituir o espaço da antiga faixa de desafios por uma seção "Resultados por Vendedor" com 3 quadrados lado a lado, atualizados em tempo real conforme os vendedores lançam dados.

## Localização

No topo do `<div class="container" id="mainContainer">`, antes da seção "Cadência Atual". Segue o padrão visual já usado no resto do dashboard: bloco `.section-meta` com `.section-label` + grid de cards no estilo `.cadence-card` (fundo `--bg-1`, borda `--border-1`, `border-radius: 20px`). Não reaproveita o visual escuro/gamificado da faixa de desafios.

Título da seção: **"Resultados por Vendedor"**, com meta-info indicando o mês/ano corrente (mesmo padrão de `cadenceMeta`/`funnelMeta`).

## Os 3 quadrados

Grid `grid-template-columns: repeat(3, 1fr)`, gap consistente com `.cadence-grid` (16px).

### Quadrado 1 — Resultado do Mês por Vendedor
- Uma linha por vendedor (`SELLERS`), com nome + dot na cor do vendedor (`s.color`).
- 3 valores por linha: Diagnósticos, Planos de Ação, Vendas — cada um usando a cor semântica já existente (`--diag`, `--plano`, `--venda`).
- Dados: `aggregateData(getMonthDates())[s.id]`.

### Quadrado 2 — Comparativo da Semana
- Gráfico de barras **horizontais** (o card ocupa só 1/3 da largura do container, então barras horizontais empilhadas verticalmente — uma por vendedor — cabem melhor que barras verticais lado a lado; mesmo padrão visual de `.week-bar-track`, mas 1 barra por vendedor em vez de 1 por semana).
- Cor da barra = `s.color` (cor do vendedor, não a cor semântica da métrica).
- Largura da barra proporcional ao total da semana: `diagnosticos + planoAcao + vendas` de `aggregateData(getWeekDates())[s.id]`, normalizado pelo maior valor entre os 4 vendedores (mesma técnica de `maxTotal` usada em `renderWeeklyComparison`).
- Valor numérico à direita de cada barra.
- Vendedores ordenados por valor decrescente (maior primeiro), pra ficar visualmente óbvio "quem está na frente".

### Quadrado 3 — Receita do Mês
- Uma linha por vendedor com o valor em R$ do mês: `aggregateData(getMonthDates())[s.id].valor * 1000`, formatado com `formatCurrency`.
- Linha de destaque no final (visualmente separada, ex. borda superior + peso maior) com o **Total geral do time**: soma do R$ de todos os vendedores no mês (via `sumTotals(aggregateData(getMonthDates())).valor * 1000`).

## Dados e tempo real

Nenhuma infraestrutura nova. Os dados já vivem em `state.records` (populados por `fetchAllRecords()` no load e mantidos atualizados por `handleRealtimeUpdate()` via o canal Supabase Realtime `lancamentos_changes`). As funções de agregação já existentes (`aggregateData`, `getWeekDates`, `getMonthDates`, `sumTotals`) cobrem tudo que os 3 quadrados precisam — nenhuma nova query ou tabela.

Implementação: uma função `renderResultsBand()` que popula os 3 quadrados, chamada dentro de `renderAll()` (mesmo lugar onde `renderChallenges()` era chamado antes de ser removido). Como `renderAll()` já roda tanto no load inicial (`initApp`) quanto em todo evento realtime (`handleRealtimeUpdate`), a seção atualiza sozinha sem código adicional de sincronização.

## Fora de escopo

- Nenhuma tabela nova no Supabase.
- Nenhuma alteração em `lancamentos` ou no schema existente.
- Sem novos campos de input — só leitura/agregação do que já é lançado.
