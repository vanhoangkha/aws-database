# Database Concepts - Part 4 (Concepts 23-35)

## 23. ETL (Extract, Transform, Load)

**Định nghĩa chi tiết**: ETL là quy trình tích hợp dữ liệu bao gồm Extract (trích xuất) dữ liệu từ multiple sources, Transform (biến đổi) theo business rules, và Load (tải) vào target system như data warehouse.

**ETL vs ELT**:
- **ETL**: Transform trước khi load (traditional approach)
- **ELT**: Load raw data trước, transform sau (modern cloud approach)

**Use Case - Retail Chain Data Integration**:
```python
import pandas as pd
import mysql.connector
import boto3
from datetime import datetime, timedelta
import json

class RetailETLPipeline:
    def __init__(self):
        # Source connections
        self.pos_db = mysql.connector.connect(
            host='pos-system.internal',
            database='retail_pos',
            user='etl_user',
            password='etl_password'
        )
        
        self.inventory_db = mysql.connector.connect(
            host='inventory-system.internal', 
            database='inventory',
            user='etl_user',
            password='etl_password'
        )
        
        # Target warehouse
        self.warehouse_db = mysql.connector.connect(
            host='warehouse.cluster-xyz.us-east-1.rds.amazonaws.com',
            database='retail_dw',
            user='warehouse_user',
            password='warehouse_password'
        )
        
        self.s3_client = boto3.client('s3')
    
    def extract_pos_data(self, start_date, end_date):
        """Extract sales data from POS systems"""
        query = """
        SELECT 
            t.transaction_id,
            t.store_id,
            t.register_id,
            t.cashier_id,
            t.customer_id,
            t.transaction_date,
            t.transaction_time,
            ti.product_id,
            ti.quantity,
            ti.unit_price,
            ti.discount_amount,
            ti.tax_amount,
            ti.line_total,
            p.product_name,
            p.category_id,
            p.brand_id
        FROM transactions t
        JOIN transaction_items ti ON t.transaction_id = ti.transaction_id
        JOIN products p ON ti.product_id = p.product_id
        WHERE t.transaction_date BETWEEN %s AND %s
          AND t.status = 'completed'
        """
        
        return pd.read_sql(query, self.pos_db, params=[start_date, end_date])
    
    def extract_inventory_data(self, start_date, end_date):
        """Extract inventory movements"""
        query = """
        SELECT 
            im.movement_id,
            im.product_id,
            im.store_id,
            im.movement_type,
            im.quantity_change,
            im.movement_date,
            im.reason_code,
            im.reference_number,
            p.current_stock,
            p.reorder_level,
            p.max_stock_level
        FROM inventory_movements im
        JOIN product_inventory p ON im.product_id = p.product_id 
                                 AND im.store_id = p.store_id
        WHERE im.movement_date BETWEEN %s AND %s
        """
        
        return pd.read_sql(query, self.inventory_db, params=[start_date, end_date])
    
    def extract_customer_data(self):
        """Extract customer master data"""
        query = """
        SELECT 
            customer_id,
            first_name,
            last_name,
            email,
            phone,
            birth_date,
            gender,
            registration_date,
            loyalty_tier,
            preferred_store_id,
            marketing_consent
        FROM customers
        WHERE status = 'active'
        """
        
        return pd.read_sql(query, self.pos_db)
    
    def transform_sales_data(self, pos_data):
        """Transform and enrich sales data"""
        # Data cleaning
        pos_data = pos_data.dropna(subset=['transaction_id', 'product_id'])
        pos_data = pos_data[pos_data['quantity'] > 0]
        pos_data = pos_data[pos_data['unit_price'] >= 0]
        
        # Create datetime column
        pos_data['transaction_datetime'] = pd.to_datetime(
            pos_data['transaction_date'].astype(str) + ' ' + 
            pos_data['transaction_time'].astype(str)
        )
        
        # Add time dimensions
        pos_data['year'] = pos_data['transaction_datetime'].dt.year
        pos_data['month'] = pos_data['transaction_datetime'].dt.month
        pos_data['quarter'] = pos_data['transaction_datetime'].dt.quarter
        pos_data['day_of_week'] = pos_data['transaction_datetime'].dt.dayofweek
        pos_data['hour'] = pos_data['transaction_datetime'].dt.hour
        pos_data['is_weekend'] = pos_data['day_of_week'].isin([5, 6])
        
        # Business calculations
        pos_data['gross_sales'] = pos_data['quantity'] * pos_data['unit_price']
        pos_data['net_sales'] = pos_data['gross_sales'] - pos_data['discount_amount']
        pos_data['total_sales'] = pos_data['net_sales'] + pos_data['tax_amount']
        
        # Profit margin calculation (simplified)
        pos_data['estimated_cost'] = pos_data['unit_price'] * 0.6  # 40% margin assumption
        pos_data['estimated_profit'] = pos_data['net_sales'] - (pos_data['quantity'] * pos_data['estimated_cost'])
        
        # Customer segmentation
        customer_stats = pos_data.groupby('customer_id').agg({
            'total_sales': 'sum',
            'transaction_id': 'nunique'
        }).reset_index()
        
        customer_stats['avg_transaction_value'] = customer_stats['total_sales'] / customer_stats['transaction_id']
        customer_stats['customer_segment'] = pd.cut(
            customer_stats['total_sales'],
            bins=[0, 1000000, 5000000, 20000000, float('inf')],
            labels=['Bronze', 'Silver', 'Gold', 'Platinum']
        )
        
        # Merge back customer segments
        pos_data = pos_data.merge(
            customer_stats[['customer_id', 'customer_segment']], 
            on='customer_id', 
            how='left'
        )
        
        return pos_data
    
    def transform_inventory_data(self, inventory_data):
        """Transform inventory data"""
        # Categorize movement types
        movement_mapping = {
            'SALE': 'Outbound',
            'RETURN': 'Inbound', 
            'RESTOCK': 'Inbound',
            'ADJUSTMENT': 'Adjustment',
            'TRANSFER_IN': 'Inbound',
            'TRANSFER_OUT': 'Outbound',
            'DAMAGED': 'Outbound',
            'EXPIRED': 'Outbound'
        }
        
        inventory_data['movement_category'] = inventory_data['movement_type'].map(movement_mapping)
        
        # Calculate stock levels after each movement
        inventory_data = inventory_data.sort_values(['product_id', 'store_id', 'movement_date'])
        inventory_data['running_stock'] = inventory_data.groupby(['product_id', 'store_id'])['quantity_change'].cumsum()
        
        # Flag potential stockouts
        inventory_data['potential_stockout'] = inventory_data['current_stock'] <= inventory_data['reorder_level']
        inventory_data['overstock'] = inventory_data['current_stock'] >= inventory_data['max_stock_level']
        
        return inventory_data
    
    def load_to_warehouse(self, transformed_data, table_name):
        """Load transformed data to warehouse"""
        cursor = self.warehouse_db.cursor()
        
        try:
            # Create staging table
            staging_table = f"{table_name}_staging"
            cursor.execute(f"DROP TABLE IF EXISTS {staging_table}")
            
            # Generate CREATE TABLE statement from DataFrame
            create_sql = self.generate_create_table_sql(transformed_data, staging_table)
            cursor.execute(create_sql)
            
            # Bulk insert data
            columns = list(transformed_data.columns)
            placeholders = ', '.join(['%s'] * len(columns))
            insert_sql = f"INSERT INTO {staging_table} ({', '.join(columns)}) VALUES ({placeholders})"
            
            data_tuples = [tuple(row) for row in transformed_data.values]
            cursor.executemany(insert_sql, data_tuples)
            
            # Merge staging to production table
            merge_sql = f"""
            INSERT INTO {table_name} 
            SELECT * FROM {staging_table}
            ON DUPLICATE KEY UPDATE
                quantity = VALUES(quantity),
                unit_price = VALUES(unit_price),
                updated_at = NOW()
            """
            cursor.execute(merge_sql)
            
            # Clean up staging
            cursor.execute(f"DROP TABLE {staging_table}")
            
            self.warehouse_db.commit()
            
        except Exception as e:
            self.warehouse_db.rollback()
            raise e
    
    def run_daily_etl(self, process_date):
        """Run complete ETL pipeline for a specific date"""
        print(f"Starting ETL for {process_date}")
        
        try:
            # Extract
            print("Extracting POS data...")
            pos_data = self.extract_pos_data(process_date, process_date)
            
            print("Extracting inventory data...")
            inventory_data = self.extract_inventory_data(process_date, process_date)
            
            print("Extracting customer data...")
            customer_data = self.extract_customer_data()
            
            # Transform
            print("Transforming sales data...")
            transformed_sales = self.transform_sales_data(pos_data)
            
            print("Transforming inventory data...")
            transformed_inventory = self.transform_inventory_data(inventory_data)
            
            # Load
            print("Loading to warehouse...")
            self.load_to_warehouse(transformed_sales, 'fact_sales')
            self.load_to_warehouse(transformed_inventory, 'fact_inventory_movements')
            self.load_to_warehouse(customer_data, 'dim_customers')
            
            # Data quality checks
            self.run_data_quality_checks(process_date)
            
            print(f"ETL completed successfully for {process_date}")
            
        except Exception as e:
            print(f"ETL failed for {process_date}: {str(e)}")
            self.send_alert(f"ETL Pipeline Failed", str(e))
            raise
    
    def run_data_quality_checks(self, process_date):
        """Run data quality validations"""
        cursor = self.warehouse_db.cursor()
        
        checks = [
            {
                'name': 'Sales Count Check',
                'query': f"""
                    SELECT COUNT(*) as record_count 
                    FROM fact_sales 
                    WHERE DATE(transaction_datetime) = '{process_date}'
                """,
                'expected_min': 1000  # Expect at least 1000 transactions per day
            },
            {
                'name': 'Revenue Range Check', 
                'query': f"""
                    SELECT SUM(total_sales) as daily_revenue
                    FROM fact_sales
                    WHERE DATE(transaction_datetime) = '{process_date}'
                """,
                'expected_min': 50000000  # Expect at least 50M revenue per day
            },
            {
                'name': 'Null Check',
                'query': f"""
                    SELECT COUNT(*) as null_count
                    FROM fact_sales
                    WHERE DATE(transaction_datetime) = '{process_date}'
                      AND (product_id IS NULL OR customer_id IS NULL)
                """,
                'expected_max': 0  # No null values allowed
            }
        ]
        
        for check in checks:
            cursor.execute(check['query'])
            result = cursor.fetchone()[0]
            
            if 'expected_min' in check and result < check['expected_min']:
                raise Exception(f"Data Quality Check Failed: {check['name']} - Got {result}, expected >= {check['expected_min']}")
            
            if 'expected_max' in check and result > check['expected_max']:
                raise Exception(f"Data Quality Check Failed: {check['name']} - Got {result}, expected <= {check['expected_max']}")
            
            print(f"✓ {check['name']}: {result}")

# AWS Glue ETL Job equivalent
glue_etl_script = """
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

args = getResolvedOptions(sys.argv, ['JOB_NAME', 'source_database', 'target_bucket'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Extract from RDS
source_df = glueContext.create_dynamic_frame.from_catalog(
    database=args['source_database'],
    table_name="transactions"
).toDF()

# Transform
transformed_df = source_df.filter(source_df.status == 'completed') \
    .withColumn('year', year(source_df.transaction_date)) \
    .withColumn('month', month(source_df.transaction_date)) \
    .withColumn('revenue', source_df.quantity * source_df.unit_price)

# Load to S3 as Parquet
transformed_df.write \
    .mode('overwrite') \
    .partitionBy('year', 'month') \
    .parquet(f"s3://{args['target_bucket']}/processed/transactions/")

job.commit()
"""
## 24. Data Lake vs Data Warehouse

**Định nghĩa chi tiết**: 
- **Data Lake**: Repository lưu trữ raw data ở native format, hỗ trợ structured, semi-structured và unstructured data với schema-on-read approach
- **Data Warehouse**: Structured repository với pre-defined schema, optimized cho analytical queries với schema-on-write approach

**Key Differences**:
| Aspect | Data Lake | Data Warehouse |
|--------|-----------|----------------|
| Schema | Schema-on-read | Schema-on-write |
| Data Types | All formats | Structured only |
| Processing | ELT | ETL |
| Cost | Lower storage cost | Higher storage cost |
| Flexibility | High | Lower |
| Performance | Variable | Optimized |

**Use Case - Media Company Architecture**:
```python
import boto3
import pandas as pd
from datetime import datetime
import json

