# Database Concepts - Part 7 (Concepts 40-50)

## 40. Streaming Data & Real-time Processing

**Định nghĩa chi tiết**: Streaming data processing xử lý continuous data flows in real-time thay vì batch processing. Enables immediate insights và responses to events as they occur.

**Use Case - Real-time Fraud Detection System**:
```python
import boto3
import json
from datetime import datetime

class RealTimeFraudDetection:
    def __init__(self):
        self.kinesis = boto3.client('kinesis')
        self.dynamodb = boto3.resource('dynamodb')
        self.fraud_rules_table = self.dynamodb.Table('FraudRules')
        
    def process_transaction_stream(self, transaction_data):
        """Process incoming transaction for fraud detection"""
        
        # Real-time fraud scoring
        fraud_score = self.calculate_fraud_score(transaction_data)
        
        # Immediate action based on score
        if fraud_score > 0.8:
            self.block_transaction(transaction_data)
        elif fraud_score > 0.5:
            self.flag_for_review(transaction_data)
        
        # Store for ML model training
        self.store_transaction_features(transaction_data, fraud_score)
        
        return fraud_score
    
    def calculate_fraud_score(self, transaction):
        """Calculate real-time fraud score"""
        score = 0.0
        
        # Velocity checks
        if transaction['amount'] > 10000:
            score += 0.3
        
        # Geographic anomaly
        if self.is_geographic_anomaly(transaction):
            score += 0.4
            
        # Time-based patterns
        if self.is_unusual_time(transaction):
            score += 0.2
            
        return min(score, 1.0)
```

## 41. Data Governance & Compliance

**Định nghĩa chi tiết**: Data governance thiết lập policies, procedures và standards để manage data assets. Compliance đảm bảo adherence to regulatory requirements như GDPR, HIPAA, SOX.

**Use Case - GDPR Compliance System**:
```sql
-- Data classification table
CREATE TABLE data_classification (
    table_name VARCHAR(100),
    column_name VARCHAR(100),
    classification ENUM('PUBLIC', 'INTERNAL', 'CONFIDENTIAL', 'RESTRICTED'),
    contains_pii BOOLEAN,
    retention_period_days INT,
    legal_basis VARCHAR(200),
    PRIMARY KEY (table_name, column_name)
);

-- Data processing log for GDPR Article 30
CREATE TABLE data_processing_log (
    log_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    processing_activity VARCHAR(200),
    data_subject_category VARCHAR(100),
    personal_data_categories TEXT,
    recipients TEXT,
    retention_period VARCHAR(100),
    security_measures TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Right to be forgotten implementation
DELIMITER //
CREATE PROCEDURE ProcessDataDeletionRequest(IN p_customer_id INT)
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE table_name VARCHAR(100);
    DECLARE deletion_cursor CURSOR FOR
        SELECT DISTINCT table_name 
        FROM data_classification 
        WHERE contains_pii = TRUE;
    
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
    
    -- Log the deletion request
    INSERT INTO data_processing_log (
        processing_activity, data_subject_category, 
        personal_data_categories, created_at
    ) VALUES (
        'Data Deletion Request', 'Customer',
        'All PII data', NOW()
    );
    
    OPEN deletion_cursor;
    
    deletion_loop: LOOP
        FETCH deletion_cursor INTO table_name;
        IF done THEN
            LEAVE deletion_loop;
        END IF;
        
        -- Anonymize or delete PII data
        CASE table_name
            WHEN 'customers' THEN
                UPDATE customers 
                SET first_name = 'DELETED',
                    last_name = 'DELETED',
                    email = CONCAT('deleted_', customer_id, '@deleted.com'),
                    phone = NULL,
                    address = 'DELETED'
                WHERE customer_id = p_customer_id;
                
            WHEN 'orders' THEN
                -- Keep order data but remove PII references
                UPDATE orders 
                SET shipping_address = 'DELETED',
                    billing_address = 'DELETED'
                WHERE customer_id = p_customer_id;
        END CASE;
        
    END LOOP;
    
    CLOSE deletion_cursor;
END //
DELIMITER ;
```

