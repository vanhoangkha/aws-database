# Database Concepts - Part 2 (Concepts 11-65)

## 11. Execution Plan (Query Plan)

**Định nghĩa chi tiết**: Kế hoạch thực thi được query optimizer tạo ra để xác định cách thức hiệu quả nhất để thực hiện một câu truy vấn. Execution plan bao gồm sequence of operations, cost estimates và resource usage.

**Query Optimizer Process**:
1. **Parse**: Kiểm tra syntax và semantic
2. **Rewrite**: Apply query rewrite rules
3. **Optimize**: Generate multiple execution plans
4. **Cost Estimation**: Calculate cost cho mỗi plan
5. **Plan Selection**: Chọn plan có cost thấp nhất

**Use Case - E-commerce Product Search với Complex Join**:
```sql
-- Complex query cần optimization
EXPLAIN FORMAT=JSON
SELECT 
    p.product_name,
    p.price,
    c.category_name,
    b.brand_name,
    AVG(r.rating) as avg_rating,
    COUNT(r.review_id) as review_count,
    i.stock_quantity
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN brands b ON p.brand_id = b.brand_id
LEFT JOIN reviews r ON p.product_id = r.product_id
JOIN inventory i ON p.product_id = i.product_id
WHERE p.price BETWEEN 1000000 AND 5000000
  AND c.category_name = 'Electronics'
  AND p.status = 'active'
  AND i.stock_quantity > 0
GROUP BY p.product_id, p.product_name, p.price, c.category_name, b.brand_name, i.stock_quantity
HAVING AVG(r.rating) >= 4.0
ORDER BY avg_rating DESC, review_count DESC
LIMIT 20;

-- Execution Plan Analysis:
{
  "query_block": {
    "select_id": 1,
    "cost_info": {
      "query_cost": "2847.65"
    },
    "nested_loop": [
      {
        "table": {
          "table_name": "c",
          "access_type": "ref",
          "possible_keys": ["idx_category_name"],
          "key": "idx_category_name",
          "used_key_parts": ["category_name"],
          "key_length": "402",
          "rows_examined_per_scan": 1,
          "rows_produced_per_join": 1,
          "filtered": "100.00",
          "cost_info": {
            "read_cost": "1.00",
            "eval_cost": "0.10",
            "prefix_cost": "1.10",
            "data_read_per_join": "416"
          }
        }
      },
      {
        "table": {
          "table_name": "p",
          "access_type": "ref",
          "possible_keys": ["idx_category_price_status"],
          "key": "idx_category_price_status",
          "used_key_parts": ["category_id", "price", "status"],
          "rows_examined_per_scan": 5000,
          "cost_info": {
            "read_cost": "1250.00",
            "eval_cost": "500.00"
          }
        }
      }
    ]
  }
}
```

**Optimization Techniques**:
```sql
-- 1. Create optimal indexes
CREATE INDEX idx_products_optimized ON products(category_id, status, price, product_id);
CREATE INDEX idx_reviews_product_rating ON reviews(product_id, rating);

-- 2. Rewrite query để tối ưu
SELECT 
    p.product_name,
    p.price,
    c.category_name,
    b.brand_name,
    COALESCE(rs.avg_rating, 0) as avg_rating,
    COALESCE(rs.review_count, 0) as review_count,
    i.stock_quantity
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN brands b ON p.brand_id = b.brand_id
JOIN inventory i ON p.product_id = i.product_id
LEFT JOIN (
    SELECT 
        product_id,
        AVG(rating) as avg_rating,
        COUNT(*) as review_count
    FROM reviews 
    GROUP BY product_id
    HAVING AVG(rating) >= 4.0
) rs ON p.product_id = rs.product_id
WHERE p.category_id = (SELECT category_id FROM categories WHERE category_name = 'Electronics')
  AND p.price BETWEEN 1000000 AND 5000000
  AND p.status = 'active'
  AND i.stock_quantity > 0
ORDER BY rs.avg_rating DESC, rs.review_count DESC
LIMIT 20;
```

## 12. Data Warehouse

**Định nghĩa chi tiết**: Hệ thống lưu trữ dữ liệu tập trung, được thiết kế đặc biệt cho analytical processing và business intelligence. Data warehouse tích hợp dữ liệu từ multiple sources, transform theo business rules và optimize cho complex queries.

**Đặc điểm Data Warehouse**:
- **Subject-Oriented**: Tổ chức theo business subjects (sales, customers, products)
- **Integrated**: Dữ liệu từ nhiều nguồn được standardized
- **Time-Variant**: Lưu trữ historical data với timestamps
- **Non-Volatile**: Dữ liệu không thay đổi sau khi load

**Architecture Components**:
- **ETL Pipeline**: Extract, Transform, Load processes
- **Staging Area**: Temporary storage during ETL
- **Data Marts**: Subject-specific subsets
- **OLAP Cubes**: Pre-aggregated data for fast queries