class MediaDataArchitecture:
    def __init__(self):
        self.s3_client = boto3.client('s3')
        self.glue_client = boto3.client('glue')
        self.athena_client = boto3.client('athena')
        self.redshift_client = boto3.client('redshift-data')
        
        # Data Lake buckets
        self.raw_bucket = 'media-datalake-raw'
        self.processed_bucket = 'media-datalake-processed'
        self.curated_bucket = 'media-datalake-curated'
        
    def ingest_to_data_lake(self, data_source, data_type):
        """Ingest various data types to Data Lake"""
        
        if data_type == 'video_metadata':
            # Video files metadata
            video_data = {
                'video_id': 'vid_12345',
                'title': 'Sample Video',
                'duration_seconds': 3600,
                'resolution': '1920x1080',
                'file_size_mb': 2048,
                'upload_timestamp': datetime.now().isoformat(),
                'tags': ['entertainment', 'comedy', 'viral'],
                'thumbnail_url': 's3://media-assets/thumbnails/vid_12345.jpg',
                'transcription': {
                    'language': 'en',
                    'confidence': 0.95,
                    'segments': [
                        {'start': 0, 'end': 5, 'text': 'Welcome to our show'},
                        {'start': 5, 'end': 10, 'text': 'Today we will discuss...'}
                    ]
                },
                'analytics': {
                    'views': 150000,
                    'likes': 12000,
                    'comments': 850,
                    'shares': 2300
                }
            }
            
            # Store as JSON in Data Lake
            key = f"raw/video_metadata/year={datetime.now().year}/month={datetime.now().month:02d}/day={datetime.now().day:02d}/vid_12345.json"
            self.s3_client.put_object(
                Bucket=self.raw_bucket,
                Key=key,
                Body=json.dumps(video_data),
                ContentType='application/json'
            )
            
        elif data_type == 'user_interactions':
            # User behavior logs (semi-structured)
            interaction_logs = [
                {
                    'timestamp': '2024-01-20T10:30:00Z',
                    'user_id': 'user_789',
                    'session_id': 'sess_abc123',
                    'event_type': 'video_play',
                    'video_id': 'vid_12345',
                    'position_seconds': 0,
                    'device_info': {
                        'type': 'mobile',
                        'os': 'iOS',
                        'version': '17.2',
                        'screen_size': '414x896'
                    },
                    'location': {
                        'country': 'Vietnam',
                        'city': 'Ho Chi Minh City',
                        'ip_address': '203.162.xxx.xxx'
                    }
                }
            ]
            
            # Store as JSONL (JSON Lines)
            key = f"raw/user_interactions/year=2024/month=01/day=20/interactions_{datetime.now().strftime('%H%M%S')}.jsonl"
            jsonl_data = '\n'.join([json.dumps(log) for log in interaction_logs])
            
            self.s3_client.put_object(
                Bucket=self.raw_bucket,
                Key=key,
                Body=jsonl_data,
                ContentType='application/x-ndjson'
            )
            
        elif data_type == 'image_analysis':
            # AI/ML analysis results (unstructured to structured)
            image_analysis = {
                'image_id': 'img_67890',
                'analysis_timestamp': datetime.now().isoformat(),
                'detected_objects': [
                    {'object': 'person', 'confidence': 0.98, 'bbox': [100, 150, 200, 400]},
                    {'object': 'car', 'confidence': 0.87, 'bbox': [300, 200, 500, 350]}
                ],
                'facial_recognition': {
                    'faces_detected': 2,
                    'emotions': [
                        {'emotion': 'happy', 'confidence': 0.92},
                        {'emotion': 'surprised', 'confidence': 0.78}
                    ]
                },
                'text_extraction': {
                    'detected_text': 'SALE 50% OFF',
                    'language': 'en',
                    'confidence': 0.95
                },
                'content_moderation': {
                    'safe_for_work': True,
                    'violence_score': 0.02,
                    'adult_content_score': 0.01
                }
            }
            
            # Store in processed bucket (cleaned/enriched)
            key = f"processed/image_analysis/year=2024/month=01/img_67890_analysis.json"
            self.s3_client.put_object(
                Bucket=self.processed_bucket,
                Key=key,
                Body=json.dumps(image_analysis),
                ContentType='application/json'
            )
    
    def process_with_glue(self):
        """Process Data Lake data with AWS Glue"""
        glue_job_script = """
        import sys
        from awsglue.transforms import *
        from awsglue.utils import getResolvedOptions
        from pyspark.context import SparkContext
        from awsglue.context import GlueContext
        from pyspark.sql.functions import *
        from pyspark.sql.types import *
        
        # Read JSON data from Data Lake
        video_df = spark.read.json("s3://media-datalake-raw/video_metadata/")
        interactions_df = spark.read.json("s3://media-datalake-raw/user_interactions/")
        
        # Transform and aggregate
        video_stats = video_df.select(
            col("video_id"),
            col("title"),
            col("duration_seconds"),
            col("analytics.views").alias("total_views"),
            col("analytics.likes").alias("total_likes"),
            col("analytics.comments").alias("total_comments"),
            (col("analytics.likes") / col("analytics.views")).alias("engagement_rate")
        )
        
        # User behavior analysis
        user_behavior = interactions_df.groupBy("user_id", "video_id").agg(
            count("*").alias("interaction_count"),
            max("position_seconds").alias("max_watch_position"),
            first("device_info.type").alias("primary_device")
        )
        
        # Write to curated bucket as Parquet
        video_stats.write.mode("overwrite").parquet("s3://media-datalake-curated/video_analytics/")
        user_behavior.write.mode("overwrite").parquet("s3://media-datalake-curated/user_behavior/")
        """
        
        return glue_job_script
    
    def query_with_athena(self):
        """Query Data Lake using Athena"""
        
        # Create external table for Athena
        create_table_sql = """
        CREATE EXTERNAL TABLE IF NOT EXISTS video_analytics (
            video_id string,
            title string,
            duration_seconds bigint,
            total_views bigint,
            total_likes bigint,
            total_comments bigint,
            engagement_rate double
        )
        STORED AS PARQUET
        LOCATION 's3://media-datalake-curated/video_analytics/'
        """
        
        # Complex analytical query
        analytics_query = """
        SELECT 
            DATE_TRUNC('day', upload_date) as date,
            COUNT(*) as videos_uploaded,
            SUM(total_views) as daily_views,
            AVG(engagement_rate) as avg_engagement,
            PERCENTILE_APPROX(duration_seconds, 0.5) as median_duration,
            
            -- Top performing videos
            ARRAY_AGG(
                CASE WHEN total_views > 100000 
                THEN STRUCT(video_id, title, total_views) 
                END
            ) as viral_videos
            
        FROM video_analytics
        WHERE upload_date >= DATE('2024-01-01')
        GROUP BY DATE_TRUNC('day', upload_date)
        ORDER BY date DESC
        """
        
        return analytics_query
    
    def load_to_data_warehouse(self):
        """Load curated data to Redshift Data Warehouse"""
        
        # Redshift optimized schema
        warehouse_schema = """
        -- Fact table: Video performance
        CREATE TABLE fact_video_performance (
            video_id VARCHAR(50) NOT NULL,
            date_id INTEGER NOT NULL,
            views_count BIGINT,
            likes_count BIGINT,
            comments_count BIGINT,
            shares_count BIGINT,
            watch_time_minutes BIGINT,
            engagement_score DECIMAL(5,4),
            revenue_usd DECIMAL(12,2)
        )
        DISTSTYLE KEY
        DISTKEY (video_id)
        SORTKEY (date_id, video_id);
        
        -- Dimension table: Videos
        CREATE TABLE dim_videos (
            video_id VARCHAR(50) PRIMARY KEY,
            title VARCHAR(500),
            description TEXT,
            category VARCHAR(100),
            duration_seconds INTEGER,
            upload_date DATE,
            creator_id VARCHAR(50),
            content_rating VARCHAR(20)
        )
        DISTSTYLE ALL;
        
        -- Dimension table: Users
        CREATE TABLE dim_users (
            user_id VARCHAR(50) PRIMARY KEY,
            registration_date DATE,
            country VARCHAR(100),
            age_group VARCHAR(20),
            subscription_tier VARCHAR(50),
            total_videos_watched INTEGER,
            total_watch_time_hours DECIMAL(10,2)
        )
        DISTSTYLE ALL;
        """
        
        # ETL from Data Lake to Warehouse
        etl_sql = """
        -- Load from S3 to Redshift
        COPY fact_video_performance
        FROM 's3://media-datalake-curated/video_performance/'
        IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftS3AccessRole'
        FORMAT AS PARQUET;
        
        -- Aggregate for OLAP queries
        INSERT INTO video_daily_summary
        SELECT 
            date_id,
            COUNT(DISTINCT video_id) as videos_active,
            SUM(views_count) as total_views,
            SUM(watch_time_minutes) as total_watch_time,
            AVG(engagement_score) as avg_engagement
        FROM fact_video_performance
        GROUP BY date_id;
        """
        
        return warehouse_schema, etl_sql