## 42. Multi-Tenancy Patterns

**Định nghĩa chi tiết**: Multi-tenancy cho phép single application instance serve multiple customers (tenants) với data isolation và customization. Critical cho SaaS applications.

**Tenancy Models**:
- **Shared Database, Shared Schema**: Tenant ID column
- **Shared Database, Separate Schema**: Schema per tenant  
- **Separate Database**: Complete isolation per tenant

**Use Case - SaaS CRM Platform**:
```sql
-- Shared schema with tenant isolation
CREATE TABLE tenants (
    tenant_id INT AUTO_INCREMENT PRIMARY KEY,
    tenant_name VARCHAR(100) NOT NULL,
    subscription_tier ENUM('BASIC', 'PREMIUM', 'ENTERPRISE'),
    max_users INT,
    max_storage_gb INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE customers (
    customer_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tenant_id INT NOT NULL,
    customer_name VARCHAR(200),
    email VARCHAR(255),
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (tenant_id) REFERENCES tenants(tenant_id),
    INDEX idx_tenant_customer (tenant_id, customer_id)
);

-- Row Level Security (RLS) implementation
CREATE TABLE user_sessions (
    session_id VARCHAR(100) PRIMARY KEY,
    user_id INT,
    tenant_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Stored procedure for tenant-aware queries
DELIMITER //
CREATE PROCEDURE GetCustomersByTenant(IN p_session_id VARCHAR(100))
BEGIN
    DECLARE v_tenant_id INT;
    
    -- Get tenant from session
    SELECT tenant_id INTO v_tenant_id
    FROM user_sessions
    WHERE session_id = p_session_id;
    
    -- Return only tenant's customers
    SELECT customer_id, customer_name, email, phone
    FROM customers
    WHERE tenant_id = v_tenant_id;
END //
DELIMITER ;
```

## 43. Database Virtualization

**Định nghĩa chi tiết**: Database virtualization tạo abstraction layer giữa applications và physical databases, enabling data federation, load balancing và transparent scaling.

**Use Case - Database Proxy for Read/Write Splitting**:
```python
import mysql.connector
import random
from typing import List, Dict

class DatabaseProxy:
    def __init__(self):
        self.write_endpoints = [
            {'host': 'master.cluster-xyz.us-east-1.rds.amazonaws.com', 'weight': 1}
        ]
        
        self.read_endpoints = [
            {'host': 'reader1.cluster-xyz.us-east-1.rds.amazonaws.com', 'weight': 3},
            {'host': 'reader2.cluster-xyz.us-east-1.rds.amazonaws.com', 'weight': 2},
            {'host': 'reader3.cluster-xyz.us-east-1.rds.amazonaws.com', 'weight': 1}
        ]
        
        self.connection_pools = {}
    
    def route_query(self, sql_query: str):
        """Route query to appropriate endpoint based on operation type"""
        
        query_type = self.analyze_query_type(sql_query)
        
        if query_type in ['INSERT', 'UPDATE', 'DELETE', 'CREATE', 'ALTER', 'DROP']:
            endpoint = self.select_write_endpoint()
        else:
            endpoint = self.select_read_endpoint()
        
        return self.execute_query(endpoint, sql_query)
    
    def select_read_endpoint(self) -> str:
        """Select read endpoint using weighted round-robin"""
        
        total_weight = sum(ep['weight'] for ep in self.read_endpoints)
        random_weight = random.randint(1, total_weight)
        
        current_weight = 0
        for endpoint in self.read_endpoints:
            current_weight += endpoint['weight']
            if random_weight <= current_weight:
                return endpoint['host']
        
        return self.read_endpoints[0]['host']  # Fallback
    
    def analyze_query_type(self, sql_query: str) -> str:
        """Analyze SQL query to determine operation type"""
        
        query_upper = sql_query.strip().upper()
        
        if query_upper.startswith('SELECT'):
            return 'SELECT'
        elif query_upper.startswith('INSERT'):
            return 'INSERT'
        elif query_upper.startswith('UPDATE'):
            return 'UPDATE'
        elif query_upper.startswith('DELETE'):
            return 'DELETE'
        else:
            return 'OTHER'
```

