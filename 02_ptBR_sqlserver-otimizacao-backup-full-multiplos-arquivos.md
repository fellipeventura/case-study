# Estudo de Caso: Otimização de Backup FULL no SQL Server com Múltiplos Arquivos

## 1. Visão Geral

Este estudo de caso documenta a otimização de backups FULL nativos do SQL Server por meio da avaliação de **backup em múltiplos arquivos (striping)**. A análise combinou testes operacionais e revisão da configuração da rotina de manutenção.

**Resultados registrados:** um teste comparativo apresentou aproximadamente **12 minutos usando 2 arquivos, 9 minutos com 4 arquivos e 8 minutos com 8 arquivos**. Em um ambiente separado de homologação, a duração do backup FULL caiu de aproximadamente **1 hora e 50 minutos para 1 hora** após otimizações. Os registros disponíveis não permitem afirmar que somente a quantidade de arquivos foi responsável pelo ganho em homologação.

> **Confidencialidade:** nomes de bancos, servidores, diretórios e clientes foram omitidos. Os tempos apresentados são observações históricas aproximadas. Este estudo não afirma a execução de testes completos de restauração sem evidência correspondente.

## 2. Contexto

O ambiente utiliza Microsoft SQL Server com backups FULL agendados por meio da procedure `dbo.DatabaseBackup`, da solução de manutenção de Ola Hallengren.

Com o aumento do volume dos bancos, a duração dos backups passou a ser uma oportunidade de melhoria da janela operacional. O objetivo era verificar se a distribuição da gravação de um único conjunto de backup entre vários arquivos poderia aumentar a vazão sem alterar a estratégia de recuperação.

Foram avaliados:

- Quantidade de arquivos de destino;
- Tempo total de execução;
- Compressão e checksums;
- Tamanho mínimo para aplicação do backup multifile;
- Validação operacional do conjunto de backup.

## 3. Problema

A rotina de backup FULL apresentava potencial de otimização, sobretudo em bancos maiores. Backups demorados podem disputar recursos com outras manutenções e reduzir a previsibilidade das janelas.

Aumentar o número de arquivos pode melhorar partes do fluxo de gravação, porém os ganhos dependem de leitura na origem, armazenamento no destino, CPU, compressão e concorrência.

A pergunta do estudo foi: **aumentar o número de arquivos reduz a duração do backup neste ambiente e em que ponto os ganhos adicionais diminuem?**

## 4. Investigação e Metodologia

Foram registrados os seguintes resultados ao comparar **2, 4 e 8 arquivos**:

| Quantidade de arquivos | Duração observada | Variação em relação a 2 arquivos |
| ---: | ---: | ---: |
| 2 | ~12 min | Referência |
| 4 | ~9 min | ~25,0% menor |
| 8 | ~8 min | ~33,3% menor |

A passagem de **4 para 8 arquivos reduziu cerca de 1 minuto adicional**, equivalente a aproximadamente **11,1%** do tempo de 4 arquivos. O resultado sugere ganho marginal decrescente.

### Limitações dos registros

As anotações históricas disponíveis não incluem tamanho exato do banco testado, datas, quantidade de repetições, arquitetura de armazenamento, carga concorrente ou comprovação de que apenas o número de arquivos variou. Portanto, são **observações operacionais**, não um benchmark controlado.

Para ensaios futuros, registrar: tamanho do banco; tamanho do backup; horário de início/fim; número de arquivos; compressão; checksum; destino; utilização de CPU; latência e throughput de I/O; processos concorrentes; e, quando possível, três execuções em condições comparáveis.

## 5. Explicação Técnica

No backup distribuído do SQL Server, os vários arquivos pertencem a **um único conjunto de backup**. Não são cópias independentes. Para restaurar esse conjunto normalmente, **todos os arquivos/stripes são necessários**.

O striping pode melhorar o aproveitamento da gravação, mas não garante aceleração proporcional à quantidade de arquivos. Oito arquivos no mesmo volume saturado, por exemplo, podem apresentar ganhos reduzidos.

A compressão diminui o volume gravado, com consumo adicional de processamento. `CHECKSUM` ajuda a detectar determinados problemas de integridade durante o backup, mas não substitui testes de restauração.

## 6. Estratégia de Configuração

A estratégia registrada para o ambiente utiliza **até 8 arquivos para backups suficientemente grandes**, evitando essa distribuição em bancos pequenos. O limite registrado na configuração mais recente foi **102.400 MB (100 GiB)** para o uso de múltiplos arquivos.

O comando abaixo é um **exemplo sanitizado**, não a cópia literal de um job de produção. Revisar diretório, seleção de bancos e versão da procedure antes de executar.

```sql
-- Exemplo ilustrativo: adaptar e validar antes de executar.
EXEC dbo.DatabaseBackup
    @Databases = 'USER_DATABASES',
    @Directory = 'X:\SQLBackups',
    @BackupType = 'FULL',
    @NumberOfFiles = 8,
    @MinBackupSizeForMultipleFiles = 102400, -- MB: 100 GiB
    @Compress = 'Y',
    @Checksum = 'Y',
    @Verify = 'Y',
    @LogToTable = 'Y';
```

O parâmetro `@MinBackupSizeForMultipleFiles` deve ser usado em conjunto com `@NumberOfFiles`. Para tamanhos abaixo do limite aplicável, a procedure utiliza apenas um arquivo. Confirmar a compatibilidade com a versão instalada.

