# Database Concepts - Part 6 (Concepts 31-40)

## 31. Locking Mechanisms

**Định nghĩa chi tiết**: Locking mechanisms kiểm soát concurrent access to database resources để maintain data consistency. Locks prevent conflicts khi multiple transactions access cùng data simultaneously.

**Lock Types**:
- **Shared Lock (S)**: Multiple readers, no writers
- **Exclusive Lock (X)**: Single writer, no readers
- **Intent Locks**: Indicate intention to lock at lower level
- **Update Lock (U)**: Prevent conversion deadlocks

**Lock Granularity**:
- **Row-level**: Lock individual rows (fine-grained)
- **Page-level**: Lock database pages
- **Table-level**: Lock entire tables (coarse-grained)
- **Database-level**: Lock entire database

**Use Case - E-commerce Inventory Management**:
```sql
-- Demonstrate different locking strategies

-- 1. Pessimistic Locking: Lock immediately
BEGIN TRANSACTION;

-- Exclusive lock on product row
SELECT product_id, stock_quantity, reserved_quantity
FROM products 
WHERE product_id = 12345
FOR UPDATE;  -- Exclusive lock until transaction ends

-- Check availability
SET @available = (SELECT stock_quantity - reserved_quantity FROM products WHERE product_id = 12345);

IF @available >= 2 THEN
    -- Reserve inventory
    UPDATE products 
    SET reserved_quantity = reserved_quantity + 2
    WHERE product_id = 12345;
    
    -- Create order
    INSERT INTO orders (customer_id, status, created_at)
    VALUES (67890, 'PROCESSING', NOW());
    
    SET @order_id = LAST_INSERT_ID();
    
    INSERT INTO order_items (order_id, product_id, quantity, unit_price)
    VALUES (@order_id, 12345, 2, (SELECT price FROM products WHERE product_id = 12345));
    
    COMMIT;
    SELECT 'Order created successfully' as result;
ELSE
    ROLLBACK;
    SELECT 'Insufficient inventory' as result;
END IF;

-- 2. Optimistic Locking: Check for conflicts at commit time
-- Add version column to products table
ALTER TABLE products ADD COLUMN version INT DEFAULT 1;

-- Optimistic locking procedure
DELIMITER //
CREATE PROCEDURE OptimisticInventoryReservation(
    IN p_product_id INT,
    IN p_quantity INT,
    IN p_customer_id INT,
    OUT p_result VARCHAR(100)
)
BEGIN
    DECLARE v_current_stock INT;
    DECLARE v_reserved_qty INT;
    DECLARE v_version INT;
    DECLARE v_rows_affected INT;
    
    -- Read current state (no locks)
    SELECT stock_quantity, reserved_quantity, version
    INTO v_current_stock, v_reserved_qty, v_version
    FROM products
    WHERE product_id = p_product_id;
    
    -- Check availability
    IF (v_current_stock - v_reserved_qty) >= p_quantity THEN
        -- Try to update with version check
        UPDATE products 
        SET reserved_quantity = reserved_quantity + p_quantity,
            version = version + 1
        WHERE product_id = p_product_id 
          AND version = v_version;  -- Optimistic lock check
        
        SET v_rows_affected = ROW_COUNT();
        
        IF v_rows_affected = 1 THEN
            -- Success - create order
            INSERT INTO orders (customer_id, status, created_at)
            VALUES (p_customer_id, 'PROCESSING', NOW());
            
            INSERT INTO order_items (order_id, product_id, quantity, unit_price)
            VALUES (LAST_INSERT_ID(), p_product_id, p_quantity, 
                   (SELECT price FROM products WHERE product_id = p_product_id));
            
            SET p_result = 'Order created successfully';
        ELSE
            -- Version conflict - someone else modified the record
            SET p_result = 'Conflict detected - please retry';
        END IF;
    ELSE
        SET p_result = 'Insufficient inventory';
    END IF;
END //
DELIMITER ;

-- 3. Lock escalation demonstration
-- Start with row-level locks
BEGIN TRANSACTION;

-- Lock individual products
SELECT * FROM products WHERE product_id IN (1, 2, 3, 4, 5) FOR UPDATE;

-- If too many row locks, MySQL may escalate to table lock
-- This depends on innodb_lock_wait_timeout and other settings

-- 4. Intent locks (automatic in InnoDB)
-- When acquiring row locks, InnoDB automatically sets intent locks on table
-- IX (Intent Exclusive) - indicates intention to set X locks on rows
-- IS (Intent Shared) - indicates intention to set S locks on rows

-- 5. Lock compatibility matrix demonstration
CREATE TABLE lock_compatibility_demo (
    session_id INT,
    lock_type VARCHAR(20),
    resource_id INT,
    acquired_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Session 1: Shared lock
-- SELECT * FROM products WHERE product_id = 100 LOCK IN SHARE MODE;

-- Session 2: Another shared lock (compatible)
-- SELECT * FROM products WHERE product_id = 100 LOCK IN SHARE MODE;

-- Session 3: Exclusive lock (will wait)
-- SELECT * FROM products WHERE product_id = 100 FOR UPDATE;
```

## 32. Query Optimization Techniques

**Định nghĩa chi tiết**: Query optimization là process cải thiện performance của database queries thông qua better execution plans, proper indexing, query rewriting và resource management.

**Optimization Techniques**:
- **Index Optimization**: Proper index design và usage
- **Query Rewriting**: Restructure queries for better performance
- **Join Optimization**: Choose optimal join algorithms
- **Subquery Optimization**: Convert to joins when beneficial
- **Partition Pruning**: Eliminate unnecessary partitions

