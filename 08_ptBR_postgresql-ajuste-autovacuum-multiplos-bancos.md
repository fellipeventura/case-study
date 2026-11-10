# Estudo de Caso: Ajuste de Autovacuum e Autoanalyze em Múltiplos Bancos PostgreSQL de Produção

## 1. Visão Geral

Este estudo de caso descreve uma iniciativa de ajuste do autovacuum e do autoanalyze realizada em diversos bancos PostgreSQL de produção.

O principal objetivo foi melhorar o comportamento das rotinas automáticas de manutenção, reduzindo execuções excessivas ou pesadas de autovacuum, aumentando a atualização das estatísticas e tornando a manutenção mais previsível.

A análise comparou logs anteriores e posteriores aos ajustes em diferentes ambientes PostgreSQL, considerando:

- Quantidade de execuções de autovacuum
- Quantidade de execuções de autoanalyze
- Tempo total de manutenção
- Tempo médio de execução
- Comportamento por tabela
- Melhorias em tabelas críticas
- Estabilidade após os ajustes

Os ambientes avaliados incluíram cargas transacionais, o ambiente Database_Production1 e uma carga de gerenciamento de documentos.

## 2. Contexto

O autovacuum do PostgreSQL é responsável por remover tuplas mortas, evitar bloat de tabelas e índices, manter os visibility maps e proteger o banco contra wraparound de IDs de transação.

O autoanalyze atualiza as estatísticas utilizadas pelo planejador de consultas.

Em bancos grandes ou com alta taxa de alterações, os parâmetros padrão podem não ser adequados. Tabelas grandes podem acumular tuplas mortas demais antes da limpeza, enquanto tabelas voláteis podem manter estatísticas desatualizadas caso o autoanalyze não execute com frequência suficiente.

A iniciativa concentrou-se em ajustar o comportamento do autovacuum de acordo com o tamanho das tabelas, o padrão da carga e o impacto operacional.

## 3. Problema

Antes dos ajustes, os ambientes apresentavam sintomas como:

- Atividade automática excessiva em alguns bancos
- Execuções pesadas de autovacuum em tabelas críticas
- Manutenção repetitiva em tabelas com alto churn
- Limpeza tardia em tabelas transacionais grandes
- Risco de estatísticas desatualizadas afetarem planos de execução
- Baixa visibilidade sobre o comportamento da manutenção por tabela

O objetivo não era simplesmente reduzir a quantidade de execuções, mas alinhar a manutenção automática à realidade de cada carga.

No PostgreSQL, menos execuções nem sempre significam melhora. Poucos autovacuums podem indicar tanto um banco saudável quanto uma manutenção chegando tarde demais, porque bancos de dados também gostam de ser mal interpretados.

## 4. Metodologia

A análise utilizou logs do PostgreSQL coletados antes e depois das alterações de parâmetros.

Foram comparados:

- Total de execuções de autovacuum e autoanalyze
- Tempo total de execução na janela analisada
- Tempo médio e mediano
- Maior execução observada
- Comportamento diário
- Tempo acumulado por tabela
- Comportamento das tabelas críticas antes e depois

Sempre que possível, foram utilizadas janelas de tempo equivalentes.

## 5. Ambiente 1: Carga Transacional

### Perfil do Banco

O primeiro ambiente era um banco PostgreSQL transacional com tabelas relacionadas a dados de pessoas físicas e jurídicas.

A comparação abrangeu o período de 9 a 18 de abril de 2026 contra 9 a 18 de junho de 2026.

### Resumo dos Resultados

| Métrica | Antes | Depois | Variação |
|---|---:|---:|---:|
| Total de execuções | 90 | 109 | +21,1% |
| Tempo total de manutenção | 16,56 min | 22,89 min | +38,2% |
| Tempo médio de execução | 11,04 s | 12,60 s | +14,1% |
| Tempo total de autovacuum | 11,84 min | 9,15 min | -22,7% |
| Tempo médio de autovacuum | 17,33 s | 11,94 s | -31,1% |
| Tempo total de autoanalyze | 4,72 min | 13,74 min | +191,1% |
| Tempo médio de autoanalyze | 5,78 s | 13,08 s | +126,4% |

### Interpretação

A quantidade total de eventos automáticos aumentou após o ajuste. Entretanto, a melhoria mais relevante ocorreu no comportamento do autovacuum.

O tempo total de autovacuum caiu de 11,84 para 9,15 minutos, enquanto o tempo médio caiu de 17,33 para 11,94 segundos.

Isso indica que a limpeza se tornou mais leve e eficiente nas tabelas críticas.

O aumento do tempo de autoanalyze era esperado, pois o ajuste tornou a atualização de estatísticas mais sensível às alterações. Na prática, as estatísticas passaram a ser renovadas com maior frequência, favorecendo melhores decisões do planejador.

