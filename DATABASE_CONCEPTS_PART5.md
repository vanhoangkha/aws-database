# Database Concepts - Part 5 (Concepts 26-40)

## 26. Sharding

**Định nghĩa chi tiết**: Sharding là kỹ thuật horizontal partitioning, chia dữ liệu thành nhiều databases độc lập (shards) dựa trên shard key. Mỗi shard chứa subset của data và có thể deploy trên different servers để scale horizontally.

**Sharding Strategies**:
- **Range-based**: Chia theo khoảng giá trị (A-M, N-Z)
- **Hash-based**: Sử dụng hash function của shard key
- **Directory-based**: Lookup service để map keys to shards
- **Geographic**: Chia theo location/region

**Use Case - Global Social Media Platform (1 billion users)**:
```python
import hashlib
import mysql.connector
from typing import Dict, List, Optional

class ShardedSocialMedia:
    def __init__(self):
        # 16 shards for horizontal scaling
        self.shards = {
            'shard_00': {'host': 'shard00.us-east.internal', 'users': 'A-C'},
            'shard_01': {'host': 'shard01.us-east.internal', 'users': 'D-F'},
            'shard_02': {'host': 'shard02.us-west.internal', 'users': 'G-I'},
            'shard_03': {'host': 'shard03.us-west.internal', 'users': 'J-L'},
            'shard_04': {'host': 'shard04.eu-west.internal', 'users': 'M-O'},
            'shard_05': {'host': 'shard05.eu-west.internal', 'users': 'P-R'},
            'shard_06': {'host': 'shard06.asia.internal', 'users': 'S-U'},
            'shard_07': {'host': 'shard07.asia.internal', 'users': 'V-Z'},
            # Additional shards for load distribution
            'shard_08': {'host': 'shard08.us-east.internal', 'users': 'hash_0-1'},
            'shard_09': {'host': 'shard09.us-east.internal', 'users': 'hash_2-3'},
            'shard_10': {'host': 'shard10.us-west.internal', 'users': 'hash_4-5'},
            'shard_11': {'host': 'shard11.us-west.internal', 'users': 'hash_6-7'},
            'shard_12': {'host': 'shard12.eu-west.internal', 'users': 'hash_8-9'},
            'shard_13': {'host': 'shard13.eu-west.internal', 'users': 'hash_A-B'},
            'shard_14': {'host': 'shard14.asia.internal', 'users': 'hash_C-D'},
            'shard_15': {'host': 'shard15.asia.internal', 'users': 'hash_E-F'}
        }
        
        self.connections = {}
        self.shard_map = {}  # Cache for user_id -> shard mapping
    
    def get_shard_by_user_id(self, user_id: str) -> str:
        """Determine shard based on user_id using consistent hashing"""
        if user_id in self.shard_map:
            return self.shard_map[user_id]
        
        # Hash-based sharding for even distribution
        hash_value = hashlib.md5(user_id.encode()).hexdigest()
        shard_index = int(hash_value[0], 16)  # Use first hex digit (0-F)
        shard_name = f"shard_{shard_index:02d}"
        
        # Cache the mapping
        self.shard_map[user_id] = shard_name
        return shard_name
    
    def get_connection(self, shard_name: str):
        """Get database connection for specific shard"""
        if shard_name not in self.connections:
            shard_info = self.shards[shard_name]
            self.connections[shard_name] = mysql.connector.connect(
                host=shard_info['host'],
                database='social_media',
                user='app_user',
                password='app_password',
                pool_size=20
            )
        return self.connections[shard_name]
    
    def create_user(self, user_id: str, username: str, email: str, location: str):
        """Create user in appropriate shard"""
        shard_name = self.get_shard_by_user_id(user_id)
        conn = self.get_connection(shard_name)
        cursor = conn.cursor()
        
        try:
            cursor.execute("""
                INSERT INTO users (user_id, username, email, location, shard_id, created_at)
                VALUES (%s, %s, %s, %s, %s, NOW())
            """, (user_id, username, email, location, shard_name))
            
            conn.commit()
            print(f"✅ User {user_id} created in {shard_name}")
            
        except Exception as e:
            conn.rollback()
            print(f"❌ Failed to create user {user_id}: {e}")
            raise
    
    def create_post(self, post_id: str, user_id: str, content: str):
        """Create post in same shard as user"""
        shard_name = self.get_shard_by_user_id(user_id)
        conn = self.get_connection(shard_name)
        cursor = conn.cursor()
        
        try:
            cursor.execute("""
                INSERT INTO posts (post_id, user_id, content, created_at, shard_id)
                VALUES (%s, %s, %s, NOW(), %s)
            """, (post_id, user_id, content, shard_name))
            
            conn.commit()
            print(f"✅ Post {post_id} created in {shard_name}")
            
        except Exception as e:
            conn.rollback()
            raise
    
    def follow_user(self, follower_id: str, following_id: str):
        """Handle cross-shard relationships"""
        follower_shard = self.get_shard_by_user_id(follower_id)
        following_shard = self.get_shard_by_user_id(following_id)
        
        # Store relationship in both shards for efficient queries
        try:
            # Store in follower's shard
            conn1 = self.get_connection(follower_shard)
            cursor1 = conn1.cursor()
            cursor1.execute("""
                INSERT INTO user_follows (follower_id, following_id, created_at, relationship_type)
                VALUES (%s, %s, NOW(), 'outgoing')
            """, (follower_id, following_id))
            conn1.commit()
            
            # Store in following user's shard (if different)
            if follower_shard != following_shard:
                conn2 = self.get_connection(following_shard)
                cursor2 = conn2.cursor()
                cursor2.execute("""
                    INSERT INTO user_follows (follower_id, following_id, created_at, relationship_type)
                    VALUES (%s, %s, NOW(), 'incoming')
                """, (follower_id, following_id))
                conn2.commit()
            
            print(f"✅ Follow relationship created: {follower_id} -> {following_id}")
            
        except Exception as e:
            print(f"❌ Failed to create follow relationship: {e}")
            raise
    
    def get_user_feed(self, user_id: str, limit: int = 20) -> List[Dict]:
        """Get user's feed - requires cross-shard queries"""
        user_shard = self.get_shard_by_user_id(user_id)
        
        # Step 1: Get users that this user follows
        conn = self.get_connection(user_shard)
        cursor = conn.cursor(dictionary=True)
        
        cursor.execute("""
            SELECT following_id 
            FROM user_follows 
            WHERE follower_id = %s AND relationship_type = 'outgoing'
            LIMIT 1000
        """, (user_id,))
        
        following_users = [row['following_id'] for row in cursor.fetchall()]
        
        # Step 2: Group following users by their shards
        shard_users = {}
        for following_user in following_users:
            shard = self.get_shard_by_user_id(following_user)
            if shard not in shard_users:
                shard_users[shard] = []
            shard_users[shard].append(following_user)
        
        # Step 3: Query each shard for posts
        all_posts = []
        for shard_name, users in shard_users.items():
            shard_conn = self.get_connection(shard_name)
            shard_cursor = shard_conn.cursor(dictionary=True)
            
            placeholders = ','.join(['%s'] * len(users))
            shard_cursor.execute(f"""
                SELECT p.post_id, p.user_id, p.content, p.created_at,
                       u.username, u.profile_image_url
                FROM posts p
                JOIN users u ON p.user_id = u.user_id
                WHERE p.user_id IN ({placeholders})
                ORDER BY p.created_at DESC
                LIMIT 100
            """, users)
            
            all_posts.extend(shard_cursor.fetchall())
        
        # Step 4: Sort and limit results
        all_posts.sort(key=lambda x: x['created_at'], reverse=True)
        return all_posts[:limit]
    
    def get_global_trending(self, limit: int = 10) -> List[Dict]:
        """Get trending posts across all shards"""
        trending_posts = []
        
        # Query each shard for trending content
        for shard_name in self.shards.keys():
            try:
                conn = self.get_connection(shard_name)
                cursor = conn.cursor(dictionary=True)
                
                cursor.execute("""
                    SELECT p.post_id, p.user_id, p.content, p.created_at,
                           COUNT(l.like_id) as like_count,
                           COUNT(c.comment_id) as comment_count,
                           u.username
                    FROM posts p
                    LEFT JOIN likes l ON p.post_id = l.post_id
                    LEFT JOIN comments c ON p.post_id = c.post_id
                    JOIN users u ON p.user_id = u.user_id
                    WHERE p.created_at >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
                    GROUP BY p.post_id
                    HAVING like_count > 100
                    ORDER BY (like_count + comment_count * 2) DESC
                    LIMIT 20
                """)
                
                shard_trending = cursor.fetchall()
                for post in shard_trending:
                    post['shard'] = shard_name
                    post['engagement_score'] = post['like_count'] + post['comment_count'] * 2
                
                trending_posts.extend(shard_trending)
                
            except Exception as e:
                print(f"⚠️  Failed to query {shard_name}: {e}")
                continue
        
        # Sort by engagement and return top posts
        trending_posts.sort(key=lambda x: x['engagement_score'], reverse=True)
        return trending_posts[:limit]
    
    def rebalance_shards(self, new_shard_count: int):
        """Rebalance data when adding new shards"""
        print(f"🔄 Rebalancing from {len(self.shards)} to {new_shard_count} shards")
        
        # This is a simplified version - real implementation would be more complex
        migration_plan = {}
        
        for user_id in self.shard_map.keys():
            old_shard = self.shard_map[user_id]
            
            # Recalculate shard with new shard count
            hash_value = hashlib.md5(user_id.encode()).hexdigest()
            new_shard_index = int(hash_value[0], 16) % new_shard_count
            new_shard = f"shard_{new_shard_index:02d}"
            
            if old_shard != new_shard:
                if new_shard not in migration_plan:
                    migration_plan[new_shard] = []
                migration_plan[new_shard].append({
                    'user_id': user_id,
                    'from_shard': old_shard,
                    'to_shard': new_shard
                })
        
        print(f"📊 Migration plan: {len(migration_plan)} shards need data migration")
        return migration_plan

# DynamoDB Auto-Sharding Example
dynamodb_sharding_example = """
# DynamoDB handles sharding automatically
import boto3

dynamodb = boto3.resource('dynamodb')

# Table with partition key for automatic sharding
table = dynamodb.Table('SocialMediaPosts')

# DynamoDB automatically distributes data based on partition key
def create_post_dynamodb(user_id, post_id, content):
    table.put_item(
        Item={
            'user_id': user_id,        # Partition key - DynamoDB shards on this
            'post_id': post_id,        # Sort key
            'content': content,
            'timestamp': int(time.time()),
            'likes': 0,
            'comments': 0
        }
    )

# Query posts for a user (single partition)
def get_user_posts(user_id):
    response = table.query(
        KeyConditionExpression=Key('user_id').eq(user_id)
    )
    return response['Items']

# Global Secondary Index for cross-partition queries
def get_recent_posts():
    response = table.scan(
        IndexName='TimestampIndex',
        FilterExpression=Attr('timestamp').gt(int(time.time()) - 86400)
    )
    return response['Items']
"""

# Usage demonstration
sharded_platform = ShardedSocialMedia()

# Create users across different shards
users = [
    ('user_001', 'alice_nguyen', 'alice@example.com', 'Vietnam'),
    ('user_002', 'bob_tran', 'bob@example.com', 'Singapore'), 
    ('user_003', 'charlie_le', 'charlie@example.com', 'Thailand'),
    ('user_004', 'diana_pham', 'diana@example.com', 'Malaysia')
]

for user_data in users:
    sharded_platform.create_user(*user_data)

# Create posts
sharded_platform.create_post('post_001', 'user_001', 'Hello from Vietnam! 🇻🇳')
sharded_platform.create_post('post_002', 'user_002', 'Singapore skyline is amazing! 🏙️')

# Create relationships
sharded_platform.follow_user('user_001', 'user_002')
sharded_platform.follow_user('user_001', 'user_003')

# Get user feed (cross-shard query)
feed = sharded_platform.get_user_feed('user_001')
print(f"📱 User feed has {len(feed)} posts")

# Get trending content (all shards)
trending = sharded_platform.get_global_trending()
print(f"🔥 Found {len(trending)} trending posts")
```

