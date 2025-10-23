# 🏅 Arquitetura Medalhão - Pipeline

Projeto demonstrando a implementação da **Arquitetura Medalhão (Medallion Architecture)** utilizando **Delta Lake** e **Apache Spark** no Databricks.

## 📊 Sobre o Projeto

Este projeto implementa um pipeline de dados em três camadas (Bronze, Silver, Gold) para processar e analisar dados, demonstrando:

- ✅ Arquitetura Medalhão (Bronze → Silver → Gold)
- ✅ Formato Delta Lake (Parquet otimizado)
- ✅ Transformações incrementais de dados
- ✅ Qualidade e governança de dados
- ✅ Análises e métricas de negócio

## 🗂️ Estrutura do Projeto

```
data-lakehouse/
│  
├── projeto/
│   ├── resource/                 # dados de origem e entrega
│   │   ├── inputs/               # arquivos CSV/Parquet recebidos da fonte
│   │   └── outputs/              # exportações (CSV, Parquet, Excel) para clientes ou setores
│   │
│   ├── bronze/
│   │   ├── dev/
│   │   │   └── 01_bronze_ingestion_dev.py
│   │   └── prod/
│   │       └── 01_bronze_ingestion_prod.py
│   │
│   ├── silver/
│   │   ├── dev/
│   │   │   └── 02_silver_transformation_dev.py
│   │   └── prod/
│   │       └── 02_silver_transformation_prod.py
│   │
│   └──── gold/
│       ├── dev/
│       │   └── 03_gold_aggregation_dev.py
│       └── prod/
│           └── 03_gold_aggregation_prod.py
│
├── pipelines/
│   ├── pipeline_dev.yaml/
│   └── pipeline_prod.yaml/
│
├── schemas/
│   └── table_schemas.py
│
└── README.md
```

## 📁 Datasets Utilizados

**Fonte:** Databricks Sample Datasets
- `fly-analysis/` - Atrasos em vôos (Apenas duas tabelas)
- `retail/` - Dados empresariais, varias tabelas (ETL, API, Medallion architecture)
- `credit-card-fraud/` - Fraude em cartões de crédito (PCA, Ideal para Machine Learning)
- `telemetria/` - Dados de telemetria (Iot)
- `mongodb/` - Dados NoSQL MongoDB

## 🥉 Camada Bronze (Raw Data)

**Objetivo:** Ingestão bruta dos dados sem transformações

```python
# Leitura e salvamento em Delta formato
df = spark.read.csv(
    f"dbfs:/databricks-datasets/{PROJETO}",
    header=True,
    inferSchema=True
)

df.write.format("delta") \
    .mode("overwrite") \
    .save(f"/mnt/delta/bronze/{PROJETO}")
```

**Características:**
- Dados originais preservados
- Formato Delta para ACID transactions
- Metadados de ingestão (timestamp, fonte)
- Um notebook para cada arquivo

## 🥈 Camada Silver (Cleaned Data)

**Objetivo:** Limpeza, validação e enriquecimento
- Regras de negócio
- Desnormalização
- Geralmente redução da quantidade de tabelas devido aos joins

```python
from pyspark.sql.functions import col, to_timestamp, when

# Leitura da camada Bronze
df_bronze = spark.read.format("delta").load(f"/mnt/delta/bronze/{PROJETO}")

# Transformações
df_silver = df_bronze \
    .filter(col("delay").isNotNull()) \
    .withColumn("date", to_timestamp(col("date"), "MMddHHmm")) \
    .withColumn("delay_category", 
        when(col("delay") <= 0, "On Time")
        .when(col("delay") <= 15, "Minor Delay")
        .when(col("delay") <= 60, "Moderate Delay")
        .otherwise("Severe Delay")
    ) \
    .dropDuplicates()

# Salvamento em Delta
df_silver.write.format("delta") \
    .mode("overwrite") \
    .option("mergeSchema", "true") \
    .save("/mnt/delta/silver/flight_delays_clean")
```

**Características:**
- Remoção de duplicatas, nulos e espaços
- Padronização de formatos
- Enriquecimento de dados
- Schema evolution habilitado
- Alguns setores consomem dados dessa camada já
- Registro de tabelas no Catalog

## 🥇 Camada Gold (Business-Level Aggregations)

**Objetivo:** Dados prontos para consumo analítico