**Use Case - E-commerce Analytics Optimization**:
```sql
-- Original slow query (5+ seconds)
SELECT 
    c.customer_name,
    c.email,
    COUNT(o.order_id) as total_orders,
    SUM(o.total_amount) as total_spent,
    AVG(o.total_amount) as avg_order_value,
    MAX(o.order_date) as last_order_date,
    (SELECT COUNT(*) FROM order_items oi 
     JOIN orders o2 ON oi.order_id = o2.order_id 
     WHERE o2.customer_id = c.customer_id) as total_items_purchased
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE c.registration_date >= '2023-01-01'
  AND c.status = 'active'
  AND EXISTS (
      SELECT 1 FROM orders o3 
      WHERE o3.customer_id = c.customer_id 
        AND o3.order_date >= DATE_SUB(NOW(), INTERVAL 90 DAY)
  )
GROUP BY c.customer_id, c.customer_name, c.email
HAVING total_spent > 1000
ORDER BY total_spent DESC;

-- Step 1: Analyze execution plan
EXPLAIN FORMAT=JSON
SELECT ... -- (same query as above)

-- Step 2: Create optimal indexes
CREATE INDEX idx_customers_reg_status ON customers(registration_date, status, customer_id);
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date, total_amount);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);

-- Step 3: Rewrite query for better performance
WITH recent_customers AS (
    -- Pre-filter customers with recent orders
    SELECT DISTINCT c.customer_id, c.customer_name, c.email
    FROM customers c
    INNER JOIN orders o ON c.customer_id = o.customer_id
    WHERE c.registration_date >= '2023-01-01'
      AND c.status = 'active'
      AND o.order_date >= DATE_SUB(NOW(), INTERVAL 90 DAY)
),
customer_stats AS (
    -- Calculate order statistics
    SELECT 
        rc.customer_id,
        rc.customer_name,
        rc.email,
        COUNT(o.order_id) as total_orders,
        SUM(o.total_amount) as total_spent,
        AVG(o.total_amount) as avg_order_value,
        MAX(o.order_date) as last_order_date
    FROM recent_customers rc
    LEFT JOIN orders o ON rc.customer_id = o.customer_id
    GROUP BY rc.customer_id, rc.customer_name, rc.email
    HAVING total_spent > 1000
),
item_counts AS (
    -- Calculate total items purchased
    SELECT 
        o.customer_id,
        COUNT(oi.item_id) as total_items_purchased
    FROM orders o
    INNER JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.customer_id IN (SELECT customer_id FROM customer_stats)
    GROUP BY o.customer_id
)
SELECT 
    cs.*,
    COALESCE(ic.total_items_purchased, 0) as total_items_purchased
FROM customer_stats cs
LEFT JOIN item_counts ic ON cs.customer_id = ic.customer_id
ORDER BY cs.total_spent DESC;

-- Step 4: Create materialized view for frequently accessed data
CREATE TABLE customer_analytics_mv (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    email VARCHAR(100),
    total_orders INT DEFAULT 0,
    total_spent DECIMAL(12,2) DEFAULT 0,
    avg_order_value DECIMAL(10,2) DEFAULT 0,
    total_items_purchased INT DEFAULT 0,
    last_order_date DATE,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_total_spent (total_spent),
    INDEX idx_last_order (last_order_date),
    INDEX idx_last_updated (last_updated)
);

-- Procedure to refresh materialized view
DELIMITER //
CREATE PROCEDURE RefreshCustomerAnalytics()
BEGIN
    -- Truncate and rebuild (for full refresh)
    TRUNCATE TABLE customer_analytics_mv;
    
    INSERT INTO customer_analytics_mv (
        customer_id, customer_name, email, total_orders, 
        total_spent, avg_order_value, total_items_purchased, last_order_date
    )
    SELECT 
        c.customer_id,
        c.customer_name,
        c.email,
        COUNT(o.order_id) as total_orders,
        COALESCE(SUM(o.total_amount), 0) as total_spent,
        COALESCE(AVG(o.total_amount), 0) as avg_order_value,
        COALESCE(SUM(oi_count.item_count), 0) as total_items_purchased,
        MAX(o.order_date) as last_order_date
    FROM customers c
    LEFT JOIN orders o ON c.customer_id = o.customer_id
    LEFT JOIN (
        SELECT order_id, COUNT(*) as item_count
        FROM order_items
        GROUP BY order_id
    ) oi_count ON o.order_id = oi_count.order_id
    WHERE c.status = 'active'
    GROUP BY c.customer_id, c.customer_name, c.email;
    
END //
DELIMITER ;

-- Step 5: Partition large tables for better performance
-- Partition orders table by year
ALTER TABLE orders 
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p2020 VALUES LESS THAN (2021),
    PARTITION p2021 VALUES LESS THAN (2022),
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Now the optimized query (< 0.1 seconds)
SELECT *
FROM customer_analytics_mv
WHERE total_spent > 1000
  AND last_order_date >= DATE_SUB(NOW(), INTERVAL 90 DAY)
ORDER BY total_spent DESC
LIMIT 100;

-- Step 6: Query hints for fine-tuning
SELECT /*+ USE_INDEX(customers, idx_customers_reg_status) */
    c.customer_name,
    o.total_amount
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
WHERE c.registration_date >= '2023-01-01'
  AND c.status = 'active';

-- Step 7: Monitoring query performance
SELECT 
    digest_text,
    count_star as execution_count,
    avg_timer_wait/1000000000 as avg_execution_time_sec,
    sum_lock_time/1000000000 as total_lock_time_sec,
    sum_rows_examined/count_star as avg_rows_examined,
    sum_rows_sent/count_star as avg_rows_returned
FROM performance_schema.events_statements_summary_by_digest
WHERE digest_text LIKE '%customers%'
  AND count_star > 10
ORDER BY avg_timer_wait DESC
LIMIT 10;
```

## 33. Data Migration Strategies

**Định nghĩa chi tiết**: Data migration là process chuyển dữ liệu từ source system sang target system, bao gồm schema conversion, data transformation và validation. Critical cho system upgrades, cloud migrations và platform changes.

**Migration Types**:
- **Big Bang**: Migrate all data at once during downtime
- **Phased**: Migrate data in stages over time
- **Parallel Run**: Run old and new systems simultaneously
- **Trickle Migration**: Continuous sync with eventual cutover

