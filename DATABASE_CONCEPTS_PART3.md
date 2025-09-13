# Database Concepts - Part 3 (Concepts 16-35)

## 16. Backup & Recovery

**Định nghĩa chi tiết**: Quá trình tạo bản sao dữ liệu để bảo vệ khỏi data loss và khôi phục hệ thống về trạng thái trước đó khi có sự cố. Backup & Recovery là foundation của business continuity và disaster recovery planning.

**Backup Types**:
- **Full Backup**: Complete copy của toàn bộ database
- **Incremental Backup**: Chỉ backup changes từ lần backup cuối
- **Differential Backup**: Changes từ lần full backup cuối
- **Transaction Log Backup**: Backup transaction logs để point-in-time recovery

**Recovery Models**:
- **Simple**: Minimal logging, không thể point-in-time recovery
- **Full**: Complete logging, full point-in-time recovery capability
- **Bulk-logged**: Optimized cho bulk operations

**Use Case - Banking System Backup Strategy**:
```sql
-- Full backup strategy cho critical banking data
-- Daily full backup at 2 AM
BACKUP DATABASE banking_prod 
TO DISK = '/backup/banking_full_20240120.bak'
WITH COMPRESSION, CHECKSUM, INIT;

-- Transaction log backup every 15 minutes
BACKUP LOG banking_prod 
TO DISK = '/backup/banking_log_20240120_1430.trn'
WITH COMPRESSION, CHECKSUM;

-- Verify backup integrity
RESTORE VERIFYONLY 
FROM DISK = '/backup/banking_full_20240120.bak';
```

**AWS RDS Automated Backup**:
```python
import boto3
from datetime import datetime, timedelta

class RDSBackupManager:
    def __init__(self):
        self.rds = boto3.client('rds')
    
    def configure_automated_backup(self, db_instance_id):
        """Configure automated backup settings"""
        response = self.rds.modify_db_instance(
            DBInstanceIdentifier=db_instance_id,
            BackupRetentionPeriod=35,  # Maximum retention
            PreferredBackupWindow='03:00-04:00',  # Low traffic window
            PreferredMaintenanceWindow='sun:04:00-sun:05:00',
            EnablePerformanceInsights=True,
            DeletionProtection=True
        )
        return response
    
    def create_manual_snapshot(self, db_instance_id, snapshot_id):
        """Create manual snapshot before major changes"""
        response = self.rds.create_db_snapshot(
            DBSnapshotIdentifier=snapshot_id,
            DBInstanceIdentifier=db_instance_id,
            Tags=[
                {'Key': 'Purpose', 'Value': 'PreMaintenance'},
                {'Key': 'CreatedBy', 'Value': 'AutomationScript'},
                {'Key': 'RetentionDays', 'Value': '90'}
            ]
        )
        return response
    
    def point_in_time_recovery(self, source_db_id, target_db_id, restore_time):
        """Restore database to specific point in time"""
        response = self.rds.restore_db_instance_to_point_in_time(
            SourceDBInstanceIdentifier=source_db_id,
            TargetDBInstanceIdentifier=target_db_id,
            RestoreTime=restore_time,
            UseLatestRestorableTime=False,
            DBSubnetGroupName='production-subnet-group',
            MultiAZ=True,
            PubliclyAccessible=False
        )
        return response
```

## 17. Transaction & ACID Properties

**Định nghĩa chi tiết**: Transaction là logical unit of work bao gồm một hoặc nhiều database operations được thực hiện như một đơn vị không thể chia cắt. ACID properties đảm bảo reliability và consistency của transactions.

**ACID Properties**:
- **Atomicity**: All-or-nothing execution
- **Consistency**: Database remains in valid state
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed changes persist permanently

**Transaction States**:
- **Active**: Transaction đang thực hiện
- **Partially Committed**: Hoàn thành nhưng chưa commit
- **Committed**: Successfully completed và persistent
- **Failed**: Cannot complete normally
- **Aborted**: Rolled back to initial state

