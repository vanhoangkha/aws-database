# Database Concepts - Complete Guide (65 Concepts)

## 1. Database (Cơ sở dữ liệu)

**Định nghĩa chi tiết**: Tập hợp có tổ chức của dữ liệu được lưu trữ điện tử, có thể truy cập, quản lý và cập nhật. Database không chỉ là file lưu trữ mà là hệ thống phức tạp với metadata, indexes, constraints và relationships.

**Đặc điểm quan trọng**:
- **Persistence**: Dữ liệu tồn tại lâu dài, không mất khi tắt máy
- **Concurrent Access**: Nhiều user/application truy cập đồng thời
- **Data Integrity**: Đảm bảo tính chính xác và nhất quán
- **Security**: Kiểm soát quyền truy cập và bảo mật

**Ví dụ thực tế chi tiết**:
- **E-commerce (Shopee/Lazada)**: 
  - Sản phẩm: 50 triệu sản phẩm với thông tin tên, giá, mô tả, hình ảnh
  - Khách hàng: 100 triệu tài khoản với profile, lịch sử mua hàng
  - Đơn hàng: 1 tỷ đơn hàng/năm với trạng thái, thanh toán, vận chuyển
  - Inventory: Real-time stock tracking cho 10,000 warehouses

- **Ngân hàng**:
  - Tài khoản: 15 triệu tài khoản với số dư, loại tài khoản
  - Giao dịch: 500 triệu giao dịch/tháng (chuyển khoản, rút tiền, nạp tiền)
  - Khách hàng: thông tin cá nhân, CMND, địa chỉ, mức tín dụng
  - Compliance: Lưu trữ 7 năm theo quy định pháp luật

**AWS Implementation**:
```
RDS MySQL: Lưu thông tin sản phẩm, đơn hàng (structured data)
DynamoDB: Lưu session user, shopping cart (key-value)
S3: Lưu hình ảnh sản phẩm, documents
DocumentDB: Lưu product catalog với nested attributes
```

## 2. DBMS (Database Management System)

**Định nghĩa chi tiết**: Phần mềm hệ thống phức tạp cung cấp interface giữa users/applications và database. DBMS không chỉ lưu trữ mà còn quản lý metadata, enforce business rules, optimize performance và đảm bảo data integrity.

**Thành phần chính của DBMS**:
- **Query Processor**: Parse, optimize và execute SQL queries
- **Storage Manager**: Quản lý physical storage, buffer pools
- **Transaction Manager**: Đảm bảo ACID properties
- **Security Manager**: Authentication, authorization, auditing
- **Recovery Manager**: Backup, restore, point-in-time recovery

**Chức năng chi tiết**:
- **CRUD Operations**: Create, Read, Update, Delete với transaction support
- **Concurrency Control**: Multi-user access với locking mechanisms
- **Security**: Role-based access, encryption at rest/in transit
- **Performance**: Query optimization, indexing, caching strategies
- **Backup/Recovery**: Automated backups, point-in-time recovery, disaster recovery

**Use Case - Hệ thống Hospital Management**:
```sql
-- Tạo bệnh nhân mới với validation
BEGIN TRANSACTION;
INSERT INTO patients (patient_id, name, dob, phone, address, insurance_id) 
VALUES (UUID(), 'Nguyen Van A', '1990-01-15', '0901234567', 'Ha Noi', 'INS123456');

-- Kiểm tra duplicate phone number
IF EXISTS (SELECT 1 FROM patients WHERE phone = '0901234567' AND patient_id != LAST_INSERT_ID()) THEN
    ROLLBACK;
    SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Phone number already exists';
END IF;

-- Đặt lịch khám với business logic
INSERT INTO appointments (appointment_id, patient_id, doctor_id, appointment_date, status, department)
VALUES (UUID(), LAST_INSERT_ID(), 67890, '2024-01-20 09:00:00', 'scheduled', 'cardiology');

-- Cập nhật kết quả khám với audit trail
UPDATE medical_records 
SET diagnosis = 'Hypertension', 
    prescription = 'Amlodipine 5mg',
    updated_by = USER(),
    updated_at = NOW()
WHERE patient_id = LAST_INSERT_ID() AND visit_date = '2024-01-20';

COMMIT;
```

**AWS DBMS Options với đặc điểm**:
- **Aurora MySQL**: 99.99% uptime, auto-scaling, 5x faster than MySQL
- **RDS PostgreSQL**: ACID compliance, complex queries, JSON support
- **DynamoDB**: NoSQL, single-digit millisecond latency, infinite scale
- **DocumentDB**: MongoDB-compatible, managed document database
- **Neptune**: Graph database, SPARQL/Gremlin support

## 3. Primary Key

**Định nghĩa chi tiết**: Cột hoặc tổ hợp cột (composite key) xác định duy nhất mỗi record trong bảng. Primary key không được NULL, không được duplicate và tự động tạo clustered index. Đây là foundation của relational model và referential integrity.

**Đặc điểm quan trọng**:
- **Uniqueness**: Mỗi giá trị chỉ xuất hiện một lần
- **Non-null**: Không được chứa giá trị NULL
- **Immutable**: Không nên thay đổi sau khi tạo
- **Minimal**: Chỉ bao gồm columns cần thiết để đảm bảo uniqueness

**Types of Primary Keys**:
- **Natural Key**: Sử dụng business data (SSN, email)
- **Surrogate Key**: Artificial key (auto-increment, UUID)
- **Composite Key**: Kết hợp nhiều columns

