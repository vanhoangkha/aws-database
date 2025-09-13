# Tóm Tắt Các File PDF - AWS Database Services

## 1. NoSQL.pdf - "Dos and don'ts of NoSQL data modeling"

**Tác giả**: Almas Shaikh & Mohammed Fazalullah Qudrath (AWS)

### Nội dung chính:

**SQL vs NoSQL Comparison**:
- **SQL**: Optimized cho storage, normalized/relational, ad hoc queries, scale vertically, tốt cho OLAP
- **NoSQL**: Optimized cho compute, denormalized/hierarchical, instantiated views, scale horizontally, built cho OLTP at scale

**NoSQL Database Types**:
- **Key-value**: Đơn giản nhất, high performance
- **Document**: Designed cho rich JSON data models, supports complex access patterns
- **Wide-column**: Flexible schema, good cho time-series data
- **Graph**: Cho relationship data
- **In-memory**: Extreme performance
- **Time-series**: Specialized cho time-based data
- **Ledger**: Immutable transaction logs

**Key Takeaways**:
- NoSQL databases được thiết kế cho specific use cases
- Data modeling trong NoSQL khác fundamentally với SQL
- Cần hiểu access patterns trước khi design schema
- Denormalization là common practice trong NoSQL

---

## 2. Amazon_Aurora.pdf - "Deep dive on Amazon Aurora"

**Tác giả**: Richard Waymire (Senior Manager, Aurora Product Management)

### Nội dung chính:

**Amazon Aurora Overview**:
- **Enterprise database at open source price**
- Drop-in compatibility với MySQL và PostgreSQL
- Managed service với simplicity và cost-effectiveness
- Throughput và availability của commercial databases

**What's New**:
- **PostgreSQL**: Babelfish support, PostgreSQL 12/13, new extensions (PostGIS 3.1, oracle_fdw, pg_partman)
- **MySQL**: MySQL 8.0 support, improved Read Replica availability
- **Both**: Graviton2 instance support (T4G, R6G, X2G)

**Architecture Highlights**:
- Shared storage architecture across multiple AZs
- Automatic failover và backup
- Read replicas với async replication
- Storage auto-scaling
- Continuous backup to S3

**Key Benefits**:
- 5x performance của MySQL, 3x performance của PostgreSQL
- 99.99% availability SLA
- Automatic scaling và patching
- Point-in-time recovery
- Global database cho cross-region replication

---

## 3. DynamoDB.pdf - "Data modeling with Amazon DynamoDB"

**Tác giả**: Alex DeBrie (Engineering Manager, Serverless, Inc.)

### Nội dung chính:

**DynamoDB Fundamentals**:
- **NoSQL database** fully managed by AWS
- **HTTPS với IAM authentication**
- Fast, consistent performance as it scales
- Built cho hyperscale và hyper-ephemeral compute

**Core Concepts**:
- **Table**: Container cho data
- **Item**: Individual records
- **Primary Key**: Simple (partition key) hoặc Composite (partition + sort key)
- **Attributes**: Data fields trong items

**API Actions**:
- **Item-based actions**: GetItem, PutItem, UpdateItem, DeleteItem (must provide entire primary key)
- **Query**: Must provide partition key, may provide sort key conditions
- **Scan**: Examines every item (avoid khi có thể!)

**Data Modeling Best Practices**:
- Understand access patterns trước
- Design cho single-table design
- Use composite keys effectively
- Leverage GSI (Global Secondary Indexes) cho additional access patterns
- Avoid hot partitions

**Key Takeaway**: DynamoDB requires different thinking - design around access patterns, not entities

---

## 4. Amazon_ElastiCache.pdf - "Deep dive on Amazon ElastiCache for Redis"

**Tác giả**: Joe Travaglini (AWS) & Lindsey Berg (Groupon)

### Nội dung chính:

**ElastiCache for Redis Overview**:
- **Redis compatible**: Fully compatible với open-source Redis
- **Fully managed**: AWS manages hardware/software setup, configuration, monitoring
- **Highly available**: Multi-AZ với auto failover, cross-region replication
- **Extreme performance**: In-memory data store với microsecond response times
- **Secure**: Network isolation, encryption at rest/in transit, RBAC
- **Scalable**: Up to 500 nodes per cluster

**Why Redis**:
- **#1 in-memory data store**
- **Most loved database** by developers
- **Blazing fast performance**
- **Versatile data structures**
- **Feature-rich và easy to use**

**Redis Data Types**:
- **String**: Sequence of bytes, good cho caching
- **Hash**: Field-value pairs
- **List**: Ordered collection
- **Set**: Unordered collection of unique strings
- **Sorted Set**: Ordered set với scores
- **Bitmap**: String treated as bit array
- **HyperLogLog**: Probabilistic data structure

