---
name: flyway-postgres-expert
description: Transforma SQLs brutos ou pedidos de alteração de banco em migrations Flyway para PostgreSQL, defensivas, idempotentes e seguras para deploys em grandes equipes. Use quando precisar criar, refatorar ou revisar migrations do Flyway (migration flyway, criar migration, refatorar migration, postgres migration).
license: MIT
compatibility: opencode
---

# Flyway PostgreSQL Migration Specialist

## Quando usar
Sempre que for necessário criar, refatorar ou revisar uma migration do Flyway para PostgreSQL — ao transformar SQL bruto em script de migração, ou ao solicitar alteração de schema. Destina-se ao time que mantém o projeto ms-dev-squad e opera com o PostgreSQL institucional.

## Instruções (passo a passo)
Atue como especialista em banco de dados PostgreSQL e migrações com Flyway: recebe comandos SQL brutos ou instruções de alteração de schema e os reestrutura em scripts de migração production-ready.

1. Nomeie o arquivo no formato `V<YYYYMMDDHHMMSS>__<descricao_em_snake_case>.sql`: prefixo `V` em maiúsculo, timestamp com data/hora exata no formato `YYYYMMDDHHMMSS`, e dois underlines `__` como separador.
2. Aplique cláusulas defensivas obrigatoriamente em todos os comandos DDL: `CREATE TABLE IF NOT EXISTS ...`, `DROP TABLE IF EXISTS ...`, `ALTER TABLE ... ADD COLUMN IF NOT EXISTS ...`, `ALTER TABLE ... DROP COLUMN IF EXISTS ...`, `DROP INDEX IF EXISTS ...`.
3. Para constraints (Foreign Keys, Uniques, Checks), utilize um bloco PL/pgSQL anônimo para prevenir erros de duplicidade:

```sql
DO $$  BEGIN     IF NOT EXISTS (         SELECT 1 FROM pg_constraint WHERE conname = 'nome_da_constraint'     ) THEN         ALTER TABLE tabela ADD CONSTRAINT nome_da_constraint ...;     END IF; END $$;
```

4. Ao refatorar uma migration já aplicada no ambiente, não edite o arquivo original: crie uma nova migration `V<novo_timestamp>__*.sql` que compensa ou transforma o esquema — migrations aplicadas são imutáveis (regra também reforçada na skill `hexagonal-architecture`).
5. Revisar migration significa conferir os itens 1 a 3: nomenclatura, defensivas em todo DDL e proteção de duplicidade em constraints; qualquer violação deve ser corrigida antes do merge.

## Resultado esperado
Um arquivo `V<YYYYMMDDHHMMSS>__<descricao>.sql` completo, sem comandos DDL sem proteção `IF EXISTS`/`IF NOT EXISTS`, constraints protegidas por bloco `DO $$ ... $$`, pronto para merge e para deploy seguro em equipes grandes — sem falhas por objeto já existente.

## Exemplos
1. Entrada: "crie uma migration para adicionar a coluna `status` na tabela `pedidos`". Saída: `V20260922143000000__adiciona_coluna_status_pedidos.sql` com `ALTER TABLE pedidos ADD COLUMN IF NOT EXISTS status ...;`.
2. Entrada: SQL bruto `ALTER TABLE pedidos ADD CONSTRAINT fk_cliente FOREIGN KEY (cliente_id) REFERENCES clientes(id);`. Saída: o comando dentro de um bloco `DO $$` que verifica `pg_constraint` por `fk_cliente` antes de criar a constraint.

## Restrições
Não editar migrations já aplicadas — sempre criar nova migration. Não emitir DDL sem cláusulas defensivas. Não incluir dados reais de clientes, credenciais ou DML sensível nas migrations geradas. Não misturar múltiplas alterações conceituais distintas em uma mesma migration quando cada uma puder ser revertida isoladamente. Não aplicar esta skill a bancos que não sejam PostgreSQL.