**Use Case - Social Media Platform (Facebook/Instagram)**:
```sql
-- Bảng Users với surrogate key
CREATE TABLE users (
    user_id BIGINT AUTO_INCREMENT PRIMARY KEY,  -- Surrogate key
    username VARCHAR(50) UNIQUE NOT NULL,       -- Natural key candidate
    email VARCHAR(100) UNIQUE NOT NULL,         -- Natural key candidate
    phone VARCHAR(15) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    status ENUM('active', 'suspended', 'deleted') DEFAULT 'active'
);

-- Bảng Posts với composite key scenario
CREATE TABLE posts (
    post_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    content TEXT,
    post_type ENUM('text', 'image', 'video'),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
}
```

**Thực tế Scale**: 
- Facebook có 3 tỷ user → user_id phải là BIGINT (8 bytes) để chứa đủ
- Instagram có 2 tỷ posts/day → cần partition strategy
- TikTok có 1 tỷ videos → UUID để distributed generation

**DynamoDB Primary Key**:
```json
{
  "TableName": "UserPosts",
  "KeySchema": [
    {
      "AttributeName": "user_id",
      "KeyType": "HASH"  // Partition Key
    },
    {
      "AttributeName": "post_timestamp", 
      "KeyType": "RANGE" // Sort Key
    }
  ]
}
```

## 4. Foreign Key

**Định nghĩa chi tiết**: Cột hoặc tổ hợp cột trong một bảng tham chiếu đến Primary Key của bảng khác, tạo mối quan hệ (relationship) giữa các bảng. Foreign key đảm bảo referential integrity - không thể có reference đến record không tồn tại.

**Referential Integrity Rules**:
- **INSERT**: Không thể insert FK value không tồn tại trong parent table
- **UPDATE**: Không thể update FK thành value không tồn tại
- **DELETE**: Không thể delete parent record nếu có child records (hoặc CASCADE)

**Cascade Options**:
- **CASCADE**: Tự động delete/update child records
- **SET NULL**: Set FK thành NULL khi parent bị delete
- **RESTRICT**: Ngăn không cho delete parent nếu có child
- **SET DEFAULT**: Set FK thành default value

**Use Case - University Management System**:
```sql
-- Bảng Majors (Parent)
CREATE TABLE majors (
    major_id INT PRIMARY KEY,
    major_name VARCHAR(100) NOT NULL,
    department VARCHAR(50),
    credit_hours INT DEFAULT 120
);

-- Bảng Students (Child)
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    major_id INT,
    enrollment_date DATE,
    gpa DECIMAL(3,2),
    FOREIGN KEY (major_id) REFERENCES majors(major_id) 
        ON DELETE SET NULL 
        ON UPDATE CASCADE
);

-- Bảng Enrollments (Many-to-Many relationship)
CREATE TABLE enrollments (
    enrollment_id INT AUTO_INCREMENT PRIMARY KEY,
    student_id INT NOT NULL,
    course_id VARCHAR(10) NOT NULL,
    semester VARCHAR(20),  -- Fall2024, Spring2025
    grade CHAR(2),         -- A+, A, B+, B, C+, C, D, F
    enrollment_date DATE DEFAULT (CURRENT_DATE),
    FOREIGN KEY (student_id) REFERENCES students(student_id) 
        ON DELETE CASCADE,
    FOREIGN KEY (course_id) REFERENCES courses(course_id) 
        ON DELETE RESTRICT,
    UNIQUE KEY unique_enrollment (student_id, course_id, semester)
);
```

## 5. Index

**Định nghĩa chi tiết**: Cấu trúc dữ liệu phụ (auxiliary data structure) được tạo từ một hoặc nhiều columns của bảng để tăng tốc độ truy vấn. Index giống như mục lục sách - giúp tìm thông tin nhanh mà không cần đọc toàn bộ.

**Types of Indexes**:
- **Clustered Index**: Sắp xếp physical data theo index order (1 per table)
- **Non-clustered Index**: Pointer đến data rows (multiple per table)
- **Unique Index**: Đảm bảo uniqueness + tăng tốc search
- **Composite Index**: Index trên nhiều columns
- **Partial Index**: Index chỉ trên subset của data
- **Functional Index**: Index trên expression/function result

**Use Case - E-commerce Search Performance**:
```sql
-- Bảng products với 50 triệu records
CREATE TABLE products (
    product_id BIGINT PRIMARY KEY,           -- Clustered index tự động
    product_name VARCHAR(200),
    category_id INT,
    brand VARCHAR(100),
    price DECIMAL(12,2),
    description TEXT,
    created_at TIMESTAMP,
    status ENUM('active', 'inactive', 'discontinued')
);

-- Slow query without index: 30 seconds (full table scan)
SELECT * FROM products 
WHERE product_name LIKE '%iPhone%' 
  AND status = 'active';

-- Create indexes for common queries
CREATE INDEX idx_product_name ON products(product_name);           -- Single column
CREATE INDEX idx_category_price ON products(category_id, price);   -- Composite
CREATE INDEX idx_brand_status ON products(brand, status);          -- Composite
CREATE INDEX idx_created_date ON products(created_at);             -- Date queries

-- Partial index: chỉ index active products
CREATE INDEX idx_active_products ON products(product_name, price) 
WHERE status = 'active';

-- Functional index: search case-insensitive
CREATE INDEX idx_product_name_lower ON products(LOWER(product_name));

-- Fast query with index: 0.01 seconds
SELECT * FROM products 
WHERE product_name LIKE '%iPhone%' 
  AND status = 'active';
```