## 27. Metadata & Data Dictionary

**Định nghĩa chi tiết**: Metadata là "data about data" - thông tin mô tả cấu trúc, meaning, relationships và characteristics của dữ liệu. Data Dictionary là centralized repository chứa metadata definitions, business rules và data lineage.

**Types of Metadata**:
- **Structural**: Schema, tables, columns, data types
- **Descriptive**: Business definitions, descriptions, examples  
- **Administrative**: Ownership, access permissions, retention policies
- **Operational**: Data quality metrics, usage statistics, lineage

**Use Case - Enterprise Data Governance Platform**:
```python
import json
from datetime import datetime
from typing import Dict, List, Optional
from dataclasses import dataclass, asdict

@dataclass
class ColumnMetadata:
    column_name: str
    data_type: str
    is_nullable: bool
    default_value: Optional[str]
    description: str
    business_name: str
    data_classification: str  # PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED
    pii_flag: bool
    data_quality_rules: List[str]
    sample_values: List[str]
    created_date: str
    last_updated: str
    owner: str

@dataclass 
class TableMetadata:
    database_name: str
    schema_name: str
    table_name: str
    table_type: str  # TABLE, VIEW, MATERIALIZED_VIEW
    description: str
    business_purpose: str
    data_source: str
    update_frequency: str
    retention_period: str
    columns: List[ColumnMetadata]
    relationships: List[Dict]
    data_lineage: List[Dict]
    access_permissions: Dict
    quality_metrics: Dict
    usage_statistics: Dict

class DataDictionary:
    def __init__(self):
        self.metadata_store = {}
        self.business_glossary = {}
        self.data_lineage_graph = {}
        
    def register_table(self, table_metadata: TableMetadata):
        """Register table metadata in data dictionary"""
        table_key = f"{table_metadata.database_name}.{table_metadata.schema_name}.{table_metadata.table_name}"
        
        # Validate metadata completeness
        self._validate_metadata(table_metadata)
        
        # Store metadata
        self.metadata_store[table_key] = table_metadata
        
        # Update business glossary
        self._update_business_glossary(table_metadata)
        
        # Update data lineage
        self._update_data_lineage(table_metadata)
        
        print(f"✅ Registered table: {table_key}")
    
    def _validate_metadata(self, table_metadata: TableMetadata):
        """Validate metadata completeness and quality"""
        required_fields = ['description', 'business_purpose', 'data_source', 'owner']
        
        for field in required_fields:
            if not getattr(table_metadata, field):
                raise ValueError(f"Required field '{field}' is missing")
        
        # Validate PII classification
        for column in table_metadata.columns:
            if column.pii_flag and column.data_classification not in ['CONFIDENTIAL', 'RESTRICTED']:
                raise ValueError(f"PII column '{column.column_name}' must be CONFIDENTIAL or RESTRICTED")
    
    def _update_business_glossary(self, table_metadata: TableMetadata):
        """Update business glossary with new terms"""
        for column in table_metadata.columns:
            if column.business_name not in self.business_glossary:
                self.business_glossary[column.business_name] = {
                    'technical_name': column.column_name,
                    'definition': column.description,
                    'data_type': column.data_type,
                    'usage_examples': column.sample_values,
                    'related_terms': [],
                    'tables_used_in': []
                }
            
            # Add table reference
            table_ref = f"{table_metadata.database_name}.{table_metadata.schema_name}.{table_metadata.table_name}"
            if table_ref not in self.business_glossary[column.business_name]['tables_used_in']:
                self.business_glossary[column.business_name]['tables_used_in'].append(table_ref)
    
    def _update_data_lineage(self, table_metadata: TableMetadata):
        """Update data lineage graph"""
        table_key = f"{table_metadata.database_name}.{table_metadata.schema_name}.{table_metadata.table_name}"
        
        self.data_lineage_graph[table_key] = {
            'upstream_sources': [],
            'downstream_targets': [],
            'transformations': []
        }
        
        # Process lineage information
        for lineage_item in table_metadata.data_lineage:
            if lineage_item['type'] == 'source':
                self.data_lineage_graph[table_key]['upstream_sources'].append(lineage_item)
            elif lineage_item['type'] == 'target':
                self.data_lineage_graph[table_key]['downstream_targets'].append(lineage_item)
    
    def search_metadata(self, search_term: str, search_type: str = 'all') -> List[Dict]:
        """Search metadata by term"""
        results = []
        
        for table_key, metadata in self.metadata_store.items():
            if search_type in ['all', 'table']:
                # Search in table metadata
                if (search_term.lower() in metadata.description.lower() or
                    search_term.lower() in metadata.business_purpose.lower() or
                    search_term.lower() in table_key.lower()):
                    
                    results.append({
                        'type': 'table',
                        'table': table_key,
                        'description': metadata.description,
                        'business_purpose': metadata.business_purpose
                    })
            
            if search_type in ['all', 'column']:
                # Search in column metadata
                for column in metadata.columns:
                    if (search_term.lower() in column.column_name.lower() or
                        search_term.lower() in column.description.lower() or
                        search_term.lower() in column.business_name.lower()):
                        
                        results.append({
                            'type': 'column',
                            'table': table_key,
                            'column': column.column_name,
                            'business_name': column.business_name,
                            'description': column.description,
                            'data_type': column.data_type
                        })
        
        return results
    
    def get_data_lineage(self, table_name: str, direction: str = 'both') -> Dict:
        """Get data lineage for a table"""
        if table_name not in self.data_lineage_graph:
            return {'error': f'Table {table_name} not found'}
        
        lineage = self.data_lineage_graph[table_name]
        
        if direction == 'upstream':
            return {'upstream_sources': lineage['upstream_sources']}
        elif direction == 'downstream':
            return {'downstream_targets': lineage['downstream_targets']}
        else:
            return lineage
    
    def generate_data_catalog(self) -> Dict:
        """Generate comprehensive data catalog"""
        catalog = {
            'metadata_summary': {
                'total_tables': len(self.metadata_store),
                'total_columns': sum(len(meta.columns) for meta in self.metadata_store.values()),
                'pii_columns': sum(1 for meta in self.metadata_store.values() 
                                for col in meta.columns if col.pii_flag),
                'business_terms': len(self.business_glossary)
            },
            'tables_by_database': {},
            'data_classification_summary': {},
            'business_glossary': self.business_glossary,
            'data_quality_issues': []
        }
        
        # Group tables by database
        for table_key, metadata in self.metadata_store.items():
            db_name = metadata.database_name
            if db_name not in catalog['tables_by_database']:
                catalog['tables_by_database'][db_name] = []
            
            catalog['tables_by_database'][db_name].append({
                'table_name': f"{metadata.schema_name}.{metadata.table_name}",
                'description': metadata.description,
                'column_count': len(metadata.columns),
                'last_updated': metadata.columns[0].last_updated if metadata.columns else None
            })
        
        # Data classification summary
        classification_counts = {}
        for metadata in self.metadata_store.values():
            for column in metadata.columns:
                classification = column.data_classification
                classification_counts[classification] = classification_counts.get(classification, 0) + 1
        
        catalog['data_classification_summary'] = classification_counts
        
        return catalog

# AWS Glue Data Catalog Integration
class GlueDataCatalogIntegration:
    def __init__(self):
        import boto3
        self.glue_client = boto3.client('glue')
        self.data_dict = DataDictionary()
    
    def sync_from_glue_catalog(self, database_name: str):
        """Sync metadata from AWS Glue Data Catalog"""
        
        # Get tables from Glue Catalog
        response = self.glue_client.get_tables(DatabaseName=database_name)
        
        for table_info in response['TableList']:
            # Convert Glue metadata to our format
            columns = []
            for col in table_info.get('StorageDescriptor', {}).get('Columns', []):
                column_meta = ColumnMetadata(
                    column_name=col['Name'],
                    data_type=col['Type'],
                    is_nullable=True,  # Default, would need additional logic
                    default_value=None,
                    description=col.get('Comment', ''),
                    business_name=col['Name'],  # Would map from business glossary
                    data_classification='INTERNAL',  # Default, would need classification rules
                    pii_flag=self._detect_pii(col['Name']),
                    data_quality_rules=[],
                    sample_values=[],
                    created_date=datetime.now().isoformat(),
                    last_updated=datetime.now().isoformat(),
                    owner='system'
                )
                columns.append(column_meta)
            
            # Create table metadata
            table_meta = TableMetadata(
                database_name=database_name,
                schema_name='default',
                table_name=table_info['Name'],
                table_type='TABLE',
                description=table_info.get('Description', ''),
                business_purpose='',  # Would need to be populated
                data_source=table_info.get('StorageDescriptor', {}).get('Location', ''),
                update_frequency='unknown',
                retention_period='unknown',
                columns=columns,
                relationships=[],
                data_lineage=[],
                access_permissions={},
                quality_metrics={},
                usage_statistics={}
            )
            
            self.data_dict.register_table(table_meta)
    
    def _detect_pii(self, column_name: str) -> bool:
        """Simple PII detection based on column name patterns"""
        pii_patterns = ['email', 'phone', 'ssn', 'credit_card', 'passport', 'address']
        return any(pattern in column_name.lower() for pattern in pii_patterns)

# Example usage
data_dict = DataDictionary()

# Register customer table metadata
customer_columns = [
    ColumnMetadata(
        column_name='customer_id',
        data_type='BIGINT',
        is_nullable=False,
        default_value=None,
        description='Unique identifier for customer',
        business_name='Customer Identifier',
        data_classification='INTERNAL',
        pii_flag=False,
        data_quality_rules=['NOT NULL', 'UNIQUE'],
        sample_values=['12345', '67890', '11111'],
        created_date='2024-01-01',
        last_updated='2024-01-20',
        owner='data_team@company.com'
    ),
    ColumnMetadata(
        column_name='email_address',
        data_type='VARCHAR(255)',
        is_nullable=False,
        default_value=None,
        description='Customer email address for communication',
        business_name='Email Address',
        data_classification='CONFIDENTIAL',
        pii_flag=True,
        data_quality_rules=['NOT NULL', 'EMAIL_FORMAT', 'UNIQUE'],
        sample_values=['john@example.com', 'jane@company.com'],
        created_date='2024-01-01',
        last_updated='2024-01-20',
        owner='data_team@company.com'
    )
]

customer_table = TableMetadata(
    database_name='ecommerce',
    schema_name='customer_data',
    table_name='customers',
    table_type='TABLE',
    description='Master table containing customer information',
    business_purpose='Store and manage customer profiles for e-commerce platform',
    data_source='Customer registration system',
    update_frequency='Real-time',
    retention_period='7 years',
    columns=customer_columns,
    relationships=[
        {'type': 'one_to_many', 'target_table': 'orders', 'foreign_key': 'customer_id'}
    ],
    data_lineage=[
        {'type': 'source', 'system': 'CRM', 'table': 'crm.contacts'},
        {'type': 'target', 'system': 'Analytics', 'table': 'warehouse.dim_customers'}
    ],
    access_permissions={
        'read': ['analytics_team', 'customer_service'],
        'write': ['customer_service'],
        'admin': ['data_team']
    },
    quality_metrics={
        'completeness': 0.98,
        'accuracy': 0.95,
        'consistency': 0.97
    },
    usage_statistics={
        'daily_queries': 1500,
        'top_users': ['analytics_dashboard', 'customer_service_app']
    }
)

data_dict.register_table(customer_table)

# Search metadata
search_results = data_dict.search_metadata('email')
print(f"Found {len(search_results)} results for 'email'")

# Generate data catalog
catalog = data_dict.generate_data_catalog()
print(f"Data Catalog: {catalog['metadata_summary']}")
```
## 28. Data Integrity & Constraints

