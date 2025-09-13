# Database Concepts - Final Part (Concepts 49-65)

## 49. Database Clustering

**Định nghĩa chi tiết**: Database clustering groups multiple database servers to work as single system, providing high availability, load distribution và fault tolerance. Enables horizontal scaling và eliminates single points of failure.

**Use Case - MySQL Cluster (NDB)**:
```sql
-- MySQL Cluster configuration
-- Data nodes store the actual data
-- SQL nodes provide MySQL interface
-- Management nodes coordinate the cluster

-- Create clustered table
CREATE TABLE customer_orders (
    order_id BIGINT AUTO_INCREMENT,
    customer_id INT NOT NULL,
    order_date DATE,
    total_amount DECIMAL(12,2),
    PRIMARY KEY (order_id),
    KEY (customer_id)
) ENGINE=NDB;

-- Partitioning across cluster nodes
ALTER TABLE customer_orders 
PARTITION BY KEY(customer_id) 
PARTITIONS 4;
```

## 50. Database Proxy & Connection Pooling

**Định nghĩa chi tiết**: Database proxy sits between applications và databases để manage connections, route queries và provide additional features như caching, load balancing và security. Connection pooling reuses database connections để reduce overhead.

**Use Case - ProxySQL Configuration**:
```sql
-- ProxySQL configuration for MySQL
INSERT INTO mysql_servers(hostgroup_id, hostname, port, weight) VALUES
(0, 'mysql-master.internal', 3306, 1000),
(1, 'mysql-slave1.internal', 3306, 900),
(1, 'mysql-slave2.internal', 3306, 800);

-- Query routing rules
INSERT INTO mysql_query_rules(rule_id, active, match_pattern, destination_hostgroup, apply) VALUES
(1, 1, '^SELECT.*', 1, 1),
(2, 1, '^INSERT|UPDATE|DELETE.*', 0, 1);

-- Connection pooling settings
UPDATE global_variables SET variable_value='200' WHERE variable_name='mysql-max_connections';
UPDATE global_variables SET variable_value='10' WHERE variable_name='mysql-default_max_connections';
```

## 51. Database Encryption

**Định nghĩa chi tiết**: Database encryption protects sensitive data bằng cách convert readable data thành encoded format. Includes encryption at rest (stored data) và in transit (data transmission).

**Use Case - Transparent Data Encryption (TDE)**:
```sql
-- Enable encryption at rest
ALTER TABLE sensitive_customer_data ENCRYPTION='Y';

-- Column-level encryption
CREATE TABLE payment_info (
    payment_id INT PRIMARY KEY,
    customer_id INT,
    credit_card_number VARBINARY(255),
    expiry_date_encrypted VARBINARY(255),
    cvv_encrypted VARBINARY(255)
);

-- Encrypt sensitive data before storage
INSERT INTO payment_info (payment_id, customer_id, credit_card_number)
VALUES (1, 12345, AES_ENCRYPT('4532-1234-5678-9012', 'encryption_key'));

-- Decrypt when retrieving
SELECT 
    payment_id,
    customer_id,
    AES_DECRYPT(credit_card_number, 'encryption_key') as card_number
FROM payment_info
WHERE customer_id = 12345;
```

## 52. Database Versioning & Schema Evolution

**Định nghĩa chi tiết**: Database versioning tracks changes to database schema over time, enabling controlled evolution của database structure while maintaining backward compatibility và data integrity.