**Use Case - Retail Chain Analytics (1000 stores, 10M customers)**:
```sql
-- Fact Table: Sales transactions
CREATE TABLE fact_sales (
    sale_id BIGINT PRIMARY KEY,
    date_id INT NOT NULL,
    store_id INT NOT NULL,
    product_id INT NOT NULL,
    customer_id INT,
    employee_id INT,
    promotion_id INT,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    tax_amount DECIMAL(10,2) NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL,
    cost_amount DECIMAL(10,2) NOT NULL,
    profit_amount DECIMAL(10,2) GENERATED ALWAYS AS (total_amount - cost_amount),
    
    -- Foreign keys to dimension tables
    FOREIGN KEY (date_id) REFERENCES dim_date(date_id),
    FOREIGN KEY (store_id) REFERENCES dim_stores(store_id),
    FOREIGN KEY (product_id) REFERENCES dim_products(product_id),
    FOREIGN KEY (customer_id) REFERENCES dim_customers(customer_id)
);

-- Dimension Tables
CREATE TABLE dim_date (
    date_id INT PRIMARY KEY,
    full_date DATE NOT NULL,
    day_of_week VARCHAR(10),
    day_of_month INT,
    day_of_year INT,
    week_of_year INT,
    month_name VARCHAR(10),
    month_number INT,
    quarter INT,
    year INT,
    is_weekend BOOLEAN,
    is_holiday BOOLEAN,
    fiscal_year INT,
    fiscal_quarter INT
);

CREATE TABLE dim_products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(200) NOT NULL,
    sku VARCHAR(50) UNIQUE,
    category_level1 VARCHAR(50),
    category_level2 VARCHAR(50),
    category_level3 VARCHAR(50),
    brand VARCHAR(100),
    supplier VARCHAR(100),
    unit_cost DECIMAL(10,2),
    standard_price DECIMAL(10,2),
    product_status VARCHAR(20),
    launch_date DATE,
    discontinue_date DATE
);

CREATE TABLE dim_stores (
    store_id INT PRIMARY KEY,
    store_name VARCHAR(100) NOT NULL,
    store_code VARCHAR(20) UNIQUE,
    address VARCHAR(200),
    city VARCHAR(50),
    state VARCHAR(50),
    postal_code VARCHAR(20),
    country VARCHAR(50),
    region VARCHAR(50),
    district VARCHAR(50),
    store_type VARCHAR(30),
    store_size_sqft INT,
    opening_date DATE,
    manager_name VARCHAR(100)
);

CREATE TABLE dim_customers (
    customer_id INT PRIMARY KEY,
    customer_code VARCHAR(50) UNIQUE,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    email VARCHAR(100),
    phone VARCHAR(20),
    birth_date DATE,
    gender VARCHAR(10),
    age_group VARCHAR(20),
    income_bracket VARCHAR(30),
    education_level VARCHAR(30),
    marital_status VARCHAR(20),
    customer_segment VARCHAR(30),
    loyalty_tier VARCHAR(20),
    registration_date DATE,
    last_purchase_date DATE
);
```

**Complex Analytics Queries**:
```sql
-- Monthly sales trend với year-over-year comparison
WITH monthly_sales AS (
    SELECT 
        d.year,
        d.month_number,
        d.month_name,
        SUM(f.total_amount) as monthly_revenue,
        SUM(f.quantity) as units_sold,
        COUNT(DISTINCT f.customer_id) as unique_customers,
        COUNT(DISTINCT f.sale_id) as transaction_count
    FROM fact_sales f
    JOIN dim_date d ON f.date_id = d.date_id
    WHERE d.year IN (2023, 2024)
    GROUP BY d.year, d.month_number, d.month_name
),
yoy_comparison AS (
    SELECT 
        month_number,
        month_name,
        SUM(CASE WHEN year = 2024 THEN monthly_revenue END) as revenue_2024,
        SUM(CASE WHEN year = 2023 THEN monthly_revenue END) as revenue_2023,
        SUM(CASE WHEN year = 2024 THEN units_sold END) as units_2024,
        SUM(CASE WHEN year = 2023 THEN units_sold END) as units_2023
    FROM monthly_sales
    GROUP BY month_number, month_name
)
SELECT 
    month_name,
    revenue_2024,
    revenue_2023,
    ROUND((revenue_2024 - revenue_2023) / revenue_2023 * 100, 2) as revenue_growth_pct,
    units_2024,
    units_2023,
    ROUND((units_2024 - units_2023) / units_2023 * 100, 2) as units_growth_pct
FROM yoy_comparison
ORDER BY month_number;

-- Customer segmentation analysis
SELECT 
    c.customer_segment,
    c.age_group,
    COUNT(DISTINCT c.customer_id) as customer_count,
    SUM(f.total_amount) as total_spent,
    AVG(f.total_amount) as avg_transaction_value,
    COUNT(f.sale_id) as total_transactions,
    ROUND(COUNT(f.sale_id) / COUNT(DISTINCT c.customer_id), 2) as avg_transactions_per_customer,
    ROUND(SUM(f.total_amount) / COUNT(DISTINCT c.customer_id), 2) as avg_spent_per_customer
FROM fact_sales f
JOIN dim_customers c ON f.customer_id = c.customer_id
JOIN dim_date d ON f.date_id = d.date_id
WHERE d.year = 2024
GROUP BY c.customer_segment, c.age_group
ORDER BY total_spent DESC;
```