# Usage comparison
media_arch = MediaDataArchitecture()

# Data Lake: Store everything, ask questions later
media_arch.ingest_to_data_lake('video_service', 'video_metadata')
media_arch.ingest_to_data_lake('mobile_app', 'user_interactions') 
media_arch.ingest_to_data_lake('ai_service', 'image_analysis')

# Data Warehouse: Structured, optimized for known queries
media_arch.load_to_data_warehouse()
```

## 25. CAP Theorem

**Định nghĩa chi tiết**: CAP Theorem (Brewer's Theorem) phát biểu rằng trong một distributed system, không thể đồng thời đảm bảo cả 3 thuộc tính: Consistency (nhất quán), Availability (sẵn sàng), và Partition tolerance (chịu lỗi phân vùng). Chỉ có thể chọn tối đa 2 trong 3.

**CAP Properties**:
- **Consistency**: Tất cả nodes thấy cùng data tại cùng thời điểm
- **Availability**: System luôn operational và responsive
- **Partition Tolerance**: System tiếp tục hoạt động dù có network failures

**CAP Trade-offs**:
- **CP Systems**: Ưu tiên Consistency + Partition tolerance (MongoDB, HBase)
- **AP Systems**: Ưu tiên Availability + Partition tolerance (DynamoDB, Cassandra)  
- **CA Systems**: Ưu tiên Consistency + Availability (Traditional RDBMS trong single node)

**Use Case - Distributed Social Media Platform**:
```python
import time
import random
from enum import Enum
from dataclasses import dataclass
from typing import Dict, List, Optional