**Use Case - Legacy Oracle to AWS Aurora Migration**:
```python
import boto3
import cx_Oracle
import mysql.connector
import pandas as pd
from datetime import datetime, timedelta
import logging
import json

class DatabaseMigration:
    def __init__(self):
        # Source Oracle connection
        self.oracle_config = {
            'user': 'legacy_user',
            'password': 'legacy_password',
            'dsn': 'legacy-oracle.company.com:1521/PROD'
        }
        
        # Target Aurora MySQL connection
        self.aurora_config = {
            'host': 'aurora-cluster.cluster-xyz.us-east-1.rds.amazonaws.com',
            'database': 'ecommerce_new',
            'user': 'migration_user',
            'password': 'migration_password'
        }
        
        # AWS DMS for ongoing replication
        self.dms_client = boto3.client('dms')
        
        self.migration_log = []
        
    def assess_source_database(self):
        """Assess source database for migration planning"""
        assessment = {
            'tables': {},
            'total_rows': 0,
            'total_size_gb': 0,
            'complex_objects': [],
            'compatibility_issues': []
        }
        
        with cx_Oracle.connect(**self.oracle_config) as conn:
            cursor = conn.cursor()
            
            # Get table information
            cursor.execute("""
                SELECT 
                    table_name,
                    num_rows,
                    ROUND(bytes/1024/1024/1024, 2) as size_gb
                FROM user_tables t
                JOIN user_segments s ON t.table_name = s.segment_name
                WHERE s.segment_type = 'TABLE'
                ORDER BY bytes DESC
            """)
            
            for table_name, num_rows, size_gb in cursor.fetchall():
                assessment['tables'][table_name] = {
                    'rows': num_rows or 0,
                    'size_gb': size_gb or 0,
                    'migration_priority': self._calculate_priority(num_rows, size_gb)
                }
                assessment['total_rows'] += (num_rows or 0)
                assessment['total_size_gb'] += (size_gb or 0)
            
            # Check for Oracle-specific features
            cursor.execute("""
                SELECT object_name, object_type 
                FROM user_objects 
                WHERE object_type IN ('PACKAGE', 'FUNCTION', 'PROCEDURE', 'TRIGGER')
            """)
            
            for obj_name, obj_type in cursor.fetchall():
                assessment['complex_objects'].append({
                    'name': obj_name,
                    'type': obj_type,
                    'conversion_needed': True
                })
        
        return assessment
    
    def _calculate_priority(self, num_rows, size_gb):
        """Calculate migration priority based on size and complexity"""
        if size_gb > 10 or num_rows > 10000000:
            return 'HIGH'
        elif size_gb > 1 or num_rows > 1000000:
            return 'MEDIUM'
        else:
            return 'LOW'
    
    def convert_schema(self, table_name):
        """Convert Oracle schema to MySQL compatible schema"""
        
        with cx_Oracle.connect(**self.oracle_config) as oracle_conn:
            oracle_cursor = oracle_conn.cursor()
            
            # Get Oracle table structure
            oracle_cursor.execute("""
                SELECT 
                    column_name,
                    data_type,
                    data_length,
                    data_precision,
                    data_scale,
                    nullable,
                    data_default
                FROM user_tab_columns
                WHERE table_name = :table_name
                ORDER BY column_id
            """, {'table_name': table_name.upper()})
            
            columns = oracle_cursor.fetchall()
            
            # Convert to MySQL DDL
            mysql_ddl = f"CREATE TABLE {table_name.lower()} (\n"
            column_definitions = []
            
            for col_name, data_type, length, precision, scale, nullable, default in columns:
                mysql_type = self._convert_data_type(data_type, length, precision, scale)
                
                col_def = f"    {col_name.lower()} {mysql_type}"
                
                if nullable == 'N':
                    col_def += " NOT NULL"
                
                if default:
                    # Convert Oracle default to MySQL
                    mysql_default = self._convert_default_value(default, data_type)
                    if mysql_default:
                        col_def += f" DEFAULT {mysql_default}"
                
                column_definitions.append(col_def)
            
            mysql_ddl += ",\n".join(column_definitions)
            
            # Add primary key if exists
            oracle_cursor.execute("""
                SELECT column_name
                FROM user_cons_columns
                WHERE constraint_name = (
                    SELECT constraint_name
                    FROM user_constraints
                    WHERE table_name = :table_name AND constraint_type = 'P'
                )
                ORDER BY position
            """, {'table_name': table_name.upper()})
            
            pk_columns = [row[0].lower() for row in oracle_cursor.fetchall()]
            if pk_columns:
                mysql_ddl += f",\n    PRIMARY KEY ({', '.join(pk_columns)})"
            
            mysql_ddl += "\n);"
            
            return mysql_ddl
    
    def _convert_data_type(self, oracle_type, length, precision, scale):
        """Convert Oracle data types to MySQL equivalents"""
        type_mapping = {
            'VARCHAR2': f'VARCHAR({length})' if length else 'VARCHAR(255)',
            'CHAR': f'CHAR({length})' if length else 'CHAR(1)',
            'NUMBER': self._convert_number_type(precision, scale),
            'DATE': 'DATETIME',
            'TIMESTAMP': 'TIMESTAMP',
            'CLOB': 'LONGTEXT',
            'BLOB': 'LONGBLOB',
            'RAW': 'VARBINARY(255)',
            'LONG RAW': 'LONGBLOB'
        }
        
        return type_mapping.get(oracle_type, 'TEXT')
    
    def _convert_number_type(self, precision, scale):
        """Convert Oracle NUMBER to appropriate MySQL numeric type"""
        if precision is None:
            return 'DECIMAL(65,30)'  # MySQL max precision
        elif scale == 0:
            if precision <= 3:
                return 'TINYINT'
            elif precision <= 5:
                return 'SMALLINT'
            elif precision <= 10:
                return 'INT'
            else:
                return 'BIGINT'
        else:
            return f'DECIMAL({precision},{scale})'
    
    def _convert_default_value(self, oracle_default, data_type):
        """Convert Oracle default values to MySQL format"""
        if not oracle_default:
            return None
        
        oracle_default = oracle_default.strip()
        
        # Common conversions
        conversions = {
            'SYSDATE': 'CURRENT_TIMESTAMP',
            'SYSTIMESTAMP': 'CURRENT_TIMESTAMP',
            'USER': 'USER()',
            'NULL': 'NULL'
        }
        
        return conversions.get(oracle_default.upper(), oracle_default)
    
    def migrate_table_data(self, table_name, batch_size=10000):
        """Migrate data from Oracle to Aurora in batches"""
        
        with cx_Oracle.connect(**self.oracle_config) as oracle_conn, \
             mysql.connector.connect(**self.aurora_config) as mysql_conn:
            
            oracle_cursor = oracle_conn.cursor()
            mysql_cursor = mysql_conn.cursor()
            
            # Get total row count
            oracle_cursor.execute(f"SELECT COUNT(*) FROM {table_name}")
            total_rows = oracle_cursor.fetchone()[0]
            
            print(f"Migrating {total_rows} rows from {table_name}")
            
            # Get column names
            oracle_cursor.execute(f"SELECT * FROM {table_name} WHERE ROWNUM = 1")
            columns = [desc[0].lower() for desc in oracle_cursor.description]
            
            # Prepare INSERT statement
            placeholders = ', '.join(['%s'] * len(columns))
            insert_sql = f"INSERT INTO {table_name.lower()} ({', '.join(columns)}) VALUES ({placeholders})"
            
            # Migrate in batches
            offset = 0
            migrated_rows = 0
            
            while offset < total_rows:
                # Fetch batch from Oracle
                oracle_cursor.execute(f"""
                    SELECT * FROM (
                        SELECT t.*, ROW_NUMBER() OVER (ORDER BY ROWID) as rn
                        FROM {table_name} t
                    ) WHERE rn > {offset} AND rn <= {offset + batch_size}
                """)
                
                batch_data = oracle_cursor.fetchall()
                
                if not batch_data:
                    break
                
                # Convert data types if needed
                converted_batch = []
                for row in batch_data:
                    converted_row = []
                    for i, value in enumerate(row[:-1]):  # Exclude ROW_NUMBER column
                        converted_value = self._convert_data_value(value, columns[i])
                        converted_row.append(converted_value)
                    converted_batch.append(converted_row)
                
                # Insert batch into MySQL
                try:
                    mysql_cursor.executemany(insert_sql, converted_batch)
                    mysql_conn.commit()
                    
                    migrated_rows += len(converted_batch)
                    print(f"Migrated {migrated_rows}/{total_rows} rows ({migrated_rows/total_rows*100:.1f}%)")
                    
                except Exception as e:
                    mysql_conn.rollback()
                    print(f"Error migrating batch at offset {offset}: {e}")
                    # Log failed batch for manual review
                    self.migration_log.append({
                        'table': table_name,
                        'offset': offset,
                        'error': str(e),
                        'timestamp': datetime.now()
                    })
                
                offset += batch_size
            
            print(f"✅ Migration completed for {table_name}: {migrated_rows} rows")
    
    def _convert_data_value(self, value, column_name):
        """Convert Oracle data values to MySQL compatible format"""
        if value is None:
            return None
        
        # Handle Oracle-specific data types
        if isinstance(value, cx_Oracle.LOB):
            return value.read()
        elif isinstance(value, datetime):
            return value
        else:
            return value
    
    def setup_dms_replication(self, table_list):
        """Setup AWS DMS for ongoing replication"""
        
        # Create replication instance
        replication_instance = self.dms_client.create_replication_instance(
            ReplicationInstanceIdentifier='oracle-to-aurora-repl',
            ReplicationInstanceClass='dms.t3.medium',
            AllocatedStorage=100,
            VpcSecurityGroupIds=['sg-12345678'],
            ReplicationSubnetGroupIdentifier='dms-subnet-group',
            MultiAZ=False,
            PubliclyAccessible=False
        )
        
        # Create source endpoint (Oracle)
        source_endpoint = self.dms_client.create_endpoint(
            EndpointIdentifier='oracle-source',
            EndpointType='source',
            EngineName='oracle',
            Username=self.oracle_config['user'],
            Password=self.oracle_config['password'],
            ServerName='legacy-oracle.company.com',
            Port=1521,
            DatabaseName='PROD'
        )
        
        # Create target endpoint (Aurora MySQL)
        target_endpoint = self.dms_client.create_endpoint(
            EndpointIdentifier='aurora-target',
            EndpointType='target',
            EngineName='mysql',
            Username=self.aurora_config['user'],
            Password=self.aurora_config['password'],
            ServerName=self.aurora_config['host'],
            Port=3306,
            DatabaseName=self.aurora_config['database']
        )
        
        # Create replication task
        table_mappings = {
            "rules": [
                {
                    "rule-type": "selection",
                    "rule-id": "1",
                    "rule-name": "1",
                    "object-locator": {
                        "schema-name": "LEGACY_SCHEMA",
                        "table-name": table_name
                    },
                    "rule-action": "include"
                } for table_name in table_list
            ]
        }
        
        replication_task = self.dms_client.create_replication_task(
            ReplicationTaskIdentifier='oracle-aurora-migration',
            SourceEndpointArn=source_endpoint['Endpoint']['EndpointArn'],
            TargetEndpointArn=target_endpoint['Endpoint']['EndpointArn'],
            ReplicationInstanceArn=replication_instance['ReplicationInstance']['ReplicationInstanceArn'],
            MigrationType='full-load-and-cdc',
            TableMappings=json.dumps(table_mappings)
        )
        
        return replication_task
    
    def validate_migration(self, table_name):
        """Validate data consistency after migration"""
        
        with cx_Oracle.connect(**self.oracle_config) as oracle_conn, \
             mysql.connector.connect(**self.aurora_config) as mysql_conn:
            
            oracle_cursor = oracle_conn.cursor()
            mysql_cursor = mysql_conn.cursor()
            
            # Row count validation
            oracle_cursor.execute(f"SELECT COUNT(*) FROM {table_name}")
            oracle_count = oracle_cursor.fetchone()[0]
            
            mysql_cursor.execute(f"SELECT COUNT(*) FROM {table_name.lower()}")
            mysql_count = mysql_cursor.fetchone()[0]
            
            validation_result = {
                'table': table_name,
                'oracle_rows': oracle_count,
                'mysql_rows': mysql_count,
                'row_count_match': oracle_count == mysql_count,
                'validation_time': datetime.now()
            }
            
            # Sample data validation
            oracle_cursor.execute(f"SELECT * FROM {table_name} WHERE ROWNUM <= 100")
            oracle_sample = oracle_cursor.fetchall()
            
            mysql_cursor.execute(f"SELECT * FROM {table_name.lower()} LIMIT 100")
            mysql_sample = mysql_cursor.fetchall()
            
            # Compare checksums or key fields
            validation_result['sample_validation'] = len(oracle_sample) == len(mysql_sample)
            
            return validation_result

# Usage example
migration = DatabaseMigration()

# Step 1: Assess source database
assessment = migration.assess_source_database()
print(f"Assessment: {assessment['total_rows']} rows, {assessment['total_size_gb']} GB")

# Step 2: Convert schemas
tables_to_migrate = ['CUSTOMERS', 'ORDERS', 'PRODUCTS', 'ORDER_ITEMS']

for table in tables_to_migrate:
    ddl = migration.convert_schema(table)
    print(f"DDL for {table}:")
    print(ddl)
    print("-" * 50)

# Step 3: Migrate data
for table in tables_to_migrate:
    migration.migrate_table_data(table)

# Step 4: Setup ongoing replication
dms_task = migration.setup_dms_replication(tables_to_migrate)

# Step 5: Validate migration
for table in tables_to_migrate:
    validation = migration.validate_migration(table)
    print(f"Validation for {table}: {validation}")
```
## 34. Data Security & Access Control

