# Estudo de Caso: Gargalo de CPU no PostgreSQL Causado por UPDATE Concorrente e Sequential Scan

## 1. Visão Geral

Este estudo de caso descreve a investigação de um gargalo de CPU no PostgreSQL causado por um comando `UPDATE` de alta frequência, executado simultaneamente por várias sessões da aplicação.

O problema foi identificado durante um período de uso elevado de CPU.

A análise concentrou-se em:

- Sessões ativas no banco de dados
- Comandos concorrentes e repetitivos
- Comportamento do plano de execução
- Custo de sequential scan
- Ausência ou inadequação de índices
- Impacto da concorrência das consultas
- Ações de ajuste recomendadas

## 2. Contexto

Um ambiente PostgreSQL de produção apresentou uso elevado de CPU em determinados períodos do dia.

A atividade do banco mostrou várias sessões concorrentes executando um padrão semelhante de `UPDATE` sobre a mesma tabela.

O padrão da consulta filtrava por identificador da conta e tipo de titularidade.

Exemplo genérico:

```sql
UPDATE application_schema.customer_holder
SET
    status = 'DELETED',
    last_load_date = localtimestamp
WHERE
    account_id = '<account_id>'
    AND ownership_type = <value>;
```

A tabela possuía grande quantidade de linhas, e o plano de execução mostrou que o PostgreSQL realizava uma varredura completa em vez de utilizar um caminho de acesso eficiente por índice.

## 3. Problema

Os principais sintomas eram:

- Uso elevado de CPU
- Várias sessões concorrentes executando o mesmo padrão de atualização
- Sequential scans em uma tabela atualizada com frequência
- Grande quantidade de linhas removidas pelo filtro
- Aumento do tempo de execução sob concorrência
- Risco de contenção de bloqueios e amplificação da carga

A consulta em si era simples, que é justamente onde os problemas gostam de se esconder. Consultas simples sem o índice adequado são apenas consultas caras vestidas de forma casual.

## 4. Investigação

A investigação utilizou monitoramento da atividade do banco e análise do plano de execução.

### Análise das sessões ativas

A primeira etapa foi identificar sessões ativas repetidas executando o mesmo comando.

Consulta útil para diagnóstico:

```sql
SELECT
    pid,
    usename,
    datname,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS runtime,
    query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY runtime DESC;
```

### Análise do plano de execução

O plano de execução apresentou um sequential scan.

Exemplo genérico:

```text
Update on customer_holder
  -> Seq Scan on customer_holder
       Filter: account_id = '<account_id>' AND ownership_type = <value>
       Rows Removed by Filter: <large_number_of_rows>
```

O ponto principal era que o PostgreSQL precisava examinar muitas linhas para localizar um conjunto muito pequeno de registros.

### Análise de concorrência

O problema se tornou mais grave porque o mesmo padrão era executado simultaneamente por várias sessões.

Uma consulta ineficiente executada uma vez pode passar despercebida. Executada repetidamente e em paralelo, transforma-se em um problema de CPU.

## 5. Causa Raiz

A causa raiz era um caminho de acesso ineficiente para o comando `UPDATE`.

O banco utilizava sequential scan porque não existia um índice adequado para o padrão do filtro, ou porque o índice existente não era seletivo nem suficientemente alinhado aos predicados da consulta.

O padrão de filtro problemático era:

```sql
WHERE account_id = ?
  AND ownership_type = ?
```

Como esse padrão era executado de forma concorrente, o custo das varreduras completas multiplicava o consumo de CPU.

## 6. Solução

A solução recomendada foi criar ou validar um índice composto alinhado aos predicados da consulta.

Exemplo:

```sql
CREATE INDEX CONCURRENTLY idx_customer_holder_account_ownership
ON application_schema.customer_holder (account_id, ownership_type);
```

Caso a atualização costume atingir apenas linhas ativas ou determinado status, pode ser avaliado um índice parcial.

Exemplo:

```sql
CREATE INDEX CONCURRENTLY idx_customer_holder_account_ownership_active
ON application_schema.customer_holder (account_id, ownership_type)
WHERE status <> 'DELETED';
```

A melhor alternativa depende da distribuição dos dados e dos padrões reais de consulta.

## 7. Etapas de Validação

