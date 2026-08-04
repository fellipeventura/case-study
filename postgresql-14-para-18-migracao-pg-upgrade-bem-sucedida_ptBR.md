# Estudo de Caso: Upgrade do PostgreSQL 14 para o PostgreSQL 18 com Restauração pgBackRest e pg_upgrade

## 1. Visão Geral

Este estudo de caso descreve um upgrade bem-sucedido de versão principal do PostgreSQL 14 para o PostgreSQL 18.

O processo incluiu a criação de um ambiente PostgreSQL 14 restaurado por meio do pgBackRest, a validação da consistência do banco, a preparação de um novo cluster PostgreSQL 18, a execução do `pg_upgrade --check`, o upgrade completo, a atualização das estatísticas do otimizador e a validação de acesso à instância atualizada.

O principal objetivo foi realizar um upgrade controlado, com verificações antes, durante e depois do processo.

A análise concentrou-se em:

- Restauração com pgBackRest
- Validação do ambiente PostgreSQL 14 restaurado
- Inicialização do cluster PostgreSQL 18
- Validação de encoding e locale
- Inventário de extensões
- Validação do risco de wraparound
- Preparação das estatísticas
- `pg_upgrade --check`
- Execução bem-sucedida do `pg_upgrade`
- Atualização das estatísticas após o upgrade
- Inicialização final da instância e validação de acesso

## 2. Contexto

Um banco PostgreSQL 14 precisava ser atualizado para o PostgreSQL 18.

Para reduzir o risco, o processo não foi executado diretamente sobre o diretório de dados original de produção. Em vez disso, foi criado um novo ambiente PostgreSQL 14 a partir de uma restauração do pgBackRest.

Depois de validar o ambiente de origem restaurado, um cluster PostgreSQL 18 limpo foi inicializado e utilizado como destino do `pg_upgrade`.

Essa abordagem permitiu validar o upgrade de forma controlada antes de transferir a instância atualizada para a porta de serviço esperada.

## 3. Preparação Inicial da Restauração

Foi criado um novo diretório de dados para armazenar a instância PostgreSQL 14 restaurada.

Exemplo:

```bash
mkdir ./14.directory_new
```

O pacote pgBackRest foi instalado no servidor de destino. Durante a instalação, foi identificado um problema de dependência.

Exemplo do erro:

```text
Failed dependencies:
    libssh2 is needed by pgbackrest
    libssh2.so.1()(64bit) is needed by pgbackrest
```

A dependência ausente foi instalada e a instalação do pgBackRest foi concluída com sucesso.

Exemplo:

```bash
dnf install -y libssh2
rpm -i pgbackrest-2.xx-1PGDG.rhel9.x86_64.rpm
```

## 4. Configuração do pgBackRest

A configuração do pgBackRest foi ajustada para utilizar o repositório, o host de backup, o usuário, o diretório de spool e a quantidade de processos corretos.

Exemplo genérico:

```ini
[global]
process-max=4
repo1-path=/backup/repository
repo1-host=<backup_host>
repo1-host-user=postgres
log-level-console=info
log-level-file=info
archive-async=y
archive-push-queue-max=2000M
spool-path=/restore/spool
```

Os metadados dos backups foram validados com:

```bash
pgbackrest --log-level-console=detail --stanza=<stanza_name> info
```

A saída confirmou que a stanza estava com status OK e que existiam backups full disponíveis.

## 5. Restauração do PostgreSQL 14

Foi realizada uma restauração do pgBackRest para o novo diretório de dados do PostgreSQL 14.

Exemplo genérico:

```bash
pgbackrest \
  --process-max=4 \
  --stanza=<stanza_name> \
  --log-level-console=detail \
  --pg1-path=<new_pg14_data_directory> \
  --delta \
  --set=<backup_set> \
  --type=immediate \
  --target-action=promote \
  restore
```

A restauração foi concluída com sucesso.

Resultado observado no processo documentado:

```text
restore size = 51.0GB
file total = 3105
restore command end: completed successfully
real    2m17s
```

## 6. Validação da Instância PostgreSQL 14 Restaurada

Após a restauração, a instância PostgreSQL 14 foi iniciada em uma porta temporária.

Exemplo:

```bash
pg_ctl -D <new_pg14_data_directory> start
```

Os logs confirmaram que o sistema estava pronto para aceitar conexões.

A instância restaurada foi acessada com `psql`, e a lista de bancos de dados foi validada.

O estado de recuperação foi verificado:

```sql
SELECT pg_is_in_recovery();
```

Resultado esperado:

```text
f
```

Isso confirmou que a instância restaurada havia sido promovida e não estava mais em recovery.

## 7. Validação Pré-Upgrade

Diversas verificações foram realizadas antes do `pg_upgrade`.