**AWS Redshift Implementation**:
```sql
-- Redshift-specific optimizations
CREATE TABLE fact_sales_redshift (
    sale_id BIGINT IDENTITY(1,1),
    date_id INT NOT NULL,
    store_id INT NOT NULL,
    product_id INT NOT NULL,
    customer_id INT,
    quantity INT NOT NULL,
    total_amount DECIMAL(12,2) NOT NULL
)
DISTSTYLE KEY
DISTKEY (customer_id)          -- Distribute by customer for customer analytics
SORTKEY (date_id, store_id);   -- Sort by date and store for time-series queries

-- Columnar storage benefits
-- - Only read columns needed for query
-- - Better compression ratios
-- - Faster aggregations
```

## 13. NoSQL Database Types

**Định nghĩa chi tiết**: NoSQL (Not Only SQL) databases được thiết kế để handle large volumes of unstructured/semi-structured data với high scalability và flexibility. Khác với relational databases, NoSQL không require fixed schema và có thể scale horizontally.

### Document Store (MongoDB, DynamoDB, DocumentDB)

**Đặc điểm**: Lưu trữ data dưới dạng documents (JSON, BSON) với nested structures và arrays.

**Use Case - Content Management System**:
```json
// User Profile Document với nested data
{
  "_id": "user_507f1f77bcf86cd799439011",
  "username": "john_doe",
  "email": "john@example.com",
  "profile": {
    "firstName": "John",
    "lastName": "Doe",
    "age": 29,
    "avatar": "https://cdn.example.com/avatars/john.jpg",
    "preferences": {
      "language": "en",
      "timezone": "UTC+7",
      "notifications": {
        "email": true,
        "push": false,
        "sms": true
      }
    },
    "address": {
      "street": "123 Main St",
      "city": "Ho Chi Minh City",
      "district": "District 1",
      "country": "Vietnam",
      "coordinates": {
        "lat": 10.7769,
        "lng": 106.7009
      }
    }
  },
  "interests": ["technology", "sports", "travel", "photography"],
  "social_links": {
    "facebook": "https://facebook.com/johndoe",
    "twitter": "@johndoe",
    "linkedin": "https://linkedin.com/in/johndoe"
  },
  "purchase_history": [
    {
      "order_id": "ord_001",
      "date": "2024-01-15T10:30:00Z",
      "amount": 1500000,
      "currency": "VND",
      "items": [
        {
          "product_id": "prod_123",
          "name": "iPhone 15 Pro",
          "quantity": 1,
          "price": 1500000
        }
      ],
      "shipping_address": {
        "street": "456 Work St",
        "city": "Ho Chi Minh City",
        "district": "District 3"
      }
    }
  ],
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-20T15:45:00Z",
  "status": "active"
}
```

**MongoDB Queries**:
```javascript
// Find users by interests và location
db.users.find({
  "interests": { $in: ["technology", "sports"] },
  "profile.address.city": "Ho Chi Minh City",
  "status": "active"
}).sort({ "updated_at": -1 }).limit(10);

// Aggregate purchase analytics
db.users.aggregate([
  {
    $unwind: "$purchase_history"
  },
  {
    $group: {
      _id: {
        city: "$profile.address.city",
        month: { $month: "$purchase_history.date" }
      },
      total_revenue: { $sum: "$purchase_history.amount" },
      avg_order_value: { $avg: "$purchase_history.amount" },
      customer_count: { $addToSet: "$_id" }
    }
  },
  {
    $project: {
      city: "$_id.city",
      month: "$_id.month",
      total_revenue: 1,
      avg_order_value: 1,
      unique_customers: { $size: "$customer_count" }
    }
  }
]);
```

### Key-Value Store (Redis, DynamoDB)

**Đặc điểm**: Simple key-value pairs với extremely fast access, thường dùng cho caching và session storage.

**Use Case - Real-time Gaming Leaderboard**:
```python
import redis
import json
from datetime import datetime, timedelta

# Redis connection
r = redis.Redis(host='elasticache-cluster.abc123.cache.amazonaws.com', port=6379, db=0)

class GameLeaderboard:
    def __init__(self, game_id):
        self.game_id = game_id
        self.leaderboard_key = f"leaderboard:{game_id}"
        self.player_stats_key = f"player_stats:{game_id}"
        
    def update_score(self, player_id, score, additional_stats=None):
        # Update leaderboard (sorted set)
        r.zadd(self.leaderboard_key, {player_id: score})
        
        # Update player detailed stats (hash)
        player_key = f"{self.player_stats_key}:{player_id}"
        stats = {
            'current_score': score,
            'last_updated': datetime.now().isoformat(),
            'games_played': r.hincrby(player_key, 'games_played', 1),
        }
        
        if additional_stats:
            stats.update(additional_stats)
            
        r.hmset(player_key, stats)
        
        # Set expiration (7 days)
        r.expire(player_key, 604800)
        
    def get_top_players(self, limit=10):
        # Get top players với scores
        top_players = r.zrevrange(
            self.leaderboard_key, 
            0, limit-1, 
            withscores=True
        )
        
        result = []
        for player_id, score in top_players:
            player_stats = r.hgetall(f"{self.player_stats_key}:{player_id.decode()}")
            result.append({
                'player_id': player_id.decode(),
                'score': int(score),
                'stats': {k.decode(): v.decode() for k, v in player_stats.items()}
            })
            
        return result
    
    def get_player_rank(self, player_id):
        rank = r.zrevrank(self.leaderboard_key, player_id)
        score = r.zscore(self.leaderboard_key, player_id)
        
        return {
            'player_id': player_id,
            'rank': rank + 1 if rank is not None else None,
            'score': int(score) if score else 0
        }

# Usage
game = GameLeaderboard("battle_royale_001")

# Update scores
game.update_score("player_123", 9500, {
    'kills': 15,
    'deaths': 3,
    'assists': 8,
    'match_duration': 1800  # seconds
})

game.update_score("player_456", 8750, {
    'kills': 12,
    'deaths': 5,
    'assists': 6,
    'match_duration': 1650
})

# Get leaderboard
top_10 = game.get_top_players(10)
print(json.dumps(top_10, indent=2))

# Session management
def store_user_session(user_id, session_data):
    session_key = f"session:{user_id}"
    r.setex(
        session_key,
        1800,  # 30 minutes expiration
        json.dumps({
            'user_id': user_id,
            'login_time': datetime.now().isoformat(),
            'cart_items': session_data.get('cart_items', []),
            'last_page': session_data.get('last_page', '/'),
            'preferences': session_data.get('preferences', {})
        })
    )

def get_user_session(user_id):
    session_key = f"session:{user_id}"
    session_data = r.get(session_key)
    return json.loads(session_data) if session_data else None
```