**Định nghĩa chi tiết**: Data Integrity đảm bảo accuracy, consistency và reliability của dữ liệu thông qua constraints, validation rules và business logic. Constraints là rules được enforce ở database level để maintain data quality.

**Types of Constraints**:
- **NOT NULL**: Column không được empty
- **UNIQUE**: Values không được duplicate  
- **PRIMARY KEY**: Unique + NOT NULL identifier
- **FOREIGN KEY**: Referential integrity
- **CHECK**: Custom validation rules
- **DEFAULT**: Default values khi không specify

**Use Case - Banking System Data Integrity**:
```sql
-- Comprehensive constraint system for banking
CREATE TABLE customers (
    customer_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    
    -- Personal Information Constraints
    first_name VARCHAR(50) NOT NULL 
        CHECK (LENGTH(TRIM(first_name)) >= 2),
    last_name VARCHAR(50) NOT NULL 
        CHECK (LENGTH(TRIM(last_name)) >= 2),
    
    -- Email with format validation
    email VARCHAR(255) NOT NULL UNIQUE
        CHECK (email REGEXP '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'),
    
    -- Phone number validation
    phone VARCHAR(20) NOT NULL UNIQUE
        CHECK (phone REGEXP '^[+]?[0-9]{10,15}$'),
    
    -- Date constraints
    birth_date DATE NOT NULL
        CHECK (birth_date >= '1900-01-01' AND birth_date <= CURDATE() - INTERVAL 18 YEAR),
    
    -- Status with allowed values
    status ENUM('ACTIVE', 'INACTIVE', 'SUSPENDED', 'CLOSED') DEFAULT 'ACTIVE',
    
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- Soft delete flag
    is_deleted BOOLEAN DEFAULT FALSE,
    
    -- Additional constraints
    CONSTRAINT chk_names_different CHECK (first_name != last_name),
    CONSTRAINT chk_valid_email_domain CHECK (
        email NOT LIKE '%@tempmail.%' AND 
        email NOT LIKE '%@10minutemail.%'
    )
);

CREATE TABLE accounts (
    account_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    
    -- Account number with format validation
    account_number VARCHAR(20) NOT NULL UNIQUE
        CHECK (account_number REGEXP '^[0-9]{10,16}$'),
    
    -- Account type validation
    account_type ENUM('CHECKING', 'SAVINGS', 'CREDIT', 'LOAN') NOT NULL,
    
    -- Balance constraints
    balance DECIMAL(15,2) NOT NULL DEFAULT 0.00
        CHECK (balance >= -999999999.99),  -- Allow negative for credit accounts
    
    -- Credit limit (only for credit accounts)
    credit_limit DECIMAL(15,2) DEFAULT 0.00
        CHECK (credit_limit >= 0),
    
    -- Interest rate validation
    interest_rate DECIMAL(5,4) DEFAULT 0.0000
        CHECK (interest_rate >= 0 AND interest_rate <= 1.0000),  -- 0% to 100%
    
    -- Status constraints
    status ENUM('ACTIVE', 'INACTIVE', 'FROZEN', 'CLOSED') DEFAULT 'ACTIVE',
    
    -- Opening date validation
    opened_date DATE NOT NULL DEFAULT (CURDATE())
        CHECK (opened_date >= '2000-01-01' AND opened_date <= CURDATE()),
    
    -- Foreign key constraint
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    
    -- Business rule constraints
    CONSTRAINT chk_credit_account_limit CHECK (
        (account_type != 'CREDIT') OR (credit_limit > 0)
    ),
    CONSTRAINT chk_savings_balance CHECK (
        (account_type != 'SAVINGS') OR (balance >= 0)
    ),
    
    -- Audit fields
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE transactions (
    transaction_id BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id BIGINT NOT NULL,
    
    -- Transaction details
    transaction_type ENUM('DEPOSIT', 'WITHDRAWAL', 'TRANSFER', 'FEE', 'INTEREST') NOT NULL,
    amount DECIMAL(15,2) NOT NULL
        CHECK (amount != 0),  -- No zero-amount transactions
    
    -- Description validation
    description VARCHAR(500) NOT NULL
        CHECK (LENGTH(TRIM(description)) >= 3),
    
    -- Reference number for tracking
    reference_number VARCHAR(50) UNIQUE
        CHECK (reference_number REGEXP '^[A-Z0-9]{8,20}$'),
    
    -- Related account for transfers
    related_account_id BIGINT,
    
    -- Transaction status
    status ENUM('PENDING', 'COMPLETED', 'FAILED', 'CANCELLED') DEFAULT 'PENDING',
    
    -- Timestamp validation
    transaction_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
        CHECK (transaction_date >= '2020-01-01' AND transaction_date <= NOW() + INTERVAL 1 HOUR),
    
    -- Processing information
    processed_by VARCHAR(100),
    processed_at TIMESTAMP NULL,
    
    -- Foreign key constraints
    FOREIGN KEY (account_id) REFERENCES accounts(account_id)
        ON DELETE RESTRICT ON UPDATE CASCADE,
    FOREIGN KEY (related_account_id) REFERENCES accounts(account_id)
        ON DELETE SET NULL ON UPDATE CASCADE,
    
    -- Business logic constraints
    CONSTRAINT chk_transfer_has_related_account CHECK (
        (transaction_type != 'TRANSFER') OR (related_account_id IS NOT NULL)
    ),
    CONSTRAINT chk_processed_status CHECK (
        (status != 'COMPLETED') OR (processed_at IS NOT NULL AND processed_by IS NOT NULL)
    ),
    CONSTRAINT chk_withdrawal_negative CHECK (
        (transaction_type != 'WITHDRAWAL') OR (amount < 0)
    ),
    CONSTRAINT chk_deposit_positive CHECK (
        (transaction_type != 'DEPOSIT') OR (amount > 0)
    )
);

-- Triggers for additional data integrity
DELIMITER //
CREATE TRIGGER validate_account_balance
BEFORE UPDATE ON accounts
FOR EACH ROW
BEGIN
    -- Prevent overdraft on savings accounts
    IF NEW.account_type = 'SAVINGS' AND NEW.balance < 0 THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Savings account balance cannot be negative';
    END IF;
    
    -- Prevent exceeding credit limit
    IF NEW.account_type = 'CREDIT' AND NEW.balance < -NEW.credit_limit THEN
        SIGNAL SQLSTATE '45000' 
        SET MESSAGE_TEXT = 'Transaction would exceed credit limit';
    END IF;
    
    -- Update timestamp
    SET NEW.updated_at = CURRENT_TIMESTAMP;
END //

CREATE TRIGGER validate_transaction_amount
BEFORE INSERT ON transactions
FOR EACH ROW
BEGIN
    DECLARE current_balance DECIMAL(15,2);
    DECLARE account_type_val VARCHAR(20);
    DECLARE credit_limit_val DECIMAL(15,2);
    
    -- Get account information
    SELECT balance, account_type, credit_limit 
    INTO current_balance, account_type_val, credit_limit_val
    FROM accounts 
    WHERE account_id = NEW.account_id;
    
    -- Validate withdrawal doesn't cause overdraft
    IF NEW.transaction_type = 'WITHDRAWAL' THEN
        IF account_type_val = 'SAVINGS' AND (current_balance + NEW.amount) < 0 THEN
            SIGNAL SQLSTATE '45000' 
            SET MESSAGE_TEXT = 'Insufficient funds for withdrawal';
        END IF;
        
        IF account_type_val = 'CREDIT' AND (current_balance + NEW.amount) < -credit_limit_val THEN
            SIGNAL SQLSTATE '45000' 
            SET MESSAGE_TEXT = 'Withdrawal would exceed credit limit';
        END IF;
    END IF;
    
    -- Generate reference number if not provided
    IF NEW.reference_number IS NULL THEN
        SET NEW.reference_number = CONCAT(
            'TXN',
            DATE_FORMAT(NOW(), '%Y%m%d'),
            LPAD(NEW.account_id, 6, '0'),
            LPAD(CONNECTION_ID(), 4, '0')
        );
    END IF;
END //
DELIMITER ;
```