### Versão do PostgreSQL

```sql
SELECT version();
```

A instância de origem restaurada foi confirmada como PostgreSQL 14.

### Inventário de Extensões

```sql
SELECT extname, extversion
FROM pg_extension
ORDER BY 1;
```

O ambiente documentado possuía apenas a extensão `plpgsql`.

Isso reduziu a complexidade do upgrade, pois não existiam extensões de terceiros que exigissem bibliotecas compatíveis com a versão de destino.

### Inventário de Objetos

```sql
SELECT relkind, count(*)
FROM pg_class
GROUP BY relkind;
```

Essa consulta ajudou a validar a quantidade de objetos antes do upgrade.

### Risco de Wraparound

A idade dos IDs de transação foi verificada:

```sql
SELECT datname, age(datfrozenxid)
FROM pg_database;
```

Os valores estavam baixos e seguros, portanto não foi necessário executar `VACUUM FREEZE` antes do upgrade.

### Revisão de Tuplas Mortas

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

O resultado documentado não apresentou acúmulo relevante de tuplas mortas.

### Tabelas sem Analyze

```sql
SELECT
    schemaname,
    relname,
    last_analyze
FROM pg_stat_user_tables
WHERE last_analyze IS NULL;
```

As tabelas afetadas foram analisadas manualmente antes do upgrade.

Exemplo:

```sql
ANALYZE schema.table_name;
```

### Encoding e Locale

```sql
SELECT
    datname,
    encoding,
    datcollate,
    datctype
FROM pg_database;
```

Essa validação era importante porque diferenças de locale ou collation entre origem e destino podem afetar ordenação e correção de índices.

## 8. Criação do Cluster PostgreSQL 18

Foi criado um novo diretório de dados para o PostgreSQL 18.

```bash
mkdir ./18.directory
```

O cluster foi inicializado com o encoding e o locale necessários:

```bash
/usr/pgsql-18/bin/initdb \
  -D <new_pg18_data_directory> \
  --encoding=UTF8 \
  --locale=pt_BR.UTF-8 \
  --no-data-checksums
```

A inicialização foi concluída com sucesso.

O novo cluster foi configurado para utilizar uma porta temporária durante a validação.

```conf
port = 5490
```

A instância PostgreSQL 18 foi iniciada e acessada com `psql` para validar os objetos padrão e a inicialização do cluster.

## 9. Verificação de Compatibilidade com pg_upgrade

Os dois clusters foram parados antes da execução do `pg_upgrade`.

```bash
/usr/pgsql-14/bin/pg_ctl -D <pg14_data_directory> stop
/usr/pgsql-18/bin/pg_ctl -D <pg18_data_directory> stop
```

A verificação de compatibilidade foi executada:

```bash
/usr/pgsql-18/bin/pg_upgrade \
  -d <pg14_data_directory> \
  -D <pg18_data_directory> \
  -b /usr/pgsql-14/bin \
  -B /usr/pgsql-18/bin \
  -c
```

A verificação foi concluída com sucesso.

Resultado principal:

```text
Checking cluster versions                                     ok
Checking database connection settings                         ok
Checking database user is the install user                    ok
Checking for prepared transactions                            ok
Checking data type usage                                      ok
Checking for not-null constraint inconsistencies              ok
Checking for presence of required libraries                   ok
Checking for new cluster tablespace directories               ok
*Clusters are compatible*
```

## 10. Execução do pg_upgrade

Após a validação de compatibilidade, o upgrade foi executado sem a opção `-c`.

```bash
/usr/pgsql-18/bin/pg_upgrade \
  -d <pg14_data_directory> \
  -D <pg18_data_directory> \
  -b /usr/pgsql-14/bin \
  -B /usr/pgsql-18/bin
```

O upgrade foi concluído com sucesso.

Entre as etapas executadas pelo `pg_upgrade` estavam:

- Verificações de consistência
- Dump de objetos globais
- Dump dos schemas dos bancos
- Configuração de locale e encoding no novo cluster
- Análise das linhas no novo cluster
- Freeze das linhas no novo cluster
- Migração dos metadados de IDs de transação e multixact
- Reset do arquivamento de WAL
- Restauração dos objetos globais
- Restauração dos schemas
- Cópia dos arquivos das relações dos usuários
- Sincronização dos diretórios de dados
- Verificação de atualização das extensões

Resultado final:

```text
Upgrade Complete
```

## 11. Atualização das Estatísticas Após o Upgrade

Após o upgrade, o PostgreSQL recomendou atualizar as estatísticas do otimizador, pois parte delas não é transferida pelo `pg_upgrade`.

A instância PostgreSQL 18 foi iniciada e os comandos recomendados foram executados:

```bash
/usr/pgsql-18/bin/vacuumdb -p <port> --all --analyze-in-stages --missing-stats-only
```

Em seguida:

```bash
/usr/pgsql-18/bin/vacuumdb -p <port> --all --analyze-only
```

Isso garantiu que as estatísticas do otimizador fossem atualizadas após a mudança de versão principal.

## 12. Migração das Configurações e Inicialização Final

Após o analyze pós-upgrade, a instância PostgreSQL 18 foi parada.

Os arquivos de configuração relevantes do PostgreSQL 14 foram copiados para o novo diretório de dados.

Exemplos:

```bash
cp <old_pg14_data_directory>/pg_hba.conf <new_pg18_data_directory>/
cp <old_pg14_data_directory>/postgresql.auto.conf <new_pg18_data_directory>/
```

O nome do cluster e a porta foram ajustados para representar a instância PostgreSQL 18 atualizada.

A instância antiga do PostgreSQL 14 foi mantida parada, e a nova instância foi iniciada na porta de serviço esperada.

O acesso foi validado com:

```bash
/usr/pgsql-18/bin/psql -p <service_port>
```

A listagem dos bancos confirmou que o banco atualizado estava disponível no PostgreSQL 18.

## 13. Resultados

| Etapa | Resultado |
|---|---|
| Instalação do pgBackRest | Concluída após a correção da dependência |
| Verificação dos backups disponíveis | Bem-sucedida |
| Restauração do PostgreSQL 14 | Bem-sucedida |
| Promoção da instância restaurada | Confirmada com `pg_is_in_recovery() = false` |
| Inventário de extensões | Apenas `plpgsql` identificado |
| Risco de wraparound | Seguro |
| Analyze manual pré-upgrade | Executado |
| `initdb` do PostgreSQL 18 | Bem-sucedido |
| `pg_upgrade --check` | Bem-sucedido |
| Execução do `pg_upgrade` | Bem-sucedida |
| Analyze pós-upgrade | Executado |
| Inicialização final do PostgreSQL 18 | Bem-sucedida |
| Validação de acesso | Bem-sucedida |

## 14. Causa Raiz e Prevenção de Riscos

Não houve falha de upgrade causada por bibliotecas de extensões no processo documentado.

O único problema de dependência ocorreu durante a instalação do pgBackRest, devido à ausência do pacote `libssh2`. A situação foi resolvida antes da restauração.

O upgrade principal foi bem-sucedido porque o processo de validação confirmou:

- Compatibilidade entre os clusters
- Presença das bibliotecas necessárias
- Idade segura dos IDs de transação
- Ausência de extensões de terceiros problemáticas
- Configuração válida dos clusters de origem e destino
- Planejamento compatível de encoding e locale
- Resultado limpo no `pg_upgrade --check`

O principal fator de sucesso não foi sorte. Foi validação. Um conceito surpreendente, aparentemente ainda opcional em alguns lugares.

## 15. Lições Aprendidas

- Sempre valide a disponibilidade dos backups antes de preparar um upgrade.
- Restaurar para um diretório PostgreSQL 14 separado reduz o risco.
- Confirme que a instância restaurada foi promovida antes de iniciar a preparação.
- Verifique as extensões antes do `pg_upgrade`.
- Valide o risco de wraparound antes de um upgrade de versão principal.
- Execute analyze nas tabelas sem estatísticas antes do upgrade.
- Planeje encoding e locale de forma intencional.
- Sempre execute `pg_upgrade --check` antes do upgrade real.
- Execute os comandos recomendados de `vacuumdb` após o upgrade.
- Não remova o cluster antigo até que a nova instância esteja completamente validada.
- Registre todas as etapas e comandos para auditoria e repetibilidade.

## 16. Impacto no Negócio

O processo reduziu o risco operacional durante o upgrade de versão principal do PostgreSQL.

Benefícios:

- Execução controlada do upgrade
- Menor risco de indisponibilidade
- Caminho de restauração validado
- Migração mais segura para o PostgreSQL 18
- Maior confiança na prontidão do rollback
- Estatísticas do otimizador atualizadas após o upgrade
- Evidências claras de validação bem-sucedida

## 17. Conclusão

O upgrade do PostgreSQL 14 para o PostgreSQL 18 foi concluído com sucesso por meio de um processo controlado baseado em restauração com pgBackRest, validação da origem, preparação do cluster de destino, `pg_upgrade --check`, execução completa do `pg_upgrade` e atualização das estatísticas.

O processo demonstrou uma abordagem segura e repetível para upgrades de versão principal do PostgreSQL, com validações em cada etapa crítica.

Este caso destaca que upgrades bem-sucedidos dependem de preparação, verificações de compatibilidade, validação dos backups e conferência pós-upgrade, não de improvisação heroica durante a janela de manutenção.
