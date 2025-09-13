Cơ sở dữ liệu (Database) là gì?

Là tập hợp có tổ chức của dữ liệu (data) được lưu trữ trong máy tính.

Dữ liệu này có thể là cấu trúc (structured, như bảng trong SQL) hoặc bán cấu trúc/phi cấu trúc (semi-structured/unstructured, như JSON, tài liệu, hình ảnh).

Mục tiêu: để nhiều người dùng hoặc ứng dụng có thể lưu, tìm kiếm, chỉnh sửa, và quản lý dữ liệu cùng lúc, một cách hiệu quả và an toàn

FCJ - Week 6 Database

.

Ví dụ đời thường:

Nghĩ cơ sở dữ liệu giống như một thư viện:

Sách = dữ liệu.

Danh mục sách = index.

Mã số sách = primary key.

Ghi chú tham khảo trong sách = foreign key.

Người mượn sách = session (khoảng thời gian bạn kết nối vào thư viện).

Điểm quan trọng của cơ sở dữ liệu:

Tổ chức: dữ liệu không lưu lộn xộn mà có mô hình rõ ràng (bảng, tài liệu, key-value…).

Truy cập đồng thời: nhiều user/app cùng làm việc với dữ liệu.

Quản lý nhất quán: đảm bảo dữ liệu chính xác, toàn vẹn dù có lỗi, mất điện, hay hệ thống sập.

👉 Nói ngắn gọn:
Cơ sở dữ liệu là nơi lưu trữ dữ liệu có tổ chức, giúp ta dễ dàng quản lý, truy vấn và đảm bảo tính toàn vẹn của thông tin.


1. Database (Cơ sở dữ liệu)

Chỉ là nơi lưu dữ liệu.

Ví dụ: file chứa bảng “Khách hàng” với tên, địa chỉ, số điện thoại…

Nó giống như kho sách trong thư viện – có rất nhiều sách (dữ liệu) nhưng bản thân kho không biết cách quản lý mượn trả.

2. DBMS (Database Management System – Hệ quản trị cơ sở dữ liệu)

Là phần mềm quản lý Database.

Nó cung cấp công cụ để:

Tạo, đọc, cập nhật, xóa dữ liệu (CRUD).

Quản lý người dùng, phân quyền.

Đảm bảo tính toàn vẹn và bảo mật.

Tối ưu truy vấn để chạy nhanh hơn.

Ví dụ: MySQL, PostgreSQL, Oracle, SQL Server, DynamoDB.

Trong ví dụ thư viện: DBMS chính là thủ thư + hệ thống mượn trả – họ quản lý sách, theo dõi ai mượn, sắp xếp lại khi trả.

3. Mối quan hệ

Database = dữ liệu (data).

DBMS = công cụ/quản lý để làm việc với dữ liệu.

Cả hai kết hợp lại mới thành hệ thống cơ sở dữ liệu mà bạn dùng hằng ngày.

👉 Mnemonic để nhớ:

Database = What (cái gì được lưu)

DBMS = How (làm sao để quản lý và truy cập dữ liệu)


🔹 1. Database

Khái niệm: Tập hợp dữ liệu có tổ chức, được lưu trữ để phục vụ cho việc truy cập, quản lý, và xử lý.

Ví dụ đời thường:

Danh bạ điện thoại lưu số liên hệ.

Thư viện sách với danh mục rõ ràng.

Use case thực tế:

Ngân hàng lưu thông tin khách hàng, số dư, lịch sử giao dịch.

E-commerce lưu sản phẩm, đơn hàng, khách hàng.

AWS: Dùng Amazon RDS (MySQL, PostgreSQL) để lưu dữ liệu ứng dụng web bán hàng.

🔹 2. DBMS (Database Management System)

Khái niệm: Phần mềm quản lý, giúp người dùng/ứng dụng tương tác với database mà không cần can thiệp trực tiếp vào file dữ liệu.

Ví dụ đời thường:

MySQL Workbench, pgAdmin (công cụ quản lý DB).

Use case thực tế:

Công ty vận tải dùng Oracle DB để quản lý vé, khách hàng, chuyến đi.

AWS: Amazon Aurora (DBMS do AWS quản lý, tương thích MySQL/Postgres, tối ưu hiệu năng đọc/ghi).