## 44. Time-Series Databases

**Định nghĩa chi tiết**: Time-series databases được optimize để store và query data points indexed by time. Ideal cho monitoring, IoT, financial data và analytics.

**Use Case - IoT Sensor Monitoring**:
```sql
-- Time-series table for sensor data
CREATE TABLE sensor_readings (
    sensor_id VARCHAR(50),
    timestamp TIMESTAMP(6),
    temperature DECIMAL(5,2),
    humidity DECIMAL(5,2),
    pressure DECIMAL(8,2),
    battery_level DECIMAL(5,2),
    
    PRIMARY KEY (sensor_id, timestamp),
    INDEX idx_timestamp (timestamp),
    INDEX idx_sensor_time (sensor_id, timestamp)
) PARTITION BY RANGE (UNIX_TIMESTAMP(timestamp)) (
    PARTITION p202401 VALUES LESS THAN (UNIX_TIMESTAMP('2024-02-01')),
    PARTITION p202402 VALUES LESS THAN (UNIX_TIMESTAMP('2024-03-01')),
    PARTITION p202403 VALUES LESS THAN (UNIX_TIMESTAMP('2024-04-01'))
);

-- Aggregated data for faster queries
CREATE TABLE sensor_hourly_stats (
    sensor_id VARCHAR(50),
    hour_timestamp TIMESTAMP,
    avg_temperature DECIMAL(5,2),
    min_temperature DECIMAL(5,2),
    max_temperature DECIMAL(5,2),
    avg_humidity DECIMAL(5,2),
    reading_count INT,
    
    PRIMARY KEY (sensor_id, hour_timestamp)
);

-- Time-series analytics queries
-- Moving average
SELECT 
    sensor_id,
    timestamp,
    temperature,
    AVG(temperature) OVER (
        PARTITION BY sensor_id 
        ORDER BY timestamp 
        ROWS BETWEEN 5 PRECEDING AND CURRENT ROW
    ) as moving_avg_temp
FROM sensor_readings
WHERE sensor_id = 'SENSOR_001'
  AND timestamp >= NOW() - INTERVAL 24 HOUR;

-- Anomaly detection
WITH sensor_stats AS (
    SELECT 
        sensor_id,
        AVG(temperature) as avg_temp,
        STDDEV(temperature) as stddev_temp
    FROM sensor_readings
    WHERE timestamp >= NOW() - INTERVAL 7 DAY
    GROUP BY sensor_id
)
SELECT 
    sr.sensor_id,
    sr.timestamp,
    sr.temperature,
    CASE 
        WHEN ABS(sr.temperature - ss.avg_temp) > 2 * ss.stddev_temp 
        THEN 'ANOMALY'
        ELSE 'NORMAL'
    END as status
FROM sensor_readings sr
JOIN sensor_stats ss ON sr.sensor_id = ss.sensor_id
WHERE sr.timestamp >= NOW() - INTERVAL 1 HOUR;
```

## 45. Vector Databases & AI Integration

**Định nghĩa chi tiết**: Vector databases store và query high-dimensional vectors, enabling similarity search cho AI/ML applications như recommendation systems, semantic search và RAG (Retrieval Augmented Generation).