## 29. Data Redundancy & Normalization

**Định nghĩa chi tiết**: Data Redundancy là việc lưu trữ cùng một thông tin ở nhiều nơi, có thể gây inconsistency và waste storage. Normalization là process loại bỏ redundancy thông qua decomposition tables theo normal forms.

**Problems with Redundancy**:
- **Update Anomalies**: Phải update multiple places
- **Insert Anomalies**: Không thể insert data without other data
- **Delete Anomalies**: Mất thông tin khi delete records
- **Storage Waste**: Duplicate data consumes space

**Use Case - E-commerce Normalization Process**:
```sql
-- 0NF: Unnormalized table with redundancy
CREATE TABLE orders_unnormalized (
    order_id INT,
    order_date DATE,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100),
    customer_phone VARCHAR(20),
    customer_address TEXT,
    customer_city VARCHAR(50),
    customer_country VARCHAR(50),
    product1_name VARCHAR(200),
    product1_price DECIMAL(10,2),
    product1_quantity INT,
    product2_name VARCHAR(200),
    product2_price DECIMAL(10,2), 
    product2_quantity INT,
    product3_name VARCHAR(200),
    product3_price DECIMAL(10,2),
    product3_quantity INT,
    shipping_method VARCHAR(50),
    shipping_cost DECIMAL(8,2),
    total_amount DECIMAL(12,2)
);

-- Problems with unnormalized design:
-- 1. Customer info repeated for each order
-- 2. Limited to 3 products per order
-- 3. Null values when order has < 3 products
-- 4. Update customer info requires updating all their orders

-- 1NF: Eliminate repeating groups
CREATE TABLE orders_1nf (
    order_id INT,
    order_date DATE,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100),
    customer_phone VARCHAR(20),
    customer_address TEXT,
    customer_city VARCHAR(50),
    customer_country VARCHAR(50),
    product_name VARCHAR(200),
    product_price DECIMAL(10,2),
    product_quantity INT,
    shipping_method VARCHAR(50),
    shipping_cost DECIMAL(8,2),
    line_total DECIMAL(10,2),
    PRIMARY KEY (order_id, product_name)
);

-- 1NF eliminates repeating groups but still has redundancy
-- Customer info still repeated for each product in order

-- 2NF: Eliminate partial dependencies
CREATE TABLE customers_2nf (
    customer_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    customer_email VARCHAR(100) UNIQUE NOT NULL,
    customer_phone VARCHAR(20),
    customer_address TEXT,
    customer_city VARCHAR(50),
    customer_country VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products_2nf (
    product_id INT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(200) NOT NULL,
    product_price DECIMAL(10,2) NOT NULL,
    category VARCHAR(100),
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders_2nf (
    order_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    shipping_method VARCHAR(50),
    shipping_cost DECIMAL(8,2),
    status VARCHAR(20) DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers_2nf(customer_id)
);

CREATE TABLE order_items_2nf (
    order_id INT,
    product_id INT,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    line_total DECIMAL(10,2) GENERATED ALWAYS AS (quantity * unit_price),
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders_2nf(order_id),
    FOREIGN KEY (product_id) REFERENCES products_2nf(product_id)
);

-- 2NF eliminates partial dependencies but may still have transitive dependencies

-- 3NF: Eliminate transitive dependencies
CREATE TABLE countries_3nf (
    country_id INT AUTO_INCREMENT PRIMARY KEY,
    country_name VARCHAR(50) UNIQUE NOT NULL,
    country_code VARCHAR(3) UNIQUE NOT NULL,
    currency VARCHAR(3),
    tax_rate DECIMAL(5,4)
);

CREATE TABLE cities_3nf (
    city_id INT AUTO_INCREMENT PRIMARY KEY,
    city_name VARCHAR(50) NOT NULL,
    country_id INT NOT NULL,
    timezone VARCHAR(50),
    FOREIGN KEY (country_id) REFERENCES countries_3nf(country_id),
    UNIQUE KEY unique_city_country (city_name, country_id)
);

CREATE TABLE categories_3nf (
    category_id INT AUTO_INCREMENT PRIMARY KEY,
    category_name VARCHAR(100) UNIQUE NOT NULL,
    parent_category_id INT,
    description TEXT,
    FOREIGN KEY (parent_category_id) REFERENCES categories_3nf(category_id)
);

CREATE TABLE shipping_methods_3nf (
    shipping_method_id INT AUTO_INCREMENT PRIMARY KEY,
    method_name VARCHAR(50) UNIQUE NOT NULL,
    base_cost DECIMAL(8,2),
    cost_per_kg DECIMAL(6,2),
    estimated_days INT,
    is_active BOOLEAN DEFAULT TRUE
);

-- Updated tables with proper 3NF structure
CREATE TABLE customers_3nf (
    customer_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    customer_email VARCHAR(100) UNIQUE NOT NULL,
    customer_phone VARCHAR(20),
    customer_address TEXT,
    city_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (city_id) REFERENCES cities_3nf(city_id)
);

CREATE TABLE products_3nf (
    product_id INT AUTO_INCREMENT PRIMARY KEY,
    product_name VARCHAR(200) NOT NULL,
    product_price DECIMAL(10,2) NOT NULL,
    category_id INT,
    description TEXT,
    weight_kg DECIMAL(6,3),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories_3nf(category_id)
);

CREATE TABLE orders_3nf (
    order_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    shipping_method_id INT,
    shipping_cost DECIMAL(8,2),
    status VARCHAR(20) DEFAULT 'PENDING',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (customer_id) REFERENCES customers_3nf(customer_id),
    FOREIGN KEY (shipping_method_id) REFERENCES shipping_methods_3nf(shipping_method_id)
);

-- Denormalization for performance (when needed)
CREATE TABLE order_summary_denormalized (
    order_id INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),        -- Denormalized for reporting
    customer_email VARCHAR(100),       -- Denormalized for notifications
    customer_city VARCHAR(50),         -- Denormalized for analytics
    customer_country VARCHAR(50),      -- Denormalized for analytics
    order_date DATE,
    item_count INT,
    total_quantity INT,
    subtotal DECIMAL(12,2),
    shipping_cost DECIMAL(8,2),
    tax_amount DECIMAL(10,2),
    total_amount DECIMAL(12,2),
    status VARCHAR(20),
    
    -- Maintain referential integrity
    FOREIGN KEY (customer_id) REFERENCES customers_3nf(customer_id),
    
    -- Update trigger to maintain consistency
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Trigger to maintain denormalized data
DELIMITER //
CREATE TRIGGER update_order_summary
AFTER INSERT ON order_items_2nf
FOR EACH ROW
BEGIN
    INSERT INTO order_summary_denormalized (
        order_id, customer_id, customer_name, customer_email, 
        customer_city, customer_country, order_date, item_count,
        total_quantity, subtotal, shipping_cost, total_amount, status
    )
    SELECT 
        o.order_id,
        o.customer_id,
        c.customer_name,
        c.customer_email,
        ci.city_name,
        co.country_name,
        o.order_date,
        COUNT(oi.product_id) as item_count,
        SUM(oi.quantity) as total_quantity,
        SUM(oi.line_total) as subtotal,
        o.shipping_cost,
        SUM(oi.line_total) + o.shipping_cost as total_amount,
        o.status
    FROM orders_3nf o
    JOIN customers_3nf c ON o.customer_id = c.customer_id
    LEFT JOIN cities_3nf ci ON c.city_id = ci.city_id
    LEFT JOIN countries_3nf co ON ci.country_id = co.country_id
    JOIN order_items_2nf oi ON o.order_id = oi.order_id
    WHERE o.order_id = NEW.order_id
    GROUP BY o.order_id
    ON DUPLICATE KEY UPDATE
        item_count = VALUES(item_count),
        total_quantity = VALUES(total_quantity),
        subtotal = VALUES(subtotal),
        total_amount = VALUES(total_amount),
        updated_at = CURRENT_TIMESTAMP;
END //
DELIMITER ;

-- Benefits analysis
SELECT 
    'Normalized Benefits' as aspect,
    'Reduced redundancy, easier updates, data consistency' as description
UNION ALL
SELECT 
    'Denormalized Benefits',
    'Faster queries, reduced joins, better reporting performance'
UNION ALL
SELECT 
    'Trade-offs',
    'Storage vs Performance, Consistency vs Speed, Complexity vs Simplicity';
```

