# Estudo de Caso: Análise de Regressão com Query Store no SQL Server

## 1. Visão Geral

Este estudo de caso descreve o uso do Query Store do SQL Server para identificar, analisar e mitigar regressões de desempenho de consultas em um ambiente de produção.

O principal objetivo foi comparar o comportamento das consultas ao longo do tempo, detectar regressões de CPU, duração e leituras lógicas e apoiar decisões como ajuste de índices, revisão de estatísticas ou forçamento de plano.

A análise concentrou-se em:

- Estatísticas de execução do Query Store
- Consultas com regressão
- Alterações nos planos de execução
- Uso de CPU
- Duração
- Leituras lógicas
- Validação de planos forçados
- Recomendações para monitoramento contínuo

## 2. Contexto

O Query Store estava habilitado em um banco de dados de produção para fornecer visibilidade histórica sobre o desempenho das consultas.

O ambiente precisava de uma forma estruturada de responder perguntas como:

- Quais consultas consumiram mais CPU?
- Quais consultas ficaram mais lentas em relação ao período anterior?
- Houve mudança no plano de execução?
- Existe algum plano forçado ativo?
- O forçamento de algum plano falhou?
- O problema deve ser resolvido com índice, atualização de estatísticas, reescrita da consulta ou plano forçado?

Sem o Query Store, esse tipo de investigação depende frequentemente do que ainda permanece no cache, o que equivale a perguntar ao SQL Server o que ele lembra depois de um dia estressante. O Query Store oferece uma visão histórica muito mais confiável.

## 3. Problema

A carga de produção apresentou períodos de maior consumo de recursos e lentidão na aplicação.

As principais preocupações eram:

- Consultas com aumento de consumo de CPU
- Consultas com maior duração média
- Consultas com mais leituras lógicas
- Mudanças de plano causando desempenho inconsistente
- Possível sensibilidade a parâmetros
- Necessidade de identificar se as regressões eram isoladas ou recorrentes
- Necessidade de validar os planos forçados existentes

O objetivo não era apenas localizar a consulta mais cara, mas identificar quais consultas haviam piorado em relação ao seu próprio comportamento anterior.

## 4. Investigação

A investigação utilizou as visões de catálogo do Query Store, incluindo:

- `sys.query_store_query_text`
- `sys.query_store_query`
- `sys.query_store_plan`
- `sys.query_store_runtime_stats`
- `sys.query_store_runtime_stats_interval`

A análise foi dividida em três verificações principais.

## 5. Consultas com Maior Consumo de CPU

A primeira etapa foi identificar as consultas que mais consumiram CPU no período analisado.

Exemplo:

```sql
SELECT TOP (50)
    DB_NAME() AS database_name,
    q.query_id,
    p.plan_id,
    OBJECT_NAME(q.object_id) AS object_name,
    SUM(rs.count_executions) AS execution_count,
    SUM(rs.avg_cpu_time * rs.count_executions) / 1000.0 AS total_cpu_ms,
    AVG(rs.avg_cpu_time) / 1000.0 AS avg_cpu_ms,
    SUM(rs.avg_duration * rs.count_executions) / 1000.0 AS total_duration_ms,
    AVG(rs.avg_duration) / 1000.0 AS avg_duration_ms,
    SUM(rs.avg_logical_io_reads * rs.count_executions) AS total_logical_reads,
    LEFT(qt.query_sql_text, 4000) AS query_text
FROM sys.query_store_query_text qt
JOIN sys.query_store_query q
    ON q.query_text_id = qt.query_text_id
JOIN sys.query_store_plan p
    ON p.query_id = q.query_id
JOIN sys.query_store_runtime_stats rs
    ON rs.plan_id = p.plan_id
JOIN sys.query_store_runtime_stats_interval rsi
    ON rsi.runtime_stats_interval_id = rs.runtime_stats_interval_id
WHERE rsi.start_time >= DATEADD(DAY, -7, SYSUTCDATETIME())
GROUP BY
    q.query_id,
    p.plan_id,
    q.object_id,
    qt.query_sql_text
ORDER BY total_cpu_ms DESC;
```

Essa consulta ajudou a identificar quais comandos eram responsáveis pelo maior consumo acumulado de CPU.

## 6. Consultas com Regressão

A segunda etapa foi comparar o comportamento atual com uma linha de base anterior.

Uma regressão pode ser identificada quando uma consulta apresenta aumento significativo em:

- Duração média
- Tempo de CPU
- Leituras lógicas
- Quantidade de execuções
- Tempo de espera

Exemplo de abordagem:

```sql
WITH query_stats AS (
    SELECT
        q.query_id,
        p.plan_id,
        CASE
            WHEN rsi.start_time >= DATEADD(DAY, -7, SYSUTCDATETIME())
                THEN 'current_period'
            WHEN rsi.start_time >= DATEADD(DAY, -14, SYSUTCDATETIME())
             AND rsi.start_time < DATEADD(DAY, -7, SYSUTCDATETIME())
                THEN 'baseline_period'
        END AS period_name,
        SUM(rs.count_executions) AS execution_count,
        AVG(rs.avg_duration) / 1000.0 AS avg_duration_ms,
        AVG(rs.avg_cpu_time) / 1000.0 AS avg_cpu_ms,
        AVG(rs.avg_logical_io_reads) AS avg_logical_reads
    FROM sys.query_store_query q
    JOIN sys.query_store_plan p
        ON p.query_id = q.query_id
    JOIN sys.query_store_runtime_stats rs
        ON rs.plan_id = p.plan_id
    JOIN sys.query_store_runtime_stats_interval rsi
        ON rsi.runtime_stats_interval_id = rs.runtime_stats_interval_id
    WHERE rsi.start_time >= DATEADD(DAY, -14, SYSUTCDATETIME())
    GROUP BY
        q.query_id,
        p.plan_id,
        CASE
            WHEN rsi.start_time >= DATEADD(DAY, -7, SYSUTCDATETIME())
                THEN 'current_period'
            WHEN rsi.start_time >= DATEADD(DAY, -14, SYSUTCDATETIME())
             AND rsi.start_time < DATEADD(DAY, -7, SYSUTCDATETIME())
                THEN 'baseline_period'
        END
)
SELECT
    current_data.query_id,
    current_data.plan_id,
    baseline_data.avg_duration_ms AS baseline_avg_duration_ms,
    current_data.avg_duration_ms AS current_avg_duration_ms,
    ((current_data.avg_duration_ms - baseline_data.avg_duration_ms)
        / NULLIF(baseline_data.avg_duration_ms, 0)) * 100 AS duration_regression_pct,
    baseline_data.avg_cpu_ms AS baseline_avg_cpu_ms,
    current_data.avg_cpu_ms AS current_avg_cpu_ms,
    baseline_data.avg_logical_reads AS baseline_avg_logical_reads,
    current_data.avg_logical_reads AS current_avg_logical_reads
FROM query_stats current_data
JOIN query_stats baseline_data
    ON baseline_data.query_id = current_data.query_id
   AND baseline_data.plan_id = current_data.plan_id
WHERE current_data.period_name = 'current_period'
  AND baseline_data.period_name = 'baseline_period'
ORDER BY duration_regression_pct DESC;
```

Essa abordagem ajuda a priorizar consultas que ficaram piores, e não apenas consultas naturalmente caras.

## 7. Validação de Planos Forçados

A terceira etapa foi validar os planos forçados.

Planos forçados podem estabilizar o desempenho, mas precisam ser monitorados, pois podem falhar ou deixar de ser apropriados após mudanças na distribuição dos dados.

Exemplo:

```sql
SELECT
    q.query_id,
    p.plan_id,
    p.is_forced_plan,
    p.force_failure_count,
    p.last_force_failure_reason_desc,
    p.count_compiles,
    p.last_compile_start_time,
    LEFT(qt.query_sql_text, 4000) AS query_text
FROM sys.query_store_plan p
JOIN sys.query_store_query q
    ON q.query_id = p.query_id
JOIN sys.query_store_query_text qt
    ON qt.query_text_id = q.query_text_id
WHERE p.is_forced_plan = 1
ORDER BY p.force_failure_count DESC;
```

Essa validação ajuda a responder:

- Quais planos estão forçados?
- Existem falhas de forçamento?
- Quando ocorreu a última compilação?
- O plano forçado ainda é seguro?

## 8. Categorias de Causa Raiz

