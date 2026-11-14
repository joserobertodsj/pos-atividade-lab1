# Incidente A — Conta comprometida apaga os clientes

## Integrantes

- `[NOME DO INTEGRANTE 1]`
- `[NOME DO INTEGRANTE 2]`

## 1. Objetivo

Este trabalho reproduz um incidente no qual as credenciais de uma conta com privilégios excessivos foram comprometidas e utilizadas para apagar todos os registros da tabela `clientes`.

O experimento teve como objetivos:

- verificar o comportamento da replicação diante de uma exclusão maliciosa;
- restaurar os dados a partir de um backup;
- medir o RTO e o RPO;
- aplicar o princípio do menor privilégio;
- demonstrar que uma réplica não substitui uma estratégia de backup.

## 2. Ambiente

O ambiente foi executado com Docker Compose e PostgreSQL 16.

| Componente | Porta no host | Função |
|---|---:|---|
| PostgreSQL primário | 5442 | Recebe escritas e alterações |
| PostgreSQL réplica | 5443 | Replica o WAL do primário |
| Replicação | Streaming replication | Replica as alterações do primário |

As credenciais do ambiente foram armazenadas em um arquivo `.env`, que não foi enviado ao repositório. O repositório contém somente o arquivo `.env.example`.

## 3. Preparação da base

A tabela `clientes` foi criada inicialmente com a seguinte estrutura:

```sql
CREATE TABLE clientes (
    id SERIAL PRIMARY KEY,
    nome VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE
);
```

Em seguida, foram cadastrados 1.000 clientes.

A contagem no primário apresentou:

```text
total_clientes
--------------
1000
```

A réplica também apresentou 1.000 clientes e retornou `pg_is_in_recovery() = true`, confirmando que era uma instância em recuperação e que a replicação estava funcionando.

Evidências:

- [Criação da base inicial](evidencias/03-criacao-base-inicial.txt)
- [Contagem no primário](evidencias/04-contagem-inicial-primario.txt)
- [Contagem na réplica](evidencias/05-contagem-inicial-replica.txt)

## 4. Backup

O backup da tabela foi iniciado em:

```text
2026-11-14 11:07:42 -03
```

Comando utilizado:

```bash
docker compose exec primary pg_dump \
  -U postgres \
  -t clientes \
  -f /tmp/backup_clientes.sql \
  postgres
```

O arquivo gerado possuía aproximadamente 38 KB e o seguinte SHA-256:

```text
e67787fb1d0846259a2d8ea8c3a1b6d79070e86841646ac74ca44e41f0838025
```

Evidências:

- [Horário do backup](evidencias/06-horario-backup.txt)
- [Verificação do backup](evidencias/07-verificacao-backup.txt)

## 5. Registros posteriores ao backup

Às `11:12:15 -03`, foram cadastrados mais 50 clientes.

Após a operação, a tabela possuía:

```text
1050 clientes
```

Esses 50 registros não estavam presentes no arquivo de backup.

Evidências:

- [Horário dos novos cadastros](evidencias/08-horario-novos-clientes.txt)
- [Inserção dos 50 clientes](evidencias/09-insercao-50-clientes.txt)
- [Contagem de 1.050 clientes](evidencias/10-contagem-1050.txt)

## 6. Reprodução do incidente

O início do incidente, denominado T0, foi registrado como:

```text
T0 = 2026-11-14 11:41:16.823 -0300
epoch_ms = 1794667276823
```

O ataque foi simulado com:

```sql
DELETE FROM clientes;
```

O PostgreSQL informou:

```text
DELETE 1050
```

Portanto, todos os 1.050 clientes foram apagados.

Evidências:

- [Registro do T0](evidencias/11-T0-inicio-incidente.txt)
- [Execução do ataque](evidencias/12-ataque-delete-clientes.txt)

## 7. Comportamento da réplica

Imediatamente após o ataque, a consulta realizada na réplica retornou:

```text
eh_replica | clientes_apos_ataque
-----------+----------------------
t          | 0
```

A réplica também ficou sem os clientes porque a replicação física transmite e reaplica os registros de WAL gerados pelo primário.

Para o PostgreSQL, o `DELETE` era uma transação válida. A replicação não consegue distinguir uma operação legítima de uma operação executada por uma conta comprometida.

Evidência:

- [Contagem na réplica após o ataque](evidencias/13-contagem-replica-apos-ataque.txt)

## 8. Restauração

A tabela vazia foi removida:

```sql
DROP TABLE clientes;
```

Em seguida, o backup foi restaurado:

```bash
docker compose exec primary psql \
  -U postgres \
  -f /tmp/backup_clientes.sql
```

A restauração recriou:

- a tabela `clientes`;
- a sequência `clientes_id_seq`;
- as restrições e índices;
- os 1.000 registros presentes no backup.

A saída apresentou:

```text
CREATE TABLE
CREATE SEQUENCE
COPY 1000
setval = 1000
```

O término da restauração, denominado T1, foi registrado como:

```text
T1 = 2026-11-14 13:31:27.384 -0300
epoch_ms = 1794673887384
```

A contagem após a restauração confirmou a recuperação de 1.000 clientes.

Evidências:

- [Remoção da tabela](evidencias/14-drop-tabela-clientes.txt)
- [Restauração do backup](evidencias/15-restauracao-backup.txt)
- [Registro do T1](evidencias/16-T1-fim-restauracao.txt)
- [Contagem após a restauração](evidencias/17-contagem-apos-restauracao.txt)