**Định nghĩa chi tiết**: Database security bảo vệ dữ liệu khỏi unauthorized access, data breaches và cyber attacks thông qua authentication, authorization, encryption và auditing mechanisms.

**Security Layers**:
- **Network Security**: Firewalls, VPNs, private subnets
- **Authentication**: Verify user identity (passwords, MFA, certificates)
- **Authorization**: Control what authenticated users can do (RBAC, ABAC)
- **Encryption**: Protect data at rest và in transit
- **Auditing**: Track và log all database activities

**Use Case - Healthcare System Security (HIPAA Compliance)**:
```sql
-- Role-based access control
CREATE ROLE doctor_role;
CREATE ROLE nurse_role;
CREATE ROLE admin_role;
CREATE ROLE patient_role;

-- Grant specific permissions to roles
GRANT SELECT, INSERT, UPDATE ON patient_records TO doctor_role;
GRANT SELECT, UPDATE ON patient_vitals TO nurse_role;
GRANT ALL PRIVILEGES ON *.* TO admin_role;
GRANT SELECT ON patient_records TO patient_role WHERE patient_id = USER_ID();

-- Create users and assign roles
CREATE USER 'dr_smith'@'%' IDENTIFIED BY 'SecurePass123!' REQUIRE SSL;
CREATE USER 'nurse_jones'@'%' IDENTIFIED BY 'NursePass456!' REQUIRE SSL;
GRANT doctor_role TO 'dr_smith'@'%';
GRANT nurse_role TO 'nurse_jones'@'%';

-- Row-level security for patient data
CREATE TABLE patient_records (
    patient_id INT PRIMARY KEY,
    ssn VARCHAR(11) ENCRYPTED,
    medical_history TEXT ENCRYPTED,
    doctor_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Encryption at rest
ALTER TABLE patient_records ENCRYPTION='Y';

-- Audit logging
CREATE TABLE security_audit_log (
    log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_name VARCHAR(100),
    action_type VARCHAR(50),
    table_name VARCHAR(100),
    record_id VARCHAR(100),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address VARCHAR(45),
    success BOOLEAN
);
```