```python
# Métricas agregadas por aeroporto de origem
df_gold_origin = spark.read.format("delta") \
    .load("/mnt/delta/silver/flight_delays_clean") \
    .groupBy("origin") \
    .agg(
        count("*").alias("total_flights"),
        avg("delay").alias("avg_delay"),
        max("delay").alias("max_delay"),
        sum(when(col("delay") > 15, 1).otherwise(0)).alias("delayed_flights")
    )

# Join com informações de aeroportos
df_airports = spark.read.format("delta").load("/mnt/delta/silver/airports_clean")

df_gold_final = df_gold_origin.join(
    df_airports,
    df_gold_origin.origin == df_airports.IATA,
    "left"
).select(
    col("origin"),
    col("City"),
    col("State"),
    col("total_flights"),
    col("avg_delay"),
    col("max_delay"),
    col("delayed_flights")
)

df_gold_final.write.format("delta") \
    .mode("overwrite") \
    .save("/mnt/delta/gold/airport_performance")
```

**Características:**
- Métricas de negócio agregadas
- Dados otimizados para dashboards
- Particionamento estratégico

## 🚀 Como Executar

### Pré-requisitos
- Databricks Workspace
- Cluster Spark configurado
- Acesso aos databricks-datasets

### Execução
1. Clone o repositório
2. Importe os notebooks para o Databricks
3. Execute na ordem: Bronze → Silver → Gold
4. Execute queries analíticas

```bash
# Ordem de execução
01_bronze_ingestion.py      # Ingestão inicial
02_silver_transformation.py  # Limpeza e padronização
03_gold_aggregation.py       # Agregações de negócio
04_analytics_queries.py      # Análises exploratórias
```

## 📈 Queries Analíticas Exemplo
- CTE são muito uteis nas querys SQL para ETL.
```sql
-- Top 10 aeroportos com maiores atrasos médios
SELECT 
    origin,
    City,
    State,
    avg_delay,
    total_flights
FROM delta.`/mnt/delta/gold/airport_performance`
ORDER BY avg_delay DESC
LIMIT 10;

-- Taxa de pontualidade por estado
SELECT 
    State,
    SUM(total_flights - delayed_flights) * 100.0 / SUM(total_flights) as on_time_rate
FROM delta.`/mnt/delta/gold/airport_performance`
GROUP BY State
ORDER BY on_time_rate DESC;
```

## 🔧 Benefícios do Delta Lake

- **ACID Transactions:** Garantia de consistência
- **Time Travel:** Versionamento de dados
- **Schema Evolution:** Alterações de schema sem quebrar pipeline
- **Upserts eficientes:** MERGE operations
- **Compactação automática:** Otimização de storage

```python
# Exemplo de Time Travel
df_versao_anterior = spark.read.format("delta") \
    .option("versionAsOf", 1) \
    .load("/mnt/delta/silver/flight_delays_clean")
```

## 📊 Otimizações Implementadas

- **Z-Ordering:** Organização de dados para queries eficientes
- **Particionamento:** Por ano/mês para queries temporais
- **Vacuum:** Limpeza de arquivos antigos
- **Optimize:** Compactação de small files

```python
# Otimização de tabela Delta
spark.sql("""
    OPTIMIZE delta.`/mnt/delta/gold/airport_performance`
    ZORDER BY (origin, State)
""")
```

## 📚 Conceitos Demonstrados

1. **Arquitetura em Camadas**: Separação clara de responsabilidades
2. **Idempotência**: Pipelines podem ser re-executados com segurança
3. **Incremental Processing**: Suporte a cargas incrementais
4. **Data Quality**: Validações e checks de qualidade
5. **Governança**: Auditoria e lineage de dados

## 🤝 Contribuindo

Contribuições são bem-vindas! Sinta-se à vontade para:
- Reportar bugs
- Sugerir melhorias
- Adicionar novas análises

## 📝 Licença

Este projeto é open source e está disponível sob a licença MIT.

## 👤 Autor

Tiago Novelli - [GitHub Tiago Novelli](https://github.com/TiagoNovelli)

## 🔗 Links Úteis

- [Documentação Delta Lake](https://docs.delta.io/)
- [Medallion Architecture](https://www.databricks.com/glossary/medallion-architecture)
- [Databricks Best Practices](https://docs.databricks.com/delta/best-practices.html)
- [Curso Databricks GitHub](https://github.com/TiagoNovelli/curso-databricks)

---

⭐ Se este projeto foi útil, considere dar uma estrela!