## 9. RTO

O RTO representa o tempo entre o início do incidente e a conclusão da restauração.

```text
RTO = T1 − T0
RTO = 1794673887384 − 1794667276823
RTO = 6610561 milissegundos
RTO = 6610,561 segundos
```

O RTO observado foi de:

```text
1 hora, 50 minutos e 10,561 segundos
```

Esse valor inclui o tempo empregado na análise, documentação e execução manual da recuperação. Ele representa o tempo real observado no exercício, não apenas o tempo de processamento do comando de restauração.

Evidência:

- [Cálculo do RTO](evidencias/18-calculo-rto.txt)

## 10. RPO

O backup continha 1.000 clientes, enquanto a tabela possuía 1.050 clientes antes do ataque.

```text
RPO = 1050 − 1000
RPO = 50 clientes
```

Portanto, foram perdidos os 50 clientes cadastrados depois do backup.

Nesse exercício, o RPO foi medido em quantidade de registros perdidos.

## 11. Correção pelo princípio do menor privilégio

Foi criado o usuário `app` com somente as permissões necessárias:

```sql
CREATE ROLE app LOGIN PASSWORD 'app';

GRANT SELECT, INSERT, UPDATE
ON clientes
TO app;

GRANT USAGE
ON SEQUENCE clientes_id_seq
TO app;
```

O usuário não recebeu permissão de `DELETE` e não se tornou proprietário da tabela.

Evidência:

- [Criação do usuário app](evidencias/19-criacao-usuario-app.txt)

### 11.1 Tentativa de DELETE

Ao executar `DELETE FROM clientes` como `app`, o banco retornou:

```text
ERROR: permission denied for table clientes
```

Evidência:

- [DELETE negado](evidencias/20-app-delete-negado.txt)

### 11.2 Tentativa de DROP TABLE

Ao executar `DROP TABLE clientes` como `app`, o banco retornou:

```text
ERROR: must be owner of table clientes
```

Evidência:

- [DROP TABLE negado](evidencias/21-app-drop-negado.txt)

### 11.3 Teste de INSERT

O usuário `app` conseguiu executar:

```sql
INSERT INTO clientes (nome, email)
VALUES ('Cliente inserido pelo app', 'app@loja.com');
```

O resultado foi:

```text
INSERT 0 1
```

A consulta de confirmação encontrou o cliente com identificador `1001`.

Evidências:

- [INSERT permitido](evidencias/22-app-insert-permitido.txt)
- [Confirmação do cadastro](evidencias/23-confirmacao-insert-app.txt)

## 12. Respostas

### Por que ter uma réplica não protegeu os dados?

A réplica não protegeu os dados porque sua função é manter uma cópia atualizada do estado do primário para disponibilidade e continuidade do serviço.

O `DELETE` foi aceito pelo primário como uma operação válida, registrado no WAL e reproduzido na réplica. Por isso, tanto o primário quanto a réplica ficaram com zero clientes.

Uma réplica ajuda em falhas de hardware ou indisponibilidade do servidor, mas não substitui backups contra exclusões acidentais, ataques ou operações maliciosas.

### Que recurso de serviço gerenciado reduziria o RPO?

O recurso indicado é o **Point-in-Time Recovery — PITR**.

O PITR combina backups com o arquivamento contínuo dos registros de transação. Isso permite restaurar o banco para um instante imediatamente anterior ao ataque, em vez de recuperar somente o último backup completo.

No cenário reproduzido, seria possível selecionar um ponto anterior ao `DELETE`, reduzindo ou até eliminando a perda dos 50 clientes cadastrados após o backup.

### Que frequência de backup daria RPO de 15 minutos?

Para garantir um RPO máximo de 15 minutos com backups periódicos, seria necessário realizar um backup a cada 15 minutos ou em intervalo menor.

No pior caso, uma falha ocorreria imediatamente antes do próximo backup, causando a perda de todas as alterações realizadas desde o backup anterior.

Na prática, realizar backups completos a cada 15 minutos pode ser caro. Uma estratégia mais apropriada seria usar backups base periódicos com arquivamento contínuo do WAL e PITR.

## 13. Recomendações

Além da restauração, as seguintes medidas reduziriam o impacto de incidentes semelhantes:

- aplicar o princípio do menor privilégio;
- não armazenar credenciais em repositórios;
- utilizar variáveis de ambiente ou um gerenciador de segredos;
- rotacionar imediatamente credenciais comprometidas;
- habilitar backups automáticos e PITR;
- testar periodicamente a restauração dos backups;
- monitorar operações destrutivas e alterações de permissões;
- registrar auditoria de acessos e comandos administrativos;
- separar contas administrativas das contas usadas pela aplicação.

## 14. Conclusão

O experimento demonstrou que replicação e backup resolvem problemas diferentes.

A réplica manteve alta disponibilidade, mas também reproduziu a exclusão maliciosa. O backup permitiu recuperar os 1.000 registros existentes no momento de sua criação, porém os 50 registros posteriores foram perdidos.

O RTO medido foi de 1 hora, 50 minutos e 10,561 segundos, e o RPO observado foi de 50 clientes.

A criação do usuário `app` demonstrou que o princípio do menor privilégio reduz o impacto de uma credencial comprometida: o usuário conseguiu realizar o `INSERT` necessário, mas não conseguiu executar `DELETE` nem `DROP TABLE`.