## 35. Performance Monitoring & Tuning

**Định nghĩa chi tiết**: Performance monitoring theo dõi database metrics để identify bottlenecks và optimize system performance thông qua query tuning, resource allocation và configuration adjustments.

**Key Metrics**:
- **Response Time**: Query execution time
- **Throughput**: Transactions per second (TPS)
- **Resource Utilization**: CPU, memory, I/O usage
- **Lock Contention**: Blocking và deadlock frequency
- **Cache Hit Ratio**: Buffer pool efficiency

**Use Case - E-commerce Performance Dashboard**:
```python
import boto3
import mysql.connector
import time
from datetime import datetime, timedelta

class DatabasePerformanceMonitor:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        self.rds = boto3.client('rds')
        
    def collect_mysql_metrics(self, connection):
        """Collect MySQL performance metrics"""
        cursor = connection.cursor(dictionary=True)
        
        # Query performance metrics
        cursor.execute("SHOW GLOBAL STATUS LIKE 'Questions'")
        questions = cursor.fetchone()['Value']
        
        cursor.execute("SHOW GLOBAL STATUS LIKE 'Uptime'")
        uptime = cursor.fetchone()['Value']
        
        qps = int(questions) / int(uptime)
        
        # Buffer pool metrics
        cursor.execute("SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests'")
        read_requests = int(cursor.fetchone()['Value'])
        
        cursor.execute("SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads'")
        disk_reads = int(cursor.fetchone()['Value'])
        
        buffer_hit_ratio = ((read_requests - disk_reads) / read_requests) * 100
        
        # Slow queries
        cursor.execute("SHOW GLOBAL STATUS LIKE 'Slow_queries'")
        slow_queries = int(cursor.fetchone()['Value'])
        
        return {
            'queries_per_second': qps,
            'buffer_hit_ratio': buffer_hit_ratio,
            'slow_queries': slow_queries,
            'uptime_hours': int(uptime) / 3600
        }
    
    def get_top_slow_queries(self, connection):
        """Get slowest queries from performance schema"""
        cursor = connection.cursor(dictionary=True)
        
        cursor.execute("""
            SELECT 
                digest_text,
                count_star as execution_count,
                avg_timer_wait/1000000000 as avg_execution_time_sec,
                sum_timer_wait/1000000000 as total_time_sec,
                sum_rows_examined/count_star as avg_rows_examined
            FROM performance_schema.events_statements_summary_by_digest
            WHERE digest_text IS NOT NULL
            ORDER BY avg_timer_wait DESC
            LIMIT 10
        """)
        
        return cursor.fetchall()
```

## 36. Disaster Recovery & High Availability

**Định nghĩa chi tiết**: Disaster Recovery đảm bảo business continuity khi có major failures thông qua backup strategies, failover mechanisms và geographic redundancy. High Availability minimizes downtime với redundant systems.

**DR Strategies**:
- **RTO (Recovery Time Objective)**: Maximum acceptable downtime
- **RPO (Recovery Point Objective)**: Maximum acceptable data loss
- **Hot Standby**: Real-time replication, instant failover
- **Warm Standby**: Periodic sync, moderate recovery time
- **Cold Standby**: Backup restoration, longer recovery time