### Wide-Column Store (Cassandra, DynamoDB)

**Đặc điểm**: Column families với dynamic columns, tối ưu cho time-series data và high write throughput.

**Use Case - IoT Sensor Data Collection**:
```sql
-- Cassandra schema cho IoT data
CREATE KEYSPACE iot_data WITH replication = {
    'class': 'SimpleStrategy',
    'replication_factor': 3
};

CREATE TABLE sensor_readings (
    device_id UUID,
    timestamp TIMESTAMP,
    sensor_type TEXT,
    value DOUBLE,
    unit TEXT,
    location TEXT,
    metadata MAP<TEXT, TEXT>,
    PRIMARY KEY (device_id, timestamp, sensor_type)
) WITH CLUSTERING ORDER BY (timestamp DESC, sensor_type ASC);

-- Insert sensor data
INSERT INTO sensor_readings (
    device_id, timestamp, sensor_type, value, unit, location, metadata
) VALUES (
    550e8400-e29b-41d4-a716-446655440000,
    '2024-01-20 10:30:00',
    'temperature',
    25.6,
    'celsius',
    'warehouse_a_zone_1',
    {'calibration_date': '2024-01-01', 'firmware_version': '1.2.3'}
);

-- Query time-series data
SELECT device_id, timestamp, sensor_type, value, unit
FROM sensor_readings
WHERE device_id = 550e8400-e29b-41d4-a716-446655440000
  AND timestamp >= '2024-01-20 00:00:00'
  AND timestamp <= '2024-01-20 23:59:59'
ORDER BY timestamp DESC;
```

### Graph Database (Neptune, Neo4j)

**Đặc điểm**: Nodes và relationships, tối ưu cho complex relationships và graph traversal queries.

**Use Case - Social Network Analysis**:
```cypher
// Create social network nodes và relationships
CREATE (alice:Person {
    name: 'Alice Nguyen',
    age: 28,
    city: 'Ho Chi Minh City',
    occupation: 'Software Engineer',
    interests: ['technology', 'travel', 'photography']
})

CREATE (bob:Person {
    name: 'Bob Tran',
    age: 32,
    city: 'Ha Noi',
    occupation: 'Product Manager',
    interests: ['business', 'sports', 'music']
})

CREATE (charlie:Person {
    name: 'Charlie Le',
    age: 25,
    city: 'Da Nang',
    occupation: 'Designer',
    interests: ['art', 'travel', 'food']
})

CREATE (techcompany:Company {
    name: 'TechViet Solutions',
    industry: 'Technology',
    size: 500,
    location: 'Ho Chi Minh City'
})

// Create relationships
CREATE (alice)-[:FRIENDS_WITH {since: '2020-01-15', strength: 'close'}]->(bob)
CREATE (bob)-[:FRIENDS_WITH {since: '2019-06-20', strength: 'medium'}]->(charlie)
CREATE (alice)-[:WORKS_FOR {position: 'Senior Developer', since: '2022-03-01'}]->(techcompany)
CREATE (alice)-[:LIKES {rating: 5}]->(:Interest {name: 'Photography'})
CREATE (alice)-[:VISITED {date: '2023-12-01', rating: 4}]->(:Place {name: 'Sapa', country: 'Vietnam'})

// Complex graph queries
// 1. Find friends of friends (2nd degree connections)
MATCH (alice:Person {name: 'Alice Nguyen'})-[:FRIENDS_WITH]->(friend)-[:FRIENDS_WITH]->(fof)
WHERE fof <> alice
RETURN DISTINCT fof.name as friend_of_friend, fof.city, fof.occupation;

// 2. Recommend friends based on mutual interests
MATCH (alice:Person {name: 'Alice Nguyen'})-[:LIKES]->(interest)<-[:LIKES]-(potential_friend)
WHERE NOT (alice)-[:FRIENDS_WITH]-(potential_friend)
  AND alice <> potential_friend
RETURN potential_friend.name, 
       potential_friend.city,
       COUNT(interest) as mutual_interests,
       COLLECT(interest.name) as shared_interests
ORDER BY mutual_interests DESC
LIMIT 5;

// 3. Find shortest path between two people
MATCH path = shortestPath(
    (alice:Person {name: 'Alice Nguyen'})-[:FRIENDS_WITH*]-(charlie:Person {name: 'Charlie Le'})
)
RETURN path, LENGTH(path) as degrees_of_separation;

// 4. Company network analysis
MATCH (company:Company {name: 'TechViet Solutions'})<-[:WORKS_FOR]-(employee)-[:FRIENDS_WITH]-(friend)
WHERE NOT (friend)-[:WORKS_FOR]->(company)
RETURN friend.name, 
       friend.occupation, 
       friend.city,
       COUNT(employee) as connections_in_company
ORDER BY connections_in_company DESC;

// 5. Influence analysis - find most connected people
MATCH (person:Person)-[r:FRIENDS_WITH]-()
RETURN person.name, 
       person.city,
       COUNT(r) as connection_count,
       AVG(CASE WHEN r.strength = 'close' THEN 3 
                WHEN r.strength = 'medium' THEN 2 
                ELSE 1 END) as avg_relationship_strength
ORDER BY connection_count DESC, avg_relationship_strength DESC
LIMIT 10;
```