**Index Strategy cho Different Query Patterns**:
```sql
-- 1. Equality search: single column index
SELECT * FROM products WHERE category_id = 123;
-- Index: idx_category (category_id)

-- 2. Range search: single column index  
SELECT * FROM products WHERE price BETWEEN 1000000 AND 5000000;
-- Index: idx_price (price)

-- 3. Multiple conditions: composite index
SELECT * FROM products WHERE category_id = 123 AND price > 1000000;
-- Index: idx_category_price (category_id, price) - category_id first!

-- 4. ORDER BY optimization
SELECT * FROM products WHERE category_id = 123 ORDER BY created_at DESC;
-- Index: idx_category_created (category_id, created_at)
```

**AWS Implementation**:
- **RDS/Aurora**: B-tree indexes, Full-text indexes, Spatial indexes
- **DynamoDB**: 
  - Global Secondary Index (GSI): Query by non-key attributes
  - Local Secondary Index (LSI): Alternative sort key for same partition
}
```

## 4. Foreign Key

**Định nghĩa**: Cột tham chiếu đến Primary Key của bảng khác, tạo mối quan hệ.

**Use Case - University Management System**:
```sql
-- Bảng Students
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(100),
    major_id INT,
    FOREIGN KEY (major_id) REFERENCES majors(major_id)
);

-- Bảng Enrollments (Many-to-Many relationship)
CREATE TABLE enrollments (
    enrollment_id INT PRIMARY KEY,
    student_id INT,
    course_id INT,
    semester VARCHAR(20),
    grade CHAR(2),
    FOREIGN KEY (student_id) REFERENCES students(student_id),
    FOREIGN KEY (course_id) REFERENCES courses(course_id)
);
```

**Referential Integrity**: Không thể xóa major nếu còn student đang học major đó.

## 5. Index

**Định nghĩa**: Cấu trúc dữ liệu tăng tốc độ tìm kiếm, giống mục lục sách.

**Use Case - E-commerce Search**:
```sql
-- Không có index: quét 50 triệu sản phẩm (30 giây)
SELECT * FROM products WHERE product_name LIKE '%iPhone%';

-- Có index: tìm trong 0.01 giây
CREATE INDEX idx_product_name ON products(product_name);
CREATE INDEX idx_category_price ON products(category_id, price);

-- Composite index cho filter phức tạp
SELECT * FROM products 
WHERE category_id = 'electronics' 
  AND price BETWEEN 10000000 AND 30000000
  AND brand = 'Apple';
```

**AWS Implementation**:
- **RDS**: B-tree indexes, Full-text search indexes
- **DynamoDB**: Global Secondary Index (GSI), Local Secondary Index (LSI)

## 6. Partition

**Định nghĩa**: Chia bảng lớn thành nhiều phần nhỏ để tăng performance.

**Use Case - Telecom Call Records**:
```sql
-- Bảng call_records có 10 tỷ records
-- Partition theo tháng
CREATE TABLE call_records (
    call_id BIGINT,
    caller_number VARCHAR(15),
    receiver_number VARCHAR(15),
    call_date DATE,
    duration INT,
    cost DECIMAL(10,2)
) PARTITION BY RANGE (YEAR(call_date)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p2025 VALUES LESS THAN (2026)
);
```

**Benefits**:
- Query chỉ scan partition cần thiết
- Backup/restore từng partition
- Drop old partitions để tiết kiệm storage

**DynamoDB Partitioning**:
```json
{
  "TableName": "UserSessions",
  "PartitionKey": "user_id",  // Phân tán theo user
  "SortKey": "session_timestamp"
}
```

## 7. Database Log

**Định nghĩa**: Nhật ký ghi lại mọi thay đổi dữ liệu để recovery và audit.

**Use Case - Banking Transaction Log**:
```
Transaction ID: TXN_20240120_001
Timestamp: 2024-01-20 14:30:25
Operation: TRANSFER
From Account: 1234567890
To Account: 0987654321
Amount: 5000000 VND
Status: COMMITTED
Checksum: ABC123DEF456
```

**AWS Implementation**:
- **RDS**: Binary logs, Transaction logs
- **Aurora**: Continuous backup to S3
- **DynamoDB**: Point-in-time recovery (PITR)

## 8. Buffer

**Định nghĩa**: Vùng nhớ RAM tạm thời để cache dữ liệu thường xuyên truy cập.

**Use Case - News Website**:
```
Hot Articles (trong buffer):
- "Breaking News" - đọc 1M lần/giờ
- "Weather Today" - đọc 500K lần/giờ

Cold Articles (trên disk):
- "News from 2020" - đọc 10 lần/ngày
```

**Buffer Hit Ratio**: 95% = chỉ 5% queries phải đọc từ disk.

## 9. OLTP vs OLAP

### OLTP (Online Transaction Processing)
**Đặc điểm**: Giao dịch nhỏ, nhanh, thường xuyên
**Use Case - ATM System**:
```sql
-- Rút tiền (< 100ms)
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 500000 WHERE account_id = '123456';
INSERT INTO transactions (account_id, type, amount, timestamp) 
VALUES ('123456', 'WITHDRAWAL', 500000, NOW());
COMMIT;
```

### OLAP (Online Analytical Processing)
**Đặc điểm**: Phân tích dữ liệu lớn, phức tạp
**Use Case - Business Intelligence**:
```sql
-- Báo cáo doanh thu theo quý (chạy 30 phút)
SELECT 
    YEAR(order_date) as year,
    QUARTER(order_date) as quarter,
    SUM(total_amount) as revenue,
    COUNT(*) as total_orders,
    AVG(total_amount) as avg_order_value