### Melhorias por Tabela

| Tabela | Tempo antes | Tempo depois | Variação |
|---|---:|---:|---:|
| `person_table` | 6,09 min | 3,01 min | -50,5% |
| `enterprise_table` | 2,02 min | 0,94 min | -53,3% |
| `classification_table` | 4,63 min | 6,00 min | +29,7% |
| `classification_alternative_table` | 0,69 min | 0,99 min | +43,4% |
| `change_table` | 3,13 min | 5,04 min | +61,1% |

Os maiores ganhos ocorreram nas tabelas transacionais, com redução superior a 50% no tempo acumulado de manutenção.

## 6. Ambiente 2: Carga Database_Production1

### Perfil do Banco

O segundo ambiente era um banco PostgreSQL Database_Production1 com tabelas de aplicação de alto volume.

A comparação abrangeu 1º a 15 de abril de 2026 contra 1º a 15 de junho de 2026.

### Resumo dos Resultados

| Métrica | Antes | Depois | Variação |
|---|---:|---:|---:|
| Total de execuções | 10.262 | 3.377 | -67,1% |
| Tempo total de manutenção | 1.661,00 min | 832,01 min | -49,9% |
| Tempo médio de execução | 9,71 s | 14,78 s | +52,2% |
| Maior execução observada | 93,66 s | 128,63 s | +37,3% |
| Execuções de autovacuum | 5.133 | 1.584 | -69,1% |
| Tempo total de autovacuum | 970,70 min | 651,81 min | -32,9% |
| Execuções de autoanalyze | 5.129 | 1.793 | -65,0% |
| Tempo total de autoanalyze | 690,30 min | 180,20 min | -73,9% |

### Interpretação

Este ambiente apresentou a maior redução geral.

Os eventos automáticos caíram 67,1%, de 10.262 para 3.377, e o tempo acumulado de manutenção caiu 49,9%, de 1.661,00 para 832,01 minutos.

Autovacuum e autoanalyze melhoraram:

- Execuções de autovacuum: -69,1%
- Tempo total de autovacuum: -32,9%
- Execuções de autoanalyze: -65,0%
- Tempo total de autoanalyze: -73,9%

Embora o tempo médio de cada execução tenha aumentado, a quantidade e o tempo acumulado diminuíram significativamente. Isso indica que o banco deixou de realizar manutenção repetitiva em excesso e passou a operar de forma mais controlada.

### Tabelas Críticas

| Tabela | Execuções antes | Tempo antes | Execuções depois | Tempo depois | Variação do tempo |
|---|---:|---:|---:|---:|---:|
| `account_table` | 4.192 | 985,01 min | 986 | 477,61 min | -51,5% |
| `person_table` | 1.574 | 342,99 min | 441 | 171,58 min | -50,0% |
| `product_table` | 657 | 176,82 min | 182 | 102,03 min | -42,3% |
| `register_table` | 760 | 47,67 min | 430 | 26,92 min | -43,5% |
| `collateral_table` | 628 | 42,02 min | 204 | 23,55 min | -44,0% |

O maior ganho absoluto ocorreu em `account_table`, cujo tempo acumulado caiu de 985,01 para 477,61 minutos.

A tabela `product_table` também melhorou. Durante a análise, um índice sem uso foi removido, reduzindo o custo de escrita e manutenção.

### Resultado Principal

O ambiente tornou-se significativamente mais eficiente, com menor recorrência excessiva e manutenção ainda ativa nas tabelas de alto volume.

## 7. Ambiente 3: Carga de Gerenciamento de Documentos

### Perfil do Banco

O terceiro ambiente era um banco PostgreSQL voltado ao gerenciamento de documentos.

A comparação avaliou o período anterior a 18 de junho de 2026 contra o período estabilizado de 20 a 22 de junho de 2026. O dia do ajuste e o dia de transição foram tratados separadamente.

### Resumo dos Resultados

| Métrica | Antes | Depois estabilizado | Variação | Interpretação |
|---|---:|---:|---:|---|
| Eventos automáticos por dia | 25,5 | 110,3 | +332,7% | Maior atividade preventiva |
| Vacuums por dia | 10,0 | 57,3 | +473,3% | Mais limpezas automáticas |
| Analyzes por dia | 15,5 | 53,0 | +241,9% | Estatísticas mais atualizadas |
| Tempo médio de autoanalyze | 24,80 s | 10,95 s | -55,9% | Redução relevante |
| Maior média de execução | 80,60 s | 39,45 s | -51,1% | Menor concentração de picos |

### Interpretação

