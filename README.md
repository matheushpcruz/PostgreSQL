# PostgreSQL

Repositório com documentações, labs e scripts que desenvolvi e mantenho sobre administração e monitoramento do PostgreSQL. O conteúdo é voltado para estudo e referência prática: cada tópico tem explicação conceitual, configuração passo a passo e, quando aplicável, um ambiente para reproduzir o lab localmente.

**Última revisão:** 19/05/2026


## Estrutura

### Alta-Disponibilidade

Documentação sobre cluster de alta disponibilidade com Patroni. Cobre instalação, configuração do arquivo `patroni.yml`, integração com etcd como DCS, failover automático, failover manual e monitoramento do cluster com `patronictl`.

| Conteúdo | PostgreSQL | Última revisão |
|---|---|---|
| Cluster Patroni | 17 | 19/05/2026 |


### Backup

Documentação sobre backup com Barman. Cobre instalação, configuração de backup via streaming e via rsync com WAL archiving, retenção, restauração e monitoramento dos backups com `barman check`.

| Conteúdo | PostgreSQL | Última revisão |
|---|---|---|
| Backup com Barman | 17 | 19/05/2026 |


### Extensions

Binários de extensões compilados para instalação manual em ambientes sem acesso à internet ou sem repositórios oficiais disponíveis. Cada subpasta contém os arquivos `.so` e `.sql` prontos para copiar para o diretório de extensões do PostgreSQL.

| Conteúdo | PostgreSQL | Última revisão |
|---|---|---|
| pg_stat_statements | 14 | 19/05/2026 |
| pgvector | 14 | 19/05/2026 |


### Ferramentas

Documentação sobre ferramentas complementares ao PostgreSQL.

**PGBouncer:** connection pooler que fica entre a aplicação e o banco, reduzindo o custo de abertura de conexões. A documentação cobre os três modos de pool (session, transaction, statement), configuração do `pgbouncer.ini` e monitoramento via console administrativo.

| Conteúdo | PostgreSQL | Última revisão |
|---|---|---|
| PGBouncer | Qualquer versão | 19/05/2026 |


### Monitoramento

Documentação sobre ferramentas e sistemas de monitoramento de queries e comportamento do banco.

**PGBadger:** analisador de logs que gera relatórios HTML com queries lentas, erros, conexões, locks, checkpoints e arquivos temporários. Inclui lab Vagrant para reproduzir o ambiente e tirar prints.

**pg_stat_statements:** extensão oficial que rastreia estatísticas de execução de todas as queries. Cobre instalação, configuração e queries úteis para identificar gargalos. Inclui o **Query Analytics**, um sistema que desenvolvi para preservar o histórico diário: uma procedure que faz snapshot antes de cada reset, uma tabela com 14 dias de retenção e uma view unificada que combina dados realtime e histórico para uso com Grafana.

| Conteúdo | PostgreSQL | Última revisão |
|---|---|---|
| PGBadger | 17 | 19/05/2026 |
| pg_stat_statements + Query Analytics | 17 | 19/05/2026 |


### Replicacao

Documentação sobre replicação lógica. Cobre a diferença entre replicação física e lógica, configuração do publisher e subscriber, criação de publicações e subscrições, monitoramento com `pg_stat_replication` e casos de uso como migração de dados e replicação seletiva de tabelas.

| Conteúdo | PostgreSQL | Última revisão |
|---|---|---|
| Replicação Lógica | 17 | 19/05/2026 |