FROM orders 
WHERE order_date >= '2020-01-01'
GROUP BY YEAR(order_date), QUARTER(order_date)
ORDER BY year, quarter;
```

**AWS Services**:
- **OLTP**: RDS, Aurora, DynamoDB
- **OLAP**: Redshift, Athena, QuickSight

## 10. Session

**Định nghĩa**: Khoảng thời gian kết nối từ client đến database.

**Use Case - Web Application**:
```python
# User login tạo session
session_id = "sess_abc123def456"
user_id = 12345
login_time = "2024-01-20 09:00:00"
last_activity = "2024-01-20 09:15:30"
session_timeout = 30  # minutes

# Session data
shopping_cart = ["product_1", "product_2"]
user_preferences = {"language": "vi", "currency": "VND"}
```

**Session Management**:
- **Active Sessions**: 50,000 concurrent users
- **Session Timeout**: 30 phút không hoạt động
- **Session Storage**: Redis/ElastiCache

## 11. Execution Plan (Query Plan)

**Định nghĩa**: Kế hoạch thực thi mà DBMS chọn để xử lý query.

**Use Case - E-commerce Product Search**:
```sql
EXPLAIN SELECT p.name, p.price, c.category_name 
FROM products p 
JOIN categories c ON p.category_id = c.category_id 
WHERE p.price BETWEEN 1000000 AND 5000000 
  AND c.category_name = 'Electronics';

-- Execution Plan:
-- 1. Index Seek on categories (category_name = 'Electronics')
-- 2. Index Seek on products (price range)  
-- 3. Nested Loop Join
-- Cost: 150 units, Rows: 5000
```

## 12. Data Warehouse

**Định nghĩa**: Kho dữ liệu tổng hợp từ nhiều nguồn để phân tích business intelligence.

**Use Case - Retail Chain Analytics**:
```sql
-- Fact Table: Sales
CREATE TABLE fact_sales (
    sale_id BIGINT,
    product_id INT,
    store_id INT,
    customer_id INT,
    date_id INT,
    quantity INT,
    unit_price DECIMAL(10,2),
    total_amount DECIMAL(12,2)
);

-- Dimension Tables
CREATE TABLE dim_products (product_id, name, category, brand);
CREATE TABLE dim_stores (store_id, name, city, region);
CREATE TABLE dim_customers (customer_id, age_group, gender, segment);
CREATE TABLE dim_date (date_id, date, month, quarter, year);
```

**Star Schema Benefits**:
- Query performance tối ưu
- Business user dễ hiểu
- Aggregation nhanh

## 13. NoSQL Database Types

### Document Store (MongoDB, DynamoDB)
```json
// User Profile Document
{
  "user_id": "user_123",
  "name": "Nguyen Van A",
  "profile": {
    "age": 25,
    "interests": ["technology", "sports"],
    "address": {
      "city": "Ho Chi Minh",
      "district": "District 1"
    }
  },
  "orders": [
    {"order_id": "ord_001", "amount": 500000},
    {"order_id": "ord_002", "amount": 750000}
  ]
}
```

### Key-Value Store (Redis, DynamoDB)
```python
# Session Cache
session:user_123 = {
    "login_time": "2024-01-20T09:00:00Z",
    "cart_items": ["item1", "item2"],
    "last_page": "/products/electronics"
}

# Real-time Leaderboard
leaderboard:game_001 = {
    "player_1": 9500,
    "player_2": 8750,
    "player_3": 8200
}
```

### Graph Database (Neptune)
```cypher
// Social Network Relationships
CREATE (alice:Person {name: 'Alice', age: 25})
CREATE (bob:Person {name: 'Bob', age: 30})
CREATE (charlie:Person {name: 'Charlie', age: 28})

CREATE (alice)-[:FRIENDS_WITH {since: '2020-01-01'}]->(bob)
CREATE (bob)-[:FRIENDS_WITH {since: '2019-06-15'}]->(charlie)

// Find friends of friends
MATCH (alice:Person {name: 'Alice'})-[:FRIENDS_WITH]->(friend)-[:FRIENDS_WITH]->(fof)
RETURN fof.name
```

## 14. Caching

**Định nghĩa**: Lưu tạm dữ liệu thường xuyên truy cập trong memory để tăng tốc.

**Use Case - News Website Caching Strategy**:
```python
# Cache Hierarchy
L1_CACHE = "Application Memory"     # 100ms TTL
L2_CACHE = "Redis Cluster"          # 5 minutes TTL  
L3_CACHE = "CDN (CloudFront)"       # 1 hour TTL
DATABASE = "Aurora MySQL"           # Source of truth

# Cache Pattern
def get_article(article_id):
    # Check L1 Cache
    article = app_cache.get(f"article:{article_id}")
    if article:
        return article
    
    # Check L2 Cache  
    article = redis.get(f"article:{article_id}")
    if article:
        app_cache.set(f"article:{article_id}", article, ttl=100)
        return article
    
    # Query Database
    article = db.query("SELECT * FROM articles WHERE id = ?", article_id)
    redis.set(f"article:{article_id}", article, ttl=300)
    return article
```

## 15. Replication

**Định nghĩa**: Sao chép dữ liệu từ database chính sang database phụ.

**Use Case - Global E-commerce Platform**:
```
Master Database (Singapore):
├── Write Operations: Orders, Payments, Inventory Updates
├── Read Replica 1 (Tokyo): Serve Japanese customers
├── Read Replica 2 (Sydney): Serve Australian customers  
└── Read Replica 3 (Mumbai): Serve Indian customers