🔹 3. Primary Key

Khái niệm: Cột (hoặc nhóm cột) duy nhất để xác định một bản ghi trong bảng.

Ví dụ:

Bảng KHÁCH HÀNG: customer_id là Primary Key.

Use case thực tế:

Ngân hàng: account_number để phân biệt từng tài khoản.

Trường đại học: student_id để phân biệt sinh viên.

AWS: DynamoDB tự động coi Partition key là Primary Key để xác định từng item.

🔹 4. Foreign Key

Khái niệm: Cột trong bảng này tham chiếu tới Primary Key của bảng khác → tạo liên kết dữ liệu.

Ví dụ:

Bảng ĐƠN HÀNG có customer_id là Foreign Key tham chiếu đến bảng KHÁCH HÀNG.

Use case thực tế:

E-commerce: liên kết đơn hàng với khách hàng.

HR system: liên kết nhân viên với phòng ban.

AWS: Trong Aurora (quan hệ), Foreign Key giúp liên kết bảng "orders" với "customers".

🔹 5. Index

Khái niệm: Cấu trúc dữ liệu đặc biệt giúp tìm kiếm nhanh hơn. Giống như mục lục trong sách.

Ví dụ:

Tìm khách hàng theo email thay vì quét hết bảng.

Use case thực tế:

App thương mại điện tử cần tìm sản phẩm theo tên rất nhanh.

AWS: DynamoDB có Global Secondary Index (GSI) để tìm item theo cột khác ngoài Primary Key.

🔹 6. Partition

Khái niệm: Chia bảng lớn thành nhiều phần nhỏ để tăng tốc truy vấn.

Ví dụ:

Bảng giao dịch ngân hàng chia theo tháng/năm.

Use case thực tế:

Log dữ liệu IoT chia theo ngày.

AWS: DynamoDB partition key để phân tán dữ liệu trên nhiều node → scale đến hàng triệu request/giây.

🔹 7. Database Log

Khái niệm: Nhật ký lưu thay đổi dữ liệu, dùng để khôi phục hoặc đồng bộ.

Ví dụ:

Ghi lại việc rút 500k từ tài khoản.

Use case thực tế:

Khôi phục dữ liệu khi hệ thống sập.

AWS: RDS có transaction log → Point-in-time Recovery (PITR).

🔹 8. Buffer

Khái niệm: Vùng nhớ tạm trong RAM để đọc/ghi dữ liệu nhanh hơn.

Ví dụ:

Copy file từ USB vào PC, trước hết nó nằm trong buffer RAM rồi mới ghi vào ổ cứng.

Use case thực tế:

Giúp xử lý truy vấn hàng ngàn user cùng lúc.

AWS: Aurora sử dụng buffer cache để tăng tốc độ đọc ghi.

🔹 9. OLTP vs OLAP

OLTP (Online Transaction Processing): Xử lý giao dịch nhỏ, nhanh, thường xuyên.

Ví dụ: rút tiền ATM, đặt hàng online.

AWS: RDS, Aurora.

OLAP (Online Analytical Processing): Phân tích dữ liệu lớn, phức tạp, thường theo batch.

Ví dụ: báo cáo doanh thu năm, phân tích hành vi khách hàng.

AWS: Redshift, Athena, Timestream.

Mnemonic để nhớ nhanh

Database = “Kho chứa dữ liệu”

DBMS = “Quản lý kho”

Primary Key = “Mã số duy nhất”

Foreign Key = “Tham chiếu chéo”

Index = “Mục lục tăng tốc”

Partition = “Chia nhỏ để nhanh”

Log = “Nhật ký thay đổi”

Buffer = “Bộ nhớ đệm tăng tốc”

OLTP vs OLAP = “Giao dịch nhanh” vs “Phân tích sâu”

🔹 10. Session (Phiên làm việc)

Khái niệm: Khoảng thời gian từ khi kết nối vào database cho đến khi ngắt kết nối.

Ví dụ: Bạn mở ứng dụng ngân hàng lúc 9h, đăng nhập, thao tác, và thoát ra lúc 9h15 → session kéo dài 15 phút.

Use case: Quản lý kết nối, bảo mật (tự động hết hạn nếu idle).

AWS: Với RDS, session được theo dõi qua Performance Insights để tối ưu connection pool.

