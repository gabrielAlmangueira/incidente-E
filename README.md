# Incidente E — Réplica isolada e queda do primário

## 1. Identificação

- Integrante(s): Victor de Almeida Mangueira [@VictorAlmeidaM](https://github.com/VictorAlmeidaM) e Gabriel de Almeida Mangueira [@gabrielAlmangueira](https://github.com/gabrielAlmangueira).
- Cenário: E — réplica isolada e queda do primário.
- Caminho: Docker Compose, projeto `lab1`, PostgreSQL 16.
- Execução: 14/11/2026; horários locais UTC−03:00, salvo indicação contrária.

## 2. Resumo do incidente

A réplica foi parada enquanto o primário continuou aceitando gravações. Foram inseridos 50 eventos no primário e, antes de a réplica voltar a receber esses registros, o primário foi derrubado. A réplica foi iniciada e promovida manualmente. Os 50 eventos não estavam no novo primário, embora os 1000 clientes estivessem preservados.

## 3. Linha do tempo

| Horário (UTC−03:00) | Ação | Resultado |
|---|---|---|
| 10:22:50 | Baseline | Primário fora de recovery; réplica em recovery; 1000 clientes em ambos; 0 eventos; estado `async`. |
| 10:25:38.853 | Parada da réplica | Réplica isolada do primário. |
| 10:25:39.541 | Inserção dos 50 eventos | Primário: `count(*) = 50`, `max(id) = 50` (M1). |
| 10:25:57.048 (T0) | Queda abrupta do primário | Container antigo parado com código de saída 137. |
| 10:26:37 | Inicialização da réplica | Ainda em recovery; `count(*) = 0`, `max(id) = NULL`. |
| 10:26:50.807 | Promoção da réplica | `pg_is_in_recovery() = false`; M2 = 0 eventos. |
| 10:27:03.616 (T1) | Primeira escrita no novo primário | Inserção aceita, ID 1. |
| 10:27:28.660 | Validação pós-failover | Novo primário fora de recovery, 1 evento e 1000 clientes. Primário antigo continua parado. |
| 10:33:01 | Standby temporário para teste síncrono | Conectado em recovery; replicação inicialmente `async`. |
| Sem horário individual salvo | Escrita com standby parado | Sessão observada em `wait_event = 'SyncRep'`. O backend foi cancelado; PostgreSQL avisou que a transação já havia sido commitada localmente. |
| Após religar o standby | Conferência do teste síncrono | A linha commitada localmente chegou ao standby; ambos tinham 2 eventos. |
| Após o teste | Reversão | `synchronous_standby_names` vazio; estado voltou a `async`. |

## 4. Medições

- **RTO:** T1 − T0 = 10:27:03.615682 − 10:25:57.048476 = **66,567 segundos** (aproximadamente 1 min 6,57 s). Mede a interrupção simulada até a primeira escrita aceita no novo primário; não inclui detecção automática, alteração de DNS ou reconexão de uma aplicação real.
- **RPO:** M1 = 50 (50 linhas; `max(id) = 50`) e M2 = 0 linhas (`max(id) = NULL`) antes da primeira escrita após promoção. **Perda observada: 50 eventos.**
- **Failover:** `lab1-replica-1`, na porta 5443, foi promovido às 10:26:50.807. A confirmação foi `pg_is_in_recovery() = false`. O container `lab1-primary-1` permaneceu parado para evitar split-brain.
- **Estado dos clientes:** 1000 linhas antes e depois da promoção.
- As consultas e saídas estão em [evidencias 01](evidencias/01-baseline-primary.txt) e [evidencias 13](evidencias/13-post-failover-validation.txt).
## 5. Causa raiz

A replicação configurada era assíncrona. Com a réplica parada, o primário aceitou e confirmou 50 inserções sem que elas fossem enviadas à réplica. A promoção recuperou apenas o estado que havia chegado ao standby; replicação não é backup e não consegue recuperar alterações que nunca recebeu.

## 6. Correção aplicada e prova

- Promoção manual do standby e isolamento do primário antigo: o novo primário aceitou uma gravação; `pg_is_in_recovery()` confirmou `false`.
- Teste síncrono sem apagar os volumes originais: foi criado o volume novo `lab1_sync_replica_data` e um standby temporário separado. Com `synchronous_standby_names = '*'`, `pg_stat_replication.sync_state` mostrou `sync`.
- Com o standby parado, a sessão de inserção aguardou em `SyncRep`. Ao cancelar o backend, o PostgreSQL informou que a transação já estava commitada localmente, porém talvez ainda não replicada. Após religar o standby, a linha estava nos dois nós. Portanto, cancelar a espera não deve ser interpretado como rollback garantido.
- A configuração foi revertida com `ALTER SYSTEM RESET synchronous_standby_names`; a verificação final mostrou valor vazio e estado `async`.

O standby temporário `lab1-sync-replica` foi mantido ativo como standby do nó promovido. Ele não é gerenciado pelo Compose e não publica uma porta no host. Os volumes originais `lab1_primary_data` e `lab1_replica_data` não foram apagados.

## 7. Comparação com serviço gerenciado

Um serviço gerenciado com failover Multi-AZ pode automatizar detecção, fencing e promoção, reduzindo o tempo operacional de failover. Replicação síncrona pode evitar a perda de commits confirmados, ao custo de latência e de depender da disponibilidade do standby. Para recuperação de alterações acidentais ou corrupção lógica, backups e recuperação point-in-time (PITR) são complementares: uma réplica reproduz alterações destrutivas e não substitui backup.

## 8. Prevenção e lições

- Alertar quando não houver standby conectado, quando `state` deixar de ser `streaming` ou quando o atraso de replay exceder o limite acordado.
- Definir RPO e RTO com base na aplicação e testar restauração e failover regularmente.
- Usar fencing para impedir que o antigo primário aceite escritas após promoção.
- Manter backups independentes e testar PITR; separar permissões de aplicação e operação.
- Replicação síncrona favorece dados que não podem perder commits confirmados; assíncrona favorece latência e disponibilidade quando algum RPO é aceitável.

## 9. Respostas do cenário

- **RPO e síncrona:** com standby síncrono e `synchronous_commit` habilitado, commits reconhecidos aguardam a confirmação configurada, podendo oferecer RPO zero para esses commits no cenário testado. Se o standby desconectar, as gravações podem aguardar; paga-se latência e disponibilidade.
- **Onde usar:** síncrona quando a perda de transações confirmadas é inaceitável e a latência entre nós é baixa; assíncrona quando menor latência/disponibilidade importa mais e a aplicação tolera um RPO diferente de zero.
- **Alarme:** monitorar ausência/estado da conexão em `pg_stat_replication`, atraso de replay e indisponibilidade do standby, com limiares coerentes com o RPO.
- **Limite deste ensaio:** o RTO medido não representa failover automático nem o tempo completo de recuperação de uma aplicação; foi medido até a primeira escrita aceita manualmente.