**AWS Services cho NoSQL**:
- **DynamoDB**: Managed key-value/document store, serverless scaling
- **DocumentDB**: MongoDB-compatible document database
- **Neptune**: Managed graph database, supports Gremlin và SPARQL
- **ElastiCache**: In-memory key-value store (Redis/Memcached)
## 14. Caching Strategies

**Định nghĩa chi tiết**: Kỹ thuật lưu trữ tạm thời dữ liệu thường xuyên được truy cập trong memory hoặc faster storage để giảm latency và database load. Caching có thể implement ở multiple levels trong architecture.

**Cache Levels**:
- **L1 Cache**: Application memory (fastest, smallest)
- **L2 Cache**: Distributed cache (Redis, Memcached)
- **L3 Cache**: CDN (CloudFront, geographic distribution)
- **Database Buffer Pool**: DBMS internal caching

**Cache Patterns**:
- **Cache-Aside (Lazy Loading)**: Application manages cache
- **Write-Through**: Write to cache và database simultaneously
- **Write-Behind (Write-Back)**: Write to cache first, database later
- **Refresh-Ahead**: Proactively refresh cache before expiration

**Use Case - High-Traffic News Website (10M daily users)**:
```python
import redis
import json
import hashlib
from datetime import datetime, timedelta
from functools import wraps

# Multi-level caching implementation
class CacheManager:
    def __init__(self):
        # L1: Application memory cache
        self.memory_cache = {}
        self.memory_cache_size = 1000
        self.memory_ttl = 60  # 1 minute
        
        # L2: Redis distributed cache
        self.redis_client = redis.Redis(
            host='elasticache-cluster.abc123.cache.amazonaws.com',
            port=6379,
            decode_responses=True
        )
        
    def get_cache_key(self, prefix, *args, **kwargs):
        """Generate consistent cache key"""
        key_data = f"{prefix}:{':'.join(map(str, args))}"
        if kwargs:
            key_data += f":{json.dumps(kwargs, sort_keys=True)}"
        return hashlib.md5(key_data.encode()).hexdigest()[:16]
    
    def get_from_memory(self, key):
        """L1 Cache: Memory lookup"""
        if key in self.memory_cache:
            data, timestamp = self.memory_cache[key]
            if datetime.now() - timestamp < timedelta(seconds=self.memory_ttl):
                return data
            else:
                del self.memory_cache[key]
        return None
    
    def set_to_memory(self, key, data):
        """L1 Cache: Memory storage với LRU eviction"""
        if len(self.memory_cache) >= self.memory_cache_size:
            # Simple LRU: remove oldest entry
            oldest_key = min(self.memory_cache.keys(), 
                           key=lambda k: self.memory_cache[k][1])
            del self.memory_cache[oldest_key]
        
        self.memory_cache[key] = (data, datetime.now())
    
    def get_from_redis(self, key):
        """L2 Cache: Redis lookup"""
        try:
            data = self.redis_client.get(key)
            return json.loads(data) if data else None
        except:
            return None
    
    def set_to_redis(self, key, data, ttl=300):
        """L2 Cache: Redis storage"""
        try:
            self.redis_client.setex(key, ttl, json.dumps(data))
        except:
            pass  # Fail silently, fallback to database
    
    def cache_decorator(self, prefix, ttl=300, use_memory=True):
        """Decorator for automatic caching"""
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                # Generate cache key
                cache_key = self.get_cache_key(prefix, *args, **kwargs)
                
                # Try L1 cache first
                if use_memory:
                    result = self.get_from_memory(cache_key)
                    if result is not None:
                        return result
                
                # Try L2 cache
                result = self.get_from_redis(cache_key)
                if result is not None:
                    if use_memory:
                        self.set_to_memory(cache_key, result)
                    return result
                
                # Cache miss: execute function
                result = func(*args, **kwargs)
                
                # Store in caches
                self.set_to_redis(cache_key, result, ttl)
                if use_memory:
                    self.set_to_memory(cache_key, result)
                
                return result
            return wrapper
        return decorator

# Usage trong News Website
cache_manager = CacheManager()

class NewsService:
    def __init__(self, db_connection):
        self.db = db_connection
    
    @cache_manager.cache_decorator('article', ttl=1800)  # 30 minutes
    def get_article(self, article_id):
        """Get single article với caching"""
        cursor = self.db.cursor()
        cursor.execute("""
            SELECT a.article_id, a.title, a.content, a.published_at,
                   a.view_count, a.category_id, c.category_name,
                   u.author_name, u.author_bio
            FROM articles a
            JOIN categories c ON a.category_id = c.category_id
            JOIN users u ON a.author_id = u.user_id
            WHERE a.article_id = %s AND a.status = 'published'
        """, (article_id,))
        
        result = cursor.fetchone()
        return dict(result) if result else None
    
    @cache_manager.cache_decorator('trending', ttl=300)  # 5 minutes
    def get_trending_articles(self, limit=10):
        """Get trending articles với shorter TTL"""
        cursor = self.db.cursor()
        cursor.execute("""
            SELECT article_id, title, view_count, published_at
            FROM articles
            WHERE status = 'published'
              AND published_at >= DATE_SUB(NOW(), INTERVAL 24 HOUR)
            ORDER BY view_count DESC, published_at DESC
            LIMIT %s
        """, (limit,))
        
        return [dict(row) for row in cursor.fetchall()]
    
    @cache_manager.cache_decorator('category_articles', ttl=600)  # 10 minutes
    def get_articles_by_category(self, category_id, page=1, per_page=20):
        """Paginated articles by category"""
        offset = (page - 1) * per_page
        
        cursor = self.db.cursor()
        cursor.execute("""
            SELECT a.article_id, a.title, a.summary, a.published_at,
                   a.view_count, u.author_name
            FROM articles a
            JOIN users u ON a.author_id = u.user_id
            WHERE a.category_id = %s AND a.status = 'published'
            ORDER BY a.published_at DESC
            LIMIT %s OFFSET %s
        """, (category_id, per_page, offset))
        
        articles = [dict(row) for row in cursor.fetchall()]
        
        # Get total count for pagination
        cursor.execute("""
            SELECT COUNT(*) as total
            FROM articles
            WHERE category_id = %s AND status = 'published'
        """, (category_id,))
        
        total = cursor.fetchone()['total']
        
        return {
            'articles': articles,
            'pagination': {
                'page': page,
                'per_page': per_page,
                'total': total,
                'total_pages': (total + per_page - 1) // per_page
            }
        }
    
    def increment_view_count(self, article_id):
        """Update view count và invalidate cache"""
        # Update database
        cursor = self.db.cursor()
        cursor.execute("""
            UPDATE articles 
            SET view_count = view_count + 1,
                last_viewed_at = NOW()
            WHERE article_id = %s
        """, (article_id,))
        
        # Invalidate related caches
        article_cache_key = cache_manager.get_cache_key('article', article_id)
        trending_cache_key = cache_manager.get_cache_key('trending', 10)
        
        cache_manager.redis_client.delete(article_cache_key)
        cache_manager.redis_client.delete(trending_cache_key)
        
        # Remove from memory cache
        if article_cache_key in cache_manager.memory_cache:
            del cache_manager.memory_cache[article_cache_key]

# Cache warming strategy
class CacheWarmer:
    def __init__(self, news_service):
        self.news_service = news_service
    
    def warm_popular_content(self):
        """Pre-load popular content into cache"""
        # Warm trending articles
        self.news_service.get_trending_articles(20)
        
        # Warm popular categories
        popular_categories = [1, 2, 3, 4, 5]  # Technology, Sports, Politics, etc.
        for category_id in popular_categories:
            self.news_service.get_articles_by_category(category_id, page=1)
        
        print("Cache warming completed")

# Performance monitoring
def monitor_cache_performance():
    """Monitor cache hit ratios"""
    redis_info = cache_manager.redis_client.info()
    
    cache_stats = {
        'redis_hits': redis_info.get('keyspace_hits', 0),
        'redis_misses': redis_info.get('keyspace_misses', 0),
        'memory_cache_size': len(cache_manager.memory_cache),
        'redis_memory_usage': redis_info.get('used_memory_human', '0B')
    }
    
    if cache_stats['redis_hits'] + cache_stats['redis_misses'] > 0:
        hit_ratio = cache_stats['redis_hits'] / (cache_stats['redis_hits'] + cache_stats['redis_misses'])
        cache_stats['hit_ratio'] = round(hit_ratio * 100, 2)
    
    return cache_stats
```