🔹 11. Execution Plan (Query Plan – Kế hoạch thực thi truy vấn)

Khái niệm: Chuỗi các bước mà DBMS chọn để lấy dữ liệu. Do query optimizer quyết định.

Ví dụ: Với câu lệnh SELECT * FROM orders WHERE amount > 100, DBMS có thể:

Quét toàn bộ bảng (table scan).

Hoặc dùng index nếu có.

Use case: Developer dùng EXPLAIN để debug và tối ưu SQL.

AWS: Aurora, RDS cho phép xem query plan để tối ưu hiệu năng.

🔹 12. Data Warehouse

Khái niệm: Kho dữ liệu lớn, chứa dữ liệu tổng hợp từ nhiều nguồn (OLTP, file log, app).

Ví dụ: Công ty bán lẻ gom dữ liệu từ POS, website, app để phân tích hành vi mua sắm.

AWS: Amazon Redshift – dùng columnar storage + MPP cho phân tích OLAP.

🔹 13. Types of NoSQL Database

Document Store: MongoDB, DynamoDB → lưu JSON/document.

Ví dụ: Hồ sơ khách hàng (profile, setting).

Key-Value Store: DynamoDB, Redis.

Ví dụ: Lưu session token, cache.

Wide-Column Store: Cassandra.

Ví dụ: Lưu dữ liệu IoT theo timestamp.

Graph Store: Neo4j, Neptune.

Ví dụ: Phân tích mạng xã hội, quan hệ giữa user.

🔹 14. Caching

Khái niệm: Lưu tạm dữ liệu hay được truy cập trong bộ nhớ để giảm tải DB.

Ví dụ: Lưu kết quả truy vấn “top 10 sản phẩm bán chạy” trong cache để không phải query DB mỗi lần.

AWS: ElastiCache (Redis/Memcached) đứng trước RDS/Aurora để tăng tốc.

🔹 15. Replication (Nhân bản dữ liệu)

Khái niệm: Sao chép dữ liệu từ DB chính sang DB phụ để tăng tính sẵn sàng hoặc phục vụ read.

Ví dụ:

Master-Slave trong MySQL.

Read Replica trong PostgreSQL.

AWS:

RDS có Multi-AZ (failover tự động).

Aurora có Global Database (multi-region).

🔹 16. Backup & Recovery

Khái niệm: Lưu bản sao DB để khôi phục khi mất mát.

Ví dụ: Công ty lưu backup hàng ngày trên tape/S3.

AWS: RDS hỗ trợ backup tự động (7–35 ngày) + Point-in-time recovery.

🔹 17. Transaction (Giao dịch)

Khái niệm: Nhóm nhiều thao tác SQL thành một đơn vị. Nếu 1 thao tác fail → rollback tất cả.

Ví dụ: Chuyển tiền 500k:

Trừ tài khoản A.

Cộng tài khoản B.

Nếu 1 bước lỗi → không ai bị mất tiền.

AWS: Aurora/RDS hỗ trợ transaction theo chuẩn ACID.

🔹 18. ACID vs BASE

ACID (SQL): Atomicity, Consistency, Isolation, Durability → đảm bảo tính toàn vẹn dữ liệu.

BASE (NoSQL): Basically Available, Soft-state, Eventual consistency → chấp nhận dữ liệu chưa đồng bộ ngay, để scale lớn.

Ví dụ:

Ngân hàng → ACID.

Mạng xã hội (like, comment) → BASE.

👉 Tóm lại: ngoài mấy khái niệm nền (PK, FK, Index, Partition…), còn có Session, Execution Plan, Replication, Caching, Transaction, ACID/BASE, Data Warehouse, NoSQL types… – tất cả đều quan trọng khi bước vào AWS Database Services.

🔹 19. Schema

Khái niệm: Bản thiết kế (blueprint) mô tả cấu trúc dữ liệu trong DB (bảng nào, cột nào, kiểu dữ liệu, quan hệ giữa bảng).

Ví dụ: Trong hệ thống trường học: schema có bảng Students, Courses, Enrollments.

AWS: Khi migrate Oracle → Aurora PostgreSQL, phải dùng AWS SCT (Schema Conversion Tool) để chuyển đổi schema.

🔹 20. Normalization & Denormalization