class ConsistencyLevel(Enum):
    STRONG = "strong"
    EVENTUAL = "eventual"
    WEAK = "weak"

@dataclass
class Post:
    post_id: str
    user_id: str
    content: str
    timestamp: float
    likes: int = 0
    version: int = 1

class DistributedSocialMedia:
    def __init__(self):
        # Simulate multiple data centers
        self.data_centers = {
            'us-east': {'posts': {}, 'status': 'healthy', 'latency': 0.01},
            'us-west': {'posts': {}, 'status': 'healthy', 'latency': 0.05}, 
            'europe': {'posts': {}, 'status': 'healthy', 'latency': 0.1},
            'asia': {'posts': {}, 'status': 'healthy', 'latency': 0.15}
        }
        self.replication_lag = 0.1  # 100ms replication lag
        
    def simulate_network_partition(self, dc1: str, dc2: str):
        """Simulate network partition between data centers"""
        print(f"🔥 Network partition between {dc1} and {dc2}")
        self.data_centers[dc1]['partitioned_from'] = dc2
        self.data_centers[dc2]['partitioned_from'] = dc1
    
    def heal_partition(self, dc1: str, dc2: str):
        """Heal network partition"""
        print(f"✅ Network partition healed between {dc1} and {dc2}")
        self.data_centers[dc1].pop('partitioned_from', None)
        self.data_centers[dc2].pop('partitioned_from', None)
    
    def write_post_cp_system(self, post: Post, primary_dc: str) -> bool:
        """CP System: Prioritize Consistency + Partition tolerance"""
        print(f"\n📝 CP System: Writing post {post.post_id} to {primary_dc}")
        
        # Check if we can reach majority of nodes
        healthy_dcs = [dc for dc, info in self.data_centers.items() 
                      if info['status'] == 'healthy' and 
                      'partitioned_from' not in info]
        
        if len(healthy_dcs) < len(self.data_centers) // 2 + 1:
            print("❌ Cannot achieve majority consensus - Write REJECTED")
            return False
        
        # Write to majority of nodes synchronously
        successful_writes = 0
        for dc in healthy_dcs[:3]:  # Write to 3 nodes for majority
            try:
                time.sleep(self.data_centers[dc]['latency'])  # Simulate network latency
                self.data_centers[dc]['posts'][post.post_id] = post
                successful_writes += 1
                print(f"✅ Written to {dc}")
            except Exception as e:
                print(f"❌ Failed to write to {dc}: {e}")
        
        if successful_writes >= 2:  # Majority achieved
            print(f"✅ Post written successfully to {successful_writes} nodes")
            return True
        else:
            print("❌ Failed to achieve majority - Rolling back")
            # Rollback writes
            for dc in healthy_dcs:
                self.data_centers[dc]['posts'].pop(post.post_id, None)
            return False
    
    def write_post_ap_system(self, post: Post, preferred_dc: str) -> bool:
        """AP System: Prioritize Availability + Partition tolerance"""
        print(f"\n📝 AP System: Writing post {post.post_id} to {preferred_dc}")
        
        # Always accept writes to available nodes
        written_dcs = []
        
        for dc, info in self.data_centers.items():
            if info['status'] == 'healthy':
                try:
                    # Write immediately to available nodes
                    self.data_centers[dc]['posts'][post.post_id] = post
                    written_dcs.append(dc)
                    print(f"✅ Written to {dc}")
                except Exception as e:
                    print(f"❌ Failed to write to {dc}: {e}")
        
        if written_dcs:
            print(f"✅ Post written to {len(written_dcs)} available nodes")
            # Asynchronous replication will eventually sync
            self.schedule_async_replication(post, written_dcs)
            return True
        else:
            print("❌ No nodes available")
            return False
    
    def read_post_strong_consistency(self, post_id: str, reader_dc: str) -> Optional[Post]:
        """Strong consistency read - must read from majority"""
        print(f"\n📖 Strong consistency read: {post_id} from {reader_dc}")
        
        # Read from majority of nodes to ensure latest version
        versions = {}
        for dc, info in self.data_centers.items():
            if info['status'] == 'healthy' and 'partitioned_from' not in info:
                post = info['posts'].get(post_id)
                if post:
                    versions[dc] = post
        
        if len(versions) >= 2:  # Majority has the data
            # Return latest version
            latest_post = max(versions.values(), key=lambda p: p.version)
            print(f"✅ Read latest version (v{latest_post.version}) from majority")
            return latest_post
        else:
            print("❌ Cannot read from majority - Data may be inconsistent")
            return None
    
    def read_post_eventual_consistency(self, post_id: str, reader_dc: str) -> Optional[Post]:
        """Eventual consistency read - read from nearest available node"""
        print(f"\n📖 Eventual consistency read: {post_id} from {reader_dc}")
        
        # Try to read from nearest DC first
        if (self.data_centers[reader_dc]['status'] == 'healthy' and 
            post_id in self.data_centers[reader_dc]['posts']):
            post = self.data_centers[reader_dc]['posts'][post_id]
            print(f"✅ Read from local DC {reader_dc} (v{post.version})")
            return post
        
        # Fallback to any available DC
        for dc, info in self.data_centers.items():
            if (info['status'] == 'healthy' and 
                post_id in info['posts']):
                post = info['posts'][post_id]
                print(f"✅ Read from fallback DC {dc} (v{post.version}) - may be stale")
                return post
        
        print("❌ Post not found in any available DC")
        return None
    
    def schedule_async_replication(self, post: Post, source_dcs: List[str]):
        """Simulate asynchronous replication for AP systems"""
        print(f"🔄 Scheduling async replication for post {post.post_id}")
        
        # In real system, this would be handled by background processes
        time.sleep(self.replication_lag)
        
        for dc in self.data_centers:
            if dc not in source_dcs and self.data_centers[dc]['status'] == 'healthy':
                # Check for conflicts and resolve
                existing_post = self.data_centers[dc]['posts'].get(post.post_id)
                if existing_post and existing_post.version > post.version:
                    print(f"⚠️  Conflict detected in {dc} - keeping newer version")
                else:
                    self.data_centers[dc]['posts'][post.post_id] = post
                    print(f"🔄 Replicated to {dc}")
    
    def demonstrate_cap_scenarios(self):
        """Demonstrate different CAP theorem scenarios"""
        
        # Create test post
        post = Post(
            post_id="post_123",
            user_id="user_456", 
            content="Hello World! #CAP",
            timestamp=time.time()
        )
        
        print("=" * 60)
        print("CAP THEOREM DEMONSTRATION")
        print("=" * 60)
        
        # Scenario 1: Normal operation (all properties work)
        print("\n🟢 SCENARIO 1: Normal Operation")
        print("All data centers healthy, no network issues")
        
        success = self.write_post_cp_system(post, 'us-east')
        if success:
            read_post = self.read_post_strong_consistency('post_123', 'europe')
        
        # Scenario 2: Network partition (CP choice)
        print("\n🟡 SCENARIO 2: Network Partition - CP System Choice")
        self.simulate_network_partition('us-east', 'europe')
        
        # CP system will reject writes if can't achieve majority
        post2 = Post("post_456", "user_789", "CP System Test", time.time())
        success = self.write_post_cp_system(post2, 'us-east')
        
        # Scenario 3: Network partition (AP choice)  
        print("\n🟠 SCENARIO 3: Network Partition - AP System Choice")
        
        # AP system will accept writes even during partition
        post3 = Post("post_789", "user_123", "AP System Test", time.time())
        success = self.write_post_ap_system(post3, 'us-west')
        
        # Read with eventual consistency
        read_post = self.read_post_eventual_consistency('post_789', 'asia')
        
        # Scenario 4: Partition healing
        print("\n🟢 SCENARIO 4: Partition Healing")
        self.heal_partition('us-east', 'europe')
        
        # Now CP system can work normally again
        post4 = Post("post_999", "user_555", "Healed Network", time.time())
        success = self.write_post_cp_system(post4, 'us-east')