**Use Case - Database Migration Framework**:
```python
class DatabaseMigration:
    def __init__(self, db_connection):
        self.db = db_connection
        self.setup_migration_table()
    
    def setup_migration_table(self):
        """Create table to track applied migrations"""
        self.db.execute("""
            CREATE TABLE IF NOT EXISTS schema_migrations (
                version VARCHAR(50) PRIMARY KEY,
                description TEXT,
                applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                checksum VARCHAR(64)
            )
        """)
    
    def apply_migration(self, version, description, sql_commands):
        """Apply database migration"""
        
        # Check if already applied
        cursor = self.db.cursor()
        cursor.execute("SELECT version FROM schema_migrations WHERE version = %s", (version,))
        
        if cursor.fetchone():
            print(f"Migration {version} already applied")
            return
        
        try:
            # Execute migration commands
            for command in sql_commands:
                cursor.execute(command)
            
            # Record migration
            cursor.execute("""
                INSERT INTO schema_migrations (version, description)
                VALUES (%s, %s)
            """, (version, description))
            
            self.db.commit()
            print(f"✅ Applied migration {version}: {description}")
            
        except Exception as e:
            self.db.rollback()
            print(f"❌ Failed to apply migration {version}: {e}")
            raise

# Migration examples
migrations = [
    {
        'version': '001_create_users_table',
        'description': 'Create initial users table',
        'commands': [
            """
            CREATE TABLE users (
                user_id INT AUTO_INCREMENT PRIMARY KEY,
                username VARCHAR(50) UNIQUE NOT NULL,
                email VARCHAR(100) UNIQUE NOT NULL,
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
            """
        ]
    },
    {
        'version': '002_add_user_profile_fields',
        'description': 'Add profile fields to users table',
        'commands': [
            "ALTER TABLE users ADD COLUMN first_name VARCHAR(50)",
            "ALTER TABLE users ADD COLUMN last_name VARCHAR(50)",
            "ALTER TABLE users ADD COLUMN birth_date DATE"
        ]
    }
]
```

## 53. Database Testing Strategies

**Định nghĩa chi tiết**: Database testing validates data integrity, performance và functionality của database operations. Includes unit testing, integration testing và performance testing.

**Use Case - Database Unit Testing**:
```python
import unittest
import mysql.connector
from datetime import datetime

class DatabaseTestCase(unittest.TestCase):
    def setUp(self):
        """Setup test database connection"""
        self.db = mysql.connector.connect(
            host='test-db.internal',
            database='test_ecommerce',
            user='test_user',
            password='test_password'
        )
        self.cursor = self.db.cursor()
        
        # Setup test data
        self.setup_test_data()
    
    def tearDown(self):
        """Cleanup after tests"""
        self.cleanup_test_data()
        self.db.close()
    
    def test_create_customer(self):
        """Test customer creation"""
        
        # Insert test customer
        customer_data = {
            'username': 'test_user',
            'email': 'test@example.com',
            'first_name': 'Test',
            'last_name': 'User'
        }
        
        self.cursor.execute("""
            INSERT INTO customers (username, email, first_name, last_name)
            VALUES (%(username)s, %(email)s, %(first_name)s, %(last_name)s)
        """, customer_data)
        
        customer_id = self.cursor.lastrowid
        
        # Verify customer was created
        self.cursor.execute("SELECT * FROM customers WHERE customer_id = %s", (customer_id,))
        result = self.cursor.fetchone()
        
        self.assertIsNotNone(result)
        self.assertEqual(result[1], 'test_user')  # username
        self.assertEqual(result[2], 'test@example.com')  # email
    
    def test_order_total_calculation(self):
        """Test order total calculation trigger"""
        
        # Create test order
        self.cursor.execute("""
            INSERT INTO orders (customer_id, status)
            VALUES (1, 'pending')
        """)
        order_id = self.cursor.lastrowid
        
        # Add order items
        items = [
            (order_id, 1, 2, 100.00),  # 2 items at $100 each
            (order_id, 2, 1, 50.00)    # 1 item at $50
        ]
        
        self.cursor.executemany("""
            INSERT INTO order_items (order_id, product_id, quantity, unit_price)
            VALUES (%s, %s, %s, %s)
        """, items)
        
        # Check calculated total
        self.cursor.execute("SELECT total_amount FROM orders WHERE order_id = %s", (order_id,))
        total = self.cursor.fetchone()[0]
        
        expected_total = (2 * 100.00) + (1 * 50.00)  # $250.00
        self.assertEqual(float(total), expected_total)
```

## 54. Database Documentation

**Định nghĩa chi tiết**: Comprehensive documentation của database schema, business rules, data dictionary và operational procedures. Essential cho maintenance, compliance và knowledge transfer.