Normalization (Chuẩn hóa): Tổ chức dữ liệu để giảm trùng lặp, tăng toàn vẹn.

Ví dụ: tách bảng Customers và Orders thay vì lưu thông tin khách hàng lặp lại trong từng order.

Denormalization: Gộp dữ liệu để tăng tốc truy vấn, chấp nhận trùng lặp.

Ví dụ: trong Data Warehouse, lưu sẵn customer_name trong bảng Orders.

AWS: OLTP DB (Aurora, RDS) thường chuẩn hóa, còn Redshift (OLAP) thường denormalize.

🔹 21. Sharding

Khái niệm: Chia nhỏ dữ liệu thành nhiều DB khác nhau để phân tán tải.

Ví dụ: App mạng xã hội có 100 triệu user → chia theo khu vực (Asia DB, Europe DB, US DB).

AWS: DynamoDB auto-shard theo partition key; Aurora có thể sharding thủ công bằng application logic.

🔹 22. Concurrency Control

Khái niệm: Quản lý nhiều transaction cùng lúc, tránh xung đột.

Ví dụ:

Hai người cùng đặt mua chiếc iPhone cuối cùng → hệ thống phải đảm bảo chỉ 1 người thành công.

Cơ chế: Locking (pessimistic), Optimistic concurrency.

AWS: Aurora hỗ trợ row-level locking; DynamoDB có conditional writes để tránh race condition.

🔹 23. Isolation Levels

Khái niệm: Mức độ tách biệt của transaction khi chạy song song.

Ví dụ:

Read Uncommitted: có thể thấy dữ liệu chưa commit → dirty read.

Serializable: mức cao nhất, như thể transaction chạy tuần tự.

Use case: Ngân hàng dùng mức cao; báo cáo analytics có thể dùng mức thấp hơn để tăng tốc.

🔹 24. Stored Procedures & Triggers

Stored Procedure: Đoạn code SQL được lưu sẵn, chạy nhiều lần.

Ví dụ: hàm calculate_bonus(employee_id) để tính thưởng cuối năm.

Trigger: Tự động chạy khi có sự kiện (INSERT, UPDATE, DELETE).

Ví dụ: khi thêm order mới, trigger gửi thông báo.

AWS: Aurora/RDS hỗ trợ cả hai (tùy engine: MySQL, PostgreSQL, SQL Server).

🔹 25. View

Khái niệm: Bảng ảo được tạo từ truy vấn SELECT.

Ví dụ: View high_value_customers chỉ hiển thị khách hàng có chi tiêu > 1 triệu.

Use case: Bảo mật (user chỉ thấy view, không truy cập bảng gốc).

🔹 26. ETL (Extract – Transform – Load)

Khái niệm: Quy trình lấy dữ liệu từ nguồn, biến đổi, rồi tải vào DB/kho dữ liệu.

Ví dụ: Lấy dữ liệu từ hệ thống POS, làm sạch, nạp vào Redshift để phân tích.

AWS: AWS Glue (ETL serverless), kết hợp với S3 + Redshift.

🔹 27. Data Lake vs Data Warehouse

Data Lake: Lưu dữ liệu thô, đủ loại định dạng (ảnh, video, JSON, log). → S3.

Data Warehouse: Dữ liệu đã xử lý, cấu trúc rõ ràng để phân tích nhanh. → Redshift.

Use case:

Data Lake = nơi chứa “nguyên liệu”.

Data Warehouse = nơi chứa “thành phẩm” cho BI/Analytics.

🔹 28. CAP Theorem

Khái niệm: Một hệ thống phân tán chỉ đảm bảo tối đa 2 trong 3:

Consistency (nhất quán).

Availability (sẵn sàng).

Partition tolerance (chịu lỗi mạng).

Ví dụ:

DynamoDB ưu tiên AP (sẵn sàng + chịu lỗi), nhưng cho chọn strong consistency khi cần.

Aurora ưu tiên CP.

👉 Như vậy, ngoài những gì đã học, còn có schema, chuẩn hóa, sharding, concurrency, isolation, stored procedure, trigger, view, ETL, data lake vs warehouse, CAP theorem… → toàn là khái niệm quan trọng khi đi sâu vào AWS Database.

🔹 19. Schema