Replication Lag: < 100ms
Failover Time: < 30 seconds
```

**AWS Aurora Global Database**:
- Primary Region: ap-southeast-1 (Singapore)
- Secondary Regions: ap-northeast-1 (Tokyo), ap-south-1 (Mumbai)
- Cross-region replication: < 1 second lag

## 16. Transaction & ACID

**Use Case - E-commerce Order Processing**:
```sql
BEGIN TRANSACTION;

-- 1. Atomicity: All or nothing
UPDATE inventory SET quantity = quantity - 1 WHERE product_id = 'iPhone15';
INSERT INTO orders (customer_id, product_id, amount) VALUES (123, 'iPhone15', 25000000);
INSERT INTO payments (order_id, amount, status) VALUES (LAST_INSERT_ID(), 25000000, 'pending');

-- 2. Consistency: Business rules maintained
IF (SELECT quantity FROM inventory WHERE product_id = 'iPhone15') < 0 THEN
    ROLLBACK;
END IF;

-- 3. Isolation: Concurrent transactions don't interfere
-- 4. Durability: Changes persist after commit
COMMIT;
```

**Real-world Impact**:
- **Success**: Customer gets iPhone, inventory decreases, payment processed
- **Failure**: No changes applied, customer notified, inventory unchanged

## 17. CAP Theorem

**Use Case - Distributed Social Media Platform**:

### Consistency + Availability (CA) - Traditional RDBMS
```
Scenario: Bank transfer between accounts
Priority: Data must be consistent (no money lost)
Trade-off: System may be unavailable during network partition
Example: Aurora MySQL with Multi-AZ
```

### Consistency + Partition Tolerance (CP) - MongoDB
```
Scenario: Inventory management system  
Priority: Stock count must be accurate
Trade-off: Some nodes may be unavailable
Example: MongoDB with majority write concern
```

### Availability + Partition Tolerance (AP) - DynamoDB
```
Scenario: Social media likes/comments
Priority: System always available for users
Trade-off: Like counts may be temporarily inconsistent
Example: DynamoDB with eventual consistency
```

## 18. Schema Design Patterns

### Normalization (OLTP)
```sql
-- 3NF Normalized Design
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10,2),
    PRIMARY KEY (order_id, product_id)
);
```

### Denormalization (OLAP)
```sql
-- Denormalized for Analytics
CREATE TABLE sales_fact (
    sale_id BIGINT PRIMARY KEY,
    customer_name VARCHAR(100),      -- Denormalized
    customer_email VARCHAR(100),     -- Denormalized  
    product_name VARCHAR(200),       -- Denormalized
    category_name VARCHAR(100),      -- Denormalized
    sale_date DATE,
    quantity INT,
    unit_price DECIMAL(10,2),
    total_amount DECIMAL(12,2)
);
```

## 19. Performance Optimization

**Use Case - High-Traffic News Website**:

### Query Optimization
```sql
-- Slow Query (5 seconds)
SELECT a.title, a.content, u.name as author
FROM articles a, users u  
WHERE a.author_id = u.user_id 
  AND a.published_date > '2024-01-01'
  AND a.category = 'Technology';

-- Optimized Query (0.1 seconds)
SELECT a.title, a.content, u.name as author
FROM articles a
INNER JOIN users u ON a.author_id = u.user_id
WHERE a.published_date > '2024-01-01'
  AND a.category = 'Technology'
  AND a.status = 'published';

-- Add indexes
CREATE INDEX idx_articles_date_category ON articles(published_date, category, status);
CREATE INDEX idx_articles_author ON articles(author_id);
```

### Connection Pooling
```python
# Without Connection Pool: 1000ms per request
def get_user_without_pool(user_id):
    conn = mysql.connect(host='db.example.com')  # 500ms
    result = conn.execute(f"SELECT * FROM users WHERE id = {user_id}")  # 100ms
    conn.close()  # 400ms
    return result

# With Connection Pool: 100ms per request  
pool = ConnectionPool(size=50, max_overflow=100)
def get_user_with_pool(user_id):
    conn = pool.get_connection()  # 1ms
    result = conn.execute(f"SELECT * FROM users WHERE id = {user_id}")  # 100ms
    pool.return_connection(conn)  # 1ms
    return result
```

## 20. Data Migration Strategies

**Use Case - Legacy System Migration**:

### Phase 1: Assessment
```
Source: Oracle 11g (On-premise)
- 500GB data
- 200 tables  
- 50 stored procedures
- 1000 concurrent users

Target: Aurora PostgreSQL (AWS)
- Compatible SQL syntax: 85%
- Manual conversion needed: 15%
```

### Phase 2: Schema Conversion
```sql
-- Oracle Source
CREATE TABLE employees (
    emp_id NUMBER(10) PRIMARY KEY,
    hire_date DATE,
    salary NUMBER(10,2)
);