**Use Case - E-commerce Order Processing với Complex Business Logic**:
```sql
-- Complex transaction: Process order với inventory, payment, shipping
DELIMITER //
CREATE PROCEDURE ProcessOrder(
    IN p_customer_id INT,
    IN p_items JSON,
    IN p_payment_method VARCHAR(50),
    IN p_shipping_address JSON,
    OUT p_order_id BIGINT,
    OUT p_status VARCHAR(50)
)
BEGIN
    DECLARE v_total_amount DECIMAL(12,2) DEFAULT 0;
    DECLARE v_item_count INT DEFAULT 0;
    DECLARE v_product_id INT;
    DECLARE v_quantity INT;
    DECLARE v_price DECIMAL(10,2);
    DECLARE v_available_stock INT;
    DECLARE v_payment_id BIGINT;
    DECLARE v_shipping_id BIGINT;
    
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_status = 'FAILED';
        RESIGNAL;
    END;
    
    -- Start transaction
    START TRANSACTION;
    
    -- 1. Validate customer
    IF NOT EXISTS (SELECT 1 FROM customers WHERE customer_id = p_customer_id AND status = 'active') THEN
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Invalid customer';
    END IF;
    
    -- 2. Create order record
    INSERT INTO orders (customer_id, status, created_at, total_amount)
    VALUES (p_customer_id, 'processing', NOW(), 0);
    
    SET p_order_id = LAST_INSERT_ID();
    
    -- 3. Process each item
    SET v_item_count = JSON_LENGTH(p_items);
    
    item_loop: WHILE v_item_count > 0 DO
        SET v_item_count = v_item_count - 1;
        
        -- Extract item details
        SET v_product_id = JSON_UNQUOTE(JSON_EXTRACT(p_items, CONCAT('$[', v_item_count, '].product_id')));
        SET v_quantity = JSON_UNQUOTE(JSON_EXTRACT(p_items, CONCAT('$[', v_item_count, '].quantity')));
        
        -- Get current price và stock
        SELECT price, stock_quantity 
        INTO v_price, v_available_stock
        FROM products 
        WHERE product_id = v_product_id AND status = 'active'
        FOR UPDATE;  -- Lock row để prevent race conditions
        
        -- Check stock availability
        IF v_available_stock < v_quantity THEN
            SIGNAL SQLSTATE '45000' 
            SET MESSAGE_TEXT = CONCAT('Insufficient stock for product ', v_product_id);
        END IF;
        
        -- Reserve inventory
        UPDATE products 
        SET stock_quantity = stock_quantity - v_quantity,
            reserved_quantity = reserved_quantity + v_quantity,
            last_updated = NOW()
        WHERE product_id = v_product_id;
        
        -- Add order item
        INSERT INTO order_items (order_id, product_id, quantity, unit_price, total_price)
        VALUES (p_order_id, v_product_id, v_quantity, v_price, v_quantity * v_price);
        
        SET v_total_amount = v_total_amount + (v_quantity * v_price);
        
    END WHILE item_loop;
    
    -- 4. Update order total
    UPDATE orders 
    SET total_amount = v_total_amount,
        updated_at = NOW()
    WHERE order_id = p_order_id;
    
    -- 5. Process payment
    INSERT INTO payments (order_id, amount, payment_method, status, created_at)
    VALUES (p_order_id, v_total_amount, p_payment_method, 'pending', NOW());
    
    SET v_payment_id = LAST_INSERT_ID();
    
    -- Simulate payment processing
    IF RAND() > 0.95 THEN  -- 5% payment failure rate
        UPDATE payments SET status = 'failed' WHERE payment_id = v_payment_id;
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Payment processing failed';
    ELSE
        UPDATE payments SET status = 'completed', processed_at = NOW() 
        WHERE payment_id = v_payment_id;
    END IF;
    
    -- 6. Create shipping record
    INSERT INTO shipping (order_id, address, status, created_at)
    VALUES (p_order_id, p_shipping_address, 'pending', NOW());
    
    -- 7. Finalize order
    UPDATE orders 
    SET status = 'confirmed',
        confirmed_at = NOW()
    WHERE order_id = p_order_id;
    
    -- 8. Update customer statistics
    UPDATE customers 
    SET total_orders = total_orders + 1,
        total_spent = total_spent + v_total_amount,
        last_order_date = NOW()
    WHERE customer_id = p_customer_id;
    
    -- Commit transaction
    COMMIT;
    SET p_status = 'SUCCESS';
    
END //
DELIMITER ;

-- Usage
CALL ProcessOrder(
    12345,  -- customer_id
    '[{"product_id": 1001, "quantity": 2}, {"product_id": 1002, "quantity": 1}]',  -- items
    'credit_card',  -- payment_method
    '{"street": "123 Main St", "city": "HCMC", "country": "Vietnam"}',  -- address
    @order_id,
    @status
);

SELECT @order_id, @status;
```

## 18. Concurrency Control

**Định nghĩa chi tiết**: Kỹ thuật quản lý multiple transactions thực hiện đồng thời để đảm bảo database consistency và isolation. Concurrency control prevents conflicts như lost updates, dirty reads và phantom reads.

**Locking Mechanisms**:
- **Shared Lock (S)**: Multiple transactions có thể read
- **Exclusive Lock (X)**: Chỉ một transaction có thể write
- **Intent Locks**: Indicate intention to acquire locks at lower level
- **Row-level vs Table-level**: Granularity của locking

