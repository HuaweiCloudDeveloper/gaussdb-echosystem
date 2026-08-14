<p align="center">
  <p align="center">
    <a href="README_en.md"><strong>English</strong></a> | <strong>简体中文</strong>
</p>

本项目收集GaussDB开源软件生态的总体信息，包括支持的开源软件、版本以及使用指南等。

> ***如果发现项目能帮助到您，别忘了点击右上角`star`表示鼓励***

> ***帮助提供信息：如果您在使用开源软件过程中发现不支持GaussDB的情况，欢迎[提交ISSUE](https://github.com/HuaweiCloudDeveloper/gaussdb-ecosystem/issues)。 您需要提供开源软件名称、使用的版本、存在的问题等信息，我们将结合您提供的信息，完善相关开源软件的支持。***

# GaussDB开源软件生态

| 大类 | 小类    | 开源</br>软件             | 基线</br>版本 | GaussDB</br>版本  | 驱动</br>版本  | 使用</br>指南                                           |
|:------|:------|:----------------------|:----------| :------------ |:-----------|:----------------------------------------------------|
| ORM框架 | Java   | MyBatis               | 3.5.14    |  505.2.0  | 506.0.0    | [使用指南](./MyBatis/3.5.x/README.md)                   |
| ORM框架 | Java   | MyBatis-Plus          | 3.5.12    |  505.2.0  | 506.0.0    | [使用指南](./MyBatis-Plus/3.5.x/README.md)              |
| ORM框架 | Java   | Spring JDBC           | 6.1.10    |  505.2.0  | 506.0.0    | [使用指南](./SpringJDBC/6.1.x/README.md)                |
| ORM框架 | Java   | Spring R2DBC          | 6.1.10    |  505.2.0  | 1.0.0.RC1  | [使用指南](./SpringR2DBC/6.1.x/README.md)               |
| ORM框架 | Java   | Spring Data JPA       | 3.4.5     |  505.2.0  | 506.0.0    | [使用指南](./SpringDataJPA/3.4.x/README.md)             |
| ORM框架 | Java   | Spring Data JPA       | 3.3.12    |  505.2.0  | 506.0.0    | [使用指南](./SpringDataJPA/3.3.x/README.md)             |
| ORM框架 | Java   | Hibernate             | 7.0.0     |  505.2.0  | 506.0.0    | [使用指南](./Hibernate/7.0.x/README.md)                 |
| ORM框架 | Java   | Hibernate             | 6.6.15    |  505.2.0  | 506.0.0    | [使用指南](./Hibernate/6.6.x/README.md)                 |
| ORM框架 | Java   | Spring Data JDBC      | 3.5.1     |  505.2.0  | 506.0.0    | [使用指南](./SpringDataJDBC/3.5.x/README.md)            |
| ORM框架 | Java   | Spring Data JDBC      | 3.4.5     |  505.2.0  | 506.0.0    | [使用指南](./SpringDataJDBC/3.4.x/README.md)            |
| ORM框架 | Java   | Spring Data R2DBC     | 3.5.1     |  505.2.0  | 1.0.0.RC1  | [使用指南](./SpringDataR2DBC/3.5.x/README.md)           |
| ORM框架 | Java   | Spring Data R2DBC     | 3.4.5     |  505.2.0  | 1.0.0.RC1  | [使用指南](./SpringDataR2DBC/3.4.x/README.md)           |
| ORM框架 | Go   | GORM                  | v1.30.0   |  505.2.0  | v1.0.0-rc1 | [使用指南](./GORM/v1.30.0/README.md)                    |
| ORM框架 | Python   | Django ORM            | 4.2.0      | 505.2.0   |  1.0.3  |  [使用指南](./DjangoORM/4.2.0/README.md)                   |
| ORM框架 | Python   | Django ORM            | 5.2.0      | 505.2.0   |  1.0.3  |  [使用指南](./DjangoORM/5.2.0/README.md)                   |
| 开发框架 | 连接池   | HikariCP              | 5.1.0     | 505.2.0  | 506.0.0    | [使用指南](./HikariCP/5.1.x/README.md)                  |
| 开发框架 | 连接池   | Druid                 | 1.2.23    | 505.2.0  | 506.0.0    | [使用指南](./Druid/1.2.x/README.md)                     |
| 开发框架 | 定时任务 | Quartz                | 2.5.0     | 505.2.0  |    506.0.0  | [使用指南](./Quartz/2.5.0/README.md)                    |
| 开发框架 | 系统集成 | Apache Camel          | 4.13.0    |505.2.0|506.0.0| [使用指南](Camel/4.13.x/README.md)                      |
| 数据库工具 | 客户端 | DBeaver               |           | 505.2.0  | 506.0.0    | [使用指南](./DBeaver/25.0.x/README.md)                  |
| 数据库工具 | ETL | Addx                  | 6.0.2     | 505.2.0  | 506.0.0    | [使用指南](Addx/README.md)                              |
| 数据库工具 | ETL | DataX                 | 4.0.0     | 505.2.0  | 506.0.0    | [使用指南](DataX/README.md)                             |
| 数据库工具 | ETL | Kettle                | 9.3.0     | 505.2.0  | 506.0.0    | [使用指南](Kettle/README.md)                            |
| 数据库工具 | ETL | Airflow               | 3.0.4    | 505.2.0  | 506.0.0    | [使用指南](Airflow/README.md)                           |
| 数据库工具 | ETL | Flink(JDBC Connector) | 3.3.0     | 505.2.0  | 506.0.0    | [使用指南](FlinkConnectorJDBC/3.3.x/README.md)          |
| 数据库工具 | ETL | Flink(JDBC Connector) | 4.0.0     | 505.2.0  | 506.0.0    | [使用指南](FlinkConnectorJDBC/4.0.x/README.md)          |
| 中间件 | 分布式任务 | DolphinScheduler      | 3.1.4     | 505.2.0  | 506.0.0    | [使用指南](DolphinScheduler/README.md)                  |
| 中间件 | 分布式任务 | ElasticJob            | 3.1.0     | 505.2.0  | 506.0.0    | [使用指南](Elasticjob/3.1.0/README.md)                  |
| AI | 向量检索 | LlamaIndex-GaussDB    | 0.1.0 | 505.2.0  |            | [使用指南](./LlamaIndex-GaussDB/0.1.0/README.md) |
| AI | 向量检索 | LangChain-GaussDB     | 0.1.0 | 505.2.0  |            | [使用指南](./LangChain-GaussDB/0.1.0/README.md) |
| AI | Agent 记忆 | Mem0-GaussDB     | 2.0.4 | 505.2.0  | - | [使用指南](./Mem0-GaussDB/2.0.4/README.md) |
| AI | 向量检索 | RAGFlow-GaussDB    | 0.26.4 | 505.2.0  | - | [使用指南](./RAGFlow-GaussDB/0.26.4/README.md) |
| AI | 向量检索 | LangGraph-GaussDB     | 0.1.0 | 505.2.0  |            | [使用指南](./LangGraph-GaussDB/0.1.0/README.md) |

* 可以通过 `select version()` 查询GaussDB版本信息。
* 可以参考 [gaussdb-drivers](https://github.com/HuaweiCloudDeveloper/gaussdb-drivers) 进一步了解驱动信息。
* 一般而言，如果版本是 `3.5.14`， 业务可以使用 `3.5.x` 任意版本；如果业务使用 `3.6.x` ，需要进一步验证；不建议业务用于 `3.4.x` 等老版本。 项目会逐步补齐不同版本的覆盖和验证，敬请期待。