-- Aurora PostgreSQL Target  
CREATE TABLE employees (
    emp_id BIGINT PRIMARY KEY,
    hire_date DATE,
    salary DECIMAL(10,2)
);
```

### Phase 3: Data Migration
```python
# AWS DMS Configuration
{
    "source_endpoint": "oracle-prod.company.com:1521",
    "target_endpoint": "aurora-cluster.cluster-xyz.us-east-1.rds.amazonaws.com",
    "migration_type": "full-load-and-cdc",
    "table_mappings": {
        "employees": "employees",
        "departments": "departments"
    }
}
```

## Summary

Database concepts tạo nền tảng cho việc thiết kế hệ thống hiệu quả:

1. **Structured Data**: RDS, Aurora cho OLTP
2. **Unstructured Data**: DynamoDB, DocumentDB cho NoSQL  
3. **Analytics**: Redshift, Athena cho OLAP
4. **Caching**: ElastiCache cho performance
5. **Global Scale**: Aurora Global, DynamoDB Global Tables

Mỗi use case cần chọn đúng database type và optimization strategy để đạt hiệu quả tối ưu.

## 6. Partition

**Định nghĩa chi tiết**: Kỹ thuật chia một bảng lớn thành nhiều phần nhỏ hơn (partitions) dựa trên một tiêu chí cụ thể. Mỗi partition có thể được lưu trữ và quản lý độc lập, giúp cải thiện performance, manageability và availability.

**Types of Partitioning**:
- **Range Partitioning**: Chia theo khoảng giá trị (dates, numbers)
- **Hash Partitioning**: Chia theo hash function của partition key
- **List Partitioning**: Chia theo danh sách giá trị cụ thể
- **Composite Partitioning**: Kết hợp nhiều phương pháp

**Use Case - Telecom Call Records (100 tỷ records)**:
```sql
-- Range Partitioning theo tháng
CREATE TABLE call_records (
    call_id BIGINT AUTO_INCREMENT,
    caller_number VARCHAR(15) NOT NULL,
    receiver_number VARCHAR(15) NOT NULL,
    call_date DATE NOT NULL,
    call_time TIME,
    duration_seconds INT,
    call_type ENUM('local', 'national', 'international'),
    cost DECIMAL(10,4),
    tower_id INT,
    INDEX idx_caller (caller_number),
    INDEX idx_receiver (receiver_number)
) PARTITION BY RANGE (YEAR(call_date) * 100 + MONTH(call_date)) (
    PARTITION p202301 VALUES LESS THAN (202302),
    PARTITION p202302 VALUES LESS THAN (202303),
    PARTITION p202303 VALUES LESS THAN (202304),
    -- ... continue for each month
    PARTITION p202412 VALUES LESS THAN (202501),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Hash Partitioning cho user data
CREATE TABLE user_activities (
    user_id BIGINT,
    activity_type VARCHAR(50),
    activity_data JSON,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
) PARTITION BY HASH(user_id) PARTITIONS 16;
```

**Benefits của Partitioning**:
- **Query Performance**: Partition pruning - chỉ scan partitions cần thiết
- **Maintenance**: Backup/restore từng partition độc lập
- **Storage Management**: Drop old partitions để tiết kiệm space
- **Parallel Processing**: Queries có thể chạy parallel trên multiple partitions

**DynamoDB Partitioning Strategy**:
```json
{
  "TableName": "UserSessions",
  "PartitionKey": "user_id",        // Phân tán theo user
  "SortKey": "session_timestamp",   // Sắp xếp theo time
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "SessionsByDate",
      "PartitionKey": "date",         // Query theo ngày
      "SortKey": "session_timestamp"
    }
  ]
}
```

## 7. Database Log

**Định nghĩa chi tiết**: Hệ thống nhật ký ghi lại tuần tự tất cả các thay đổi dữ liệu (transactions) để đảm bảo durability, recovery và auditing. Database log là foundation của ACID properties và disaster recovery.

**Types of Database Logs**:
- **Transaction Log (Redo Log)**: Ghi các thay đổi để recovery
- **Undo Log**: Ghi trạng thái cũ để rollback
- **Binary Log**: Replication và point-in-time recovery
- **Audit Log**: Security và compliance tracking
- **Error Log**: System errors và warnings

**Use Case - Banking Transaction Log**:
```sql
-- Transaction log entry structure
CREATE TABLE transaction_log (
    log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    transaction_id VARCHAR(50) NOT NULL,
    timestamp TIMESTAMP(6) DEFAULT CURRENT_TIMESTAMP(6),
    operation_type ENUM('BEGIN', 'INSERT', 'UPDATE', 'DELETE', 'COMMIT', 'ROLLBACK'),
    table_name VARCHAR(100),
    record_id VARCHAR(100),
    old_values JSON,
    new_values JSON,
    user_id VARCHAR(50),
    session_id VARCHAR(100),
    checksum VARCHAR(64)
);

-- Example log entries for money transfer
INSERT INTO transaction_log VALUES
(1, 'TXN_20240120_001', '2024-01-20 14:30:25.123456', 'BEGIN', NULL, NULL, NULL, NULL, 'user123', 'sess_abc', NULL),
(2, 'TXN_20240120_001', '2024-01-20 14:30:25.234567', 'UPDATE', 'accounts', '1234567890', 
 '{"balance": 10000000}', '{"balance": 5000000}', 'user123', 'sess_abc', 'hash1'),
(3, 'TXN_20240120_001', '2024-01-20 14:30:25.345678', 'UPDATE', 'accounts', '0987654321',
 '{"balance": 2000000}', '{"balance": 7000000}', 'user123', 'sess_abc', 'hash2'),
(4, 'TXN_20240120_001', '2024-01-20 14:30:25.456789', 'INSERT', 'transactions', '999888777',
 NULL, '{"from_account": "1234567890", "to_account": "0987654321", "amount": 5000000}', 'user123', 'sess_abc', 'hash3'),
