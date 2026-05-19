# Replicação Lógica no PostgreSQL

## Ambiente do Lab

Este documento foi escrito para um laboratório local com **PostgreSQL 17** rodando em máquinas virtuais Debian provisionadas com **Vagrant** e **VirtualBox**.

- [VirtualBox](https://www.virtualbox.org/) — hypervisor para rodar as VMs localmente
- [Vagrant](https://www.vagrantup.com/) — ferramenta para provisionar e gerenciar as VMs via linha de comando
- [Documentação oficial: Logical Replication (PostgreSQL 17)](https://www.postgresql.org/docs/17/logical-replication.html)

Os hostnames e IPs abaixo são referência para identificar o papel de cada máquina nos comandos. Adapte-os ao seu ambiente.

> Os prints exibidos neste documento foram gerados em ambiente de laboratório local. Usuários, senhas e configurações mostrados nas imagens são apenas ilustrativos e não devem ser replicados em produção.

| Hostname | Papel | IP (referência) |
|---|---|---|
| srv0.local | Publisher (origem) | 192.168.56.70 |
| srv1.local | Subscriber (destino) | 192.168.56.71 |

---

## Conceito

Replicação lógica envia alterações no nível de linha (INSERT, UPDATE, DELETE) de tabelas específicas, ao contrário da replicação física que copia blocos de disco inteiros. Isso permite replicar apenas o que interessa: tabelas selecionadas, com filtro de linha (WHERE), entre versões diferentes do PostgreSQL e até para outros bancos.

Casos de uso comuns:

- Banco de leitura para relatórios sem sobrecarregar o banco de produção
- Migração com zero (ou mínimo) downtime entre versões do PostgreSQL
- Replicação parcial: apenas tabelas ou linhas relevantes para um serviço
- Consolidação de dados de múltiplas origens em um banco analítico

---

## Pré-requisitos

### Versões suportadas

| Funcionalidade | Versão mínima |
|---|---|
| Replicação lógica básica | PostgreSQL 10 |
| Filtro de linha (WHERE) | PostgreSQL 15 |
| Filtro de coluna | PostgreSQL 15 |
| Replicação bidirecional | PostgreSQL 16 |

### Parâmetros do servidor (publisher)

```sql
-- Verificar configuração atual
SHOW wal_level;
SHOW max_replication_slots;
SHOW max_wal_senders;
```

O `wal_level` precisa ser `logical`. Para alterar, edite o `postgresql.conf`:

```
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

Após alterar, reinicie o PostgreSQL:

```bash
sudo systemctl restart postgresql
```

---

## Tabelas do Lab

> Os nomes das tabelas, colunas, dados e a estrutura usados aqui foram criados de forma aleatória apenas para ilustrar o funcionamento da replicação lógica. Em um ambiente real, substitua pelo modelo de dados do seu projeto.

Crie as tabelas abaixo nos dois servidores antes de configurar a replicação.

```sql
-- Executar em srv0.local E srv1.local

CREATE TABLE clientes (
    id        SERIAL PRIMARY KEY,
    nome      TEXT NOT NULL,
    email     TEXT,
    ativo     BOOLEAN DEFAULT true,
    regiao    TEXT
);

CREATE TABLE pedidos (
    id          SERIAL PRIMARY KEY,
    cliente_id  INT REFERENCES clientes(id),
    valor       NUMERIC(10,2),
    status      TEXT DEFAULT 'pendente',
    criado_em   TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE produtos (
    id      SERIAL PRIMARY KEY,
    nome    TEXT NOT NULL,
    preco   NUMERIC(10,2),
    estoque INT DEFAULT 0,
    ativo   BOOLEAN DEFAULT true
);
```

Configure `REPLICA IDENTITY FULL` no publisher antes de criar publicações com filtro WHERE em colunas fora da PK. Sem isso, UPDATE e DELETE falham com o erro `column used in the publication WHERE expression is not part of the replica identity`:

```sql
-- Executar em srv0.local (publisher)

ALTER TABLE clientes REPLICA IDENTITY FULL;
ALTER TABLE pedidos  REPLICA IDENTITY FULL;
ALTER TABLE produtos REPLICA IDENTITY FULL;
```

Insira dados de teste no publisher (srv0.local):

```sql
-- Executar em srv0.local

INSERT INTO clientes (nome, email, ativo, regiao) VALUES
    ('Ana Silva',    'ana@lab.local',    true,  'Sul'),
    ('Bruno Costa',  'bruno@lab.local',  true,  'Sudeste'),
    ('Carla Lima',   'carla@lab.local',  false, 'Sul'),
    ('Diego Matos',  'diego@lab.local',  true,  'Norte');

INSERT INTO produtos (nome, preco, estoque, ativo) VALUES
    ('Teclado',  150.00, 20, true),
    ('Mouse',     80.00, 35, true),
    ('Monitor', 1200.00,  5, true),
    ('Cabo USB',  15.00,  0, false);

INSERT INTO pedidos (cliente_id, valor, status) VALUES
    (1, 230.00, 'confirmado'),
    (1,  80.00, 'entregue'),
    (2, 1200.00, 'confirmado'),
    (3,  15.00, 'cancelado'),
    (4, 310.00, 'pendente');
```

---

## Configuração do Publisher (srv0.local)

### 1. Usuário de replicação

```sql
-- Executar em srv0.local como superusuário

CREATE USER replicador WITH REPLICATION LOGIN PASSWORD 'sua_senha_aqui';

GRANT SELECT ON TABLE clientes, pedidos, produtos TO replicador;
```

### 2. Liberar acesso no pg_hba.conf

Adicionar em `/etc/postgresql/<versao>/main/pg_hba.conf`:

```
host  replication  replicador  192.168.56.71/32  scram-sha-256
```

Recarregar:

```bash
sudo pg_ctlcluster <versao> main reload
```

Ou via SQL:

```sql
SELECT pg_reload_conf();
```

### 3. Criar a publicação

**Publicar todas as tabelas:**

```sql
CREATE PUBLICATION pub_lab FOR ALL TABLES;
```

**Publicar tabelas específicas:**

```sql
CREATE PUBLICATION pub_lab FOR TABLE clientes, pedidos, produtos;
```

**Publicar com filtro de linha (PostgreSQL 15+):**

```sql
-- Apenas clientes ativos e pedidos confirmados/entregues
CREATE PUBLICATION pub_relatorio
FOR TABLE clientes WHERE (ativo = true),
          pedidos  WHERE (status IN ('confirmado', 'entregue')),
          produtos;
```

> Tabelas com filtro WHERE em colunas fora da chave primária precisam de `REPLICA IDENTITY FULL` no publisher, caso contrário UPDATE e DELETE são bloqueados. Consulte a seção [REPLICA IDENTITY](#prevencao-replica-identity).

**Publicar apenas operações específicas (sem DELETE):**

```sql
CREATE PUBLICATION pub_append
FOR TABLE pedidos
WITH (publish = 'insert, update');
```

**Publicar colunas específicas (PostgreSQL 15+):**

```sql
CREATE PUBLICATION pub_parcial
FOR TABLE clientes (id, nome, email);
```

---

## Configuração do Subscriber (srv1.local)

### 1. Criar a assinatura

```sql
-- Executar em srv1.local

CREATE SUBSCRIPTION sub_lab
CONNECTION 'host=192.168.56.70 port=5432 dbname=postgres user=replicador password=sua_senha_aqui'
PUBLICATION pub_lab;
```

Com opções explícitas:

```sql
CREATE SUBSCRIPTION sub_lab
CONNECTION 'host=192.168.56.70 port=5432 dbname=postgres user=replicador password=sua_senha_aqui'
PUBLICATION pub_lab
WITH (
    copy_data          = true,   -- copia dados já existentes ao criar
    synchronous_commit = off     -- melhor performance; mínimo risco em crash
);
```

### 2. Verificar sincronização inicial

```sql
-- srsubstate = 'r' (ready) significa que a tabela está sincronizada
-- colunas renomeadas no PostgreSQL 17: relid -> srrelid, state -> srsubstate
SELECT srrelid::regclass AS tabela, srsubstate AS state
FROM pg_subscription_rel;
```

Com `copy_data = true`, os dados já existentes no publisher são copiados automaticamente. O resultado abaixo mostra as três tabelas populadas em srv1 sem nenhuma ação manual.

![Estado inicial no subscriber](Imagens/Replicacao%20Logica/teste1_estado_inicial.png)

---

## Monitoramento

### No publisher (srv0.local)

```sql
-- Slots de replicação
SELECT slot_name, plugin, active, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots;

-- Conexões ativas e lag
SELECT application_name, state,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;

-- Publicações criadas
SELECT pubname, puballtables, pubinsert, pubupdate, pubdelete
FROM pg_publication;

-- Tabelas e filtros de cada publicação
SELECT pubname, schemaname, tablename, rowfilter
FROM pg_publication_tables;
```

Slot de replicação com lag zero, subscriber em estado `streaming` e filtros `rowfilter` de cada publicação visíveis.

![Monitoramento no publisher — slot, stat_replication e publication_tables](Imagens/Replicacao%20Logica/teste6_monitoramento_publisher.png)

### No subscriber (srv1.local)

```sql
-- Assinaturas ativas
SELECT subname, subenabled, subpublications
FROM pg_subscription;

-- Status por tabela (PostgreSQL 17+: srrelid, srsubstate, srsublsn)
SELECT srrelid::regclass AS tabela, srsubstate AS state, srsublsn AS received_lsn
FROM pg_subscription_rel;

-- Lag de tempo
SELECT now() - pg_last_xact_replay_timestamp() AS lag_tempo;
```

Subscription ativa, as três tabelas com `srsubstate = r` (ready) e lag praticamente nulo.

![Monitoramento no subscriber — subscription, subscription_rel e lag](Imagens/Replicacao%20Logica/teste7_monitoramento_subscriber.png)

---

## Filtro de Linha (Row Filter) para Relatórios

Disponível a partir do PostgreSQL 15. O filtro é definido no publisher e determina quais linhas chegam ao subscriber.

**Por status:**

```sql
CREATE PUBLICATION pub_relatorio
FOR TABLE pedidos WHERE (status IN ('confirmado', 'entregue'));
```

**Por regiao:**

```sql
CREATE PUBLICATION pub_sul
FOR TABLE clientes WHERE (regiao = 'Sul'),
          pedidos;
```

**Apenas registros ativos com estoque:**

```sql
CREATE PUBLICATION pub_ativos
FOR TABLE produtos WHERE (ativo = true AND estoque > 0);
```

**Regras do filtro WHERE:**

O filtro aceita apenas expressões imutáveis (sem funções de sessão).

```sql
-- Correto: comparacao simples
FOR TABLE pedidos WHERE (status != 'cancelado')

-- Correto: funcao imutavel
FOR TABLE pedidos WHERE (EXTRACT(YEAR FROM criado_em) = 2025)

-- Errado: NOW() nao e imutavel em filtros de publicacao
FOR TABLE pedidos WHERE (criado_em > NOW())
```

O filtro WHERE captura linhas no momento do DML. Linhas que existiam antes da publicacao e nao se encaixam no filtro nao sao enviadas. Linhas que deixam de atender ao filtro apos um UPDATE geram um DELETE no subscriber.

### Usuarios especificos por contexto no subscriber

Uma vantagem prática do filtro de linha é que o servidor subscriber recebe apenas o subconjunto de dados definido na publicação. Com isso, é possível criar usuários no subscriber com acesso restrito às tabelas replicadas, garantindo que cada consumidor enxergue somente o que é relevante para ele.

**Exemplo: subscriber de relatório regional**

O publisher envia apenas pedidos da região Sul. No subscriber, um usuário de relatório tem acesso somente a essa tabela:

```sql
-- No publisher (srv0): publicar apenas pedidos do Sul
CREATE PUBLICATION pub_sul
FOR TABLE pedidos  WHERE (regiao = 'Sul'),
          clientes WHERE (regiao = 'Sul');

-- No subscriber (srv1): criar usuario restrito
CREATE USER relatorio_sul WITH LOGIN PASSWORD 'senha';
GRANT CONNECT ON DATABASE labdb TO relatorio_sul;
GRANT SELECT ON TABLE pedidos, clientes TO relatorio_sul;
```

O usuário `relatorio_sul` consegue fazer SELECT na tabela `pedidos`, mas os únicos dados que existem ali são os da região Sul. Ele não tem acesso a nenhuma outra tabela e não vê dados de outras regiões nem no banco.

**Exemplo: multiplos subscribers, cada um com seu escopo**

É possível ter subscribers diferentes recebendo recortes distintos do mesmo publisher:

```
Publisher (produção)
  ├── pub_sul       → subscriber_sul   (pedidos da região Sul)
  ├── pub_financeiro → subscriber_fin  (apenas tabela financeiro, todos os registros)
  └── pub_relatorio  → subscriber_rel  (clientes ativos + pedidos confirmados)
```

Cada subscriber tem seus próprios usuários e permissões, isolados uns dos outros. O publisher não precisa saber quem vai consumir cada publicação.

Esse padrão é útil para:

- **Relatórios**: réplica com apenas os dados relevantes para o time de BI, sem expor o banco de produção
- **Segregação por região ou unidade**: cada filial recebe apenas os seus dados
- **Isolamento de serviços**: um microserviço recebe somente as tabelas que ele precisa, sem acesso ao restante do schema

No exemplo abaixo, dois clientes são inseridos no publisher: Gabriel Lins (`ativo = true`) e Helena Cruz (`ativo = false`). Com a subscription apontando para `pub_relatorio` (que filtra `clientes WHERE ativo = true`), apenas Gabriel chega ao subscriber.

![Filtro WHERE — apenas cliente ativo replicado](Imagens/Replicacao%20Logica/teste5_filtro_where.png)

---

## Migração de Versão sem Downtime

A replicação lógica é uma das poucas formas de atualizar o PostgreSQL para uma versão maior (por exemplo, 15 para 17) com downtime mínimo ou zero. A replicação física não serve para isso pois exige que publisher e subscriber estejam na mesma versão principal. A lógica opera no nível de SQL, por isso funciona entre versões diferentes.

**Fluxo geral:**

```
[PG 15 — produção]  →  replicação lógica  →  [PG 17 — novo servidor]
       ↑                                              ↑
  aplicação aponta aqui               (sincronizando em paralelo)
                         ↓
              janela de manutenção curta:
              1. pausar escrita na aplicação
              2. aguardar lag zerar
              3. redirecionar conexão para PG 17
              4. desligar PG 15
```

**Requisitos obrigatórios:**

Para que a replicação lógica funcione como caminho de migração, todas as tabelas que serão replicadas precisam atender a pelo menos uma das condições abaixo:

- Ter **chave primária** (recomendado)
- Ou ter `REPLICA IDENTITY FULL` configurado (necessário para tabelas sem PK e para tabelas com filtro WHERE em colunas fora da PK)

Tabelas sem PK e sem `REPLICA IDENTITY FULL` não conseguem replicar UPDATE e DELETE, o que torna a migração incompleta e perigosa.

Verifique antes de começar:

```sql
-- Tabelas sem chave primaria no publisher
SELECT schemaname, tablename
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
  AND tablename NOT IN (
      SELECT table_name
      FROM information_schema.table_constraints
      WHERE constraint_type = 'PRIMARY KEY'
  );
```

**O que NÃO é replicado pela replicação lógica:**

| Item | Observação |
|---|---|
| DDL (CREATE, ALTER, DROP) | Qualquer alteração de estrutura precisa ser aplicada manualmente no novo servidor antes de ligar a subscription |
| Sequences | O valor atual da sequence não é sincronizado. Após a migração, ajuste manualmente com `setval()` |
| Large Objects | Não são replicados |
| Dados de tabelas sem PK e sem REPLICA IDENTITY | UPDATE e DELETE são bloqueados nessas tabelas |

**Passo a passo resumido:**

```sql
-- 1. No novo servidor (PG 17): criar o schema identico ao PG 15
--    (use pg_dump --schema-only para exportar a estrutura)

-- 2. No PG 15 (publisher): criar publicacao de todas as tabelas
CREATE PUBLICATION pub_migracao FOR ALL TABLES;

-- 3. No PG 17 (subscriber): criar a subscription
CREATE SUBSCRIPTION sub_migracao
CONNECTION 'host=<IP_PG15> port=5432 dbname=<banco> user=replicador password=<senha>'
PUBLICATION pub_migracao
WITH (copy_data = true);

-- 4. Acompanhar o progresso da copia inicial
SELECT srrelid::regclass AS tabela, srsubstate AS state
FROM pg_subscription_rel;
-- Aguardar todas com srsubstate = 'r'

-- 5. Monitorar o lag ate zerar
SELECT now() - pg_last_xact_replay_timestamp() AS lag;

-- 6. Janela de corte: pausar aplicacao, confirmar lag zero,
--    redirecionar conexoes para o PG 17

-- 7. Apos corte: ajustar sequences no PG 17
SELECT 'SELECT setval(' || quote_literal(sequence_name) ||
       ', (SELECT MAX(' || quote_ident(column_name) ||
       ') FROM ' || quote_ident(table_name) || '));'
FROM information_schema.columns
WHERE column_default LIKE 'nextval%';
```

---

## Alterar Publicacoes e Assinaturas

### Adicionar tabela

```sql
ALTER PUBLICATION pub_lab ADD TABLE produtos;
```

### Remover tabela

```sql
ALTER PUBLICATION pub_lab DROP TABLE produtos;
```

### Alterar filtro de linha

```sql
ALTER PUBLICATION pub_relatorio
ALTER TABLE pedidos WHERE (status = 'entregue');
```

### Atualizar subscriber apos mudancas na publicacao

```sql
-- Executar em srv1.local sempre que adicionar tabelas ao publisher
ALTER SUBSCRIPTION sub_lab REFRESH PUBLICATION;
```

### Pausar e retomar

```sql
ALTER SUBSCRIPTION sub_lab DISABLE;

ALTER SUBSCRIPTION sub_lab ENABLE;
```

---

## Remover Replicacao

```sql
-- Em srv1.local: remove a assinatura e o slot no publisher
DROP SUBSCRIPTION sub_lab;

-- Em srv0.local: remove a publicacao
DROP PUBLICATION pub_lab;

-- Em srv0.local: remover slot manualmente se o subscriber caiu sem cleanup
SELECT pg_drop_replication_slot('sub_lab');
```

Slots inativos acumulam WAL e podem esgotar o disco do publisher. Monitore:

```sql
SELECT slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag
FROM pg_replication_slots
WHERE active = false;
```

---

## Resolucao de Conflitos

Conflitos ocorrem quando o subscriber ja tem uma linha com a mesma chave primaria que esta sendo inserida. O comportamento padrao e parar e logar o erro.

### Ver o erro

```sql
-- Em srv1.local
SELECT * FROM pg_stat_subscription_stats;
```

O log do PostgreSQL tambem mostra:

```
ERROR: duplicate key value violates unique constraint "clientes_pkey"
DETAIL: Key (id)=(1) already exists.
```

### Resolver

```sql
-- Opcao 1: remover o registro conflitante no subscriber
DELETE FROM clientes WHERE id = 1;

-- Opcao 2: pular a transacao problematica (PostgreSQL 15+)
SELECT pg_replication_origin_advance('pg_<subid>', '<lsn>');
```

### Prevencao: REPLICA IDENTITY

O `REPLICA IDENTITY` define quais colunas o PostgreSQL grava no WAL para identificar a linha original em operações UPDATE e DELETE. Existem dois casos em que o padrão (somente a PK) é insuficiente:

**Caso 1: tabela sem chave primária**

Sem PK, o PostgreSQL não consegue identificar a linha no subscriber para aplicar UPDATE ou DELETE. A solução é `REPLICA IDENTITY FULL`, que grava todas as colunas no WAL:

```sql
ALTER TABLE logs REPLICA IDENTITY FULL;
```

**Caso 2: publicação com filtro WHERE em coluna fora da PK**

Quando uma publicação tem `WHERE (coluna = valor)` e essa coluna não faz parte da PK, o PostgreSQL precisa avaliar o filtro na linha *antes* da alteração para saber se ela estava sendo publicada. Sem `REPLICA IDENTITY FULL`, ele não tem essa informação disponível e bloqueia o UPDATE e DELETE com o erro:

```
ERROR: cannot update table "clientes"
DETAIL: Column used in the publication WHERE expression is not part of the replica identity.
```

```sql
-- Aplicar nas tabelas cujas publicacoes filtram por colunas fora da PK
ALTER TABLE clientes REPLICA IDENTITY FULL;  -- filtra por: ativo
ALTER TABLE pedidos  REPLICA IDENTITY FULL;  -- filtra por: status
ALTER TABLE produtos REPLICA IDENTITY FULL;
```

Execute no publisher **antes** de criar as publicações com filtro.

**Verificar o REPLICA IDENTITY atual de cada tabela:**

```sql
SELECT relname AS tabela,
       CASE relreplident
           WHEN 'd' THEN 'default (PK)'
           WHEN 'f' THEN 'full'
           WHEN 'i' THEN 'index'
           WHEN 'n' THEN 'nothing'
       END AS replica_identity
FROM pg_class
WHERE relnamespace = 'public'::regnamespace
  AND relkind = 'r'
ORDER BY relname;
```

**Verificar tabelas sem PK:**

```sql
SELECT schemaname, tablename
FROM pg_tables
WHERE schemaname = 'public'
  AND tablename NOT IN (
      SELECT table_name
      FROM information_schema.table_constraints
      WHERE constraint_type = 'PRIMARY KEY'
  );
```

---

## Exercicio Completo do Lab

Execute em ordem para subir a replicacao do zero.

### Passo 1: srv0.local (publisher)

```sql
-- Criar usuario
CREATE USER replicador WITH REPLICATION LOGIN PASSWORD 'sua_senha_aqui';
GRANT SELECT ON TABLE clientes, pedidos, produtos TO replicador;

-- REPLICA IDENTITY FULL: necessario pois pub_relatorio filtra
-- por colunas fora da PK (ativo, status)
ALTER TABLE clientes REPLICA IDENTITY FULL;
ALTER TABLE pedidos  REPLICA IDENTITY FULL;
ALTER TABLE produtos REPLICA IDENTITY FULL;

-- Criar publicacao com filtro
CREATE PUBLICATION pub_relatorio
FOR TABLE clientes WHERE (ativo = true),
          pedidos  WHERE (status IN ('confirmado', 'entregue')),
          produtos;
```

Editar `/etc/postgresql/<versao>/main/pg_hba.conf`:

```
host  replication  replicador  192.168.56.71/32  scram-sha-256
```

```sql
SELECT pg_reload_conf();
```

### Passo 2: srv1.local (subscriber)

```sql
-- As tabelas ja foram criadas no inicio do lab

-- Criar assinatura
CREATE SUBSCRIPTION sub_lab
CONNECTION 'host=192.168.56.70 port=5432 dbname=postgres user=replicador password=sua_senha_aqui'
PUBLICATION pub_lab
WITH (copy_data = true);

-- Verificar estado
SELECT srrelid::regclass AS tabela, srsubstate AS state FROM pg_subscription_rel;
```

### Passo 3: testar no publisher

```sql
-- Em srv0.local: inserir novo cliente ativo
INSERT INTO clientes (nome, email, ativo, regiao)
VALUES ('Fernanda Rocha', 'fernanda@lab.local', true, 'Sul');
```

![INSERT no publisher — Fernanda Rocha inserida](Imagens/Replicacao%20Logica/teste2_insert_replicado.png)

```sql
-- Em srv0.local: atualizar regiao de um cliente
UPDATE clientes SET regiao = 'Centro-Oeste' WHERE nome = 'Diego Matos';
```

![UPDATE no publisher — Diego Matos com regiao Centro-Oeste no subscriber](Imagens/Replicacao%20Logica/teste3_update_replicado.png)

```sql
-- Em srv0.local: remover pedido cancelado
DELETE FROM pedidos WHERE status = 'cancelado';
```

![DELETE no publisher — pedido cancelado removido no subscriber](Imagens/Replicacao%20Logica/teste4_delete_replicado.png)

### Passo 4: conferir no subscriber

```sql
-- Em srv1.local: dados devem ter chegado
SELECT * FROM clientes;
SELECT * FROM pedidos;
```

---

## Referencia Rapida

| Acao | Comando |
|---|---|
| Criar publicacao completa | `CREATE PUBLICATION nome FOR ALL TABLES` |
| Criar publicacao filtrada | `CREATE PUBLICATION nome FOR TABLE t WHERE (cond)` |
| Criar assinatura | `CREATE SUBSCRIPTION nome CONNECTION '...' PUBLICATION pub` |
| Ver publicacoes | `SELECT * FROM pg_publication` |
| Ver tabelas publicadas | `SELECT * FROM pg_publication_tables` |
| Ver assinaturas | `SELECT * FROM pg_subscription` |
| Ver lag (publisher) | `SELECT * FROM pg_stat_replication` |
| Ver lag (subscriber) | `SELECT now() - pg_last_xact_replay_timestamp()` |
| Atualizar apos ADD TABLE | `ALTER SUBSCRIPTION nome REFRESH PUBLICATION` |
| Pausar subscriber | `ALTER SUBSCRIPTION nome DISABLE` |
| Remover assinatura | `DROP SUBSCRIPTION nome` |