## 30. Deadlock Detection & Prevention

**Định nghĩa chi tiết**: Deadlock xảy ra khi hai hoặc nhiều transactions chờ nhau mãi mãi để acquire locks, tạo thành circular dependency. Database systems cần detect và resolve deadlocks để maintain system availability.

**Deadlock Conditions (All 4 must be present)**:
- **Mutual Exclusion**: Resources không thể shared
- **Hold and Wait**: Process giữ resource và chờ others
- **No Preemption**: Resources không thể forcibly taken
- **Circular Wait**: Circular chain of waiting processes

**Use Case - Banking System Deadlock Scenarios**:
```python
import threading
import time
import mysql.connector
from contextlib import contextmanager
import logging

class DeadlockDemo:
    def __init__(self):
        self.db_config = {
            'host': 'localhost',
            'database': 'banking',
            'user': 'app_user',
            'password': 'app_password',
            'autocommit': False
        }
        self.deadlock_count = 0
        self.retry_count = 0
        
    @contextmanager
    def get_connection(self):
        """Context manager for database connections"""
        conn = mysql.connector.connect(**self.db_config)
        try:
            yield conn
        finally:
            conn.close()
    
    def transfer_money_unsafe(self, from_account: str, to_account: str, amount: float, thread_id: int):
        """Unsafe transfer that can cause deadlocks"""
        with self.get_connection() as conn:
            cursor = conn.cursor()
            
            try:
                print(f"Thread {thread_id}: Starting transfer {from_account} -> {to_account}: ${amount}")
                
                # Lock accounts in order they appear (BAD - can cause deadlock)
                cursor.execute("BEGIN")
                
                # Lock first account
                cursor.execute("""
                    SELECT balance FROM accounts 
                    WHERE account_number = %s 
                    FOR UPDATE
                """, (from_account,))
                
                from_balance = cursor.fetchone()[0]
                print(f"Thread {thread_id}: Locked {from_account}, balance: ${from_balance}")
                
                # Simulate processing time
                time.sleep(0.1)
                
                # Lock second account (potential deadlock here)
                cursor.execute("""
                    SELECT balance FROM accounts 
                    WHERE account_number = %s 
                    FOR UPDATE
                """, (to_account,))
                
                to_balance = cursor.fetchone()[0]
                print(f"Thread {thread_id}: Locked {to_account}, balance: ${to_balance}")
                
                # Validate sufficient funds
                if from_balance >= amount:
                    # Update balances
                    cursor.execute("""
                        UPDATE accounts SET balance = balance - %s 
                        WHERE account_number = %s
                    """, (amount, from_account))
                    
                    cursor.execute("""
                        UPDATE accounts SET balance = balance + %s 
                        WHERE account_number = %s
                    """, (amount, to_account))
                    
                    # Record transaction
                    cursor.execute("""
                        INSERT INTO transactions (from_account, to_account, amount, status, created_at)
                        VALUES (%s, %s, %s, 'COMPLETED', NOW())
                    """, (from_account, to_account, amount))
                    
                    conn.commit()
                    print(f"Thread {thread_id}: ✅ Transfer completed successfully")
                else:
                    conn.rollback()
                    print(f"Thread {thread_id}: ❌ Insufficient funds")
                    
            except mysql.connector.Error as e:
                conn.rollback()
                if e.errno == 1213:  # Deadlock error code
                    self.deadlock_count += 1
                    print(f"Thread {thread_id}: 🔒 DEADLOCK DETECTED! Error: {e}")
                else:
                    print(f"Thread {thread_id}: ❌ Database error: {e}")
                raise
    
    def transfer_money_safe(self, from_account: str, to_account: str, amount: float, thread_id: int, max_retries: int = 3):
        """Safe transfer with deadlock prevention and retry logic"""
        
        # Prevention: Always lock accounts in consistent order (alphabetical)
        first_account = min(from_account, to_account)
        second_account = max(from_account, to_account)
        
        for attempt in range(max_retries):
            with self.get_connection() as conn:
                cursor = conn.cursor()
                
                try:
                    print(f"Thread {thread_id}: Attempt {attempt + 1} - Transfer {from_account} -> {to_account}: ${amount}")
                    
                    cursor.execute("BEGIN")
                    
                    # Lock accounts in consistent order to prevent deadlock
                    cursor.execute("""
                        SELECT account_number, balance FROM accounts 
                        WHERE account_number IN (%s, %s)
                        ORDER BY account_number
                        FOR UPDATE
                    """, (first_account, second_account))
                    
                    account_balances = {row[0]: row[1] for row in cursor.fetchall()}
                    
                    print(f"Thread {thread_id}: Locked both accounts in order")
                    
                    from_balance = account_balances[from_account]
                    to_balance = account_balances[to_account]
                    
                    # Validate and execute transfer
                    if from_balance >= amount:
                        cursor.execute("""
                            UPDATE accounts SET balance = balance - %s 
                            WHERE account_number = %s
                        """, (amount, from_account))
                        
                        cursor.execute("""
                            UPDATE accounts SET balance = balance + %s 
                            WHERE account_number = %s
                        """, (amount, to_account))
                        
                        cursor.execute("""
                            INSERT INTO transactions (from_account, to_account, amount, status, created_at)
                            VALUES (%s, %s, %s, 'COMPLETED', NOW())
                        """, (from_account, to_account, amount))
                        
                        conn.commit()
                        print(f"Thread {thread_id}: ✅ Safe transfer completed")
                        return True
                    else:
                        conn.rollback()
                        print(f"Thread {thread_id}: ❌ Insufficient funds")
                        return False
                        
                except mysql.connector.Error as e:
                    conn.rollback()
                    if e.errno == 1213:  # Deadlock
                        self.deadlock_count += 1
                        self.retry_count += 1
                        wait_time = (2 ** attempt) * 0.1  # Exponential backoff
                        print(f"Thread {thread_id}: 🔒 Deadlock on attempt {attempt + 1}, retrying in {wait_time}s...")
                        time.sleep(wait_time)
                        continue
                    else:
                        print(f"Thread {thread_id}: ❌ Database error: {e}")
                        raise
        
        print(f"Thread {thread_id}: ❌ Max retries exceeded")
        return False
    
    def simulate_concurrent_transfers(self, use_safe_method: bool = True):
        """Simulate concurrent transfers that can cause deadlocks"""
        
        # Create test accounts
        self.setup_test_accounts()
        
        # Define transfer scenarios that can cause deadlocks
        transfers = [
            ('ACC001', 'ACC002', 100.0),
            ('ACC002', 'ACC001', 50.0),   # Reverse direction - potential deadlock
            ('ACC001', 'ACC003', 75.0),
            ('ACC003', 'ACC001', 25.0),   # Reverse direction - potential deadlock
            ('ACC002', 'ACC003', 60.0),
            ('ACC003', 'ACC002', 40.0),   # Reverse direction - potential deadlock
        ]
        
        threads = []
        
        # Start concurrent transfers
        for i, (from_acc, to_acc, amount) in enumerate(transfers):
            if use_safe_method:
                target_func = self.transfer_money_safe
            else:
                target_func = self.transfer_money_unsafe
                
            thread = threading.Thread(
                target=target_func,
                args=(from_acc, to_acc, amount, i + 1)
            )
            threads.append(thread)
            thread.start()
            
            # Small delay to increase chance of deadlock
            time.sleep(0.05)
        
        # Wait for all transfers to complete
        for thread in threads:
            thread.join()
        
        print(f"\n📊 Results:")
        print(f"Deadlocks detected: {self.deadlock_count}")
        print(f"Retries performed: {self.retry_count}")
        print(f"Method used: {'Safe (with prevention)' if use_safe_method else 'Unsafe (deadlock prone)'}")
    
    def setup_test_accounts(self):
        """Setup test accounts for demonstration"""
        with self.get_connection() as conn:
            cursor = conn.cursor()
            
            # Create accounts table if not exists
            cursor.execute("""
                CREATE TABLE IF NOT EXISTS accounts (
                    account_id INT AUTO_INCREMENT PRIMARY KEY,
                    account_number VARCHAR(20) UNIQUE NOT NULL,
                    balance DECIMAL(15,2) NOT NULL DEFAULT 0.00,
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            """)
            
            cursor.execute("""
                CREATE TABLE IF NOT EXISTS transactions (
                    transaction_id INT AUTO_INCREMENT PRIMARY KEY,
                    from_account VARCHAR(20),
                    to_account VARCHAR(20),
                    amount DECIMAL(15,2),
                    status VARCHAR(20),
                    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
                )
            """)
            
            # Insert test accounts
            test_accounts = [
                ('ACC001', 1000.00),
                ('ACC002', 1500.00),
                ('ACC003', 2000.00)
            ]
            
            for account_number, initial_balance in test_accounts:
                cursor.execute("""
                    INSERT INTO accounts (account_number, balance)
                    VALUES (%s, %s)
                    ON DUPLICATE KEY UPDATE balance = VALUES(balance)
                """, (account_number, initial_balance))
            
            conn.commit()
            print("✅ Test accounts setup completed")

# Database-level deadlock detection
deadlock_detection_sql = """
-- MySQL deadlock detection settings
SHOW VARIABLES LIKE 'innodb_deadlock_detect';
SHOW VARIABLES LIKE 'innodb_lock_wait_timeout';

-- Enable deadlock detection (default is ON)
SET GLOBAL innodb_deadlock_detect = ON;

-- Set lock wait timeout (default is 50 seconds)
SET GLOBAL innodb_lock_wait_timeout = 10;

-- Monitor deadlocks
SHOW ENGINE INNODB STATUS;

-- Deadlock information is in the LATEST DETECTED DEADLOCK section
-- Shows:
-- - Transactions involved
-- - Locks held and requested
-- - Which transaction was rolled back
-- - SQL statements that caused deadlock
"""

# AWS RDS deadlock monitoring
aws_rds_monitoring = """
-- CloudWatch metrics for deadlock monitoring
Deadlocks/sec - Number of deadlocks per second
DatabaseConnections - Active connections
ReadLatency/WriteLatency - Performance impact

-- Performance Insights queries
SELECT 
    sql_id,
    digest_text,
    count_star as execution_count,
    sum_lock_time/1000000 as total_lock_time_ms,
    avg_lock_time/1000000 as avg_lock_time_ms
FROM performance_schema.events_statements_summary_by_digest
WHERE digest_text LIKE '%FOR UPDATE%'
ORDER BY sum_lock_time DESC;
"""

# Run demonstration
print("🔒 DEADLOCK DEMONSTRATION")
print("=" * 50)

demo = DeadlockDemo()

print("\n1. Running UNSAFE transfers (deadlock-prone):")
demo.simulate_concurrent_transfers(use_safe_method=False)

# Reset counters
demo.deadlock_count = 0
demo.retry_count = 0

print("\n2. Running SAFE transfers (with prevention):")
demo.simulate_concurrent_transfers(use_safe_method=True)
```