(5, 'TXN_20240120_001', '2024-01-20 14:30:25.567890', 'COMMIT', NULL, NULL, NULL, NULL, 'user123', 'sess_abc', NULL);
```

**AWS Implementation**:
- **RDS**: Automated backups với binary logs, transaction logs
- **Aurora**: Continuous backup to S3, log-structured storage
- **DynamoDB**: Point-in-time recovery (PITR) với change streams

## 8. Buffer Pool

**Định nghĩa chi tiết**: Vùng nhớ RAM được DBMS sử dụng để cache data pages, index pages và other database objects. Buffer pool giảm disk I/O bằng cách giữ frequently accessed data trong memory.

**Buffer Pool Components**:
- **Data Pages**: Actual table data
- **Index Pages**: B-tree index nodes  
- **Log Buffer**: Transaction log entries chờ flush to disk
- **Dictionary Cache**: Metadata về tables, columns, constraints

**Buffer Pool Algorithms**:
- **LRU (Least Recently Used)**: Evict pages ít được dùng nhất
- **Clock Algorithm**: Circular buffer với reference bits
- **LFU (Least Frequently Used)**: Evict pages có frequency thấp nhất

**Use Case - High-Traffic News Website**:
```sql
-- Buffer pool configuration
SET GLOBAL innodb_buffer_pool_size = 8589934592;  -- 8GB
SET GLOBAL innodb_buffer_pool_instances = 8;       -- 8 instances for concurrency