**Concurrency Problems**:
- **Lost Update**: Two transactions update same data, one update lost
- **Dirty Read**: Read uncommitted data từ another transaction
- **Non-repeatable Read**: Same query returns different results
- **Phantom Read**: New rows appear between reads

**Use Case - Banking Concurrent Transfers**:
```sql
-- Scenario: Hai transactions đồng thời transfer money
-- Transaction 1: Transfer $500 from Account A to Account B
-- Transaction 2: Transfer $300 from Account B to Account A

-- Transaction 1 (Session 1)
BEGIN TRANSACTION;

-- Lock accounts in consistent order để avoid deadlock
SELECT balance INTO @balance_a 
FROM accounts 
WHERE account_id = 'ACC_001' 
FOR UPDATE;

SELECT balance INTO @balance_b
FROM accounts 
WHERE account_id = 'ACC_002'
FOR UPDATE;

-- Validate sufficient funds
IF @balance_a >= 500 THEN
    UPDATE accounts SET balance = balance - 500 WHERE account_id = 'ACC_001';
    UPDATE accounts SET balance = balance + 500 WHERE account_id = 'ACC_002';
    
    INSERT INTO transactions (from_account, to_account, amount, type, status)
    VALUES ('ACC_001', 'ACC_002', 500, 'transfer', 'completed');
    
    COMMIT;
ELSE
    ROLLBACK;
END IF;

-- Transaction 2 (Session 2) - runs concurrently
BEGIN TRANSACTION;

-- Same locking order để prevent deadlock
SELECT balance INTO @balance_a
FROM accounts 
WHERE account_id = 'ACC_001'
FOR UPDATE;  -- Will wait for Transaction 1 to complete

SELECT balance INTO @balance_b
FROM accounts 
WHERE account_id = 'ACC_002'
FOR UPDATE;

IF @balance_b >= 300 THEN
    UPDATE accounts SET balance = balance - 300 WHERE account_id = 'ACC_002';
    UPDATE accounts SET balance = balance + 300 WHERE account_id = 'ACC_001';
    
    INSERT INTO transactions (from_account, to_account, amount, type, status)
    VALUES ('ACC_002', 'ACC_001', 300, 'transfer', 'completed');
    
    COMMIT;
ELSE
    ROLLBACK;
END IF;
```

**Optimistic vs Pessimistic Concurrency**:
```python
# Pessimistic Locking: Lock resources upfront
class PessimisticInventoryManager:
    def __init__(self, db_connection):
        self.db = db_connection
    
    def reserve_product(self, product_id, quantity):
        cursor = self.db.cursor()
        
        try:
            cursor.execute("BEGIN TRANSACTION")
            
            # Lock row immediately
            cursor.execute("""
                SELECT stock_quantity, reserved_quantity
                FROM products 
                WHERE product_id = %s
                FOR UPDATE
            """, (product_id,))
            
            result = cursor.fetchone()
            if not result:
                raise Exception("Product not found")
            
            available = result['stock_quantity'] - result['reserved_quantity']
            
            if available >= quantity:
                cursor.execute("""
                    UPDATE products 
                    SET reserved_quantity = reserved_quantity + %s
                    WHERE product_id = %s
                """, (quantity, product_id))
                
                cursor.execute("COMMIT")
                return True
            else:
                cursor.execute("ROLLBACK")
                return False
                
        except Exception as e:
            cursor.execute("ROLLBACK")
            raise e

# Optimistic Locking: Check for conflicts at commit time
class OptimisticInventoryManager:
    def __init__(self, db_connection):
        self.db = db_connection
    
    def reserve_product(self, product_id, quantity, max_retries=3):
        for attempt in range(max_retries):
            cursor = self.db.cursor()
            
            try:
                # Read current state với version
                cursor.execute("""
                    SELECT stock_quantity, reserved_quantity, version
                    FROM products 
                    WHERE product_id = %s
                """, (product_id,))
                
                result = cursor.fetchone()
                if not result:
                    raise Exception("Product not found")
                
                current_version = result['version']
                available = result['stock_quantity'] - result['reserved_quantity']
                
                if available >= quantity:
                    # Try to update với version check
                    cursor.execute("""
                        UPDATE products 
                        SET reserved_quantity = reserved_quantity + %s,
                            version = version + 1
                        WHERE product_id = %s AND version = %s
                    """, (quantity, product_id, current_version))
                    
                    if cursor.rowcount == 1:
                        self.db.commit()
                        return True
                    else:
                        # Version conflict, retry
                        continue
                else:
                    return False
                    
            except Exception as e:
                self.db.rollback()
                if attempt == max_retries - 1:
                    raise e
                continue
        
        raise Exception("Max retries exceeded due to conflicts")
```
## 19. Schema Design & Normalization