**Use Case - E-commerce Recommendation System**:
```python
import numpy as np
import boto3
from typing import List, Tuple

class VectorRecommendationSystem:
    def __init__(self):
        self.opensearch = boto3.client('opensearchserverless')
        self.bedrock = boto3.client('bedrock-runtime')
        
    def generate_product_embeddings(self, product_data: dict) -> List[float]:
        """Generate embeddings for product using Amazon Bedrock"""
        
        # Combine product features into text
        product_text = f"""
        Product: {product_data['name']}
        Category: {product_data['category']}
        Description: {product_data['description']}
        Brand: {product_data['brand']}
        Price: ${product_data['price']}
        """
        
        # Call Bedrock Titan Embeddings
        response = self.bedrock.invoke_model(
            modelId='amazon.titan-embed-text-v1',
            body=json.dumps({
                'inputText': product_text
            })
        )
        
        embeddings = json.loads(response['body'].read())['embedding']
        return embeddings
    
    def store_product_vector(self, product_id: str, embeddings: List[float], metadata: dict):
        """Store product vector in OpenSearch"""
        
        document = {
            'product_id': product_id,
            'embeddings': embeddings,
            'metadata': metadata,
            'timestamp': datetime.now().isoformat()
        }
        
        # Index in OpenSearch with vector field
        self.opensearch.index(
            index='product-vectors',
            id=product_id,
            body=document
        )
    
    def find_similar_products(self, query_vector: List[float], k: int = 10) -> List[dict]:
        """Find similar products using vector similarity search"""
        
        search_query = {
            'size': k,
            'query': {
                'knn': {
                    'embeddings': {
                        'vector': query_vector,
                        'k': k
                    }
                }
            },
            '_source': ['product_id', 'metadata']
        }
        
        response = self.opensearch.search(
            index='product-vectors',
            body=search_query
        )
        
        similar_products = []
        for hit in response['hits']['hits']:
            similar_products.append({
                'product_id': hit['_source']['product_id'],
                'similarity_score': hit['_score'],
                'metadata': hit['_source']['metadata']
            })
        
        return similar_products
    
    def get_user_recommendations(self, user_id: str, num_recommendations: int = 5) -> List[dict]:
        """Generate personalized recommendations for user"""
        
        # Get user's purchase history
        user_purchases = self.get_user_purchase_history(user_id)
        
        # Generate user preference vector (average of purchased products)
        user_vectors = []
        for purchase in user_purchases:
            product_vector = self.get_product_vector(purchase['product_id'])
            if product_vector:
                user_vectors.append(product_vector)
        
        if not user_vectors:
            return self.get_popular_products(num_recommendations)
        
        # Calculate user preference vector
        user_preference_vector = np.mean(user_vectors, axis=0).tolist()
        
        # Find similar products
        similar_products = self.find_similar_products(
            user_preference_vector, 
            k=num_recommendations * 3  # Get more to filter out already purchased
        )
        
        # Filter out already purchased products
        purchased_product_ids = {p['product_id'] for p in user_purchases}
        recommendations = [
            p for p in similar_products 
            if p['product_id'] not in purchased_product_ids
        ]
        
        return recommendations[:num_recommendations]

# PostgreSQL with pgvector extension
pgvector_example = """
-- Install pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Create table with vector column
CREATE TABLE product_embeddings (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    category VARCHAR(100),
    embedding vector(1536),  -- 1536 dimensions for OpenAI embeddings
    created_at TIMESTAMP DEFAULT NOW()
);

-- Create vector index for fast similarity search
CREATE INDEX ON product_embeddings USING ivfflat (embedding vector_cosine_ops);

-- Insert product with embedding
INSERT INTO product_embeddings (name, category, embedding)
VALUES ('iPhone 15 Pro', 'Electronics', '[0.1, 0.2, 0.3, ...]');

-- Find similar products using cosine similarity
SELECT 
    product_id,
    name,
    category,
    1 - (embedding <=> '[0.1, 0.2, 0.3, ...]') AS similarity_score
FROM product_embeddings
ORDER BY embedding <=> '[0.1, 0.2, 0.3, ...]'
LIMIT 10;

-- Semantic search with filters
SELECT 
    p.product_id,
    p.name,
    p.category,
    1 - (p.embedding <=> $1) AS similarity_score
FROM product_embeddings p
WHERE p.category = 'Electronics'
  AND 1 - (p.embedding <=> $1) > 0.8  -- Similarity threshold
ORDER BY p.embedding <=> $1
LIMIT 5;
"""

# Usage example
recommender = VectorRecommendationSystem()

# Generate and store product embeddings
product_data = {
    'name': 'iPhone 15 Pro',
    'category': 'Electronics',
    'description': 'Latest iPhone with advanced camera system',
    'brand': 'Apple',
    'price': 999
}

embeddings = recommender.generate_product_embeddings(product_data)
recommender.store_product_vector('PROD_001', embeddings, product_data)

# Get recommendations for user
recommendations = recommender.get_user_recommendations('USER_123', 5)
print(f"Found {len(recommendations)} recommendations")
```
## 46. Database Federation