Khái niệm: Bản thiết kế (blueprint) mô tả cấu trúc dữ liệu trong DB (bảng nào, cột nào, kiểu dữ liệu, quan hệ giữa bảng).

Ví dụ: Trong hệ thống trường học: schema có bảng Students, Courses, Enrollments.

AWS: Khi migrate Oracle → Aurora PostgreSQL, phải dùng AWS SCT (Schema Conversion Tool) để chuyển đổi schema.

🔹 20. Normalization & Denormalization

Normalization (Chuẩn hóa): Tổ chức dữ liệu để giảm trùng lặp, tăng toàn vẹn.

Ví dụ: tách bảng Customers và Orders thay vì lưu thông tin khách hàng lặp lại trong từng order.

Denormalization: Gộp dữ liệu để tăng tốc truy vấn, chấp nhận trùng lặp.

Ví dụ: trong Data Warehouse, lưu sẵn customer_name trong bảng Orders.

AWS: OLTP DB (Aurora, RDS) thường chuẩn hóa, còn Redshift (OLAP) thường denormalize.

🔹 21. Sharding

Khái niệm: Chia nhỏ dữ liệu thành nhiều DB khác nhau để phân tán tải.

Ví dụ: App mạng xã hội có 100 triệu user → chia theo khu vực (Asia DB, Europe DB, US DB).

AWS: DynamoDB auto-shard theo partition key; Aurora có thể sharding thủ công bằng application logic.

🔹 22. Concurrency Control

Khái niệm: Quản lý nhiều transaction cùng lúc, tránh xung đột.

Ví dụ:

Hai người cùng đặt mua chiếc iPhone cuối cùng → hệ thống phải đảm bảo chỉ 1 người thành công.

Cơ chế: Locking (pessimistic), Optimistic concurrency.

AWS: Aurora hỗ trợ row-level locking; DynamoDB có conditional writes để tránh race condition.

🔹 23. Isolation Levels

Khái niệm: Mức độ tách biệt của transaction khi chạy song song.

Ví dụ:

Read Uncommitted: có thể thấy dữ liệu chưa commit → dirty read.

Serializable: mức cao nhất, như thể transaction chạy tuần tự.

Use case: Ngân hàng dùng mức cao; báo cáo analytics có thể dùng mức thấp hơn để tăng tốc.

🔹 24. Stored Procedures & Triggers

Stored Procedure: Đoạn code SQL được lưu sẵn, chạy nhiều lần.

Ví dụ: hàm calculate_bonus(employee_id) để tính thưởng cuối năm.

Trigger: Tự động chạy khi có sự kiện (INSERT, UPDATE, DELETE).

Ví dụ: khi thêm order mới, trigger gửi thông báo.

AWS: Aurora/RDS hỗ trợ cả hai (tùy engine: MySQL, PostgreSQL, SQL Server).

🔹 25. View

Khái niệm: Bảng ảo được tạo từ truy vấn SELECT.

Ví dụ: View high_value_customers chỉ hiển thị khách hàng có chi tiêu > 1 triệu.

Use case: Bảo mật (user chỉ thấy view, không truy cập bảng gốc).

🔹 26. ETL (Extract – Transform – Load)

Khái niệm: Quy trình lấy dữ liệu từ nguồn, biến đổi, rồi tải vào DB/kho dữ liệu.

Ví dụ: Lấy dữ liệu từ hệ thống POS, làm sạch, nạp vào Redshift để phân tích.

AWS: AWS Glue (ETL serverless), kết hợp với S3 + Redshift.

🔹 27. Data Lake vs Data Warehouse

Data Lake: Lưu dữ liệu thô, đủ loại định dạng (ảnh, video, JSON, log). → S3.

Data Warehouse: Dữ liệu đã xử lý, cấu trúc rõ ràng để phân tích nhanh. → Redshift.

Use case:

Data Lake = nơi chứa “nguyên liệu”.

Data Warehouse = nơi chứa “thành phẩm” cho BI/Analytics.

🔹 28. CAP Theorem

Khái niệm: Một hệ thống phân tán chỉ đảm bảo tối đa 2 trong 3:

Consistency (nhất quán).

Availability (sẵn sàng).

Partition tolerance (chịu lỗi mạng).

Ví dụ:

