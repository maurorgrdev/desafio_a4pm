
# Desafio técnico A4PM

O projeto desenvolvido tem como objetivo propor soluções para bucar os dados solicitados no desafio.


## 1ª Parte - Desafios e Soluções

#### 1º Liste o maior salário entre todos os funcionários.

Solução:
```sql
SELECT SALARIO 
FROM FUNCIONARIOS F 
ORDER BY SALARIO DESC LIMIT 1;
```
 
#### 2º Liste o menor salário entre todos os funcionários.

Solução:
```sql
SELECT SALARIO 
FROM FUNCIONARIOS F 
ORDER BY SALARIO ASC LIMIT 1;
```

#### 3 º Liste o maior salário entre os Desenvolvedores.
Solução:
```sql
SELECT SALARIO
FROM FUNCIONARIOS F 
WHERE EXISTS (
    SELECT *
    FROM DEPARTAMENTO D 
    WHERE NOME_DEPARTAMENTO = 'Desenvolvedor'
        AND F.DEPARTAMENTO_ID = D.ID_DEPARTAMENTO
)
ORDER BY SALARIO ASC  LIMIT 1;
```


#### 4º Liste o nome de todos os funcionários é o nome de seus respectivos departamentos.
Solução:
```sql
SELECT 
    NOME_FUNCIONARIO, 
    COALESCE(NOME_DEPARTAMENTO, 'SEM DEPARTAMENTO')
FROM FUNCIONARIOS F
    LEFT JOIN DEPARTAMENTO D ON F.DEPARTAMENTO_ID = D.ID_DEPARTAMENTO;
```

#### 5º Liste a média salarial entre todos os funcionários.
Solução:
```sql
SELECT 
    AVG(SALARIO) AS MEDIA_SALARIAL_AVG, 
    (SUM(SALARIO) / COUNT(DISTINCT ID_FUNCIONARIO)) AS MEDIA_SALARIAL_CALCULADA
FROM FUNCIONARIOS F;
```

#### 6º Atualize o salário do funcionário João para R$ 2000.
Solução:
```sql
UPDATE FUNCIONARIOS 
SET SALARIO = 2000
WHERE ID_FUNCIONARIO = 1
```

#### 7º Faça uma subquery para listar todos os funcionários que são do departamento “Desenvolvedor”.
Solução:
```sql
SELECT * 
FROM FUNCIONARIOS 
WHERE DEPARTAMENTO_ID = (
    SELECT ID_DEPARTAMENTO 
    FROM DEPARTAMENTO 
    WHERE NOME_DEPARTAMENTO = 'Desenvolvedor'
);
```


#### 8º Incrementar ao modelo uma tabela para endereço e outra para telefone, insira algumas informações e em seguida faça a relação (foreign key) com a tabela de "funcionarios".
Solução para tabela endereço:
```sql
CREATE TABLE endereco (
    id_endereco SERIAL PRIMARY KEY,
    rua VARCHAR(100) NOT NULL,
    cidade VARCHAR(50) NOT NULL,
    estado VARCHAR(50) NOT NULL,
    cep VARCHAR(20) NOT NULL
);
```
Solução para tabela telefone:
```sql
CREATE TABLE telefone (
    id_telefone SERIAL PRIMARY KEY,
    numero VARCHAR(20) NOT NULL,
    tipo VARCHAR(20) CHECK (tipo IN ('Residencial', 'Comercial', 'Celular')),
    funcionario_id INTEGER,
    FOREIGN KEY (funcionario_id) REFERENCES funcionarios(id_funcionario) ON DELETE CASCADE
);
```

#### 9º Um funcionário pode ter mais do que um endereço e um endereço pode pertencer a mais do que um funcionário, liste os endereços com mais de um funcionário e funcionários com mais de um endereço.
Solução vinculo entre Funcionário e Endereço:
```sql
CREATE TABLE funcionario_endereco (
    funcionario_id INTEGER,
    endereco_id INTEGER,
    PRIMARY KEY (funcionario_id, endereco_id),
    FOREIGN KEY (funcionario_id) REFERENCES funcionarios(id_funcionario) ON DELETE CASCADE,
    FOREIGN KEY (endereco_id) REFERENCES endereco(id_endereco) ON DELETE CASCADE
);
```

Solução para listar os funcionários com mais de 1 endereço:
```sql
SELECT 
    E.*, 
    COUNT(FE.FUNCIONARIO_ID) AS QTD_FUNCIONARIOS
FROM ENDERECO E
    JOIN FUNCIONARIO_ENDERECO FE ON E.ID_ENDERECO = FE.ENDERECO_ID
GROUP BY E.ID_ENDERECO
HAVING COUNT(FE.FUNCIONARIO_ID) > 1;
```

Solução para listar endereços com mais de 1 funcionário:
```sql
SELECT 
    F.*, 
    COUNT(FE.ENDERECO_ID) AS QTD_ENDERECOS
FROM FUNCIONARIOS F
    JOIN FUNCIONARIO_ENDERECO FE ON F.ID_FUNCIONARIO = FE.FUNCIONARIO_ID
GROUP BY F.ID_FUNCIONARIO
HAVING COUNT(FE.ENDERECO_ID) > 1;
```