**Định nghĩa chi tiết**: Schema là blueprint định nghĩa cấu trúc database bao gồm tables, columns, data types, constraints và relationships. Normalization là quá trình tổ chức dữ liệu để giảm redundancy và improve data integrity.

**Normal Forms**:
- **1NF**: Atomic values, no repeating groups
- **2NF**: 1NF + no partial dependencies on composite keys
- **3NF**: 2NF + no transitive dependencies
- **BCNF**: 3NF + every determinant is a candidate key
- **4NF**: BCNF + no multi-valued dependencies

**Use Case - E-commerce Schema Evolution**:
```sql
-- Unnormalized table (0NF) - Bad design
CREATE TABLE orders_bad (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100),
    customer_phone VARCHAR(20),
    customer_address TEXT,
    product_names TEXT,  -- "iPhone,iPad,MacBook"
    product_prices TEXT, -- "1000,800,2000"
    quantities TEXT,     -- "1,2,1"
    order_total DECIMAL(10,2),
    order_date DATE
);

-- 1NF: Atomic values
CREATE TABLE orders_1nf (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100),
    customer_phone VARCHAR(20),
    customer_address TEXT,
    product_name VARCHAR(200),
    product_price DECIMAL(10,2),
    quantity INT,
    line_total DECIMAL(10,2),
    order_total DECIMAL(10,2),
    order_date DATE
);

-- 2NF: Remove partial dependencies
CREATE TABLE customers_2nf (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100),
    customer_phone VARCHAR(20),
    customer_address TEXT
);

CREATE TABLE products_2nf (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(200),
    product_price DECIMAL(10,2),
    category VARCHAR(100)
);

CREATE TABLE orders_2nf (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_total DECIMAL(10,2),
    order_date DATE,
    FOREIGN KEY (customer_id) REFERENCES customers_2nf(customer_id)
);

CREATE TABLE order_items_2nf (
    order_id INT,
    product_id INT,
    quantity INT,
    line_total DECIMAL(10,2),
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders_2nf(order_id),
    FOREIGN KEY (product_id) REFERENCES products_2nf(product_id)
);

-- 3NF: Remove transitive dependencies
CREATE TABLE categories_3nf (
    category_id INT PRIMARY KEY,
    category_name VARCHAR(100),
    category_description TEXT
);

CREATE TABLE products_3nf (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(200),
    product_price DECIMAL(10,2),
    category_id INT,
    FOREIGN KEY (category_id) REFERENCES categories_3nf(category_id)
);

-- Denormalization for performance (Data Warehouse)
CREATE TABLE sales_fact_denormalized (
    sale_id BIGINT PRIMARY KEY,
    order_date DATE,
    customer_id INT,
    customer_name VARCHAR(100),        -- Denormalized
    customer_segment VARCHAR(50),      -- Denormalized
    product_id INT,
    product_name VARCHAR(200),         -- Denormalized
    category_name VARCHAR(100),        -- Denormalized
    brand_name VARCHAR(100),           -- Denormalized
    quantity INT,
    unit_price DECIMAL(10,2),
    total_amount DECIMAL(12,2),
    profit_margin DECIMAL(5,2)
);
```

## 20. Isolation Levels

**Định nghĩa chi tiết**: Isolation levels xác định mức độ tách biệt giữa concurrent transactions, balance giữa consistency và performance. Mỗi level cho phép hoặc ngăn chặn các concurrency phenomena khác nhau.

**Isolation Levels (từ thấp đến cao)**:
- **READ UNCOMMITTED**: Cho phép dirty reads
- **READ COMMITTED**: Ngăn dirty reads, cho phép non-repeatable reads
- **REPEATABLE READ**: Ngăn dirty và non-repeatable reads, cho phép phantom reads
- **SERIALIZABLE**: Ngăn tất cả concurrency phenomena