# AWS Services CAP Classification
aws_services_cap = {
    'DynamoDB': {
        'type': 'AP',
        'description': 'Prioritizes Availability and Partition tolerance',
        'consistency': 'Eventual consistency by default, strong consistency available',
        'use_case': 'High-scale applications, gaming leaderboards, IoT'
    },
    'Aurora': {
        'type': 'CP', 
        'description': 'Prioritizes Consistency and Partition tolerance',
        'consistency': 'Strong consistency within region',
        'use_case': 'OLTP applications, financial systems'
    },
    'DocumentDB': {
        'type': 'CP',
        'description': 'MongoDB-compatible, prioritizes consistency',
        'consistency': 'Strong consistency with configurable read preferences',
        'use_case': 'Content management, catalogs'
    },
    'ElastiCache': {
        'type': 'AP',
        'description': 'In-memory cache, prioritizes availability',
        'consistency': 'Eventual consistency in cluster mode',
        'use_case': 'Session storage, real-time analytics'
    }
}

# Run demonstration
demo = DistributedSocialMedia()
demo.demonstrate_cap_scenarios()

print("\n" + "=" * 60)
print("AWS SERVICES CAP CLASSIFICATION")
print("=" * 60)

for service, props in aws_services_cap.items():
    print(f"\n{service} ({props['type']} System):")
    print(f"  Description: {props['description']}")
    print(f"  Consistency: {props['consistency']}")
    print(f"  Use Case: {props['use_case']}")
```