**Atenção:** Não combinar `@MaxFileSize` com a suposição de quantidade fixa de arquivos sem verificar previamente o comportamento e as restrições da versão instalada. O cenário deste case está baseado em `@NumberOfFiles` e `@MinBackupSizeForMultipleFiles`.

## 7. Resultado em Ambiente Separado de Homologação

Após otimizações da rotina de backup FULL em um ambiente de homologação, foi observado:

| Métrica | Antes | Depois | Variação |
| --- | ---: | ---: | ---: |
| Tempo do backup FULL | ~110 min (1h50) | ~60 min (1h) | ~50 min a menos (~45,5%) |

Esse ganho representa redução importante da janela operacional. Entretanto, os registros disponíveis **não isolam o efeito** da quantidade de arquivos em relação a compressão, armazenamento e carga no momento do teste. Trata-se de um **comparativo operacional antes/depois**, não de prova de causalidade exclusiva do striping.

## 8. Validação e Recuperabilidade

Reduzir a duração do backup não basta: é necessário garantir sua recuperabilidade. Backups distribuídos dependem da disponibilidade de **todos os arquivos do conjunto**. Mesmo `RESTORE VERIFYONLY` não substitui a restauração completa.

Procedimento recomendado:

1. Conferir sucesso da job, erros e avisos no histórico;
2. Confirmar quantidade de arquivos e acesso a todos eles;
3. Consultar `msdb.dbo.backupset` e `msdb.dbo.backupmediafamily`;
4. Executar `RESTORE VERIFYONLY` sobre todos os arquivos, quando apropriado;
5. Testar periodicamente a restauração integral em instância não produtiva e validar integridade;
6. Validar separadamente a cadeia de recuperação com backups diferenciais e de log, quando aplicáveis.

Consulta de apoio para analisar duração e número de famílias do backup:

```sql
SELECT TOP (30)
    bs.database_name,
    bs.backup_start_date,
    bs.backup_finish_date,
    DATEDIFF(SECOND, bs.backup_start_date, bs.backup_finish_date) AS duration_seconds,
    CAST(bs.backup_size / 1048576.0 AS decimal(18,2)) AS backup_size_mb,
    CAST(bs.compressed_backup_size / 1048576.0 AS decimal(18,2)) AS compressed_size_mb,
    COUNT(DISTINCT bmf.family_sequence_number) AS media_families
FROM msdb.dbo.backupset AS bs
JOIN msdb.dbo.backupmediafamily AS bmf
    ON bmf.media_set_id = bs.media_set_id
WHERE bs.type = 'D' -- Backup FULL de banco de dados
GROUP BY
    bs.backup_set_id, bs.database_name,
    bs.backup_start_date, bs.backup_finish_date,
    bs.backup_size, bs.compressed_backup_size
ORDER BY bs.backup_start_date DESC;
```

> A consulta apresenta o histórico registrado no `msdb`, mas **não comprova** que os arquivos ainda existem, estão íntegros ou já foram restaurados com sucesso.

## 9. Resultados e Impacto no Negócio

**Resultados registrados:**

- Teste com 2 versus 8 arquivos: **~12 para ~8 minutos**, redução de **~33,3%**;
- Teste com 4 versus 8 arquivos: **~9 para ~8 minutos**, benefício incremental menor;
- Ambiente separado de homologação: **~110 para ~60 minutos**, redução de **~45,5%** na janela do backup FULL após otimização.

**Benefícios operacionais possíveis, não mensurados separadamente:**

- Maior previsibilidade da janela de backup;
- Mais flexibilidade para outras manutenções;
- Potencial redução de concorrência operacional quando há capacidade de I/O;
- Critérios objetivos para selecionar a quantidade de arquivos.

Não se afirma melhoria em tempo de restauração, latência da aplicação, RPO ou RTO, pois esses resultados não foram medidos nos registros originais.

## 10. Lições Aprendidas

- Testar striping na arquitetura de armazenamento real;
- Não assumir que o dobro de arquivos reduz o tempo pela metade;
- Separar comparativos curtos de testes e resultados gerais de um ambiente;
- Aplicar múltiplos arquivos seletivamente em bancos maiores;
- Preservar todo o conjunto de arquivos e executar testes de restauração;
- Coletar histórico das jobs e evidências do `msdb` imediatamente após cada alteração.

## 11. Conclusão

O estudo demonstrou que a configuração da saída de backups FULL pode influenciar de forma relevante a duração da execução no ambiente observado. O teste entre 2, 4 e 8 arquivos apresentou redução aproximada de 12 para 8 minutos, com ganho marginal decrescente. Em uma otimização separada, a janela de homologação caiu de aproximadamente 110 para 60 minutos.

A principal evolução foi adotar uma abordagem orientada por medições e tratar **performance do backup e capacidade de restauração como validações independentes**.

## Referências

- [SQL Server Maintenance Solution — DatabaseBackup](https://ola.hallengren.com/sql-server-backup.html)
- [Microsoft Learn — Visão geral de backup do SQL Server](https://learn.microsoft.com/pt-br/sql/relational-databases/backup-restore/backup-overview-sql-server)
- [Microsoft Learn — RESTORE VERIFYONLY](https://learn.microsoft.com/pt-br/sql/t-sql/statements/restore-statements-verifyonly-transact-sql)