**Use Case - Financial Services DR Architecture**:
```python
import boto3
from datetime import datetime

class DisasterRecoveryManager:
    def __init__(self):
        self.rds = boto3.client('rds')
        self.route53 = boto3.client('route53')
        
    def setup_multi_region_dr(self):
        """Setup cross-region disaster recovery"""
        
        # Primary region: us-east-1
        primary_config = {
            'region': 'us-east-1',
            'cluster_id': 'banking-prod-primary',
            'rto_minutes': 5,
            'rpo_minutes': 1
        }
        
        # DR region: us-west-2  
        dr_config = {
            'region': 'us-west-2',
            'cluster_id': 'banking-prod-dr',
            'rto_minutes': 15,
            'rpo_minutes': 5
        }
        
        # Create Aurora Global Database
        global_cluster = self.rds.create_global_cluster(
            GlobalClusterIdentifier='banking-global-cluster',
            SourceDBClusterIdentifier='banking-prod-primary',
            Engine='aurora-mysql',
            EngineVersion='8.0.mysql_aurora.3.02.0'
        )
        
        return global_cluster
    
    def initiate_failover(self, target_region):
        """Initiate disaster recovery failover"""
        
        # 1. Promote secondary cluster to primary
        response = self.rds.failover_global_cluster(
            GlobalClusterIdentifier='banking-global-cluster',
            TargetDbClusterIdentifier='banking-prod-dr'
        )
        
        # 2. Update DNS to point to DR region
        self.route53.change_resource_record_sets(
            HostedZoneId='Z123456789',
            ChangeBatch={
                'Changes': [{
                    'Action': 'UPSERT',
                    'ResourceRecordSet': {
                        'Name': 'banking-db.company.com',
                        'Type': 'CNAME',
                        'TTL': 60,
                        'ResourceRecords': [{'Value': 'banking-prod-dr.cluster-xyz.us-west-2.rds.amazonaws.com'}]
                    }
                }]
            }
        )
        
        return response
```
## 37. Data Archiving & Lifecycle Management

**Định nghĩa chi tiết**: Data archiving là process di chuyển old/inactive data từ production systems sang cheaper, long-term storage để optimize performance và reduce costs. Lifecycle management tự động hóa data movement dựa trên age, access patterns và business rules.

**Archiving Strategies**:
- **Time-based**: Archive data older than X months/years
- **Usage-based**: Archive rarely accessed data
- **Size-based**: Archive when database reaches size limits
- **Compliance-based**: Archive theo regulatory requirements

**Use Case - E-commerce Order Archiving System**:
```sql
-- Current year orders (hot data)
CREATE TABLE orders_current (
    order_id BIGINT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    total_amount DECIMAL(12,2),
    status VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_order_date (order_date),
    INDEX idx_customer_id (customer_id)
);

-- Archive table for old orders (cold data)
CREATE TABLE orders_archive (
    order_id BIGINT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    total_amount DECIMAL(12,2),
    status VARCHAR(20),
    created_at TIMESTAMP,
    archived_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_order_date (order_date),
    INDEX idx_archived_at (archived_at)
) ENGINE=ARCHIVE;  -- Compressed storage engine

-- Automated archiving procedure
DELIMITER //
CREATE PROCEDURE ArchiveOldOrders()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE batch_size INT DEFAULT 10000;
    DECLARE archived_count INT DEFAULT 0;
    DECLARE cutoff_date DATE;
    
    -- Archive orders older than 2 years
    SET cutoff_date = DATE_SUB(CURDATE(), INTERVAL 2 YEAR);
    
    -- Move data in batches
    archive_loop: LOOP
        -- Insert batch into archive
        INSERT INTO orders_archive (
            order_id, customer_id, order_date, total_amount, status, created_at
        )
        SELECT order_id, customer_id, order_date, total_amount, status, created_at
        FROM orders_current
        WHERE order_date < cutoff_date
        LIMIT batch_size;
        
        SET archived_count = ROW_COUNT();
        
        IF archived_count = 0 THEN
            LEAVE archive_loop;
        END IF;
        
        -- Delete from current table
        DELETE FROM orders_current
        WHERE order_date < cutoff_date
        LIMIT batch_size;
        
        -- Log progress
        INSERT INTO archive_log (table_name, archived_count, archive_date)
        VALUES ('orders_current', archived_count, NOW());
        
    END LOOP;
    
END //
DELIMITER ;

-- Schedule archiving job
CREATE EVENT archive_orders_monthly
ON SCHEDULE EVERY 1 MONTH
STARTS '2024-01-01 02:00:00'
DO
    CALL ArchiveOldOrders();
```

**AWS S3 Lifecycle Integration**:
```python
import boto3
import pandas as pd
from datetime import datetime, timedelta

class DataLifecycleManager:
    def __init__(self):
        self.s3 = boto3.client('s3')
        self.glue = boto3.client('glue')
        
    def setup_s3_lifecycle_policy(self, bucket_name):
        """Setup S3 lifecycle policy for data archiving"""
        
        lifecycle_config = {
            'Rules': [
                {
                    'ID': 'DatabaseArchiveRule',
                    'Status': 'Enabled',
                    'Filter': {'Prefix': 'database-archives/'},
                    'Transitions': [
                        {
                            'Days': 30,
                            'StorageClass': 'STANDARD_IA'  # Infrequent Access
                        },
                        {
                            'Days': 90,
                            'StorageClass': 'GLACIER'      # Long-term archive
                        },
                        {
                            'Days': 365,
                            'StorageClass': 'DEEP_ARCHIVE' # Lowest cost
                        }
                    ],
                    'Expiration': {
                        'Days': 2555  # 7 years retention
                    }
                }
            ]
        }
        
        self.s3.put_bucket_lifecycle_configuration(
            Bucket=bucket_name,
            LifecycleConfiguration=lifecycle_config
        )
    
    def export_to_s3_archive(self, table_name, cutoff_date):
        """Export old data to S3 for archiving"""
        
        # Export data as Parquet for efficient storage
        export_path = f"s3://company-data-archive/database-archives/{table_name}/year={cutoff_date.year}/"
        
        # Use AWS Glue for large data exports
        job_response = self.glue.start_job_run(
            JobName='database-archive-export',
            Arguments={
                '--table_name': table_name,
                '--cutoff_date': cutoff_date.isoformat(),
                '--output_path': export_path,
                '--format': 'parquet',
                '--compression': 'snappy'
            }
        )
        
        return job_response
```