**Cache Invalidation Strategies**:
```python
# 1. TTL-based expiration
cache_manager.redis_client.setex('user:123', 3600, user_data)  # 1 hour

# 2. Tag-based invalidation
def invalidate_user_caches(user_id):
    patterns = [
        f'user:{user_id}',
        f'user_posts:{user_id}:*',
        f'user_profile:{user_id}',
        'trending_users'  # Global cache affected by user changes
    ]
    
    for pattern in patterns:
        if '*' in pattern:
            keys = cache_manager.redis_client.keys(pattern)
            if keys:
                cache_manager.redis_client.delete(*keys)
        else:
            cache_manager.redis_client.delete(pattern)

# 3. Event-driven invalidation
class CacheInvalidator:
    def __init__(self, cache_manager):
        self.cache = cache_manager
    
    def on_article_published(self, article_id, category_id):
        """Invalidate caches when new article published"""
        keys_to_invalidate = [
            'trending',
            f'category_articles:{category_id}:*',
            'homepage_articles',
            'latest_articles'
        ]
        
        for key_pattern in keys_to_invalidate:
            if '*' in key_pattern:
                keys = self.cache.redis_client.keys(key_pattern)
                if keys:
                    self.cache.redis_client.delete(*keys)
            else:
                self.cache.redis_client.delete(key_pattern)
```

