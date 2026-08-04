# Estudo de Caso: Monitoramento de Freeze Age e Risco de Wraparound de IDs de Transação no PostgreSQL

## 1. Visão Geral

Este estudo de caso descreve a investigação e a mitigação de tabelas PostgreSQL que estavam se aproximando do risco de freeze dos IDs de transação.

O principal objetivo foi reduzir o risco de wraparound, identificando tabelas com idade elevada de XID, avaliando a necessidade de intervenção manual emergencial e definindo uma estratégia de monitoramento após o ajuste do autovacuum.

A análise concentrou-se em:

- Idade dos IDs de transação
- Limites de freeze age
- Comportamento de freeze do autovacuum
- `VACUUM FREEZE` emergencial
- Monitoramento após o ajuste
- Manutenção preventiva por tabela

## 2. Contexto

O PostgreSQL utiliza IDs de transação para controlar a visibilidade das linhas.

Como os IDs são finitos, tuplas antigas precisam ser congeladas em algum momento. Caso isso não ocorra dentro do prazo adequado, o banco pode se aproximar do risco de wraparound.

Normalmente, o autovacuum executa esse processo automaticamente. Entretanto, tabelas grandes ou atividade insuficiente do autovacuum podem permitir que a idade do XID das relações cresça além do aceitável.

Esse não é o tipo de problema que pede atenção educadamente. Ele espera, cresce em silêncio e depois ameaça o banco como um vilão com paciência admirável.

## 3. Problema

Algumas tabelas foram identificadas com idade elevada de XID, próximas de limites perigosos de freeze.

As principais preocupações eram:

- Tabelas próximas de `autovacuum_freeze_max_age`
- Risco de vacuum agressivo forçado
- Possibilidade de manutenção emergencial fora das janelas planejadas
- Tabelas grandes exigindo operações longas de freeze
- Necessidade de determinar se um `VACUUM FREEZE` manual era apropriado
- Necessidade de validar se os novos parâmetros de autovacuum seriam suficientes

## 4. Investigação

A investigação começou pela identificação das relações com maior idade de ID de transação.

Consulta útil:

```sql
SELECT
    n.nspname AS schema_name,
    c.relname AS table_name,
    age(c.relfrozenxid) AS xid_age,
    current_setting('autovacuum_freeze_max_age')::bigint AS autovacuum_freeze_max_age,
    round(
        100.0 * age(c.relfrozenxid)
        / current_setting('autovacuum_freeze_max_age')::numeric,
        2
    ) AS freeze_age_pct,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
FROM pg_class c
JOIN pg_namespace n
    ON n.oid = c.relnamespace
WHERE c.relkind IN ('r', 'p')
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(c.relfrozenxid) DESC;
```

Em seguida, foi verificada a atividade de autovacuum por tabela:

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    last_analyze,
    last_autoanalyze,
    analyze_count,
    autoanalyze_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

## 5. Causa Raiz

O principal risco era que algumas tabelas acumulavam idade de XID mais rapidamente do que o autovacuum conseguia reduzi-la.

Possíveis fatores contribuintes:

- Tamanho elevado das tabelas
- Alta atividade de escrita
- Autovacuum disparando tarde demais
- Operações de vacuum incapazes de acompanhar a carga
- Transações de longa duração atrasando a limpeza
- Limites de custo ou quantidade de workers insuficientes
- Parâmetros por tabela desalinhados à carga
- Limites de freeze altos demais para a realidade operacional

## 6. Ação Emergencial: VACUUM FREEZE Manual

Para tabelas já próximas de limites perigosos, pode ser necessária intervenção manual.

Exemplo:

```sql
VACUUM (FREEZE, VERBOSE) schema.table_name;
```

O `VACUUM FREEZE` manual deve ser considerado quando:

- A idade do XID estiver próxima do limite crítico
- O autovacuum não estiver conseguindo acompanhar
- A tabela for grande e exigir execução planejada
- Houver risco de vacuum agressivo forçado durante o horário comercial
- O monitoramento mostrar crescimento contínuo mesmo após o ajuste dos parâmetros

Entretanto, o freeze manual deve ser tratado como ação emergencial ou corretiva, não como estratégia principal de longo prazo.

Se o freeze manual virar rotina, a configuração do autovacuum provavelmente está pedindo ajuda à maneira peculiar do PostgreSQL.

## 7. Solução de Longo Prazo

A solução definitiva é ajustar o autovacuum para acompanhar a carga antes que as tabelas se aproximem de uma idade crítica.

Parâmetros importantes:

```sql
ALTER TABLE schema.table_name SET (
    autovacuum_vacuum_threshold = <value>,
    autovacuum_vacuum_scale_factor = <value>,
    autovacuum_analyze_threshold = <value>,
    autovacuum_analyze_scale_factor = <value>,
    autovacuum_freeze_min_age = <value>,
    autovacuum_freeze_table_age = <value>
);
```

A estratégia deve considerar:

- Tamanho da tabela
- Taxa de churn
- Geração de tuplas mortas
- Idade atual do XID
- Tempo necessário para o vacuum
- Capacidade de I/O disponível
- Janelas de manutenção
- Impacto no negócio

## 8. Estratégia de Monitoramento

Após o ajuste, o monitoramento deve verificar se a idade do XID está estabilizando ou continua crescendo.

Monitoramento diário recomendado:

```sql
SELECT
    n.nspname AS schema_name,
    c.relname AS table_name,
    age(c.relfrozenxid) AS xid_age,
    round(
        100.0 * age(c.relfrozenxid)
        / current_setting('autovacuum_freeze_max_age')::numeric,
        2
    ) AS freeze_age_pct,
    s.last_autovacuum,
    s.autovacuum_count,
    s.n_dead_tup,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
FROM pg_class c
JOIN pg_namespace n
    ON n.oid = c.relnamespace
LEFT JOIN pg_stat_user_tables s
    ON s.relid = c.oid
WHERE c.relkind IN ('r', 'p')
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 50;
```

Classificação recomendada:

| Percentual de Freeze Age | Classificação | Ação |
|---:|---|---|
| < 50% | Normal | Monitorar |
| 50% a 70% | Atenção | Revisar o comportamento do autovacuum |
| 70% a 85% | Alerta | Validar necessidade de ajuste ou vacuum manual |
| 85% a 95% | Crítico | Planejar vacuum ou freeze manual |
| > 95% | Emergência | Ação imediata necessária |

## 9. Validação Após o Ajuste

Após o tuning, valide:

- Se `xid_age` está reduzindo ou estabilizando
- Se o autovacuum está sendo executado nas tabelas críticas
- Se as tuplas mortas permanecem controladas
- Se vacuums agressivos deixaram de ocorrer inesperadamente
- Se a freeze age permanece abaixo dos limites de alerta
- Se os parâmetros por tabela estão adequados

Exemplo:

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    autovacuum_count,
    last_autoanalyze,
    autoanalyze_count
FROM pg_stat_user_tables
WHERE relname IN ('<critical_table_1>', '<critical_table_2>')
ORDER BY relname;
```

## 10. Resultados Esperados

| Métrica | Antes | Depois |
|---|---|---|
| Idade do XID | Crescente | Estabilizada ou reduzida |
| Risco de vacuum agressivo | Alto | Reduzido |
| Dependência de freeze manual | Possível | Reduzida |
| Atividade do autovacuum | Insuficiente ou tardia | Mais consistente |
| Controle de tuplas mortas | Fraco | Melhorado |
| Previsibilidade operacional | Baixa | Melhorada |

## 11. Lições Aprendidas

As principais lições foram:

- A freeze age deve ser monitorada proativamente.
- O tuning do autovacuum não se limita ao controle de tuplas mortas.
- Tabelas grandes podem exigir uma estratégia de freeze específica.
- O `VACUUM FREEZE` manual é útil em emergências, mas não substitui uma configuração adequada do autovacuum.
- O monitoramento deve continuar após alterações de parâmetros.
- Uma tendência estável da idade do XID é mais importante que uma leitura isolada.
- Tabelas próximas do risco de wraparound devem ser priorizadas antes de otimizações rotineiras.

## 12. Impacto no Negócio

Prevenir o risco de wraparound protege a disponibilidade do banco de dados.

Benefícios para o negócio:

- Menor risco de indisponibilidade emergencial
- Maior previsibilidade da manutenção
- Menor probabilidade de vacuum agressivo forçado durante o horário comercial
- Maior controle operacional sobre tabelas grandes
- Ambiente PostgreSQL de produção mais seguro
- Postura mais forte de prevenção de desastres

## 13. Conclusão

A investigação mostrou que o tuning do autovacuum deve contemplar a prevenção de wraparound, e não apenas a limpeza de tuplas mortas.

Para tabelas já próximas de limites críticos, o `VACUUM FREEZE` manual pode ser necessário. Entretanto, a solução de longo prazo é ajustar e monitorar o autovacuum para manter automaticamente a idade dos XIDs sob controle.

Este caso destaca a importância do monitoramento proativo de freeze age como parte das operações de produção do PostgreSQL.