**Định nghĩa chi tiết**: Database federation tạo unified view của multiple heterogeneous databases, cho phép query across different systems như chúng là single database. Enables data integration without physical consolidation.

**Use Case - Enterprise Data Federation**:
```sql
-- MySQL Federated Engine example
CREATE TABLE federated_orders (
    order_id INT,
    customer_id INT,
    order_date DATE,
    total_amount DECIMAL(12,2)
) ENGINE=FEDERATED
CONNECTION='mysql://user:password@remote-server:3306/ecommerce/orders';

-- Query across federated tables
SELECT 
    fo.order_id,
    fo.total_amount,
    lc.customer_name,
    lc.email
FROM federated_orders fo
JOIN local_customers lc ON fo.customer_id = lc.customer_id
WHERE fo.order_date >= '2024-01-01';
```

## 47. Change Data Capture (CDC)

**Định nghĩa chi tiết**: CDC tracks và captures changes made to database tables in real-time, enabling downstream systems to react to data modifications immediately. Critical for data synchronization và event-driven architectures.

**Use Case - Real-time Data Synchronization**:
```python
import boto3
import json
from datetime import datetime

class CDCProcessor:
    def __init__(self):
        self.kinesis = boto3.client('kinesis')
        self.lambda_client = boto3.client('lambda')
        
    def process_cdc_event(self, event_data):
        """Process CDC event from database"""
        
        change_event = {
            'event_type': event_data['eventName'],  # INSERT, UPDATE, DELETE
            'table_name': event_data['tableName'],
            'timestamp': datetime.now().isoformat(),
            'old_image': event_data.get('oldImage', {}),
            'new_image': event_data.get('newImage', {}),
            'keys': event_data.get('keys', {})
        }
        
        # Route to appropriate handler
        if change_event['table_name'] == 'orders':
            self.handle_order_change(change_event)
        elif change_event['table_name'] == 'customers':
            self.handle_customer_change(change_event)
            
        # Send to downstream systems
        self.publish_to_stream(change_event)
    
    def handle_order_change(self, change_event):
        """Handle order table changes"""
        
        if change_event['event_type'] == 'INSERT':
            # New order created - trigger fulfillment
            self.trigger_order_fulfillment(change_event['new_image'])
            
        elif change_event['event_type'] == 'UPDATE':
            # Order updated - check status changes
            old_status = change_event['old_image'].get('status')
            new_status = change_event['new_image'].get('status')
            
            if old_status != new_status:
                self.handle_status_change(change_event['keys']['order_id'], old_status, new_status)
```

## 48. Database Monitoring & Alerting

**Định nghĩa chi tiết**: Continuous monitoring của database performance, health và security metrics với automated alerting khi thresholds are exceeded. Essential cho proactive database management.