-- Monitor buffer pool efficiency
SELECT 
    ROUND((1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100, 2) AS buffer_hit_ratio,
    Innodb_buffer_pool_read_requests AS total_requests,
    Innodb_buffer_pool_reads AS disk_reads,
    ROUND(Innodb_buffer_pool_bytes_data / 1024 / 1024, 2) AS data_mb,
    ROUND(Innodb_buffer_pool_bytes_dirty / 1024 / 1024, 2) AS dirty_mb
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME IN (
    'Innodb_buffer_pool_read_requests',
    'Innodb_buffer_pool_reads',
    'Innodb_buffer_pool_bytes_data', 
    'Innodb_buffer_pool_bytes_dirty'
);
```

**Performance Impact**:
```
Hot Articles (trong buffer pool):
- "Breaking News" - đọc 1M lần/giờ - 1ms response time
- "Weather Today" - đọc 500K lần/giờ - 1ms response time

Cold Articles (disk reads):
- "News from 2020" - đọc 10 lần/ngày - 50ms response time

Buffer Hit Ratio: 98% = chỉ 2% queries phải đọc từ disk
```

## 9. OLTP vs OLAP

### OLTP (Online Transaction Processing)

**Định nghĩa chi tiết**: Hệ thống xử lý giao dịch trực tuyến, tối ưu cho việc thực hiện nhiều transactions nhỏ, nhanh và concurrent. OLTP systems support day-to-day business operations.

**Đặc điểm OLTP**:
- **High Concurrency**: Hàng nghìn users đồng thời
- **Low Latency**: Response time < 100ms
- **ACID Compliance**: Đảm bảo data consistency
- **Normalized Schema**: Giảm redundancy
- **Small Transactions**: Insert/Update/Delete ít records

**Use Case - ATM Banking System**:
```sql
-- Typical OLTP transaction: ATM withdrawal (< 100ms)
BEGIN TRANSACTION;

-- Check account balance
SELECT balance, account_status, daily_limit_remaining 
FROM accounts 
WHERE account_number = '1234567890' 
FOR UPDATE;  -- Row-level lock

-- Validate withdrawal
IF balance >= 500000 AND account_status = 'active' AND daily_limit_remaining >= 500000 THEN
    -- Update account balance
    UPDATE accounts 
    SET balance = balance - 500000,
        daily_limit_remaining = daily_limit_remaining - 500000,
        last_transaction_date = NOW()
    WHERE account_number = '1234567890';
    
    -- Record transaction
    INSERT INTO transactions (
        account_number, transaction_type, amount, 
        atm_id, transaction_date, status
    ) VALUES (
        '1234567890', 'WITHDRAWAL', 500000,
        'ATM_001', NOW(), 'COMPLETED'
    );
    
    COMMIT;
ELSE
    ROLLBACK;
END IF;
```

### OLAP (Online Analytical Processing)

**Định nghĩa chi tiết**: Hệ thống phân tích trực tuyến, tối ưu cho việc thực hiện complex queries trên large datasets để business intelligence và reporting. OLAP systems support decision-making processes.

**Đặc điểm OLAP**:
- **Complex Queries**: Aggregations, joins, subqueries
- **Large Data Volumes**: Terabytes to petabytes
- **Denormalized Schema**: Star/snowflake schema
- **Batch Processing**: ETL jobs, scheduled reports
- **Read-Heavy**: Mostly SELECT operations

**Use Case - Retail Business Intelligence**:
```sql
-- Complex OLAP query: Quarterly sales analysis (chạy 30 phút)
SELECT 
    p.category_name,
    p.brand,
    d.year,
    d.quarter,
    COUNT(DISTINCT s.customer_id) as unique_customers,
    SUM(s.quantity) as total_units_sold,
    SUM(s.total_amount) as total_revenue,
    AVG(s.total_amount) as avg_order_value,
    SUM(s.total_amount) - SUM(s.cost_amount) as gross_profit,
    ROUND((SUM(s.total_amount) - SUM(s.cost_amount)) / SUM(s.total_amount) * 100, 2) as profit_margin_pct
FROM fact_sales s
JOIN dim_products p ON s.product_id = p.product_id
JOIN dim_date d ON s.date_id = d.date_id
JOIN dim_stores st ON s.store_id = st.store_id
WHERE d.year >= 2022
  AND st.region IN ('North', 'South', 'Central')
GROUP BY p.category_name, p.brand, d.year, d.quarter
HAVING SUM(s.total_amount) > 1000000  -- Categories with >1M revenue
ORDER BY d.year DESC, d.quarter DESC, total_revenue DESC;
```

**AWS Services Comparison**:
```
OLTP Services:
- RDS (MySQL, PostgreSQL, SQL Server): Traditional relational OLTP
- Aurora: Cloud-native, 5x faster than MySQL, auto-scaling
- DynamoDB: NoSQL, single-digit millisecond latency

OLAP Services:  
- Redshift: Petabyte-scale data warehouse, columnar storage
- Athena: Serverless query service for S3 data
- QuickSight: Business intelligence và visualization
```

## 10. Session Management

**Định nghĩa chi tiết**: Quản lý kết nối và trạng thái của users/applications với database system. Session bao gồm authentication, connection pooling, transaction state và resource allocation.

**Session Lifecycle**:
1. **Connection Establishment**: Authentication, SSL handshake
2. **Session Initialization**: Set session variables, character sets
3. **Transaction Processing**: Execute queries, maintain state
4. **Session Cleanup**: Release locks, close cursors
5. **Connection Termination**: Cleanup resources

**Use Case - E-commerce Web Application**:
```python
# Session management với connection pooling
import mysql.connector.pooling
from datetime import datetime, timedelta

# Connection pool configuration
pool_config = {
    'pool_name': 'ecommerce_pool',
    'pool_size': 50,           # Normal connections
    'pool_reset_session': True, # Reset session variables
    'host': 'aurora-cluster.cluster-xyz.us-east-1.rds.amazonaws.com',
    'database': 'ecommerce',
    'user': 'app_user',
    'password': 'secure_password',
    'autocommit': False        # Explicit transaction control
}

connection_pool = mysql.connector.pooling.MySQLConnectionPool(**pool_config)

class SessionManager:
    def __init__(self):
        self.active_sessions = {}
        
    def create_session(self, user_id, ip_address):
        session_id = f"sess_{user_id}_{int(datetime.now().timestamp())}"
        
        # Get connection from pool
        connection = connection_pool.get_connection()
        
        session_data = {
            'session_id': session_id,
            'user_id': user_id,
            'connection': connection,
            'created_at': datetime.now(),
            'last_activity': datetime.now(),
            'ip_address': ip_address,
            'shopping_cart': [],
            'transaction_count': 0
        }
        
        # Set session-specific variables
        cursor = connection.cursor()
        cursor.execute("SET SESSION sql_mode = 'STRICT_TRANS_TABLES'")
        cursor.execute("SET SESSION transaction_isolation = 'READ COMMITTED'")
        cursor.execute(f"SET @user_id = {user_id}")
        cursor.execute(f"SET @session_id = '{session_id}'")
        
        self.active_sessions[session_id] = session_data
        return session_id
    
    def get_session(self, session_id):
        if session_id in self.active_sessions:
            session = self.active_sessions[session_id]
            
            # Check session timeout (30 minutes)
            if datetime.now() - session['last_activity'] > timedelta(minutes=30):
                self.cleanup_session(session_id)
                return None
                
            # Update last activity
            session['last_activity'] = datetime.now()
            return session
        return None
    
    def cleanup_session(self, session_id):
        if session_id in self.active_sessions:
            session = self.active_sessions[session_id]
            
            # Rollback any uncommitted transactions
            try:
                session['connection'].rollback()
                session['connection'].close()
            except:
                pass
                
            del self.active_sessions[session_id]

# Usage example
session_mgr = SessionManager()

def process_order(session_id, order_data):
    session = session_mgr.get_session(session_id)
    if not session:
        raise Exception("Invalid or expired session")
    
    connection = session['connection']
    cursor = connection.cursor()
    
    try:
        # Start transaction
        cursor.execute("START TRANSACTION")
        
        # Process order logic
        cursor.execute("""
            INSERT INTO orders (user_id, total_amount, status, session_id)
            VALUES (@user_id, %s, 'pending', @session_id)
        """, (order_data['total'],))
        
        order_id = cursor.lastrowid
        
        # Insert order items
        for item in order_data['items']:
            cursor.execute("""
                INSERT INTO order_items (order_id, product_id, quantity, price)
                VALUES (%s, %s, %s, %s)
            """, (order_id, item['product_id'], item['quantity'], item['price']))
        
        # Update inventory
        for item in order_data['items']:
            cursor.execute("""
                UPDATE products 
                SET stock_quantity = stock_quantity - %s
                WHERE product_id = %s AND stock_quantity >= %s
            """, (item['quantity'], item['product_id'], item['quantity']))
            
            if cursor.rowcount == 0:
                raise Exception(f"Insufficient stock for product {item['product_id']}")
        
        # Commit transaction
        connection.commit()
        session['transaction_count'] += 1
        
        return {'order_id': order_id, 'status': 'success'}
        
    except Exception as e:
        # Rollback on error
        connection.rollback()
        raise e
```

**AWS RDS Session Monitoring**:
```sql
-- Monitor active sessions
SELECT 
    ID as session_id,
    USER as username,
    HOST as client_host,
    DB as database_name,
    COMMAND as current_command,
    TIME as duration_seconds,
    STATE as session_state,
    INFO as current_query
FROM information_schema.PROCESSLIST
WHERE COMMAND != 'Sleep'
ORDER BY TIME DESC;

-- Session statistics
SELECT 
    VARIABLE_NAME,
    VARIABLE_VALUE
FROM information_schema.GLOBAL_STATUS
WHERE VARIABLE_NAME IN (
    'Threads_connected',      -- Current connections
    'Threads_running',        -- Active threads
    'Max_used_connections',   -- Peak connections
    'Connection_errors_max_connections'  -- Connection errors
);
```
