# Comandos executados

Horários registrados em UTC−03:00. Saídas reais estão em `evidencias/`. Para este workspace, os containers existentes pertenciam ao projeto Compose `lab1`; por isso foram usados nomes de container, evitando criar outro projeto a partir do nome da pasta.

## Inspeção e baseline

1. `git status --short; git rev-parse --show-toplevel; docker compose ps -a; docker compose config --volumes; docker volume ls` — antes do incidente. O Compose no diretório não listou containers por padrão; `docker compose -p lab1 ps` identificou os ativos.
2. `docker volume inspect lab1_primary_data lab1_replica_data` e `docker ps -a --filter label=com.docker.compose.project=lab1` — volumes persistentes existentes e containers ativos.
3. Leitura somente leitura dos volumes: `docker run --rm --entrypoint sh -v lab1_primary_data:/data:ro postgres:16 -c 'test -s /data/PG_VERSION; cat /data/PG_VERSION; ls -1 /data'` e o mesmo comando com `lab1_replica_data`.
4. Consultas SQL de verificação em `lab1-primary-1` e `lab1-replica-1`: recovery, existência de tabelas, contagem de eventos e `pg_stat_replication`. As tabelas ainda não existiam.
5. Inicialização no primário: `CREATE TABLE clientes (id serial PRIMARY KEY, nome text, email text);`; `INSERT INTO clientes (nome, email) SELECT 'cliente ' || g, 'c' || g || '@loja.com' FROM generate_series(1, 1000) g;`; `CREATE TABLE eventos (id bigserial PRIMARY KEY, ts timestamptz DEFAULT now());`.
6. Validação e baseline: queries `pg_is_in_recovery()`, `count(*)` de clientes/eventos, `max(id)` e `pg_stat_replication`. Evidências: `01-baseline-primary.txt`, `02-baseline-replica.txt`.

## Incidente e failover

| Horário (UTC−03:00) | Comando/ação | Evidência |
|---|---|---|
| 10:25:38.853 | `docker stop lab1-replica-1` | `03-horario-isolamento-replica.txt`, `04-stop-replica.txt` |
| 10:25:39.541 | `docker exec lab1-primary-1 psql -v ON_ERROR_STOP=1 -U postgres -d postgres -c "INSERT INTO eventos (ts) SELECT now() FROM generate_series(1, 50);" -c "SELECT now(), count(*), max(id) FROM eventos;"` | `05-insert-50-M1.txt` |
| 10:25:57.048 (T0) | `docker kill lab1-primary-1` | `06-T0-primary-crash.txt`, `07-kill-primary.txt` |
| 10:26:37 | `docker start lab1-replica-1`; consultas de recovery, contagem e max ID | `08-start-replica.txt`, `09-replica-before-promotion.txt` |
| 10:26:50.807 | `docker exec lab1-replica-1 psql -v ON_ERROR_STOP=1 -U postgres -d postgres -c "SELECT pg_promote();" -c "SELECT now(), pg_is_in_recovery();" -c "SELECT count(*), max(id) FROM eventos;" -c "SELECT count(*) FROM clientes;"` | `10-promote-and-M2.txt` |
| 10:27:03.616 (T1) | `docker exec lab1-replica-1 psql -v ON_ERROR_STOP=1 -U postgres -d postgres -c "INSERT INTO eventos DEFAULT VALUES RETURNING id;"` | `11-first-write-new-primary.txt`, `12-T1-write-accepted.txt` |
| 10:27:28.660 | Consultas finais de recovery, contagem e clientes; `docker ps -a` | `13-post-failover-validation.txt` |

## Teste síncrono isolado

1. Criado volume exclusivo: `docker volume create lab1_sync_replica_data`.
2. Tentativa de criar standby com `pg_basebackup` via `docker run ... --entrypoint bash postgres:16 -c 'pg_basebackup ...; exec postgres'` falhou ao iniciar: permissões 0755. Na reinicialização, o comando repetiu o basebackup e encontrou diretório não vazio. Nenhum volume original foi alterado.
3. Correção somente do volume de teste: `docker run --rm --user root -v lab1_sync_replica_data:/data --entrypoint chmod postgres:16 700 /data`. O `postgresql.auto.conf` do clone foi inspecionado para confirmar `primary_conninfo` e slot `sync_test_slot`.
4. Container de teste recriado sobre a cópia já existente: `docker rm lab1-sync-replica`; `docker run -d --name lab1-sync-replica --network lab1_default --user postgres -e PGPASSWORD=repl -v lab1_sync_replica_data:/var/lib/postgresql/data --entrypoint postgres postgres:16`.
5. Consultas confirmaram standby em recovery, mesma contagem de eventos e conexão `streaming`/`async`. Evidência: `16-sync-standby-validation.txt`.
6. Configuração: `ALTER SYSTEM SET synchronous_standby_names = '*';` e `SELECT pg_reload_conf();`. Verificação com `SHOW synchronous_standby_names;` e `SELECT application_name, state, sync_state FROM pg_stat_replication;` mostrou `*` e `sync`. Evidências: `17-enable-sync.txt`, `18-sync-state.txt`.
7. Standby parado às 10:34:44.548: `docker stop lab1-sync-replica`. Evidências: `19-sync-standby-stop-time.txt`, `20-stop-sync-standby.txt`.
8. Tentativa de escrita: `docker exec lab1-replica-1 psql -v ON_ERROR_STOP=1 -U postgres -d postgres -c "INSERT INTO eventos DEFAULT VALUES;"`. Consulta em `pg_stat_activity` mostrou `wait_event_type = 'IPC'`, `wait_event = 'SyncRep'`. Evidência: `26-sync-wait-observed.txt`.
9. Cancelamento: `SELECT pg_cancel_backend(105);`. O comando informou que o commit ocorreu localmente, mas poderia não ter sido replicado. Após `docker start lab1-sync-replica`, ambos os nós tinham 2 eventos. Evidências: `21-cancel-sync-write.txt`, `22-after-cancel-primary.txt`, `23-after-cancel-standby.txt`.
10. Reversão: `ALTER SYSTEM RESET synchronous_standby_names;` e `SELECT pg_reload_conf();`. Verificação final mostrou parâmetro vazio, recovery `false` no nó promovido e replicação `async`. Evidências: `24-reset-sync.txt`, `25-post-reset-validation.txt`.

## Estado final observado

- `lab1-primary-1`: parado; volume original preservado.
- `lab1-replica-1`: nó promovido, atende na porta 5443.
- `lab1-sync-replica`: standby temporário ativo, sem porta publicada no host.
- `lab1_sync_replica_data`: volume de teste separado, mantido para preservar a topologia validada.
- Configuração síncrona: removida; replicação final `async`.