DynamoDB ưu tiên AP (sẵn sàng + chịu lỗi), nhưng cho chọn strong consistency khi cần.

Aurora ưu tiên CP.

👉 Như vậy, ngoài những gì đã học, còn có schema, chuẩn hóa, sharding, concurrency, isolation, stored procedure, trigger, view, ETL, data lake vs warehouse, CAP theorem… → toàn là khái niệm quan trọng khi đi sâu vào AWS Database.

🔹 29. Data Model

Khái niệm: Cách tổ chức dữ liệu (entities, quan hệ, thuộc tính).

Loại chính: Relational, Document, Key-Value, Graph, Time-series.

AWS: Aurora (relational), DynamoDB (key-value/document), Neptune (graph), Timestream (time-series).

🔹 30. Metadata

Khái niệm: Dữ liệu mô tả dữ liệu (data about data).

Ví dụ: file ảnh có metadata về ngày chụp, camera.

AWS: Glue Data Catalog lưu metadata để query qua Athena.

🔹 31. Data Integrity

Khái niệm: Đảm bảo dữ liệu chính xác, không bị sai hoặc mâu thuẫn.

Ví dụ: Một đơn hàng không thể có số lượng âm.

AWS: Aurora/RDS enforce constraints (NOT NULL, CHECK, UNIQUE).

🔹 32. Constraints

Khái niệm: Quy tắc áp dụng trên cột/bảng để đảm bảo toàn vẹn.

Ví dụ:

NOT NULL → cột bắt buộc có giá trị.

UNIQUE → giá trị không được trùng.

AWS: Aurora/MySQL/Postgres hỗ trợ constraints đầy đủ.

🔹 33. Data Redundancy

Khái niệm: Dữ liệu bị lặp lại. Có thể gây lãng phí và sai lệch.

Ví dụ: thông tin khách hàng xuất hiện lặp trong nhiều bảng.

Giải pháp: Normalization hoặc caching hợp lý.

🔹 34. Deadlock

Khái niệm: Hai transaction chờ nhau mãi mãi vì đã khóa tài nguyên mà bên kia cần.

Ví dụ: Transaction A giữ khóa bảng X, chờ bảng Y; Transaction B giữ Y, chờ X.

AWS: Aurora phát hiện và rollback transaction để tránh deadlock.

🔹 35. Data Replication Modes

Synchronous: Ghi dữ liệu vào bản chính và bản sao cùng lúc (an toàn nhưng chậm hơn).

Asynchronous: Ghi bản chính trước, đồng bộ bản sao sau (nhanh hơn nhưng có thể mất dữ liệu khi fail).

AWS:

Multi-AZ RDS = synchronous.

Read Replica = asynchronous.

🔹 36. Materialized View

Khái niệm: View nhưng được lưu trữ dữ liệu thực, không phải bảng ảo.

Ví dụ: báo cáo doanh số theo tháng, lưu sẵn để query nhanh.

AWS: Redshift hỗ trợ materialized views để tăng tốc phân tích OLAP.

🔹 37. Star Schema & Snowflake Schema

Khái niệm: Cách tổ chức dữ liệu trong Data Warehouse.

Star: bảng fact ở trung tâm, liên kết dimension.

Snowflake: dimension được chuẩn hóa thành nhiều bảng nhỏ.

AWS: Thiết kế schema trong Redshift/Data Warehouse.

🔹 38. Data Migration

Khái niệm: Chuyển dữ liệu từ hệ thống này sang hệ thống khác.

AWS: DMS (Database Migration Service) hỗ trợ di chuyển Oracle → Aurora, SQL Server → PostgreSQL…

🔹 39. Data Governance

Khái niệm: Bộ quy tắc quản lý dữ liệu (ai được xem, ai được sửa, tuân thủ chuẩn gì).

AWS: Lake Formation, IAM policies cho S3/DB.

🔹 40. Data Security

Khái niệm: Bảo vệ dữ liệu khỏi truy cập trái phép.

Cách thực hiện: mã hóa (at rest, in transit), phân quyền.

AWS: KMS, IAM, Security Group cho RDS.

🔹 41. Entity & Attribute

Entity: đối tượng cần quản lý (VD: Khách hàng, Nhân viên).

Attribute: thuộc tính mô tả entity (VD: tên, tuổi, địa chỉ).

