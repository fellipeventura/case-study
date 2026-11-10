# Estudo de Caso: Solução de Direcionamento Incorreto em Servidor FDW do PostgreSQL

## 1. Visão Geral

Este estudo de caso descreve a investigação de um problema no Foreign Data Wrapper do PostgreSQL, no qual uma tabela estrangeira apontava para um servidor remoto incorreto.

O principal objetivo foi identificar por que as consultas à tabela estrangeira estavam chegando ao destino errado, validar os metadados do FDW e corrigir com segurança a configuração do servidor estrangeiro.

A análise concentrou-se em:

- `postgres_fdw`
- Tabelas estrangeiras
- Servidores estrangeiros
- Mapeamentos de usuários
- Opções de conexão
- Validação de host e porta
- Comportamento de senha e `.pgpass`
- Validação após a alteração

## 2. Contexto

Um ambiente PostgreSQL utilizava `postgres_fdw` para acessar tabelas de outra instância PostgreSQL.

Esperava-se que uma tabela estrangeira recuperasse dados de determinado banco remoto, mas a investigação mostrou que a configuração do servidor FDW apontava para um host ou serviço incorreto.

Esse tipo de problema pode ser sutil, pois o nome da tabela estrangeira local pode parecer correto enquanto o servidor estrangeiro subjacente aponta para outro lugar. Muito elegante, da mesma forma que um alçapão escondido sob o tapete.

## 3. Problema

As consultas à tabela estrangeira não se comportavam como esperado.

Os principais sintomas eram:

- Tabela estrangeira acessando o servidor remoto incorreto
- Destino de conexão inesperado
- Erros de autenticação após a correção do servidor
- Inconsistência no mapeamento de usuários
- Necessidade de validar a configuração do FDW sem expor credenciais
- Risco de consultar dados remotos desatualizados ou incorretos

O principal risco operacional era o direcionamento incorreto dos dados: um objeto local poderia estar tecnicamente válido, mas logicamente conectado à origem errada.

## 4. Investigação

A investigação começou pela verificação dos metadados locais da tabela estrangeira.

### Identificar a tabela estrangeira

```sql
SELECT
    n.nspname AS schema_name,
    c.relname AS foreign_table_name,
    ft.ftserver,
    s.srvname AS foreign_server_name
FROM pg_foreign_table ft
JOIN pg_class c
    ON c.oid = ft.ftrelid
JOIN pg_namespace n
    ON n.oid = c.relnamespace
JOIN pg_foreign_server s
    ON s.oid = ft.ftserver
WHERE n.nspname = '<schema_name>'
  AND c.relname = '<foreign_table_name>';
```

### Verificar as opções do servidor estrangeiro

```sql
SELECT
    s.srvname,
    s.srvtype,
    s.srvversion,
    fdw.fdwname,
    s.srvoptions
FROM pg_foreign_server s
JOIN pg_foreign_data_wrapper fdw
    ON fdw.oid = s.srvfdw
WHERE s.srvname = '<foreign_server_name>';
```

Essa consulta valida o host, a porta e o banco remoto configurados nas opções do servidor estrangeiro.

Exemplo das opções esperadas:

```text
host=<remote_host>, port=<remote_port>, dbname=<remote_database>
```

### Verificar os mapeamentos de usuários

```sql
SELECT
    s.srvname,
    r.rolname AS local_user,
    um.umoptions
FROM pg_user_mapping um
JOIN pg_foreign_server s
    ON s.oid = um.umserver
JOIN pg_roles r
    ON r.oid = um.umuser
WHERE s.srvname = '<foreign_server_name>';
```

O mapeamento de usuários determina qual usuário remoto será utilizado quando uma role local consultar a tabela estrangeira.

## 5. Causa Raiz

A causa raiz foi uma divergência entre o destino remoto esperado e a configuração real do servidor FDW.

A tabela estrangeira estava associada a um servidor cujas opções de conexão apontavam para o host ou a porta remota incorretos.

Após a correção do destino do servidor, surgiu um problema de autenticação porque o mapeamento de usuário ou a configuração da senha não estava alinhado ao destino correto.

Isso revelou dois problemas distintos:

1. O servidor FDW apontava para o destino incorreto.
2. As credenciais do destino correto também precisavam ser validadas.

## 6. Solução

A primeira etapa foi corrigir as opções do servidor estrangeiro.

Exemplo:

```sql
ALTER SERVER <foreign_server_name>
OPTIONS (
    SET host '<correct_host>',
    SET port '<correct_port>',
    SET dbname '<correct_database>'
);
```

Caso uma opção ainda não exista, utilize `ADD` em vez de `SET`:

```sql
ALTER SERVER <foreign_server_name>
OPTIONS (
    ADD host '<correct_host>',
    ADD port '<correct_port>',
    ADD dbname '<correct_database>'
);
```