**Use Case - Comprehensive Database Monitoring**:
```python
import boto3
import mysql.connector
from datetime import datetime, timedelta

class DatabaseMonitor:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        self.sns = boto3.client('sns')
        
    def collect_performance_metrics(self, db_connection):
        """Collect comprehensive performance metrics"""
        
        cursor = db_connection.cursor(dictionary=True)
        
        metrics = {}
        
        # Connection metrics
        cursor.execute("SHOW GLOBAL STATUS LIKE 'Threads_connected'")
        metrics['active_connections'] = int(cursor.fetchone()['Value'])
        
        cursor.execute("SHOW GLOBAL STATUS LIKE 'Max_used_connections'")
        metrics['max_connections_used'] = int(cursor.fetchone()['Value'])
        
        # Query performance
        cursor.execute("SHOW GLOBAL STATUS LIKE 'Queries'")
        total_queries = int(cursor.fetchone()['Value'])
        
        cursor.execute("SHOW GLOBAL STATUS LIKE 'Uptime'")
        uptime = int(cursor.fetchone()['Value'])
        
        metrics['queries_per_second'] = total_queries / uptime if uptime > 0 else 0
        
        # Buffer pool efficiency
        cursor.execute("SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read_requests'")
        read_requests = int(cursor.fetchone()['Value'])
        
        cursor.execute("SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads'")
        disk_reads = int(cursor.fetchone()['Value'])
        
        metrics['buffer_hit_ratio'] = ((read_requests - disk_reads) / read_requests * 100) if read_requests > 0 else 0
        
        # Slow queries
        cursor.execute("SHOW GLOBAL STATUS LIKE 'Slow_queries'")
        metrics['slow_queries'] = int(cursor.fetchone()['Value'])
        
        return metrics
    
    def check_alert_conditions(self, metrics):
        """Check metrics against alert thresholds"""
        
        alerts = []
        
        # High connection usage
        if metrics['active_connections'] > 80:
            alerts.append({
                'severity': 'WARNING',
                'metric': 'active_connections',
                'value': metrics['active_connections'],
                'threshold': 80,
                'message': 'High number of active database connections'
            })
        
        # Low buffer hit ratio
        if metrics['buffer_hit_ratio'] < 95:
            alerts.append({
                'severity': 'WARNING',
                'metric': 'buffer_hit_ratio',
                'value': metrics['buffer_hit_ratio'],
                'threshold': 95,
                'message': 'Low buffer pool hit ratio - consider increasing buffer pool size'
            })
        
        # High query rate
        if metrics['queries_per_second'] > 1000:
            alerts.append({
                'severity': 'INFO',
                'metric': 'queries_per_second',
                'value': metrics['queries_per_second'],
                'threshold': 1000,
                'message': 'High query rate detected'
            })
        
        return alerts
    
    def send_cloudwatch_metrics(self, metrics, db_instance_id):
        """Send custom metrics to CloudWatch"""
        
        metric_data = []
        
        for metric_name, value in metrics.items():
            metric_data.append({
                'MetricName': metric_name,
                'Value': value,
                'Unit': 'Count',
                'Dimensions': [
                    {
                        'Name': 'DBInstanceIdentifier',
                        'Value': db_instance_id
                    }
                ],
                'Timestamp': datetime.utcnow()
            })
        
        # Send metrics in batches (CloudWatch limit: 20 metrics per call)
        for i in range(0, len(metric_data), 20):
            batch = metric_data[i:i+20]
            
            self.cloudwatch.put_metric_data(
                Namespace='Custom/Database',
                MetricData=batch
            )
    
    def create_cloudwatch_alarms(self, db_instance_id):
        """Create CloudWatch alarms for database monitoring"""
        
        alarms = [
            {
                'AlarmName': f'{db_instance_id}-HighConnections',
                'MetricName': 'DatabaseConnections',
                'Threshold': 80,
                'ComparisonOperator': 'GreaterThanThreshold'
            },
            {
                'AlarmName': f'{db_instance_id}-HighCPU',
                'MetricName': 'CPUUtilization',
                'Threshold': 80,
                'ComparisonOperator': 'GreaterThanThreshold'
            },
            {
                'AlarmName': f'{db_instance_id}-LowFreeSpace',
                'MetricName': 'FreeStorageSpace',
                'Threshold': 2000000000,  # 2GB in bytes
                'ComparisonOperator': 'LessThanThreshold'
            }
        ]
        
        for alarm_config in alarms:
            self.cloudwatch.put_metric_alarm(
                AlarmName=alarm_config['AlarmName'],
                ComparisonOperator=alarm_config['ComparisonOperator'],
                EvaluationPeriods=2,
                MetricName=alarm_config['MetricName'],
                Namespace='AWS/RDS',
                Period=300,
                Statistic='Average',
                Threshold=alarm_config['Threshold'],
                ActionsEnabled=True,
                AlarmActions=[
                    'arn:aws:sns:us-east-1:123456789012:database-alerts'
                ],
                AlarmDescription=f'Alarm for {alarm_config["MetricName"]} on {db_instance_id}',
                Dimensions=[
                    {
                        'Name': 'DBInstanceIdentifier',
                        'Value': db_instance_id
                    }
                ]
            )
```