| Categoria | Descrição |
|---|---|
| Regressão de plano | O SQL Server escolheu um plano de execução pior |
| Estatísticas desatualizadas | As estimativas de cardinalidade ficaram imprecisas |
| Sensibilidade a parâmetros | Valores diferentes de parâmetros produziram planos instáveis |
| Alteração de índice | Índices ausentes, alterados ou removidos afetaram o plano |
| Crescimento dos dados | O aumento da tabela alterou o custo da consulta |
| Mudança na carga | O volume ou o padrão de acesso mudou |
| Bloqueios ou esperas | O tempo de execução foi afetado pela concorrência, não apenas pelo plano |

## 9. Solução

A solução depende do tipo de regressão identificado.

### Atualização de estatísticas

Quando as estimativas estiverem imprecisas:

```sql
UPDATE STATISTICS schema.table_name;
```

### Ajuste de índices

Quando as leituras lógicas estiverem altas ou houver excesso de scans, avalie alterações nos índices.

```sql
CREATE INDEX ix_table_filter_columns
ON schema.table_name (filter_column_1, filter_column_2)
INCLUDE (select_column_1, select_column_2);
```

### Reescrita da consulta

Quando a estrutura da consulta produzir planos ruins, revise joins, predicados, funções em cláusulas `WHERE`, conversões implícitas e colunas desnecessárias.

### Forçamento de plano

Quando existir um plano conhecido e eficiente e a carga for estável, o forçamento pode ser considerado.

```sql
EXEC sp_query_store_force_plan
    @query_id = <query_id>,
    @plan_id = <plan_id>;
```

O plano forçado deve ser monitorado após a implementação.

## 10. Validação Após a Correção

Após aplicar a correção, valide:

- Uso de CPU
- Duração
- Leituras lógicas
- Quantidade de execuções
- Estatísticas de espera
- Estabilidade do plano
- Estatísticas de execução do Query Store
- Comportamento da aplicação

Consulta de validação:

```sql
SELECT
    q.query_id,
    p.plan_id,
    SUM(rs.count_executions) AS execution_count,
    AVG(rs.avg_duration) / 1000.0 AS avg_duration_ms,
    AVG(rs.avg_cpu_time) / 1000.0 AS avg_cpu_ms,
    AVG(rs.avg_logical_io_reads) AS avg_logical_reads
FROM sys.query_store_query q
JOIN sys.query_store_plan p
    ON p.query_id = q.query_id
JOIN sys.query_store_runtime_stats rs
    ON rs.plan_id = p.plan_id
JOIN sys.query_store_runtime_stats_interval rsi
    ON rsi.runtime_stats_interval_id = rs.runtime_stats_interval_id
WHERE q.query_id = <query_id>
  AND rsi.start_time >= DATEADD(DAY, -1, SYSUTCDATETIME())
GROUP BY
    q.query_id,
    p.plan_id
ORDER BY avg_duration_ms DESC;
```

## 11. Resultados Esperados

| Métrica | Antes | Depois |
|---|---|---|
| Duração média | Aumentada | Reduzida ou estabilizada |
| Uso de CPU | Alto | Reduzido |
| Leituras lógicas | Altas | Reduzidas |
| Estabilidade do plano | Instável | Melhorada |
| Visibilidade das regressões | Baixa | Melhorada |
| Tempo de diagnóstico | Maior | Menor |

## 12. Lições Aprendidas

- O Query Store é essencial para análises históricas de desempenho.
- As consultas que mais consomem recursos nem sempre são as que sofreram regressão.
- Planos forçados devem ser monitorados continuamente.
- Estatísticas e sensibilidade a parâmetros são causas comuns de instabilidade de planos.
- Decisões de tuning devem ser baseadas em evidências de antes e depois.
- Uma boa análise precisa separar problemas de plano de problemas de carga ou bloqueio.

## 13. Impacto no Negócio

O uso do Query Store melhorou a capacidade de diagnosticar e priorizar problemas de desempenho.

Benefícios:

- Análise de causa raiz mais rápida
- Redução da lentidão da aplicação
- Melhor priorização das ações de tuning
- Menor risco ao utilizar planos forçados
- Maior estabilidade em produção
- Melhor comunicação entre as equipes de DBA e aplicação

## 14. Conclusão

A análise de regressões com Query Store forneceu uma forma estruturada de identificar consultas que se tornaram mais lentas ou mais caras ao longo do tempo.

Ao combinar consumo de recursos, detecção de regressões e validação de planos forçados, a equipe de DBA passou a contar com um método mais confiável para investigar desempenho no SQL Server.

Essa abordagem apoia decisões de tuning mais seguras e um comportamento mais previsível em produção.