## 15. Database Replication

**Định nghĩa chi tiết**: Quá trình sao chép dữ liệu từ một database (master/primary) sang một hoặc nhiều databases khác (slave/replica) để đảm bảo high availability, load distribution và disaster recovery.

**Replication Types**:
- **Master-Slave**: Một master, nhiều read-only slaves
- **Master-Master**: Multiple masters có thể write
- **Circular Replication**: Daisy-chain replication
- **Hierarchical**: Multi-level replication topology

**Replication Methods**:
- **Synchronous**: Master chờ confirmation từ slaves trước khi commit
- **Asynchronous**: Master commit ngay, replicate sau
- **Semi-synchronous**: Chờ ít nhất 1 slave confirm

**Use Case - Global E-commerce Platform (Multi-region)**:
```sql
-- Master Database Configuration (Singapore - ap-southeast-1)
-- Primary region handling all writes
CREATE DATABASE ecommerce_master;

-- Configure binary logging for replication
SET GLOBAL log_bin = ON;
SET GLOBAL server_id = 1;
SET GLOBAL binlog_format = 'ROW';  -- Safer for replication
SET GLOBAL sync_binlog = 1;        -- Durability
SET GLOBAL innodb_flush_log_at_trx_commit = 1;

-- Create replication user
CREATE USER 'repl_user'@'%' IDENTIFIED BY 'secure_replication_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl_user'@'%';
FLUSH PRIVILEGES;

-- Show master status
SHOW MASTER STATUS;
-- +------------------+----------+--------------+------------------+
-- | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
-- +------------------+----------+--------------+------------------+
-- | mysql-bin.000001 |      154 |              |                  |
-- +------------------+----------+--------------+------------------+
```

```sql
-- Read Replica Configuration (Tokyo - ap-northeast-1)
-- Serve Japanese customers với low latency
SET GLOBAL server_id = 2;
SET GLOBAL read_only = ON;  -- Prevent writes to replica

-- Configure replication
CHANGE MASTER TO
    MASTER_HOST = 'master.ecommerce.internal',
    MASTER_USER = 'repl_user',
    MASTER_PASSWORD = 'secure_replication_password',
    MASTER_LOG_FILE = 'mysql-bin.000001',
    MASTER_LOG_POS = 154,
    MASTER_SSL = 1;

-- Start replication
START SLAVE;

-- Monitor replication status
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: 0
-- Slave_IO_Running: Yes
-- Slave_SQL_Running: Yes
-- Last_Error: 
```

