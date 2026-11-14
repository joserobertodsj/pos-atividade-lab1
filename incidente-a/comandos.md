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