🔹 42. Entity-Relationship Diagram (ERD)

Bản vẽ mô tả các bảng (entity) và quan hệ (relationship) giữa chúng.

Thường dùng khi thiết kế hệ thống.

🔹 43. Relational Algebra & Relational Calculus

Nền tảng lý thuyết của SQL.

Cho thấy cách dữ liệu được thao tác bằng phép toán tập hợp.

🔹 44. Data Dictionary / Catalog

Kho siêu dữ liệu (metadata) mô tả cấu trúc DB, quyền, quan hệ.

AWS: Glue Data Catalog.

🔹 45. Denormalization Techniques

Lặp dữ liệu để tối ưu đọc, ví dụ thêm cột customer_name trực tiếp trong bảng orders.

🔹 46. Concurrency Anomalies

Dirty Read, Non-repeatable Read, Phantom Read → vấn đề khi isolation level thấp.

🔹 47. Locking Mechanisms

Table-level lock, Row-level lock, Page lock.

AWS Aurora dùng row-level locking để tránh tắc nghẽn.

🔹 48. Query Optimization

Kỹ thuật cải thiện tốc độ: dùng index, rewrite query, partitioning.

DBMS dùng query optimizer để tự chọn execution plan.

🔹 49. Data Archiving

Lưu dữ liệu cũ sang hệ thống rẻ hơn (ví dụ: S3 Glacier) để giảm tải DB chính.

🔹 50. Data Lifecycle Management

Quản lý dữ liệu từ lúc sinh ra → dùng → lưu trữ → hủy.

AWS: S3 Lifecycle Policy, RDS backup retention.

🔹 51. Eventual Consistency vs Strong Consistency

NoSQL thường eventual (DynamoDB), SQL thường strong.

🔹 52. Polyglot Persistence

Dùng nhiều loại DB cùng lúc trong một hệ thống (VD: RDS cho giao dịch, DynamoDB cho cache, Redshift cho phân tích).

🔹 53. Big Data Integration

Database truyền thống kết hợp với hệ thống Big Data (Hadoop, Spark).

AWS: Redshift Spectrum query dữ liệu trong S3.

🔹 54. Streaming Data

Dữ liệu đến liên tục (IoT, log).

AWS: Kinesis + Timestream cho time-series.

🔹 55. Data Federation / Virtualization

Query nhiều nguồn dữ liệu khác nhau như một DB duy nhất.

AWS: Athena + Redshift Spectrum query trực tiếp trên S3.

🔹 56. High Availability (HA) & Fault Tolerance

DB phải luôn online, ngay cả khi có lỗi.

AWS: RDS Multi-AZ, Aurora Global DB.

🔹 57. Scalability

Vertical scaling (tăng cấu hình máy).

Horizontal scaling (thêm node, sharding, replicas).

🔹 58. Disaster Recovery (DR)

Chiến lược khôi phục DB khi có sự cố lớn.

AWS: Cross-region snapshot, Aurora Global Database.

🔹 59. Data Masking / Anonymization

Che giấu dữ liệu nhạy cảm (VD: số thẻ tín dụng chỉ hiện 4 số cuối).

🔹 60. Data Auditing & Logging

Theo dõi ai truy cập, sửa đổi dữ liệu.

AWS: CloudTrail + RDS/Aurora logs.

🔹 61. Hot / Warm / Cold Data

Hot: thường xuyên truy cập (lưu trong DB chính).

Warm: thỉnh thoảng dùng (cache/secondary DB).

Cold: ít dùng (archive, Glacier).

🔹 62. Multi-Tenancy

Một DB phục vụ nhiều khách hàng (tenant) khác nhau.

SaaS apps thường áp dụng.

🔹 63. Data Lakehouse

Kết hợp Data Lake (dữ liệu thô) và Data Warehouse (dữ liệu cấu trúc) → vừa linh hoạt vừa nhanh.

AWS: S3 + Redshift Spectrum.

🔹 64. Vector Database

DB tối ưu cho dữ liệu embedding AI/ML.

AWS: Aurora PostgreSQL + pgvector, OpenSearch vector search.

🔹 65. Change Data Capture (CDC)

Theo dõi thay đổi dữ liệu theo thời gian thực.

AWS: DMS hỗ trợ CDC để replicate từ on-premise → AWS.