**Application-level Read/Write Splitting**:
```python
import mysql.connector
from mysql.connector import pooling
import random

class DatabaseManager:
    def __init__(self):
        # Master connection pool (writes)
        self.master_pool = pooling.MySQLConnectionPool(
            pool_name="master_pool",
            pool_size=20,
            pool_reset_session=True,
            host="master.ecommerce.internal",
            database="ecommerce",
            user="app_user",
            password="app_password",
            autocommit=False
        )
        
        # Read replica pools (reads)
        self.replica_pools = {
            'tokyo': pooling.MySQLConnectionPool(
                pool_name="tokyo_pool",
                pool_size=30,
                host="replica-tokyo.ecommerce.internal",
                database="ecommerce",
                user="readonly_user",
                password="readonly_password",
                autocommit=True
            ),
            'sydney': pooling.MySQLConnectionPool(
                pool_name="sydney_pool", 
                pool_size=25,
                host="replica-sydney.ecommerce.internal",
                database="ecommerce",
                user="readonly_user",
                password="readonly_password",
                autocommit=True
            ),
            'mumbai': pooling.MySQLConnectionPool(
                pool_name="mumbai_pool",
                pool_size=35,
                host="replica-mumbai.ecommerce.internal", 
                database="ecommerce",
                user="readonly_user",
                password="readonly_password",
                autocommit=True
            )
        }
    
    def get_write_connection(self):
        """Get connection to master for writes"""
        return self.master_pool.get_connection()
    
    def get_read_connection(self, region_preference=None):
        """Get connection to read replica"""
        if region_preference and region_preference in self.replica_pools:
            return self.replica_pools[region_preference].get_connection()
        
        # Load balance across all replicas
        replica_name = random.choice(list(self.replica_pools.keys()))
        return self.replica_pools[replica_name].get_connection()
    
    def execute_write_query(self, query, params=None):
        """Execute write query on master"""
        conn = self.get_write_connection()
        cursor = conn.cursor(dictionary=True)
        
        try:
            cursor.execute(query, params)
            conn.commit()
            
            if cursor.lastrowid:
                return cursor.lastrowid
            return cursor.rowcount
            
        except Exception as e:
            conn.rollback()
            raise e
        finally:
            cursor.close()
            conn.close()
    
    def execute_read_query(self, query, params=None, region=None):
        """Execute read query on replica"""
        conn = self.get_read_connection(region)
        cursor = conn.cursor(dictionary=True)
        
        try:
            cursor.execute(query, params)
            return cursor.fetchall()
        finally:
            cursor.close()
            conn.close()

# Usage examples
db = DatabaseManager()

class OrderService:
    def __init__(self, db_manager):
        self.db = db_manager
    
    def create_order(self, customer_id, items, region='singapore'):
        """Write operation - always goes to master"""
        order_query = """
            INSERT INTO orders (customer_id, total_amount, status, created_at, region)
            VALUES (%s, %s, 'pending', NOW(), %s)
        """
        
        total_amount = sum(item['price'] * item['quantity'] for item in items)
        order_id = self.db.execute_write_query(
            order_query, 
            (customer_id, total_amount, region)
        )
        
        # Insert order items
        for item in items:
            item_query = """
                INSERT INTO order_items (order_id, product_id, quantity, price)
                VALUES (%s, %s, %s, %s)
            """
            self.db.execute_write_query(
                item_query,
                (order_id, item['product_id'], item['quantity'], item['price'])
            )
        
        return order_id
    
    def get_order_history(self, customer_id, region='tokyo'):
        """Read operation - can use regional replica"""
        query = """
            SELECT o.order_id, o.total_amount, o.status, o.created_at,
                   COUNT(oi.item_id) as item_count
            FROM orders o
            LEFT JOIN order_items oi ON o.order_id = oi.order_id
            WHERE o.customer_id = %s
            GROUP BY o.order_id
            ORDER BY o.created_at DESC
            LIMIT 20
        """
        
        return self.db.execute_read_query(query, (customer_id,), region)
    
    def get_order_analytics(self, start_date, end_date, region='mumbai'):
        """Analytics query - use replica to avoid impacting master"""
        query = """
            SELECT 
                DATE(o.created_at) as order_date,
                o.region,
                COUNT(*) as order_count,
                SUM(o.total_amount) as total_revenue,
                AVG(o.total_amount) as avg_order_value
            FROM orders o
            WHERE o.created_at BETWEEN %s AND %s
              AND o.status IN ('completed', 'shipped')
            GROUP BY DATE(o.created_at), o.region
            ORDER BY order_date DESC, total_revenue DESC
        """
        
        return self.db.execute_read_query(
            query, 
            (start_date, end_date), 
            region
        )
```

**AWS Aurora Global Database**:
```python
# Aurora Global Database configuration
aurora_global_config = {
    "primary_region": "ap-southeast-1",  # Singapore
    "secondary_regions": [
        "ap-northeast-1",  # Tokyo
        "ap-south-1",      # Mumbai  
        "ap-southeast-2"   # Sydney
    ],
    "cross_region_replication_lag": "< 1 second",
    "rpo": "1 second",     # Recovery Point Objective
    "rto": "< 1 minute"    # Recovery Time Objective
}

# Connection strings for Aurora Global
connections = {
    "primary_writer": "aurora-global-cluster.cluster-xyz.ap-southeast-1.rds.amazonaws.com",
    "primary_reader": "aurora-global-cluster.cluster-ro-xyz.ap-southeast-1.rds.amazonaws.com",
    "tokyo_reader": "aurora-global-cluster-tokyo.cluster-ro-abc.ap-northeast-1.rds.amazonaws.com",
    "mumbai_reader": "aurora-global-cluster-mumbai.cluster-ro-def.ap-south-1.rds.amazonaws.com"
}

# Failover scenario
def handle_primary_region_failure():
    """Promote secondary region to primary"""
    # 1. Detect primary region failure
    # 2. Promote Tokyo region to primary
    # 3. Update application configuration
    # 4. Redirect write traffic to new primary
    
    new_primary_config = {
        "writer_endpoint": "aurora-global-cluster-tokyo.cluster-xyz.ap-northeast-1.rds.amazonaws.com",
        "failover_time": "< 60 seconds",
        "data_loss": "< 1 second of transactions"
    }
    
    return new_primary_config
```

**Monitoring Replication Health**:
```sql
-- Monitor replication lag
SELECT 
    CASE 
        WHEN Seconds_Behind_Master IS NULL THEN 'Replication Stopped'
        WHEN Seconds_Behind_Master = 0 THEN 'Up to Date'
        WHEN Seconds_Behind_Master < 5 THEN 'Minimal Lag'
        WHEN Seconds_Behind_Master < 30 THEN 'Moderate Lag'
        ELSE 'High Lag'
    END as replication_status,
    Seconds_Behind_Master as lag_seconds,
    Slave_IO_Running,
    Slave_SQL_Running,
    Last_Error
FROM information_schema.REPLICA_HOST_STATUS;

-- Monitor binary log position
SHOW MASTER STATUS;
SHOW SLAVE STATUS\G
```
