## The Entire Platform

The full detail of the platform can get quite complex, but at a high level the structure is fairly simple.

```mermaid
graph LR
  Producers[Data Producers] --> Ingestion
  Ingestion --> Storage[Long-term Storage]
  Ingestion --> Stream[Stream Processing]
  Stream --> Storage
  Batch[Batch Processing] --> Storage
  Storage --> Batch
  Self[Self Serve] -.- Stream
  Self -.- Batch
  Stream -.-> Visualization
  Batch -.-> Visualization
  Stream --> Export
  Batch --> Export
```

Each of these high-level parts of the platform are described in more detail below.

## Ingestion

```mermaid
graph LR
  subgraph HTTP
    tee
    lb[Load Balancer]
    mozingest
  end
  subgraph Kafka
    kafka_unvalidated[Kafka unvalidated]
    kafka_validated[Kafka validated]
    zookeeper[ZooKeeper] -.- kafka_unvalidated
    zookeeper -.- kafka_validated
  end
  subgraph Storage
    s3_heka[S3 Heka Protobuf Storage]
    s3_parquet[S3 Parquet Storage]
  end
  subgraph Data Producers
    Firefox --> lb
    more_producers[Other Producers] --> lb
  end

  lb --> tee
  tee --> mozingest
  mozingest --> kafka_unvalidated
  mozingest --> Landfill
  kafka_unvalidated --> dwl[Data Store Loader]
  kafka_validated --> cep[Hindsight CEP]
  kafka_validated --> sparkstreaming[Spark Streaming]
  Schemas -.->|validation| dwl
  dwl --> kafka_validated
  dwl --> s3_heka
  dwl --> s3_parquet
  sparkstreaming --> s3_parquet
```

##### Data flow for valid submissions

```mermaid
sequenceDiagram
    participant Fx as Firefox
    participant lb as Load Balancer
    participant mi as mozingest
    participant lf as Landfill
    participant k as Kafka
    participant dwl as Data Store Loader
    participant dl as Data Lake

    Fx->>lb: HTTPS POST
    lb->>mi: forward
    mi-->>lf: failsafe store
    mi->>k: enqueue
    k->>dwl: validate, decode
    dwl->>k: enqueue validated
    dwl->>dl: store durably
```

Hindsight is used for [ingestion of logs][hslog] from applications and services, it supports parsing of log lines and appending similar metadata as the HTTP ingestion above (timestamp, source, and so on).

```mermaid
graph TD
  subgraph RDBMS
    PostgreSQL
    Redshift
    MySQL
    BigQuery
  end
  subgraph NoSQL
    DynamoDB
  end
  subgraph S3
    landfill[Landfill]
    s3_heka[Heka Data Lake]
    s3_parquet[Parquet Data Lake]
    s3_analysis[Analysis Outputs]
    s3_public[Public Outputs]
  end

  Ingestion --> s3_heka
  Ingestion --> s3_parquet
  Ingestion --> landfill
  Ingestion -.-> stream[Stream Processing]
  stream --> s3_parquet
  batch[Batch Processing] --> s3_parquet
  batch --> PostgreSQL
  batch --> DynamoDB
  batch --> s3_public
  selfserve[Self Serve] --> s3_analysis
  s3_analysis --> selfserve
  Hive -->|Presto| redash[Re:dash]
  PostgreSQL --> redash
  Redshift --> redash
  MySQL --> redash
  BigQuery --> redash

  s3_parquet -.- Hive
```



Job scheduling and dependency management is done using [Airflow]. Most jobs run once a day, processing data from "yesterday" on each run. A typical job launches a cluster, which fetches the specified ETL code as part of its bootstrap on startup, runs the ETL code, then shuts down upon completion. If something goes wrong, a job may time out or fail, and in this case it is retried automatically.


```mermaid
graph TD
  subgraph Storage
    lake[Data Lake]
    s3_output_public[Public Outputs]
    s3_output_private[Analysis Outputs]
  end
  subgraph ATMO
    Jupyter -->|runs on| emr_cluster[Ad hoc EMR Cluster]
    Zeppelin -->|runs on| emr_cluster
    atmo_service[ATMO Service] -->|launch| emr_cluster
    atmo_service -->|schedule| emr_job[fa:fa-clock-o Scheduled EMR Job]
    emr_cluster -->|mount| EFS
    emr_cluster -->|read + write| lake
    emr_job -->|read + write| s3_output_public
    emr_job -->|read + write| s3_output_private
  end
  subgraph STMO
    redash[Re:dash] -->|read| lake
  end
  subgraph TMO
    evo[Evolution Dashboard]
    histo[Histogram Dashboard]
    agg[Telemetry Aggregates]
    evo -.- agg
    histo -.- agg
  end
  subgraph Databricks
    db_notebook[Notebook]
    db_notebook -->|read + write| lake
  end
```

Most of the data analysis tooling has been developed with the goal of being "self-serve". This means that people should be able to access and analyze data on their own, without involving data engineers or operations. Thus can data access scale beyond a small set of people with specialized knowledge of the entire pipeline.

## Bringing it all together

Finally, here is a more detailed view of the entire platform. Some connections are omitted for clarity.

```mermaid
graph LR
 subgraph Data Producers
  Firefox
  more_producers[...]
 end
 subgraph Storage
  Landfill
  warehouse_heka[Heka Data Lake]
  warehouse_parquet[Parquet Data Lake]
  warehouse_analysis[Analysis Outputs]
  PostgreSQL
  Redshift
  MySQL
  hive[Hive] -.- warehouse_parquet
 end
 subgraph Stream Processing
  cep[Hindsight Streaming]
  dwl[Data Store Loader] --> warehouse_heka
  dwl --> warehouse_parquet
  sparkstreaming[Spark Streaming] --> warehouse_parquet
 end
 subgraph Ingestion
  Firefox --> lb[Load Balancer]
  more_producers --> lb
  lb --> tee
  tee --> mozingest
  mozingest --> kafka
  mozingest --> Landfill
  ZooKeeper -.- kafka[Kafka]
  kafka --> dwl
  kafka --> cep
  kafka --> sparkstreaming
 end
 subgraph Batch Processing
  Airflow -.->|spark|tbv[telemetry-batch-view]
  Airflow -.->|spark|python_mozetl
  warehouse_heka --> tbv
  warehouse_parquet --> tbv
  warehouse_heka --> python_mozetl
  warehouse_parquet --> python_mozetl
  tmo_agg[Telemetry Aggregates]
 end
 subgraph Visualization
  Hindsight
  Jupyter
  Zeppelin
  TMO
  redash_graphs[Re:dash]
  MissionControl
  bespoke_viz[Bespoke Viz]
 end
 subgraph Export
  tbv --> Amplitude
  sparkstreaming --> Amplitude
 end
 subgraph Self Serve
  redash[Re:dash] -.-> Presto
  Presto --> hive
  redash -.-> Athena
  Athena --> hive
  ATMO -.-> spcluster[Spark Cluster]
  warehouse_heka --> spcluster
  warehouse_parquet --> spcluster
  spcluster --> warehouse_analysis
 end
 Schemas -.->|validation| dwl
```