**Use Cases**:
- **Caching**: Session storage, database query caching
- **Real-time analytics**: Leaderboards, counting, metrics
- **Message queuing**: Pub/Sub patterns
- **Geospatial**: Location-based services

**Groupon Case Study**:
- Migration từ self-managed Redis to ElastiCache
- Deal curation system using Redis
- Improved performance và reduced operational overhead

---

## 5. PostgreSQL.pdf - "Deep dive on PostgreSQL databases on Amazon RDS"

**Tác giả**: Jim Mlodgenski (Principal Database Engineer, AWS)

### Nội dung chính:

**Why PostgreSQL**:
- **Open source**: 30+ years active development, community-controlled
- **Performance và scale**: Extensive index support, parallel processing, native partitioning
- **Advanced features**: JSON support, full-text search, geospatial data
- **ACID compliance**: Strong consistency guarantees

**Amazon RDS for PostgreSQL**:
- PostgreSQL community version với easy management
- Supports versions 9.5, 9.6, 10, 11, 12 (13 in preview)
- **High availability**: Multi-AZ deployment across availability zones
- **Automated backups**: Point-in-time recovery
- **Read replicas**: Scale read workloads

**Amazon Aurora PostgreSQL**:
- Built from ground up để leverage AWS services
- **3x performance** của standard PostgreSQL
- **Shared storage architecture**
- **Automatic scaling** và failover
- **Global database** cho cross-region replication

**Key Features**:
- **Extensions**: PostGIS, pg_stat_statements, pg_hint_plan
- **Performance Insights**: Database performance monitoring
- **Parameter groups**: Easy configuration management
- **Security**: Encryption, VPC, IAM integration

**Migration Strategies**:
- **pg_dump/pg_restore**: For smaller databases
- **AWS DMS**: For larger, complex migrations
- **Logical replication**: For minimal downtime migrations

---

## 6. RDS-Ad.pdf - "What's new with Amazon RDS"

**Tác giả**: Bill Jacobi & Alex Zarenin (AWS Principal Solutions Architects)

### Nội dung chính:

**Amazon RDS Overview**:
- **Managed relational database service**
- **Choice of popular database engines**:
  - MySQL, MariaDB, PostgreSQL (open source)
  - Oracle, SQL Server, Db2 (commercial)
  - Aurora (MySQL/PostgreSQL compatible)

**Performance Monitoring & Tuning**:
- **Performance Insights**: Database performance dashboard
- **Enhanced monitoring**: OS-level metrics
- **Query optimization**: Slow query analysis
- **Automated tuning**: Parameter recommendations

**High Availability Features**:
- **Multi-AZ deployments**: Automatic failover
- **Read replicas**: Scale read workloads
- **Backup và restore**: Automated backups, snapshots
- **Point-in-time recovery**: Restore to specific timestamp

**Read Scaling & Cloning**:
- **Read replicas**: Up to 15 replicas
- **Cross-region replicas**: Global read scaling
- **Aurora cloning**: Fast database copies
- **Serverless**: Automatic scaling based on demand

**AI/ML Enhancements**:
- **SageMaker integration**: ML model inference
- **Comprehend integration**: Natural language processing
- **Native ML functions**: Built-in ML capabilities
- **Data export**: Easy integration với ML pipelines

**Recent Innovations**:
- **RDS Proxy**: Connection pooling và management
- **Blue/Green deployments**: Zero-downtime updates
- **Optimized reads**: Improved query performance
- **Storage optimization**: Intelligent tiering

---

## Tổng Kết Chung

### AWS Database Portfolio Strategy:
1. **Purpose-built databases**: Mỗi database type cho specific use cases
2. **Managed services**: Reduce operational overhead
3. **Performance optimization**: Built-in monitoring và tuning
4. **High availability**: Multi-AZ, automatic failover
5. **Security**: Encryption, VPC, IAM integration
6. **AI/ML integration**: Native ML capabilities

### Key Decision Factors:
- **OLTP vs OLAP**: RDS/Aurora cho OLTP, Redshift cho OLAP
- **SQL vs NoSQL**: RDS cho structured data, DynamoDB cho flexible schema
- **Performance requirements**: ElastiCache cho sub-millisecond latency
- **Scale requirements**: Aurora cho high throughput, DynamoDB cho massive scale
- **Operational complexity**: Managed services để reduce overhead

### Best Practices Summary:
1. **Understand access patterns** trước khi chọn database
2. **Use appropriate data modeling** cho each database type
3. **Leverage managed services** để reduce operational burden
4. **Implement proper monitoring** và alerting
5. **Plan for high availability** và disaster recovery
6. **Consider cost optimization** strategies
7. **Integrate security** từ đầu trong design