Caso a senha do mapeamento de usuário também precise ser corrigida:

```sql
ALTER USER MAPPING FOR <local_user>
SERVER <foreign_server_name>
OPTIONS (
    SET user '<remote_user>',
    SET password '<remote_password>'
);
```

Em ambientes que utilizam `.pgpass`, valide:

- Localização do arquivo
- Proprietário do arquivo
- Permissões do arquivo
- Host
- Porta
- Banco de dados
- Usuário
- Formato da entrada de senha

Formato esperado do `.pgpass`:

```text
hostname:port:database:username:password
```

Permissão necessária:

```bash
chmod 600 ~/.pgpass
```

## 7. Validação

Após a correção, valide os metadados do servidor estrangeiro:

```sql
SELECT
    s.srvname,
    s.srvoptions
FROM pg_foreign_server s
WHERE s.srvname = '<foreign_server_name>';
```

Valide o mapeamento de usuário:

```sql
SELECT
    s.srvname,
    r.rolname AS local_user,
    um.umoptions
FROM pg_user_mapping um
JOIN pg_foreign_server s
    ON s.oid = um.umserver
JOIN pg_roles r
    ON r.oid = um.umuser
WHERE s.srvname = '<foreign_server_name>';
```

Valide o acesso à tabela estrangeira:

```sql
SELECT *
FROM <schema_name>.<foreign_table_name>
LIMIT 10;
```

Quando possível, confirme a identidade remota consultando um objeto conhecido e não sensível que permita verificar se os dados pertencem à origem esperada.

## 8. Verificações Operacionais Recomendadas

Antes de alterar uma configuração FDW em produção:

| Verificação | Finalidade |
|---|---|
| Mapeamento da tabela estrangeira para o servidor | Confirmar qual servidor é utilizado pela tabela |
| Opções do servidor | Validar host, porta e `dbname` |
| Mapeamento de usuário | Validar a associação entre usuário local e remoto |
| Conectividade | Confirmar o alcance de rede |
| Permissões | Confirmar que o usuário remoto possui o acesso necessário |
| Validação dos dados | Confirmar que a tabela retorna dados da origem esperada |
| Plano de rollback | Restaurar as opções anteriores do servidor, se necessário |

## 9. Estratégia de Rollback

Antes de alterar o servidor estrangeiro, registre a configuração atual:

```sql
SELECT
    s.srvname,
    s.srvoptions
FROM pg_foreign_server s
WHERE s.srvname = '<foreign_server_name>';
```

Exemplo de rollback:

```sql
ALTER SERVER <foreign_server_name>
OPTIONS (
    SET host '<previous_host>',
    SET port '<previous_port>',
    SET dbname '<previous_database>'
);
```

Caso o mapeamento de usuário tenha sido alterado, registre e restaure também suas opções, conforme necessário.

## 10. Resultados Esperados

| Métrica ou comportamento | Antes | Depois |
|---|---|---|
| Destino da tabela estrangeira | Servidor incorreto | Servidor correto |
| Fonte dos dados | Incorreta ou inesperada | Origem remota esperada |
| Autenticação | Com falhas ou inconsistente | Validada |
| Metadados do FDW | Desalinhados | Corrigidos |
| Confiança operacional | Baixa | Melhorada |

## 11. Lições Aprendidas

As principais lições foram:

- O nome de uma tabela estrangeira pode ocultar um mapeamento incorreto de servidor.
- A investigação de FDW deve começar pelos metadados dos catálogos do PostgreSQL.
- As opções do servidor e os mapeamentos de usuários devem ser validados em conjunto.
- A correção do servidor de destino pode revelar problemas de credenciais.
- O `.pgpass` exige permissões rígidas e correspondência exata de host, porta, banco e usuário.
- Alterações em FDW devem sempre possuir um plano de rollback.
- A validação precisa confirmar tanto a conectividade quanto a origem correta dos dados.

## 12. Impacto no Negócio

A correção da configuração FDW reduziu o risco de consultas a dados remotos incorretos.

Os benefícios para o negócio incluem:

- Integração entre bancos de dados mais confiável
- Menor risco de inconsistência de dados
- Diagnóstico mais rápido de problemas em FDW
- Maior visibilidade operacional
- Maior confiança na arquitetura de integração do PostgreSQL

## 13. Conclusão

A investigação identificou que a tabela estrangeira local estava associada a um servidor estrangeiro configurado para o destino remoto incorreto.

Após corrigir as opções do servidor e validar as credenciais do mapeamento de usuário, a conexão FDW pôde ser alinhada à origem remota esperada.

Este caso destaca a importância de verificar os metadados dos catálogos do PostgreSQL ao investigar tabelas estrangeiras e acessos a dados remotos.