Antes de criar o índice, valide as estatísticas das colunas:

```sql
SELECT
    attname,
    n_distinct,
    correlation
FROM pg_stats
WHERE schemaname = 'application_schema'
  AND tablename = 'customer_holder'
  AND attname IN ('account_id', 'ownership_type', 'status');
```

Revise os índices existentes:

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = 'application_schema'
  AND tablename = 'customer_holder';
```

Após criar o índice, valide novamente o plano de execução:

```sql
EXPLAIN (ANALYZE, BUFFERS)
UPDATE application_schema.customer_holder
SET
    status = 'DELETED',
    last_load_date = localtimestamp
WHERE
    account_id = '<account_id>'
    AND ownership_type = <value>;
```

Melhorias esperadas:

- Index Scan ou Bitmap Index Scan no lugar de Seq Scan
- Menor quantidade de linhas examinadas
- Redução do uso de CPU
- Menor tempo de execução
- Menor impacto sob concorrência

## 8. Recomendações Adicionais

### Controlar a concorrência

Caso a aplicação execute muitas atualizações idênticas em paralelo, deve-se avaliar se a concorrência pode ser limitada.

O ajuste do banco ajuda, mas permitir que oito sessões ataquem a mesma tabela ao mesmo tempo continua sendo uma escolha arquitetural bastante criativa.

### Processamento em lotes

Caso a operação afete muitas linhas, atualizações em lotes podem reduzir a duração dos bloqueios e os picos de consumo.

Exemplo:

```sql
UPDATE application_schema.customer_holder
SET
    status = 'DELETED',
    last_load_date = localtimestamp
WHERE ctid IN (
    SELECT ctid
    FROM application_schema.customer_holder
    WHERE account_id = '<account_id>'
      AND ownership_type = <value>
    LIMIT 1000
);
```

### Manter as estatísticas atualizadas

Caso a tabela sofra alterações frequentes, os parâmetros de autoanalyze devem ser revisados.

Estatísticas desatualizadas podem levar o PostgreSQL a escolher planos de execução inadequados.

### Monitorar após a alteração

O monitoramento pós-alteração deve incluir:

- Uso de CPU
- Duração da consulta
- Quantidade de linhas examinadas
- Leituras de buffers
- Uso do índice
- Esperas por bloqueio
- Frequência das atualizações

## 9. Resultados Esperados

| Métrica | Antes | Depois |
|---|---|---|
| Método de acesso | Sequential Scan | Index Scan ou Bitmap Index Scan |
| Linhas examinadas | Alto volume | Baixo volume |
| Uso de CPU | Alto sob concorrência | Reduzido |
| Duração da consulta | Maior | Menor |
| Impacto da concorrência | Significativo | Reduzido |
| Estabilidade do plano | Fraca | Melhorada |

## 10. Lições Aprendidas

As principais lições foram:

- Comandos `UPDATE` simples podem se tornar caros sem índices adequados.
- Sequential scans sob concorrência podem rapidamente se transformar em gargalos de CPU.
- O desenho dos índices deve seguir os predicados reais das consultas.
- Os planos de execução devem ser validados com `EXPLAIN ANALYZE`.
- A concorrência da aplicação pode amplificar ineficiências do banco de dados.
- Estatísticas atualizadas são essenciais para a qualidade dos planos.

## 11. Impacto no Negócio

A abordagem de ajuste reduz o risco de lentidão na aplicação causada pela saturação de CPU no banco de dados.

Benefícios esperados:

- Operações de atualização mais rápidas
- Menor pressão sobre a CPU
- Melhor tempo de resposta da aplicação
- Menor risco de incidentes em horários de pico
- Comportamento mais previsível do banco de dados
- Melhor escalabilidade para cargas concorrentes

## 12. Conclusão

A investigação mostrou que o gargalo de CPU era causado por um `UPDATE` repetitivo e concorrente, executado por meio de um sequential scan ineficiente.

A solução recomendada foi adicionar um índice composto alinhado ao predicado da consulta, validar o plano de execução e monitorar a concorrência após a implementação.

Este caso destaca a importância de combinar monitoramento de sessões, análise de planos de execução e desenho de índices ao investigar problemas de desempenho no PostgreSQL.