## 38. Big Data Integration

**Định nghĩa chi tiết**: Big Data integration kết nối traditional databases với big data platforms để process large volumes, variety và velocity của data. Enables analytics trên structured và unstructured data sources.

**Integration Patterns**:
- **ETL to Data Lake**: Extract từ DB, load vào S3/HDFS
- **Real-time Streaming**: Kafka, Kinesis cho live data feeds
- **Federated Queries**: Query across DB và big data stores
- **Lambda Architecture**: Batch và stream processing layers

**Use Case - IoT Data Processing Pipeline**:
```python
import boto3
import json
from datetime import datetime

class BigDataIntegration:
    def __init__(self):
        self.kinesis = boto3.client('kinesis')
        self.firehose = boto3.client('firehose')
        self.athena = boto3.client('athena')
        
    def setup_streaming_pipeline(self):
        """Setup real-time data pipeline for IoT sensors"""
        
        # 1. Kinesis Data Stream for real-time ingestion
        stream_config = {
            'StreamName': 'iot-sensor-data',
            'ShardCount': 10,
            'RetentionPeriod': 168  # 7 days
        }
        
        # 2. Kinesis Data Firehose for S3 delivery
        firehose_config = {
            'DeliveryStreamName': 'iot-to-s3',
            'DeliveryStreamType': 'KinesisStreamAsSource',
            'KinesisStreamSourceConfiguration': {
                'KinesisStreamARN': 'arn:aws:kinesis:us-east-1:123456789012:stream/iot-sensor-data',
                'RoleARN': 'arn:aws:iam::123456789012:role/firehose-delivery-role'
            },
            'ExtendedS3DestinationConfiguration': {
                'BucketARN': 'arn:aws:s3:::iot-data-lake',
                'Prefix': 'year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/hour=!{timestamp:HH}/',
                'BufferingHints': {
                    'SizeInMBs': 128,
                    'IntervalInSeconds': 60
                },
                'CompressionFormat': 'GZIP',
                'DataFormatConversionConfiguration': {
                    'Enabled': True,
                    'OutputFormatConfiguration': {
                        'Serializer': {
                            'ParquetSerDe': {}
                        }
                    }
                }
            }
        }
        
        return firehose_config
    
    def process_sensor_data(self, sensor_readings):
        """Process and enrich sensor data"""
        
        enriched_data = []
        
        for reading in sensor_readings:
            # Enrich with metadata
            enriched_reading = {
                'sensor_id': reading['sensor_id'],
                'timestamp': reading['timestamp'],
                'temperature': reading['temperature'],
                'humidity': reading['humidity'],
                'location': reading['location'],
                
                # Calculated fields
                'heat_index': self.calculate_heat_index(
                    reading['temperature'], 
                    reading['humidity']
                ),
                'alert_level': self.determine_alert_level(reading),
                'processing_time': datetime.utcnow().isoformat()
            }
            
            enriched_data.append(enriched_reading)
        
        return enriched_data
    
    def federated_analytics_query(self):
        """Query across RDS and S3 data lake using Athena"""
        
        query = """
        WITH recent_orders AS (
            SELECT customer_id, order_date, total_amount
            FROM "rds_database"."orders"
            WHERE order_date >= current_date - interval '30' day
        ),
        customer_behavior AS (
            SELECT 
                customer_id,
                avg(session_duration) as avg_session_duration,
                count(*) as page_views
            FROM "data_lake"."web_analytics"
            WHERE year = '2024' AND month = '01'
            GROUP BY customer_id
        )
        SELECT 
            ro.customer_id,
            ro.total_amount,
            cb.avg_session_duration,
            cb.page_views,
            CASE 
                WHEN cb.avg_session_duration > 300 AND ro.total_amount > 1000 
                THEN 'High Value'
                ELSE 'Standard'
            END as customer_segment
        FROM recent_orders ro
        LEFT JOIN customer_behavior cb ON ro.customer_id = cb.customer_id
        """
        
        response = self.athena.start_query_execution(
            QueryString=query,
            ResultConfiguration={
                'OutputLocation': 's3://athena-query-results/'
            }
        )
        
        return response
```

## 39. Polyglot Persistence

**Định nghĩa chi tiết**: Polyglot Persistence sử dụng multiple database technologies trong cùng application để leverage strengths của each database type cho specific use cases. Different data models cho different requirements.

**Database Selection Criteria**:
- **Relational (MySQL/PostgreSQL)**: ACID transactions, complex queries
- **Document (MongoDB/DocumentDB)**: Flexible schema, nested data
- **Key-Value (Redis/DynamoDB)**: High performance, simple operations
- **Graph (Neptune)**: Relationships, network analysis
- **Search (Elasticsearch)**: Full-text search, analytics