**Use Case - Automated Schema Documentation**:
```python
class DatabaseDocumentationGenerator:
    def __init__(self, db_connection):
        self.db = db_connection
    
    def generate_schema_documentation(self):
        """Generate comprehensive schema documentation"""
        
        cursor = self.db.cursor(dictionary=True)
        
        # Get all tables
        cursor.execute("SHOW TABLES")
        tables = [row[f'Tables_in_{self.db.database}'] for row in cursor.fetchall()]
        
        documentation = {
            'database_name': self.db.database,
            'generated_at': datetime.now().isoformat(),
            'tables': {}
        }
        
        for table_name in tables:
            table_doc = self.document_table(table_name)
            documentation['tables'][table_name] = table_doc
        
        return documentation
    
    def document_table(self, table_name):
        """Document individual table structure"""
        
        cursor = self.db.cursor(dictionary=True)
        
        # Get table structure
        cursor.execute(f"DESCRIBE {table_name}")
        columns = cursor.fetchall()
        
        # Get table comment
        cursor.execute(f"""
            SELECT table_comment 
            FROM information_schema.tables 
            WHERE table_schema = DATABASE() AND table_name = '{table_name}'
        """)
        table_comment = cursor.fetchone()['table_comment']
        
        # Get indexes
        cursor.execute(f"SHOW INDEX FROM {table_name}")
        indexes = cursor.fetchall()
        
        # Get foreign keys
        cursor.execute(f"""
            SELECT 
                column_name,
                referenced_table_name,
                referenced_column_name
            FROM information_schema.key_column_usage
            WHERE table_schema = DATABASE() 
              AND table_name = '{table_name}'
              AND referenced_table_name IS NOT NULL
        """)
        foreign_keys = cursor.fetchall()
        
        return {
            'description': table_comment,
            'columns': columns,
            'indexes': indexes,
            'foreign_keys': foreign_keys,
            'estimated_rows': self.get_table_row_count(table_name)
        }
```

## 55-65. Advanced Concepts Summary

**55. Database Containerization**: Docker containers cho database deployment và testing
**56. Microservices Data Patterns**: Database per service, Saga pattern, Event sourcing
**57. Database DevOps**: CI/CD pipelines cho database changes, automated testing
**58. Cloud-Native Databases**: Serverless databases, auto-scaling, managed services
**59. Database Security Scanning**: Vulnerability assessment, compliance checking
**60. Data Mesh Architecture**: Decentralized data ownership, domain-oriented data
**61. Database Observability**: Distributed tracing, metrics correlation, log analysis
**62. Edge Databases**: Data replication to edge locations, offline-first applications
**63. Database Automation**: Self-healing systems, automated optimization, ML-driven tuning
**64. Quantum-Safe Cryptography**: Future-proof encryption methods cho databases
**65. Database Sustainability**: Green computing practices, energy-efficient database operations

## Summary - Complete Database Concepts Guide

Chúng ta đã hoàn thành 65 database concepts covering:

**Foundation (1-15)**:
- Core concepts: Database, DBMS, Keys, Indexes
- Architecture: Partitioning, Logging, Buffering
- Processing: OLTP vs OLAP, Sessions, Query Plans
- Storage: Data Warehouses, NoSQL Types, Caching, Replication

**Advanced Operations (16-30)**:
- Reliability: Backup & Recovery, Transactions, ACID
- Concurrency: Isolation Levels, Stored Procedures, Views
- Integration: ETL, Data Lakes, CAP Theorem
- Scaling: Sharding, Metadata Management

**Enterprise Features (31-45)**:
- Performance: Locking, Query Optimization, Migration
- Security: Data Integrity, Constraints, Deadlock Prevention
- Architecture: Multi-tenancy, Virtualization, Time-series
- AI Integration: Vector Databases, Embeddings, Similarity Search

**Modern Practices (46-65)**:
- Integration: Federation, CDC, Monitoring
- Operations: Clustering, Proxies, Encryption
- Development: Versioning, Testing, Documentation
- Emerging: Edge Computing, Automation, Sustainability

**AWS Services Mapping**:
- **OLTP**: RDS, Aurora, DynamoDB
- **OLAP**: Redshift, Athena, QuickSight
- **NoSQL**: DynamoDB, DocumentDB, Neptune
- **Caching**: ElastiCache, DAX
- **Analytics**: EMR, Glue, Kinesis
- **ML/AI**: SageMaker, Bedrock, OpenSearch

Mỗi concept được minh họa với real-world use cases, complete code examples và AWS implementation patterns để provide comprehensive understanding cho database professionals.