**Use Case - Banking System với Different Isolation Requirements**:
```sql
-- READ UNCOMMITTED: Fast reporting (có thể chấp nhận dirty data)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN TRANSACTION;

-- Quick dashboard query - có thể thấy uncommitted data
SELECT 
    account_type,
    COUNT(*) as account_count,
    SUM(balance) as total_balance,
    AVG(balance) as avg_balance
FROM accounts
WHERE status = 'active'
GROUP BY account_type;

COMMIT;

-- READ COMMITTED: Normal business operations
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN TRANSACTION;

-- ATM withdrawal - cần thấy committed data
SELECT balance, daily_limit_remaining
FROM accounts 
WHERE account_number = '1234567890';

-- Update balance nếu sufficient funds
UPDATE accounts 
SET balance = balance - 500000,
    daily_limit_remaining = daily_limit_remaining - 500000
WHERE account_number = '1234567890'
  AND balance >= 500000
  AND daily_limit_remaining >= 500000;

COMMIT;

-- REPEATABLE READ: Financial calculations
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRANSACTION;

-- Calculate interest - cần consistent reads
SELECT @total_balance := SUM(balance)
FROM savings_accounts
WHERE account_type = 'premium';

-- Multiple reads phải return same result
SELECT account_id, balance, 
       (balance / @total_balance * 100) as percentage_of_total
FROM savings_accounts
WHERE account_type = 'premium'
ORDER BY balance DESC;

-- Apply interest based on calculations
UPDATE savings_accounts
SET balance = balance * 1.05,  -- 5% interest
    last_interest_date = CURDATE()
WHERE account_type = 'premium';

COMMIT;

-- SERIALIZABLE: Critical operations (end-of-day processing)
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRANSACTION;

-- End-of-day balance reconciliation
SELECT @system_total := SUM(balance) FROM accounts;
SELECT @ledger_total := SUM(amount) FROM general_ledger WHERE date = CURDATE();

IF @system_total != @ledger_total THEN
    -- Log discrepancy
    INSERT INTO audit_log (event_type, description, amount_difference, created_at)
    VALUES ('BALANCE_MISMATCH', 
            CONCAT('System: ', @system_total, ', Ledger: ', @ledger_total),
            @system_total - @ledger_total,
            NOW());
    
    ROLLBACK;
ELSE
    -- Mark day as reconciled
    INSERT INTO daily_reconciliation (date, total_balance, status, reconciled_at)
    VALUES (CURDATE(), @system_total, 'RECONCILED', NOW());
    
    COMMIT;
END IF;
```

**Performance vs Consistency Trade-offs**:
```python
import mysql.connector
import time
from concurrent.futures import ThreadPoolExecutor

class IsolationLevelDemo:
    def __init__(self, connection_config):
        self.config = connection_config
    
    def test_isolation_level(self, isolation_level, query_type):
        """Test performance của different isolation levels"""
        conn = mysql.connector.connect(**self.config)
        cursor = conn.cursor()
        
        start_time = time.time()
        
        try:
            cursor.execute(f"SET TRANSACTION ISOLATION LEVEL {isolation_level}")
            cursor.execute("BEGIN")
            
            if query_type == 'read_heavy':
                # Simulate read-heavy workload
                for i in range(100):
                    cursor.execute("""
                        SELECT account_id, balance, account_type
                        FROM accounts 
                        WHERE balance > %s
                        LIMIT 10
                    """, (i * 1000,))
                    cursor.fetchall()
            
            elif query_type == 'write_heavy':
                # Simulate write-heavy workload
                for i in range(50):
                    cursor.execute("""
                        UPDATE accounts 
                        SET last_accessed = NOW()
                        WHERE account_id = %s
                    """, (i + 1,))
            
            cursor.execute("COMMIT")
            
        except Exception as e:
            cursor.execute("ROLLBACK")
            raise e
        finally:
            cursor.close()
            conn.close()
        
        end_time = time.time()
        return end_time - start_time
    
    def concurrent_test(self, isolation_level, num_threads=10):
        """Test concurrent performance"""
        with ThreadPoolExecutor(max_workers=num_threads) as executor:
            futures = []
            
            for i in range(num_threads):
                query_type = 'read_heavy' if i % 2 == 0 else 'write_heavy'
                future = executor.submit(
                    self.test_isolation_level, 
                    isolation_level, 
                    query_type
                )
                futures.append(future)
            
            results = [future.result() for future in futures]
            
        return {
            'isolation_level': isolation_level,
            'avg_time': sum(results) / len(results),
            'max_time': max(results),
            'min_time': min(results)
        }

# Performance comparison
demo = IsolationLevelDemo({
    'host': 'localhost',
    'database': 'banking',
    'user': 'test_user',
    'password': 'test_password'
})

isolation_levels = [
    'READ UNCOMMITTED',
    'READ COMMITTED', 
    'REPEATABLE READ',
    'SERIALIZABLE'
]

print("Isolation Level Performance Comparison:")
print("-" * 50)

for level in isolation_levels:
    result = demo.concurrent_test(level)
    print(f"{level:20} | Avg: {result['avg_time']:.3f}s | Max: {result['max_time']:.3f}s")
```

## 21. Stored Procedures & Triggers

**Định nghĩa chi tiết**: Stored procedures là pre-compiled SQL code được lưu trong database server, có thể nhận parameters và return values. Triggers là special stored procedures tự động execute khi có specific database events.

**Stored Procedure Benefits**:
- **Performance**: Pre-compiled, execution plan cached
- **Security**: Encapsulation, parameter validation
- **Maintainability**: Centralized business logic
- **Network Traffic**: Reduced client-server communication

