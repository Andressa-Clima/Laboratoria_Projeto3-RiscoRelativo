# Projeto SuperCaja - Limpeza, Tratamento e Análise de Dados

## 🔵 Conectar/Importar Dados para Ferramentas

Descompactei os documentos e incluí no Google Drive em sheets, em planilhas separadas. Importei as tabelas com ajuda desse vídeo:
[Eliabe Silva - Como criar uma tabela no BigQuery vinculada ao Google Sheets](https://www.youtube.com/watch?v=link_exemplo)

A partir disso, fiz uma tabela para cada planilha e importei todas para o mesmo projeto.

---

## 🔵 Identificar e Tratar Valores Nulos

Foi utilizada a seguinte consulta para identificar valores nulos. Também utilizei `IS NOT NULL` para conferir os resultados das outras colunas.

Antes de prosseguir, ocorreu um erro, pois a coluna 3 da tabela `user_info`, nas linhas 4932 e 9466, continha valores em formato string, enquanto a coluna estava definida como `integer`, causando falhas na consulta. Os valores problemáticos foram substituídos por `NULL`, pois não foi possível identificar seus valores reais.

```sql
SELECT
    COUNT(*) AS total_nulos
FROM
    projeto3-riscorelativo-467822.supercaja.user_info
WHERE
    user_id IS NULL;
```

**Tabela `user_info`, nulos identificados:**

* `last_month_salary` = 7201 valores nulos
* `number_dependents` = 943 valores nulos

**Tabela `loans_outstanding`, nulos identificados:**
Não foram identificados nulos nesta tabela.

**Tabela `default`, nulos identificados:**
Não foram identificados nulos nesta tabela.

**Tabela `loans_detail`, nulos identificados:**
Não foram identificados nulos nesta tabela.

Além disso, a tabela `default` foi unificada à tabela `user_info` como sugestão do marco para entendermos os nulos de `last_month_salary`. Foi usado `LEFT JOIN` para manter os dados da tabela `user_info`.

```sql
CREATE OR REPLACE TABLE projeto3-riscorelativo-467822.supercaja.user_info_unificada AS
SELECT
    A.*,
    B.* EXCEPT (user_id)
FROM
    projeto3-riscorelativo-467822.supercaja.user_info A
LEFT JOIN
    projeto3-riscorelativo-467822.supercaja.default  B
ON
    A.user_id = B.user_id;
```

A correlação entre `last_month_salary` e `default` foi de `-0,003`, indicando que a ausência do salário informado não está relacionada à inadimplência.

```sql
SELECT
    CORR(
        CAST(default_flag AS FLOAT64),
        CAST(IF(last_month_salary IS NULL, 1, 0) AS FLOAT64)
    ) AS correlacao_inadimplencia_salario_nulo
FROM
    projeto3-riscorelativo-467822.supercaja.user_info_unificada;
```

---

## 🔵 Identificar e Tratar Valores Duplicados

### Tabela `user_info`

Foram identificados 21 dados duplicados, sem IDs compatíveis.

```sql
WITH duplicados AS (
    SELECT age, sex, last_month_salary, number_dependents, default_flag, COUNT() AS quantidade
    FROM projeto3-riscorelativo-467822.supercaja.user_info_unificada
    GROUP BY age, sex, last_month_salary, number_dependents, default_flag
    HAVING COUNT() > 1
)
SELECT t.*, d.quantidade
FROM projeto3-riscorelativo-467822.supercaja.user_info_unificada t
JOIN duplicados d
ON t.age = d.age AND t.sex = d.sex AND t.last_month_salary = d.last_month_salary AND t.number_dependents = d.number_dependents AND t.default_flag = d.default_flag
ORDER BY d.quantidade DESC, t.age;
```

### Tabela `loans_detail`

Foram encontrados 416 usuários com informações duplicadas, mas IDs distintos.

```sql
WITH duplicados AS (
    SELECT more_90_days_overdue, using_lines_not_secured_personal_assets, number_times_delayed_payment_loan_30_59_days, debt_ratio, number_times_delayed_payment_loan_60_89_days, COUNT() AS quantidade
    FROM projeto3-riscorelativo-467822.supercaja.loans_detail
    GROUP BY more_90_days_overdue, using_lines_not_secured_personal_assets, number_times_delayed_payment_loan_30_59_days, debt_ratio, number_times_delayed_payment_loan_60_89_days
    HAVING COUNT() > 1
)
SELECT t.*, d.quantidade
FROM projeto3-riscorelativo-467822.supercaja.loans_detail t
JOIN duplicados d
ON t.more_90_days_overdue = d.more_90_days_overdue AND t.using_lines_not_secured_personal_assets = d.using_lines_not_secured_personal_assets AND t.number_times_delayed_payment_loan_30_59_days = d.number_times_delayed_payment_loan_30_59_days AND t.debt_ratio = d.debt_ratio AND t.number_times_delayed_payment_loan_60_89_days = d.number_times_delayed_payment_loan_60_89_days
ORDER BY d.quantidade DESC;
```

### Tabela `loans_outstanding`

Não foram encontrados dados duplicados.

```sql
WITH loan_duplicados AS (
    SELECT loan_id, COUNT() AS quantidade
    FROM projeto3-riscorelativo-467822.supercaja.loans_outstanding
    GROUP BY loan_id
    HAVING COUNT() > 1
)
SELECT t.*, d.quantidade
FROM projeto3-riscorelativo-467822.supercaja.loans_outstanding t
JOIN loan_duplicados d
ON t.loan_id = d.loan_id
ORDER BY d.quantidade DESC, t.loan_id;
```

---

## 🔵 Identificar e Gerenciar Dados Fora do Escopo de Análise

A variável `sex` foi retirada por não ser aplicável à análise de crédito.

```sql
CREATE OR REPLACE TABLE projeto3-riscorelativo-467822.supercaja.user_info_unificada AS
SELECT * EXCEPT(sex)
FROM projeto3-riscorelativo-467822.supercaja.user_info_unificada;
```

Variáveis altamente correlacionadas foram analisadas e a de menor desvio padrão excluída:

```sql
CREATE OR REPLACE TABLE projeto3-riscorelativo-467822.supercaja.loans_detail_limpo AS
SELECT * EXCEPT(number_times_delayed_payment_loan_60_89_days)
FROM projeto3-riscorelativo-467822.supercaja.loans_detail;
```

---

## 🔵 Identificar e Tratar Dados Discrepantes

### Variáveis Categóricas

Padronização de `loan_type`:

* "REAL ESTATE" → "real estate"
* "OTHERS"/"Others" → "other"

```sql
CREATE OR REPLACE TABLE projeto3-riscorelativo-467822.supercaja.loans_outstanding_V2 AS
SELECT * EXCEPT (loan_type) FROM projeto3-riscorelativo-467822.supercaja.loans_outstanding;
```

### Variáveis Numéricas

Tratamento de `using_lines_not_secured_personal_assets` e `debt_ratio` para formato NUMERIC:

```sql
CREATE OR REPLACE TABLE projeto3-riscorelativo-467822.supercaja.user_info_unificada_V2 AS
SELECT user_id, age, last_month_salary, number_dependents, default_flag, total_emprestimos, more_90_days_overdue, CAST(REPLACE(REPLACE(using_lines_not_secured_personal_assets, '.', ''), ',', '.') AS NUMERIC) AS using_lines_not_secured_personal_assets, number_times_delayed_payment_loan_30_59_days, CAST(REPLACE(REPLACE(debt_ratio, '.', ''), ',', '.') AS NUMERIC) AS debt_ratio, qtd_real_estate, qtd_other
FROM projeto3-riscorelativo-467822.supercaja.user_info_unificada_V2;
```

Outliers em `last_month_salary` foram identificados e estatísticas calculadas (mínimo, máximo, média, desvio padrão, quartis).

---

## 🔵 Unir Tabelas e Criar Novas Variáveis

Tabelas `user_info_unificada`, `loans_outstanding_V3` e `loans_detail_limpo` foram unificadas via `LEFT JOIN`.

Empréstimos foram agrupados por cliente, classificando por tipo (`real estate` e `other`) e substituindo `loan_id`.

```sql
SELECT user_id, COUNT(loan_id) AS total_emprestimos, COUNTIF(loan_type_padronizado = 'real estate') AS qtd_real_estate, COUNTIF(loan_type_padronizado = 'other') AS qtd_other
FROM projeto3-riscorelativo-467822.supercaja.loans_outstanding_V2
GROUP BY user_id
ORDER BY total_emprestimos;
```

A coluna `default_flag` foi convertida para string para melhor visualização.

Faixas etárias foram classificadas de 15 em 15 anos:

```sql
SELECT CASE
  WHEN age BETWEEN 18 AND 32 THEN '18-32'
  WHEN age BETWEEN 33 AND 47 THEN '33-47'
  WHEN age BETWEEN 48 AND 62 THEN '48-62'
  WHEN age BETWEEN 63 AND 77 THEN '63-77'
  WHEN age >= 78 THEN '78+'
  ELSE 'Não informado'
END AS faixa_etaria, COUNT(*) AS quantidade
FROM projeto3-riscorelativo-467822.supercaja.user_info_unificada_V2
GROUP BY faixa_etaria
ORDER BY faixa_etaria;
```

---

## 🔵 Tabelas Auxiliares e Quartis

Valores nulos foram substituídos por 0 nas colunas de empréstimos.

Quartis foram calculados para idade, salário, dependentes, total de empréstimos, atrasos >90 dias, dívida não garantida, atrasos 30-59 dias e razão de dívida.

```sql
WITH quartis_calculados AS (
  SELECT *, NTILE(4) OVER (ORDER BY age) AS quartil_idade, NTILE(4) OVER (ORDER BY last_month_salary) AS quartil_salario, NTILE(4) OVER (ORDER BY number_dependents) AS quartil_dependentes, NTILE(4) OVER (ORDER BY total_emprestimos) AS quartil_emprestimos, NTILE(4) OVER (ORDER BY more_90_days_overdue) AS quartil_atrasos_90dias, NTILE(4) OVER (ORDER BY using_lines_not_secured_personal_assets) AS quartil_divida_nao_garantida, NTILE(4) OVER (ORDER BY number_times_delayed_payment_loan_30_59_days) AS quartil_atrasos_30_59dias, NTILE(4) OVER (ORDER BY debt_ratio) AS quartil_debt_ratio
  FROM projeto3-riscorelativo-467822.supercaja.user_info_V2
)
SELECT * FROM quartis_calculados;
```

---

## Agrupar dados de acordo com variáveis categóricas

Foram identificadas as variáveis e incluídas para a visualização.

## Visualizar variáveis categóricas

Foram incluídos gráficos para a apresentação do dashboard além dos scorecards.

## Aplicar medidas de tendência central

Foram aplicadas medidas de tendência central em idade, dependentes, salários e empréstimos por cliente.

## Ver distribuição

✔

## Aplicar medidas de dispersão

✔

## Calcular quartis, decis ou percentis

Foram calculados quartis das seguintes variáveis numéricas:

```sql
WITH quartis_calculados AS (
SELECT
user_id,
age,
last_month_salary,
number_dependents,
total_emprestimos,
more_90_days_overdue,
using_lines_not_secured_personal_assets,
number_times_delayed_payment_loan_30_59_days,
debt_ratio,
qtd_real_estate,
qtd_other,
faixa_etaria,
default_flag,

-- Calculando quartis para idade
NTILE(4) OVER (ORDER BY age) AS quartil_idade,

-- Calculando quartis para salário
NTILE(4) OVER (ORDER BY last_month_salary) AS quartil_salario,

-- Calculando quartis para dependentes
NTILE(4) OVER (ORDER BY number_dependents) AS quartil_dependentes,

-- Calculando quartis para empréstimos
NTILE(4) OVER (ORDER BY total_emprestimos) AS quartil_emprestimos,

-- Calculando quartis para atrasos >90 dias
NTILE(4) OVER (ORDER BY more_90_days_overdue) AS quartil_atrasos_90dias,

-- Calculando quartis para dívida não garantida
NTILE(4) OVER (ORDER BY using_lines_not_secured_personal_assets) AS quartil_divida_nao_garantida,

-- Calculando quartis para atrasos 30-59 dias
NTILE(4) OVER (ORDER BY number_times_delayed_payment_loan_30_59_days) AS quartil_atrasos_30_59dias,

-- Calculando quartis para razão de dívida
NTILE(4) OVER (ORDER BY debt_ratio) AS quartil_debt_ratio

FROM projeto3-riscorelativo-467822.supercaja.user_info_V2
)

SELECT * FROM quartis_calculados;
```

## Calcular correlação entre variáveis

✔

## Calcular o risco relativo

Alguns cálculos de risco relativo:

* 🔴 Atrasos >90 dias (maior risco geral)

  * Quartil 4: Risco 3,80x (🔴 Muito Alto)
* 🟠 Dívida Não Garantida

  * Quartil 4: Risco 2,46x (🔴 Muito Alto)
* 🟠 Empréstimos

  * Quartil 4: Risco 1,86x (🔴 Muito Alto)
* 🟠 Idade

  * Quartil 1: Risco 1,80x
  * Quartil 4: Risco 0,28x (🟢 Baixo)
* 🟠 Salário

  * Quartil 4: Risco 0,44x (🟢 Baixo)
* 🔴 Atrasos >90 dias

  * Quartil 1: Risco 0,04x (🟢 Baixo)

```sql
risco_por_variavel AS (
SELECT
'idade' AS variavel,
quartil_idade AS quartil,
COUNT() AS total_grupo,
SUM(CASE WHEN default_flag = 'Sim' THEN 1 ELSE 0 END) AS maus_pagadores,
SUM(CASE WHEN default_flag = 'Sim' THEN 1 ELSE 0 END) / COUNT() AS risco_absoluto
FROM quartis
GROUP BY quartil_idade

UNION ALL

SELECT
'salario' AS variavel,
quartil_salario AS quartil,
COUNT() AS total_grupo,
SUM(CASE WHEN default_flag = 'Sim' THEN 1 ELSE 0 END) AS maus_pagadores,
SUM(CASE WHEN default_flag = 'Sim' THEN 1 ELSE 0 END) / COUNT() AS risco_absoluto
FROM quartis
GROUP BY quartil_salario

UNION ALL

SELECT
'dependentes' AS variavel,
quartil_dependentes AS quartil,
COUNT() AS total_grupo,
SUM(CASE WHEN default_flag = 'Sim' THEN 1 ELSE 0 END) AS maus_pagadores,
SUM(CASE WHEN default_flag = 'Sim' THEN 1 ELSE 0 END) / COUNT() AS risco_absoluto
FROM quartis
GROUP BY quartil_dependentes

UNION ALL

SELECT
'emprestimos' AS variavel,
quartil_emprestimos AS quartil,
COUNT() AS total_grupo,
SUM(CASE WHEN default_flag = 'Sim' THEN 1 ELSE 0 END) AS maus_pagadores,
SUM(CASE WHEN default_flag = 'Sim' THEN 1 ELSE 0 END) / COUNT() AS risco_absoluto
FROM quartis
GROUP BY quartil_emprestimos

UNION ALL

SELECT
'divida_nao_garantida' AS variavel,
quartil_divida_nao_garantida AS quartil,
COUNT() AS total_grupo,
SUM(CASE WHEN default_flag = 'Sim' THEN 1 ELSE 0 END) AS maus_pagadores,
SUM(CASE WHEN default_flag = 'Sim' THEN 1 ELSE 0 END) / COUNT() AS risco_absoluto
FROM quartis
GROUP BY quartil_divida_nao_garantida

UNION ALL

SELECT
'debt_ratio' AS variavel,
quartil_debt_ratio AS quartil,
COUNT() AS total_grupo,
SUM(CASE WHEN default_flag = 'Sim' THEN 1 ELSE 0 END) AS maus_pagadores,
SUM(CASE WHEN default_flag = 'Sim' THEN 1 ELSE 0 END) / COUNT() AS risco_absoluto
FROM quartis
GROUP BY quartil_debt_ratio
)
```

## 🔴 Cálculo de Risco Relativo e Score

O risco relativo foi calculado para diversas variáveis, sendo os dias de atraso a maior influência.

A tabela final de score foi ajustada com pesos relativos e classificação de risco.

```sql
-- Exemplo de criação de score
CREATE OR REPLACE TABLE projeto3-riscorelativo-467822.supercaja.tabela_final_completa AS
WITH dummy AS (
  SELECT *, CASE WHEN quartil_idade IN (1,2) THEN 3 ELSE 0 END AS age_score, CASE WHEN quartil_salario = 2 THEN 1 ELSE 0 END AS salary_score, CASE WHEN quartil_emprestimos IN (3,4) THEN 2 ELSE 0 END AS loans_score, CASE WHEN quartil_atrasos_90 = 4 THEN 5 ELSE 0 END AS atraso_score, CASE WHEN quartil_credito_sem_garantia = 4 THEN 2 ELSE 0 END AS credito_score, CASE WHEN quartil_endividamento IN (4,2) THEN 1 ELSE 0 END AS divida_score, CASE WHEN quartil_dependentes = 4 THEN 1 ELSE 0 END AS dependentes_score
  FROM projeto3-riscorelativo-467822.supercaja.quartis_cliente
),
score_calculo AS (
  SELECT *, (age_score + salary_score + loans_score + atraso_score + credito_score + divida_score + dependentes_score) AS score_risco_total
  FROM dummy
)
SELECT user_id, age, last_month_salary, number_dependents, total_emprestimos, more_90_days_overdue, using_lines_not_secured_personal_assets, debt_ratio, default_flag_binario, CASE WHEN score_risco_total >= 10 THEN 'Alto Risco' ELSE 'Baixo Risco' END AS classificacao_risco, score_risco_total
FROM score_calculo;
```

Matriz de confusão e métricas (recall, precisão, acurácia, F1-score) foram calculadas para avaliação da classificação.