**Use Case - Social Media Platform Architecture**:
```python
import mysql.connector
import boto3
import redis
import json
from datetime import datetime

class SocialMediaPlatform:
    def __init__(self):
        # Relational DB for user profiles and transactions
        self.mysql_conn = mysql.connector.connect(
            host='aurora-cluster.cluster-xyz.us-east-1.rds.amazonaws.com',
            database='social_media',
            user='app_user',
            password='app_password'
        )
        
        # Document DB for posts and content
        self.dynamodb = boto3.resource('dynamodb')
        self.posts_table = self.dynamodb.Table('Posts')
        
        # Key-value store for sessions and cache
        self.redis_client = redis.Redis(
            host='elasticache-cluster.abc123.cache.amazonaws.com',
            port=6379,
            decode_responses=True
        )
        
        # Graph DB for relationships
        self.neptune_endpoint = 'neptune-cluster.cluster-xyz.us-east-1.neptune.amazonaws.com'
        
        # Search engine for content discovery
        self.elasticsearch_endpoint = 'search-social-xyz.us-east-1.es.amazonaws.com'
    
    def create_user_profile(self, user_data):
        """Store user profile in relational database"""
        cursor = self.mysql_conn.cursor()
        
        cursor.execute("""
            INSERT INTO users (username, email, first_name, last_name, birth_date, created_at)
            VALUES (%s, %s, %s, %s, %s, %s)
        """, (
            user_data['username'],
            user_data['email'], 
            user_data['first_name'],
            user_data['last_name'],
            user_data['birth_date'],
            datetime.now()
        ))
        
        user_id = cursor.lastrowid
        self.mysql_conn.commit()
        
        return user_id
    
    def create_post(self, user_id, content, media_urls=None):
        """Store post content in document database"""
        
        post_data = {
            'post_id': f"post_{int(datetime.now().timestamp())}_{user_id}",
            'user_id': user_id,
            'content': content,
            'media_urls': media_urls or [],
            'created_at': datetime.now().isoformat(),
            'likes_count': 0,
            'comments_count': 0,
            'shares_count': 0,
            'hashtags': self.extract_hashtags(content),
            'mentions': self.extract_mentions(content)
        }
        
        # Store in DynamoDB
        self.posts_table.put_item(Item=post_data)
        
        # Index in Elasticsearch for search
        self.index_post_for_search(post_data)
        
        return post_data['post_id']
    
    def create_user_session(self, user_id, session_data):
        """Store session data in Redis cache"""
        
        session_key = f"session:{user_id}"
        session_info = {
            'user_id': user_id,
            'login_time': datetime.now().isoformat(),
            'last_activity': datetime.now().isoformat(),
            'device_info': session_data.get('device_info', {}),
            'preferences': session_data.get('preferences', {})
        }
        
        # Store with 24-hour expiration
        self.redis_client.setex(
            session_key,
            86400,  # 24 hours
            json.dumps(session_info)
        )
        
        return session_key
    
    def follow_user(self, follower_id, following_id):
        """Create relationship in graph database"""
        
        # This would use Neptune/Gremlin queries
        gremlin_query = f"""
        g.addV('follows')
         .property('follower_id', {follower_id})
         .property('following_id', {following_id})
         .property('created_at', '{datetime.now().isoformat()}')
        """
        
        # Also cache in Redis for fast lookups
        self.redis_client.sadd(f"following:{follower_id}", following_id)
        self.redis_client.sadd(f"followers:{following_id}", follower_id)
        
        return True
    
    def get_user_feed(self, user_id, limit=20):
        """Aggregate feed from multiple data sources"""
        
        # 1. Get following list from Redis cache
        following_users = self.redis_client.smembers(f"following:{user_id}")
        
        if not following_users:
            # Fallback to graph database
            following_users = self.get_following_from_graph(user_id)
        
        # 2. Get recent posts from DynamoDB
        recent_posts = []
        for following_user_id in following_users:
            response = self.posts_table.query(
                IndexName='UserIdIndex',
                KeyConditionExpression='user_id = :uid',
                ExpressionAttributeValues={':uid': following_user_id},
                ScanIndexForward=False,  # Latest first
                Limit=10
            )
            recent_posts.extend(response['Items'])
        
        # 3. Sort by timestamp and limit
        recent_posts.sort(key=lambda x: x['created_at'], reverse=True)
        feed_posts = recent_posts[:limit]
        
        # 4. Enrich with user data from MySQL
        enriched_feed = []
        for post in feed_posts:
            user_info = self.get_user_info(post['user_id'])
            post['user_info'] = user_info
            enriched_feed.append(post)
        
        return enriched_feed
    
    def search_content(self, query, filters=None):
        """Search posts using Elasticsearch"""
        
        search_body = {
            'query': {
                'bool': {
                    'must': [
                        {
                            'multi_match': {
                                'query': query,
                                'fields': ['content', 'hashtags', 'mentions']
                            }
                        }
                    ]
                }
            },
            'sort': [
                {'created_at': {'order': 'desc'}}
            ],
            'size': 50
        }
        
        # Add filters if provided
        if filters:
            search_body['query']['bool']['filter'] = []
            
            if 'date_range' in filters:
                search_body['query']['bool']['filter'].append({
                    'range': {
                        'created_at': filters['date_range']
                    }
                })
            
            if 'user_ids' in filters:
                search_body['query']['bool']['filter'].append({
                    'terms': {
                        'user_id': filters['user_ids']
                    }
                })
        
        # Execute search (would use elasticsearch-py client)
        # results = es_client.search(index='posts', body=search_body)
        
        return search_body  # Return query for demonstration
    
    def get_analytics_data(self):
        """Aggregate analytics from multiple sources"""
        
        analytics = {}
        
        # User metrics from MySQL
        cursor = self.mysql_conn.cursor(dictionary=True)
        cursor.execute("""
            SELECT 
                COUNT(*) as total_users,
                COUNT(CASE WHEN created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY) THEN 1 END) as new_users_30d
            FROM users
        """)
        analytics['users'] = cursor.fetchone()
        
        # Content metrics from DynamoDB
        # (Would use DynamoDB scan/query with aggregation)
        analytics['content'] = {
            'total_posts': 1500000,
            'posts_today': 5000
        }
        
        # Activity metrics from Redis
        active_sessions = len(self.redis_client.keys("session:*"))
        analytics['activity'] = {
            'active_sessions': active_sessions
        }
        
        return analytics

# Usage demonstration
platform = SocialMediaPlatform()

# Create user (MySQL)
user_id = platform.create_user_profile({
    'username': 'john_doe',
    'email': 'john@example.com',
    'first_name': 'John',
    'last_name': 'Doe',
    'birth_date': '1990-01-15'
})

# Create post (DynamoDB + Elasticsearch)
post_id = platform.create_post(
    user_id, 
    "Just launched my new app! #startup #tech @jane_doe",
    ['https://cdn.example.com/image1.jpg']
)

# Create session (Redis)
session_key = platform.create_user_session(user_id, {
    'device_info': {'type': 'mobile', 'os': 'iOS'},
    'preferences': {'theme': 'dark', 'language': 'en'}
})

# Follow user (Neptune + Redis)
platform.follow_user(user_id, 12345)

# Get personalized feed (Multi-database aggregation)
feed = platform.get_user_feed(user_id)

print(f"Created user {user_id}, post {post_id}, session {session_key}")
print(f"Feed contains {len(feed)} posts")
```