**Trigger Types**:
- **BEFORE**: Execute trước khi event occurs
- **AFTER**: Execute sau khi event completes
- **INSTEAD OF**: Replace the triggering event (views only)

**Use Case - E-commerce Audit System với Complex Business Logic**:
```sql
-- Stored procedure: Calculate customer loyalty tier
DELIMITER //
CREATE PROCEDURE CalculateCustomerTier(
    IN p_customer_id INT,
    OUT p_current_tier VARCHAR(20),
    OUT p_next_tier VARCHAR(20),
    OUT p_points_needed INT
)
BEGIN
    DECLARE v_total_spent DECIMAL(12,2);
    DECLARE v_order_count INT;
    DECLARE v_avg_order_value DECIMAL(10,2);
    DECLARE v_days_since_first_order INT;
    DECLARE v_loyalty_points INT DEFAULT 0;
    
    -- Get customer statistics
    SELECT 
        COALESCE(SUM(total_amount), 0),
        COUNT(*),
        COALESCE(AVG(total_amount), 0),
        COALESCE(DATEDIFF(NOW(), MIN(created_at)), 0)
    INTO v_total_spent, v_order_count, v_avg_order_value, v_days_since_first_order
    FROM orders
    WHERE customer_id = p_customer_id 
      AND status IN ('completed', 'delivered');
    
    -- Calculate loyalty points
    SET v_loyalty_points = 
        (v_total_spent / 100000) +           -- 1 point per 100k spent
        (v_order_count * 10) +               -- 10 points per order
        (v_avg_order_value / 50000) +        -- Bonus for high-value orders
        GREATEST(0, (v_days_since_first_order - 365) / 30);  -- Tenure bonus
    
    -- Determine tier
    CASE
        WHEN v_loyalty_points >= 1000 THEN
            SET p_current_tier = 'DIAMOND';
            SET p_next_tier = 'DIAMOND';
            SET p_points_needed = 0;
        WHEN v_loyalty_points >= 500 THEN
            SET p_current_tier = 'PLATINUM';
            SET p_next_tier = 'DIAMOND';
            SET p_points_needed = 1000 - v_loyalty_points;
        WHEN v_loyalty_points >= 200 THEN
            SET p_current_tier = 'GOLD';
            SET p_next_tier = 'PLATINUM';
            SET p_points_needed = 500 - v_loyalty_points;
        WHEN v_loyalty_points >= 50 THEN
            SET p_current_tier = 'SILVER';
            SET p_next_tier = 'GOLD';
            SET p_points_needed = 200 - v_loyalty_points;
        ELSE
            SET p_current_tier = 'BRONZE';
            SET p_next_tier = 'SILVER';
            SET p_points_needed = 50 - v_loyalty_points;
    END CASE;
    
    -- Update customer record
    UPDATE customers 
    SET loyalty_tier = p_current_tier,
        loyalty_points = v_loyalty_points,
        tier_updated_at = NOW()
    WHERE customer_id = p_customer_id;
    
END //
DELIMITER ;

-- Audit trigger: Track all changes to sensitive tables
CREATE TABLE audit_log (
    audit_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    table_name VARCHAR(100) NOT NULL,
    operation_type ENUM('INSERT', 'UPDATE', 'DELETE') NOT NULL,
    record_id VARCHAR(100),
    old_values JSON,
    new_values JSON,
    changed_by VARCHAR(100),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    session_id VARCHAR(100),
    ip_address VARCHAR(45),
    user_agent TEXT
);

-- Trigger for accounts table
DELIMITER //
CREATE TRIGGER accounts_audit_trigger
AFTER UPDATE ON accounts
FOR EACH ROW
BEGIN
    DECLARE v_changes JSON DEFAULT JSON_OBJECT();
    DECLARE v_old_values JSON DEFAULT JSON_OBJECT();
    DECLARE v_new_values JSON DEFAULT JSON_OBJECT();
    
    -- Track specific field changes
    IF OLD.balance != NEW.balance THEN
        SET v_old_values = JSON_SET(v_old_values, '$.balance', OLD.balance);
        SET v_new_values = JSON_SET(v_new_values, '$.balance', NEW.balance);
    END IF;
    
    IF OLD.status != NEW.status THEN
        SET v_old_values = JSON_SET(v_old_values, '$.status', OLD.status);
        SET v_new_values = JSON_SET(v_new_values, '$.status', NEW.status);
    END IF;
    
    IF OLD.credit_limit != NEW.credit_limit THEN
        SET v_old_values = JSON_SET(v_old_values, '$.credit_limit', OLD.credit_limit);
        SET v_new_values = JSON_SET(v_new_values, '$.credit_limit', NEW.credit_limit);
    END IF;
    
    -- Insert audit record only if there are actual changes
    IF JSON_LENGTH(v_old_values) > 0 THEN
        INSERT INTO audit_log (
            table_name, operation_type, record_id, 
            old_values, new_values, changed_by, session_id
        ) VALUES (
            'accounts', 'UPDATE', NEW.account_id,
            v_old_values, v_new_values, 
            COALESCE(@audit_user, USER()), 
            CONNECTION_ID()
        );
    END IF;
END //
DELIMITER ;

-- Business logic trigger: Auto-apply discounts
DELIMITER //
CREATE TRIGGER order_discount_trigger
BEFORE INSERT ON orders
FOR EACH ROW
BEGIN
    DECLARE v_customer_tier VARCHAR(20);
    DECLARE v_order_count INT;
    DECLARE v_discount_rate DECIMAL(5,4) DEFAULT 0;
    
    -- Get customer information
    SELECT loyalty_tier INTO v_customer_tier
    FROM customers 
    WHERE customer_id = NEW.customer_id;
    
    -- Count previous orders this month
    SELECT COUNT(*) INTO v_order_count
    FROM orders
    WHERE customer_id = NEW.customer_id
      AND YEAR(created_at) = YEAR(NOW())
      AND MONTH(created_at) = MONTH(NOW());
    
    -- Apply tier-based discount
    CASE v_customer_tier
        WHEN 'DIAMOND' THEN SET v_discount_rate = 0.15;  -- 15%
        WHEN 'PLATINUM' THEN SET v_discount_rate = 0.10; -- 10%
        WHEN 'GOLD' THEN SET v_discount_rate = 0.05;     -- 5%
        WHEN 'SILVER' THEN SET v_discount_rate = 0.02;   -- 2%
        ELSE SET v_discount_rate = 0;
    END CASE;
    
    -- Additional discount for frequent buyers
    IF v_order_count >= 5 THEN
        SET v_discount_rate = v_discount_rate + 0.05;  -- Extra 5%
    END IF;
    
    -- Apply maximum discount limit
    SET v_discount_rate = LEAST(v_discount_rate, 0.25);  -- Max 25%
    
    -- Calculate discounted amount
    SET NEW.discount_rate = v_discount_rate;
    SET NEW.discount_amount = NEW.subtotal * v_discount_rate;
    SET NEW.total_amount = NEW.subtotal - NEW.discount_amount + NEW.tax_amount;
    
END //
DELIMITER ;

-- Usage examples
-- Calculate customer tier
CALL CalculateCustomerTier(12345, @tier, @next_tier, @points_needed);
SELECT @tier as current_tier, @next_tier as next_tier, @points_needed as points_needed;

-- Set audit context before sensitive operations
SET @audit_user = 'admin_user_123';
UPDATE accounts SET balance = balance - 1000000 WHERE account_id = 'ACC_001';
```

