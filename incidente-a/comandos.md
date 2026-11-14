# Comandos executados

Todos os comandos foram executados em `~/lab1/incidente-a`.

## 1. Registro do horário inicial + Verificação inicial da tabela

Horário registrado em `evidencias/00-horario-inicial.txt`.

```bash
date '+%Y-%m-%d %H:%M:%S %Z' \
  | tee evidencias/00-horario-inicial.txt
```

```bash
docker compose exec primary psql -U postgres -d postgres \
  -c "\dt clientes" \
  2>&1 | tee evidencias/01-verificacao-tabela.txt
```

###A verificação demonstrou que a tabela `clientes` ainda não existia.

## 2. Registro do horário de criação da base + contagens

Horário registrado em `evidencias/02-horario-criacao-base.txt`.

```bash
date '+%Y-%m-%d %H:%M:%S %Z' \
  | tee evidencias/02-horario-criacao-base.txt
```

```bash
docker compose exec primary psql -U postgres -d postgres \
  -v ON_ERROR_STOP=1 \
  -c "
CREATE TABLE clientes (
    id SERIAL PRIMARY KEY,
    nome VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE
);

###inserindo

INSERT INTO clientes (nome, email)
SELECT
    'cliente ' || g,
    'cliente' || g || '@loja.com'
FROM generate_series(1, 1000) AS g;
" 2>&1 | tee evidencias/03-criacao-base-inicial.txt
```

###contagens

```bash
docker compose exec primary psql -U postgres -d postgres \
  -c "SELECT count(*) AS total_clientes FROM clientes;" \
  2>&1 | tee evidencias/04-contagem-inicial-primario.txt
```

```bash
docker compose exec replica psql -U postgres -d postgres \
  -c "
SELECT
    pg_is_in_recovery() AS eh_replica,
    count(*) AS total_clientes
FROM clientes;
" 2>&1 | tee evidencias/05-contagem-inicial-replica.txt
```

## 3. Registro do horário + backup + verificacao

Horário registrado em `evidencias/06-horario-backup.txt`.

```bash
date '+%Y-%m-%d %H:%M:%S %Z' \
  | tee evidencias/06-horario-backup.txt
```

###backup

```bash
docker compose exec primary pg_dump \
  -U postgres \
  -t clientes \
  -f /tmp/backup_clientes.sql \
  postgres
```

###verificacao

```bash
docker compose exec primary bash -c \
  'ls -lh /tmp/backup_clientes.sql && sha256sum /tmp/backup_clientes.sql' \
  2>&1 | tee evidencias/07-verificacao-backup.txt
```

## 4. Registro do horário + novos cadastros + confirmacao

Horário registrado em `evidencias/08-horario-novos-clientes.txt`.

```bash
date '+%Y-%m-%d %H:%M:%S %Z' \
  | tee evidencias/08-horario-novos-clientes.txt
```

###inserindo

```bash
docker compose exec primary psql -U postgres -d postgres \
  -c "
INSERT INTO clientes (nome, email)
SELECT
    'novo ' || g,
    'n' || g || '@loja.com'
FROM generate_series(1, 50) AS g;
" 2>&1 | tee evidencias/09-insercao-50-clientes.txt
```

###confirmando quantidade

```bash
docker compose exec primary psql -U postgres -d postgres \
  -c "SELECT count(*) AS total_clientes_antes_ataque FROM clientes;" \
  2>&1 | tee evidencias/10-contagem-1050.txt
```
## 5. Registro do T0 — início do incidente + exclusao dos clientes

O T0 representa o momento imediatamente anterior à exclusão dos clientes.

Horário registrado em `evidencias/11-T0-inicio-incidente.txt`.

```bash
date '+T0=%Y-%m-%d %H:%M:%S.%3N %z | epoch_ms=%s%3N' \
  | tee evidencias/11-T0-inicio-incidente.txt
```

### exclusao

```bash
docker compose exec primary psql -U postgres -d postgres \
  -c "DELETE FROM clientes;" \
  2>&1 | tee evidencias/12-ataque-delete-clientes.txt
```

O comando apagou os 1.050 clientes existentes no primário.

### contagem replica


```bash
docker compose exec replica psql -U postgres -d postgres \
  -c "
SELECT
    pg_is_in_recovery() AS eh_replica,
    count(*) AS clientes_apos_ataque
FROM clientes;
" 2>&1 | tee evidencias/13-contagem-replica-apos-ataque.txt
```

A consulta verificou se a exclusão realizada no primário também foi reproduzida na réplica.

## 6. Remoção da tabela antes da restauração + restauracao + contagem


```bash
docker compose exec primary psql -U postgres -d postgres \
  -v ON_ERROR_STOP=1 \
  -c "DROP TABLE clientes;" \
  2>&1 | tee evidencias/14-drop-tabela-clientes.txt
```

A tabela vazia foi removida para permitir sua recriação a partir do backup.

### Restauração do backup


```bash
docker compose exec primary psql -U postgres -d postgres \
  -v ON_ERROR_STOP=1 \
  -f /tmp/backup_clientes.sql \
  2>&1 | tee evidencias/15-restauracao-backup.txt
```

A restauração recriou a tabela, a sequência e as restrições, além de copiar os 1.000 clientes presentes no backup.

### Registro do T1 — fim da restauração

Horário registrado em `evidencias/16-T1-fim-restauracao.txt`.

```bash
date '+T1=%Y-%m-%d %H:%M:%S.%3N %z | epoch_ms=%s%3N' \
  | tee evidencias/16-T1-fim-restauracao.txt
```

Valor registrado:

```text
T1=2026-11-14 13:31:27.384 -0300
epoch_ms=1794673887384
```

###Contagem após a restauração

```bash
docker compose exec primary psql -U postgres -d postgres \
  -c "SELECT count(*) AS clientes_apos_restauracao FROM clientes;" \
  2>&1 | tee evidencias/17-contagem-apos-restauracao.txt
```

A contagem final apresentou 1.000 clientes. Os 50 clientes cadastrados após o backup não foram recuperados.

## 7. Cálculo do RTO

Horário: após o registro do T1.

Os valores em milissegundos foram extraídos das evidências de T0 e T1.

```bash
incidente_t0_ms=$(sed -n 's/.*epoch_ms=\([0-9]*\).*/\1/p' \
  evidencias/11-T0-inicio-incidente.txt)

incidente_t1_ms=$(sed -n 's/.*epoch_ms=\([0-9]*\).*/\1/p' \
  evidencias/16-T1-fim-restauracao.txt)

incidente_rto_ms=$((incidente_t1_ms - incidente_t0_ms))

printf 'T0 epoch_ms: %s\nT1 epoch_ms: %s\nRTO: %s ms\n' \
  "$incidente_t0_ms" "$incidente_t1_ms" "$incidente_rto_ms" \
  | tee evidencias/18-calculo-rto.txt

awk -v ms="$incidente_rto_ms" \
  'BEGIN { printf "RTO: %.3f segundos\n", ms / 1000 }' \
  | tee -a evidencias/18-calculo-rto.txt
```

Resultado:

```text
T0 epoch_ms: 1794667276823
T1 epoch_ms: 1794673887384
RTO: 6610561 ms
RTO: 6610,561 segundos
```

O RTO medido foi de **6.610,561 segundos**, equivalente a **1 hora, 50 minutos e 10,561 segundos**.

O RPO observado foi de **50 clientes**, correspondente aos registros cadastrados após o backup e ausentes na restauração.