Diferentemente do ambiente Database_Production1, este melhorou por meio do aumento da frequência da manutenção automática.

Os eventos diários subiram de 25,5 para 110,3, o que foi positivo porque a manutenção passou a executar mais cedo, antes de grandes acúmulos.

Ao mesmo tempo, a duração média do autoanalyze caiu de 24,80 para 10,95 segundos, redução de 55,9%.

O padrão tornou-se mais preventivo: mais execuções, porém mais leves e menos concentradas.

### Tabelas Críticas

As principais tabelas monitoradas foram:

- `document_identifier`
- `document_folder_identifier`

Em `document_identifier`, a manutenção permaneceu frequente, mas as maiores execuções de autoanalyze ficaram abaixo dos picos anteriores.

Em `document_folder_identifier`, o tempo médio de autoanalyze caiu fortemente após a estabilização, especialmente em 20 e 21 de junho.

## 8. Causa Raiz

A causa principal era o desalinhamento dos parâmetros anteriores com o tamanho das tabelas, a taxa de churn e o comportamento da carga.

Em alguns ambientes, a manutenção era frequente e repetitiva demais. Em outros, era tardia, fazendo com que cada execução precisasse processar um volume maior.

Uma única configuração não funciona igualmente bem para todas as tabelas. Tabelas grandes, voláteis ou analisadas com frequência exigem limites e `scale_factor` próprios.

## 9. Solução

A solução envolveu revisar o comportamento do autovacuum e aplicar ajustes por tabela.

A estratégia considerou:

- Tamanho da tabela
- Quantidade estimada de linhas
- Geração de tuplas mortas
- Volume de updates e deletes
- Frequência das alterações
- Histórico de autovacuum
- Comportamento do autoanalyze
- Risco de estatísticas desatualizadas
- Risco de freeze dos IDs de transação
- Impacto durante o horário comercial

Exemplo:

```sql
ALTER TABLE schema.table_name SET (
    autovacuum_vacuum_threshold = <value>,
    autovacuum_vacuum_scale_factor = <value>,
    autovacuum_analyze_threshold = <value>,
    autovacuum_analyze_scale_factor = <value>
);
```

## 10. Resultados

### Banco transacional

- Tempo total de autovacuum reduzido em 22,7%
- Tempo médio de autovacuum reduzido em 31,1%
- Tabelas transacionais com redução superior a 50% no tempo acumulado
- Autoanalyze mais ativo e estatísticas mais atualizadas

### Banco Database_Production1

- Execuções totais reduzidas em 67,1%
- Tempo total de manutenção reduzido em 49,9%
- Execuções de autovacuum reduzidas em 69,1%
- Tempo total de autoanalyze reduzido em 73,9%
- Reduções significativas nas tabelas críticas

### Banco de gerenciamento de documentos

- Eventos automáticos diários aumentados em 332,7%
- Vacuums diários aumentados em 473,3%
- Tempo médio de autoanalyze reduzido em 55,9%
- Maior média de execução reduzida em 51,1%
- Manutenção mais preventiva e menos concentrada

## 11. Lições Aprendidas

- O tuning de autovacuum deve considerar cada tabela.
- Reduzir execuções não é o único indicador de sucesso.
- Mais execuções podem ser positivas quando evitam acúmulos excessivos.
- Autovacuum e autoanalyze devem ser avaliados em conjunto.
- Tabelas grandes normalmente exigem thresholds e scale factors específicos.
- Índices sem uso aumentam o custo de manutenção e devem ser revisados.
- A análise dos logs é essencial para comprovar o comportamento real.
- Comparações de antes e depois são necessárias para demonstrar melhoria.

## 12. Impacto no Negócio

Os ajustes aumentaram a eficiência e a previsibilidade da manutenção.

Principais benefícios:

- Menor custo de manutenção em bancos críticos
- Redução de recorrência automática excessiva
- Estatísticas mais atualizadas para o planejador
- Menor risco de planos de execução baseados em estatísticas antigas
- Comportamento mais previsível
- Menor risco operacional em tabelas de alto volume
- Maior visibilidade para decisões futuras

## 13. Conclusão

A iniciativa melhorou o comportamento da manutenção em diferentes cargas PostgreSQL de produção.

Database_Production1 apresentou forte redução de execuções e tempo acumulado. A carga transacional passou a ter autovacuums mais leves, mesmo com maior atividade de autoanalyze. O ambiente documental migrou para um modelo mais preventivo, com execuções mais frequentes e leves.

Os resultados demonstram que o autovacuum não deve ser tratado como uma configuração global e única. Um tuning eficaz exige análise por tabela, compreensão da carga e monitoramento contínuo.

O resultado final foi um processo de manutenção PostgreSQL mais previsível, eficiente e controlado.