## 22. Views & Materialized Views

**Định nghĩa chi tiết**: Views là virtual tables được định nghĩa bởi SQL query, không lưu trữ data mà dynamically generate results. Materialized views lưu trữ query results physically và refresh periodically để improve performance.

**View Benefits**:
- **Security**: Hide sensitive columns, row-level filtering
- **Simplification**: Complex joins presented as simple tables
- **Abstraction**: Isolate applications from schema changes
- **Reusability**: Common queries defined once

**Materialized View Benefits**:
- **Performance**: Pre-computed results for complex aggregations
- **Consistency**: Snapshot of data at specific point in time
- **Reduced Load**: Avoid expensive computations on base tables

**Use Case - E-commerce Reporting System**:
```sql
-- Security view: Customer information without sensitive data
CREATE VIEW customer_public_view AS
SELECT 
    customer_id,
    CONCAT(LEFT(first_name, 1), '***') as first_name_masked,
    CONCAT(LEFT(last_name, 1), '***') as last_name_masked,
    CONCAT(LEFT(email, 3), '***@', SUBSTRING_INDEX(email, '@', -1)) as email_masked,
    registration_date,
    loyalty_tier,
    total_orders,
    CASE 
        WHEN total_spent > 10000000 THEN 'High Value'
        WHEN total_spent > 5000000 THEN 'Medium Value'
        ELSE 'Regular'
    END as customer_segment
FROM customers
WHERE status = 'active';

-- Complex analytical view: Order analytics
CREATE VIEW order_analytics_view AS
SELECT 
    o.order_id,
    o.customer_id,
    c.loyalty_tier,
    o.order_date,
    YEAR(o.order_date) as order_year,
    MONTH(o.order_date) as order_month,
    QUARTER(o.order_date) as order_quarter,
    DAYNAME(o.order_date) as order_day_name,
    o.subtotal,
    o.discount_amount,
    o.tax_amount,
    o.total_amount,
    o.total_amount - o.subtotal as additional_charges,
    (o.discount_amount / o.subtotal * 100) as discount_percentage,
    
    -- Order composition
    (SELECT COUNT(*) FROM order_items oi WHERE oi.order_id = o.order_id) as item_count,
    (SELECT SUM(oi.quantity) FROM order_items oi WHERE oi.order_id = o.order_id) as total_quantity,
    
    -- Customer context
    (SELECT COUNT(*) FROM orders o2 WHERE o2.customer_id = o.customer_id AND o2.order_date <= o.order_date) as customer_order_sequence,
    
    -- Time-based metrics
    DATEDIFF(o.order_date, c.registration_date) as days_since_registration,
    
    -- Category breakdown
    (SELECT GROUP_CONCAT(DISTINCT cat.category_name) 
     FROM order_items oi 
     JOIN products p ON oi.product_id = p.product_id
     JOIN categories cat ON p.category_id = cat.category_id
     WHERE oi.order_id = o.order_id) as categories_purchased

FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE o.status IN ('completed', 'delivered');

-- Materialized view simulation (MySQL doesn't have native materialized views)
-- Create table to store materialized results
CREATE TABLE mv_daily_sales_summary (
    summary_date DATE PRIMARY KEY,
    total_orders INT,
    total_revenue DECIMAL(15,2),
    total_customers INT,
    avg_order_value DECIMAL(10,2),
    top_category VARCHAR(100),
    top_product VARCHAR(200),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Stored procedure to refresh materialized view
DELIMITER //
CREATE PROCEDURE RefreshDailySalesSummary(IN p_date DATE)
BEGIN
    DECLARE v_total_orders INT;
    DECLARE v_total_revenue DECIMAL(15,2);
    DECLARE v_total_customers INT;
    DECLARE v_avg_order_value DECIMAL(10,2);
    DECLARE v_top_category VARCHAR(100);
    DECLARE v_top_product VARCHAR(200);
    
    -- Calculate metrics for the date
    SELECT 
        COUNT(*),
        SUM(total_amount),
        COUNT(DISTINCT customer_id),
        AVG(total_amount)
    INTO v_total_orders, v_total_revenue, v_total_customers, v_avg_order_value
    FROM orders
    WHERE DATE(order_date) = p_date
      AND status IN ('completed', 'delivered');
    
    -- Find top category
    SELECT cat.category_name INTO v_top_category
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    JOIN categories cat ON p.category_id = cat.category_id
    WHERE DATE(o.order_date) = p_date
      AND o.status IN ('completed', 'delivered')
    GROUP BY cat.category_id, cat.category_name
    ORDER BY SUM(oi.quantity * oi.unit_price) DESC
    LIMIT 1;
    
    -- Find top product
    SELECT p.product_name INTO v_top_product
    FROM orders o
    JOIN order_items oi ON o.order_id = oi.order_id
    JOIN products p ON oi.product_id = p.product_id
    WHERE DATE(o.order_date) = p_date
      AND o.status IN ('completed', 'delivered')
    GROUP BY p.product_id, p.product_name
    ORDER BY SUM(oi.quantity) DESC
    LIMIT 1;
    
    -- Insert or update summary
    INSERT INTO mv_daily_sales_summary (
        summary_date, total_orders, total_revenue, total_customers,
        avg_order_value, top_category, top_product
    ) VALUES (
        p_date, v_total_orders, v_total_revenue, v_total_customers,
        v_avg_order_value, v_top_category, v_top_product
    ) ON DUPLICATE KEY UPDATE
        total_orders = v_total_orders,
        total_revenue = v_total_revenue,
        total_customers = v_total_customers,
        avg_order_value = v_avg_order_value,
        top_category = v_top_category,
        top_product = v_top_product,
        updated_at = NOW();
        
END //
DELIMITER ;

-- Automated refresh using events
CREATE EVENT refresh_daily_summary
ON SCHEDULE EVERY 1 HOUR
STARTS CURRENT_TIMESTAMP
DO
BEGIN
    -- Refresh yesterday's data (complete day)
    CALL RefreshDailySalesSummary(DATE_SUB(CURDATE(), INTERVAL 1 DAY));
    
    -- Refresh today's data (partial day)
    CALL RefreshDailySalesSummary(CURDATE());
END;

-- Query materialized view for fast reporting
SELECT 
    summary_date,
    total_orders,
    FORMAT(total_revenue, 0) as revenue_formatted,
    total_customers,
    FORMAT(avg_order_value, 0) as avg_order_formatted,
    top_category,
    top_product
FROM mv_daily_sales_summary
WHERE summary_date >= DATE_SUB(CURDATE(), INTERVAL 30 DAY)
ORDER BY summary_date DESC;
```