#### 10º Faça uma query que retorne a porcentagem de funcionários cadastrados nos últimos 30 dias .
Solução:
```sql
SELECT 
    (COUNT(*) * 100.0) / (SELECT COUNT(*) FROM funcionarios) AS percentual
FROM funcionarios
WHERE dh_criacao >= CURRENT_DATE - INTERVAL '30 days';
```

#### 11º Crie uma Procedure que receba como parâmetro os campos da tabela “funcionarios” e em seguida insira dois novos funcionários com dados fictícios.

Para garantir que o ID seja incrementado corretamente, é necessário ajustar a sequência do campo id_funcionario, que por padrão é nomeada como funcionarios_id_funcionario_seq seguindo o formato:
```sql
<nome_da_tabela>_<nome_da_coluna>_seq
```
O ajuste pode ser feito com o seguinte comando:
```sql
SELECT setval('funcionarios_id_funcionario_seq', (SELECT MAX(id_funcionario) FROM funcionarios));
```

Solução:
```sql
CREATE OR REPLACE PROCEDURE adicionar_funcionarios(
    new_nome VARCHAR, new_salario INTEGER, new_departamento_id INTEGER
) LANGUAGE plpgsql AS $$
BEGIN
    INSERT INTO funcionarios (nome_funcionario, salario, departamento_id, dh_criacao) 
    VALUES (new_nome, new_salario, new_departamento_id, CURRENT_TIMESTAMP);
END;
$$;
```

Adicionar 2 funcionarios com a procedure:
```sql
CALL adicionar_funcionarios('Mauro', 3500, 2);
CALL adicionar_funcionarios('Roger', 2500, 1);
```
## 2ª Parte - Desafios e Soluções

### Você recebeu a seguinte query SQL utilizada para extrair dados da tabela “funcionarios”. O objetivo dessa query é retornar todos os funcionários que não tenham status igual a Excluir, Inativar ou esteja nulo.

```sql
SELECT 
    log.funl_id_funcionario,
    log.funl_nome_funcionario,
    log.funl_id_departamento
FROM (
    SELECT 
        funl_id_funcionario,
        funl_nome_funcionario,
        funl_id_departamento,
        funl_created_at,
        funl_alteracao
    FROM (
        SELECT *
        FROM funcionarios_log AS fnlog
        ORDER BY funl_created_at DESC
    ) AS l
    WHERE l.funl_created_at >= ‘2025-01-01’
    GROUP BY funl_id_funcionario
) AS log
INNER JOIN
    funcionarios AS fn ON log.funl_id_funcionario = fn.fun_id_funcionario
WHERE log.alteracao NOT IN ('Excluir','Inativar')
    OR log.alteracao IS NOT NULL
```

#### 1º Identifique o motivo da query retornar registros do tipo Excluir e Inativar.
O problema está na cláusula WHERE:
```sql
WHERE log.alteracao NOT IN ('Excluir', 'Inativar') OR log.alteracao IS NOT NULL
```
✅ A condição log.alteracao NOT IN ('Excluir', 'Inativar') retorna TRUE para registros que não possuem os valores 'Excluir' ou 'Inativar', mas retorna FALSE para esses valores.

✅ A condição log.alteracao IS NOT NULL retorna TRUE para todos os registros que não são nulos, inclusive aqueles com 'Excluir' ou 'Inativar'.

❌ Como qualquer valor TRUE em um OR torna a condição verdadeira, todos os registros acabam sendo incluídos no resultado.


#### Correção
Para corrigir esse erro, podemos usar AND ou envolver a condição com NOT(...):

Opção 1: Uso do AND (Mais Legível e Recomendada)
```sql
WHERE log.alteracao NOT IN ('Excluir', 'Inativar') 
AND log.alteracao IS NOT NULL
```

Opção 2: Uso do NOT(...)
```sql
WHERE NOT (log.alteracao IN ('Excluir', 'Inativar') OR log.alteracao IS NULL)
```

#### 2º Proponha uma solução para otimizar a query.

Solução:
```sql
SELECT *
FROM funcionarios_log fnlog
WHERE EXISTS (
    SELECT 1
    FROM funcionarios f 
    WHERE fnlog.funl_id_funcionario = f.fun_id_funcionario
)
AND fnlog.funl_created_at >= '2025-01-01'
AND fnlog.funl_alteracao NOT IN ('Excluir', 'Inativar')
AND fnlog.funl_alteracao IS NOT NULL;

```
✅ Remoção do ORDER BY dentro da subquery → Como não influenciava na agregação, ele poderia ser realizado externamente, caso necessário.

✅ Eliminação do GROUP BY desnecessário → A agregação não era necessária para essa consulta, reduzindo a complexidade da query.

✅ Uso do EXISTS → Melhor performance ao verificar a existência de um funcionário sem trazer dados desnecessários.

✅ Filtros mais eficientes → Agora a query retorna apenas os registros dentro do período desejado (funl_created_at >= '2025-01-01') e exclui os status 'Excluir' e 'Inativar', além de ignorar valores NULL.
