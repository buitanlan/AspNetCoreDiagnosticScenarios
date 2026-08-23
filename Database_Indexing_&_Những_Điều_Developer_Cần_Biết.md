# Database Indexing & Developer Should Known

## Mục lục

- [Nền tảng: Hiểu Index từ gốc rễ](#nền-tảng-hiểu-index-từ-gốc-rễ)
  - [B+ Tree](#b-tree-không-cần-hiểu-chi-tiết-chỉ-cần-hiểu-ý-tưởng)
  - [Có sắp xếp hay không: Chuyện của Primary Key](#có-sắp-xếp-hay-không-chuyện-của-primary-key)
  - [Heap Table và Clustered Index](#hai-cách-lưu-trữ-khác-nhau-heap-table-và-clustered-index)
  - [Có index chưa chắc query nhanh](#có-index-chưa-chắc-query-nhanh-hiểu-lầm-phổ-biến-nhất)
- [Bốn nguyên tắc vàng khi sử dụng Index](#bốn-nguyên-tắc-vàng-khi-sử-dụng-index)
- [Index hoạt động thế nào với từng thao tác SQL](#index-hoạt-động-thế-nào-với-từng-thao-tác-sql)
- [Tại sao Database không dùng Index của tôi](#tại-sao-database-không-dùng-index-của-tôi)
  - [Parameter sniffing](#parameter-sniffing-plan-đúng-với-lần-chạy-đầu-sai-với-lần-sau)
  - [Tạo index không khóa bảng](#tạo-và-bảo-trì-index-mà-không-khóa-bảng)
- [Cạm bẫy và mẹo nâng cao về Indexing](#cạm-bẫy-và-mẹo-nâng-cao-về-indexing)
- [Kỹ thuật thao tác dữ liệu hiệu quả](#kỹ-thuật-thao-tác-dữ-liệu-hiệu-quả)
- [Viết query như chuyên gia](#viết-query-như-chuyên-gia)
- [Thiết kế Schema](#thiết-kế-schema-nền-móng-vững-chắc)

> **Quy ước trong ebook:** Ví dụ lấy **PostgreSQL** và **SQL Server (MSSQL)** làm hệ chính. Cú pháp MySQL, Oracle, MariaDB được **giữ nguyên** khi khác biệt — không thay thế, chỉ bổ sung bên cạnh.



## Nền tảng: Hiểu Index từ gốc rễ



### B+ Tree: Không cần hiểu chi tiết, chỉ cần hiểu ý tưởng

Mọi cuốn sách về database đều bắt đầu bằng việc giải thích chi tiết cấu trúc B+ tree: leaf nodes, internal nodes, thuật toán insert, delete, rebalance... Thành thật mà nói, bạn không cần biết tất cả những thứ đó để tạo được index tốt. Số người thực sự cần hiểu chi tiết kỹ thuật B+ tree là rất nhỏ, và bạn không phải một trong số đó. Kiến thức quá chi tiết về internal thậm chí còn gây hại, vì bạn sẽ mắc kẹt trong cảm giác "mình chưa hiểu đủ" và không dám áp dụng.

Thay vào đó, hãy nhớ hai điều cốt lõi về B+ tree:

#### Index là một danh sách đã được sắp xếp (sorted list) + bảng tóm tắt phân cấp

Hãy tưởng tượng bạn có một cuốn từ điển dày 2000 trang. Bạn muốn tìm từ "performance". Bạn sẽ:

- Nhìn vào phần gáy sách → thấy P nằm khoảng trang 1200
- Lật đến trang 1200, nhìn header → thấy "PER" bắt đầu từ trang 1245
- Lật đến 1245, scan vài trang → tìm thấy "performance"

Bạn chỉ cần 3 bước thay vì đọc 2000 trang. Index trong database hoạt động y hệt:

- Leaf nodes (tầng dưới cùng) = các trang từ điển, chứa danh sách giá trị đã sorted
- Internal nodes (các tầng trên) = phần gáy/header sách, chứa "tóm tắt phạm vi" để nhảy nhanh
- Kể cả bảng có hàng tỷ row, số bước nhảy qua internal nodes cũng chỉ khoảng 3-4 lần (vì mỗi tầng phân chia dữ liệu ra hàng trăm/hàng nghìn nhánh)

Từ đây bạn có thể hình dung đơn giản: index = sorted list + bảng tóm tắt giúp nhảy nhanh. Không cần phức tạp hơn thế.

#### Database tự quản lý index hoàn toàn

Mỗi khi bạn INSERT, UPDATE hay DELETE một row, database tự động cập nhật tất cả các index liên quan:

- Thêm row → tạo entry mới trong index, đặt đúng vị trí sorted
- Xóa row → xóa entry tương ứng trong index
- Sửa row → xóa entry cũ, thêm entry mới (chỉ khi cột trong index bị thay đổi). Nếu bạn chỉ update một cột không nằm trong bất kỳ index nào, index không bị ảnh hưởng gì.

**Điều này dẫn đến một trade-off quan trọng:** càng nhiều index → write càng chậm (vì mỗi lần ghi phải cập nhật nhiều index hơn). Để bạn hình dung cụ thể:

Bảng users có 5 index:

- INSERT 1 row → database phải thêm entry vào CẢ 5 index
- UPDATE cột email (nằm trong 2 index) → 2 index bị cập nhật
- UPDATE cột bio (không nằm trong index nào) → 0 index bị cập nhật
- DELETE 1 row → xóa entry khỏi CẢ 5 index

Nhưng đừng lo lắng quá: trong thực tế, hầu hết ứng dụng đọc nhiều hơn ghi rất nhiều (tỷ lệ read:write thường là 90:10 hoặc cao hơn), và mỗi bảng thường chỉ cần 3-7 index. Chi phí write thêm cho index hầu như không đáng kể so với lợi ích read mà nó mang lại.

Câu hỏi hay gặp: "Vậy tạo bao nhiêu index là quá nhiều?" Không có con số cố định, nhưng nếu bảng có hơn 10 index và bạn thấy write performance giảm đáng kể, đó là lúc nên review lại. Dùng query kiểm tra unused index (sẽ học ở Phần 5) để tìm và xóa index thừa.

### Có sắp xếp hay không: Chuyện của Primary Key

Vì index là sorted list, vị trí insert entry mới ảnh hưởng đến performance:

- Thêm vào cuối (giá trị luôn tăng): nhanh, vì vị trí cuối luôn được cache trong memory
- Thêm vào giữa (giá trị random): chậm hơn, vì phải tìm vị trí đúng, có thể phải di chuyển entries xung quanh, và trang chứa vị trí đó có thể không nằm trong memory

Đây là lý do lựa chọn kiểu primary key quan trọng:


| Loại PK        | Giá trị mẫu                    | Thứ tự insert           | Tốc độ     |
| -------------- | ------------------------------ | ----------------------- | ---------- |
| Auto-increment | 1, 2, 3, 4...                  | Luôn tăng → cuối list   | Nhanh nhất |
| UUIDv4         | `a3f8b2c1-...` (random)        | Random → giữa list      | Chậm nhất  |
| UUIDv7 / ULID  | `019abc12-...` (time-based)    | Gần như tăng → gần cuối | Nhanh      |
| Snowflake ID   | `142506829879...` (time-based) | Luôn tăng → cuối list   | Nhanh      |


Để bạn hình dung mức độ ảnh hưởng: trên một bảng MySQL hoặc SQL Server 10 triệu row (clustered index trên PK), insert UUIDv4 / `NEWID()` có thể chậm hơn 3–5 lần so với auto-increment / `IDENTITY` / `NEWSEQUENTIALID()`. Khoảng cách càng lớn khi bảng càng to, vì index không fit trong memory nữa và mỗi random insert có thể trigger disk I/O + page split.

Lưu ý quan trọng: Vấn đề random key nặng nhất với **SQL Server** và **MySQL (InnoDB)** vì mặc định PK là Clustered Index (table được sắp theo PK). Với **PostgreSQL** (Heap Table) ảnh hưởng nhẹ hơn: dữ liệu bảng append vào cuối bất kể PK là gì, chỉ secondary index bị phân mảnh. Nhưng nếu bảng hàng triệu row và insert liên tục, bạn nên dùng key tăng dần cho cả ba.

### Hai cách lưu trữ khác nhau: Heap Table và Clustered Index

Đây là kiến thức nền quan trọng mà nhiều developer bỏ qua, nhưng nó ảnh hưởng trực tiếp đến cách bạn thiết kế schema và chọn primary key. Có hai cách database lưu trữ dữ liệu trên disk:

#### Heap Table (PostgreSQL mặc định)

Dữ liệu được append vào cuối file, không quan tâm thứ tự. Bạn insert row theo thứ tự nào, nó nằm ở vị trí đó luôn: kể cả PK = 999 được insert trước PK = 1. Tất cả index (kể cả primary key) đều lưu "địa chỉ vật lý" (physical location: gọi là tuple ID hoặc ctid trong PostgreSQL) của row.

```
┌─────────────────────────┐      ┌─────────────────────────┐
│   INDEX (email)         │      │     TABLE (heap)        │
│                         │      │                         │
│ alice@... → row ở page 5│──────│ Page 5: [alice, 28, ...]│
│ bob@...   → row ở page 2│──────│ Page 2: [bob, 35, ...]  │
│ charlie@..→ row ở page 7│──────│ Page 7: [charlie, 22,..]│
└─────────────────────────┘      └─────────────────────────┘
```

Primary key index cũng hoạt động y hệt: chỉ khác là có UNIQUE constraint.

Điểm quan trọng: vì tất cả index đều bình đẳng (đều trỏ đến vị trí vật lý), không có sự khác biệt performance giữa lookup bằng PK hay bằng secondary index: cả hai đều cần 1 bước nhảy từ index vào table.

#### Clustered Index (SQL Server và MySQL/InnoDB mặc định)

Ở đây mọi thứ khác hẳn. Primary key chính là bảng: dữ liệu row được lưu ngay trong leaf node của primary key index, sắp xếp theo thứ tự PK. Không có "bảng heap" riêng.

```
┌────────────────────────────────────────┐
│ PRIMARY KEY INDEX = TABLE              │
│                                        │
│ PK=1 → [alice, alice@..., 28, ...]     │
│ PK=2 → [bob, bob@..., 35, ...]         │
│ PK=3 → [charlie, charlie@..., 22, ...] │
└────────────────────────────────────────┘
            ▲
│ (tìm PK=2 → có ngay dữ liệu, không cần bước nào thêm!)
```

Còn secondary index thì không trỏ đến vị trí vật lý (vì không có heap), mà trỏ đến giá trị primary key:

```
┌──────────────────────────┐      ┌──────────────────────────┐
│ SECONDARY INDEX (email)  │      │ PRIMARY KEY INDEX = TABLE│
│                          │      │                          │
│ alice@... → PK=1         │──┐   │ PK=1 → [alice, ...]      │
│ bob@...   → PK=2         │──┤   │ PK=2 → [bob, ...]        │
│ charlie@..→ PK=3         │──┘   │ PK=3 → [charlie, ...]    │
└──────────────────────────┘      └──────────────────────────┘
```

Secondary index lookup = 2 bước: tìm PK trong secondary index → tìm row trong PK index.

So sánh hai cách tiếp cận:


|                                 | Heap Table (PostgreSQL mặc định) | Clustered Index (SQL Server, MySQL/InnoDB)                        |
| ------------------------------- | -------------------------------- | ----------------------------------------------------------------- |
| PK lookup                       | 2 bước: PK index → heap table    | 1 bước: clustered / PK index = table                              |
| Secondary / nonclustered lookup | 2 bước: index → heap (`ctid`)    | 2 bước: secondary → PK (SQL Server gọi **Key Lookup**)            |
| Insert random PK                | Nhanh (append vào heap)          | Chậm (insert đúng vị trí + page split)                            |
| PK size ảnh hưởng?              | Ít (chỉ PK index)                | Nhiều (clustering key / PK được copy vào **mọi** secondary index) |


SQL Server vẫn *có thể* tạo heap (bảng không clustered index), nhưng hầu hết bảng production đều có `PRIMARY KEY CLUSTERED`. MySQL InnoDB luôn clustered theo PK (hoặc hidden row ID nếu không có PK). PostgreSQL cũng có `CLUSTER` nhưng đó là sắp xếp một lần, không được duy trì tự động.

#### Hệ quả thực tế cho Clustered Index (SQL Server, MySQL/InnoDB)



##### Hệ quả 1: KHÔNG dùng UUIDv4 làm primary key!

Vì table = PK index (sorted), mỗi row mới với UUIDv4 random phải insert vào một vị trí random trong tree. Với bảng lớn không fit trong RAM, mỗi insert có thể trigger đọc disk → insert chậm gấp 3-10 lần so với auto-increment.

##### Hệ quả 2: Primary key size ảnh hưởng toàn bộ database

PK value được copy vào mỗi secondary index entry. Tính toán cụ thể:

```
Bảng: 1 triệu row, 5 secondary index
PK = BIGINT (8 bytes):       5 index × 1M × 8 bytes  =  40 MB overhead
PK = ULID string (26 bytes): 5 index × 1M × 26 bytes = 130 MB overhead
Chênh lệch: 90 MB — chỉ cho MỘT bảng!
Với 50 bảng tương tự: chênh lệch 4.5 GB
→ Có thể là khác biệt giữa "fit trong RAM" và "phải đọc disk"
```

Thực tiễn: auto-increment integer hoặc UUIDv7/ULID (binary, không phải string) là lựa chọn an toàn nhất.

- **PostgreSQL:** `BIGINT GENERATED ALWAYS AS IDENTITY` hoặc UUIDv7 kiểu `UUID` (16 bytes), không lưu `CHAR(36)`.
- **SQL Server:** `BIGINT IDENTITY` hoặc `UNIQUEIDENTIFIER` với `NEWSEQUENTIALID()` / UUIDv7. Tránh `NEWID()` trên clustered PK. Đừng lưu UUID dạng `CHAR(36)` / `NVARCHAR(36)`.
- **MySQL:** `BIGINT AUTO_INCREMENT` hoặc ULID/UUIDv7 dạng `BINARY(16)`, không dùng `CHAR(36)`.



##### Hệ quả 3: PK lookup cực nhanh — tận dụng cho CRUD apps

Nếu app chủ yếu là CRUD (tìm user theo ID, lấy order theo ID...), clustered index cho performance rất tốt vì data có sẵn ngay khi tìm thấy PK. Đây là lý do SQL Server và MySQL/InnoDB phổ biến với web CRUD.

Bù lại, secondary lookup phải nhảy thêm một bước: **Key Lookup** (SQL Server) hoặc PK lookup (MySQL). Nếu query lấy nhiều cột ngoài index, bước này hàng loạt có thể đắt hơn table scan. Giải pháp: covering index với `INCLUDE` (PostgreSQL, SQL Server) hoặc đưa đủ cột vào index (MySQL 8.0 cũng có `INCLUDE` từ 8.0.13).

### Có index chưa chắc query nhanh: Hiểu lầm phổ biến nhất

Đây là điều nhiều người không nhận ra: dùng index không đảm bảo query nhanh. Cảm giác "tôi đã tạo index rồi mà vẫn chậm" xuất phát từ việc không hiểu quy trình thực tế.

Quy trình khi database dùng index:

```
┌──────────────────────────┐
│ 1. Tìm matching entries  │ ← Index giúp bước này nhanh
│    trong index           │
└──────────┬───────────────┘
           │ (danh sách row IDs / PK values)
           ▼
┌──────────────────────────┐
│ 2. Load từng row từ table│ ← CHẬM nếu quá nhiều row!
│    (random I/O)          │   Mỗi row ở vị trí khác nhau
└──────────┬───────────────┘
           ▼
┌──────────────────────────┐
│ 3. Check thêm các điều   │ ← Lãng phí nếu nhiều row bị loại
│    kiện KHÔNG trong index│
└──────────┬───────────────┘
           ▼
┌──────────────────────────┐
│ 4. Trả kết quả           │
└──────────────────────────┘
```

Bước 2 và 3 chính là nơi query có thể chậm. Hãy xem một ví dụ thực tế:

```sql
-- Bảng orders: 5 triệu rows
-- Index: chỉ có trên (status)
SELECT * FROM orders
WHERE status = 'pending'     -- index lọc: 5M → 200,000 rows
  AND region = 'southeast'   -- KHÔNG trong index
  AND total > 1000000;       -- KHÔNG trong index

-- Chuyện gì xảy ra:
-- 1. Index tìm 200,000 entries có status = 'pending'   ← nhanh
-- 2. Load 200,000 rows từ table (random I/O)           ← CHẬM!
-- 3. Check region = 'southeast' → còn 15,000 rows      ← lãng phí 185,000 lần đọc
-- 4. Check total > 1000000 → còn 500 rows              ← lãng phí thêm 14,500 lần
-- Kết quả: chỉ cần 500 rows nhưng đã load 200,000 rows!
```



#### Giải pháp: Tạo index bao phủ nhiều điều kiện hơn

```sql
-- Index tốt hơn: (status, region, total)
-- 1. status = 'pending' → fast lookup
-- 2. region = 'southeast' → tiếp tục lọc trong index (không cần load row)
-- 3. total > 1000000 → scan range trên index (vẫn không load row)
-- 4. Chỉ load ~500 rows thực sự cần → nhanh hơn RẤT NHIỀU

-- Hoặc thậm chí index-only nếu chỉ cần vài cột:
-- Index: (status, region, total) INCLUDE (order_id, customer_id)
-- → Không cần load row từ table nào cả!
```

Quy tắc thực hành: Sau khi tạo index, luôn xem execution plan trước khi deploy.

- **PostgreSQL:** `EXPLAIN (ANALYZE, BUFFERS)` — xem `Index Scan` / `Index Only Scan` / `Seq Scan`, và `rows` ước lượng so với thực tế.
- **SQL Server:** Actual Execution Plan trong SSMS (Ctrl+M), hoặc `SET STATISTICS IO, TIME ON`. Tìm `Index Seek` (tốt), `Index Scan`, `Clustered Index Scan`, `Key Lookup` (đắt nếu số lần lớn), `Table Scan`.
- **MySQL:** `EXPLAIN ANALYZE` (8.0.18+) hoặc `EXPLAIN`. Cột `type`: `ALL` = full table scan, `index` = full index scan, `range` = range scan, `ref` = index lookup, `const` = đúng 1 row qua unique index.

Nếu số row đọc >> số row trả về, index chưa đủ tốt (thiếu cột filter, hoặc chưa covering).

## Bốn nguyên tắc vàng khi sử dụng Index

Đây là phần quan trọng nhất của cả cuốn ebook. Bốn nguyên tắc này là tất cả những gì bạn cần để tạo index tốt cho bất kỳ query nào. Khi bạn gặp một query chậm, hãy vẽ ra giấy (hoặc hình dung trong đầu) cách query đó map vào index theo 4 nguyên tắc này: đó là cách tốt nhất để xác định index cần tạo.

### Nguyên tắc 1: Tra cứu nhanh — Nhảy thẳng đến vị trí cần tìm

Thao tác cơ bản nhất của index: tìm một giá trị cụ thể gần như tức thì bằng cách nhảy qua các tầng internal node, thay vì scan từ đầu đến cuối.

```sql
SELECT * FROM movies WHERE release_year = 2019;
```

Hình dung index trên `release_year` là một cuốn sách sorted:

```
Index summary (internal nodes):
[... | 2015-2017 | 2018-2020 | 2021-2023 | ...]
                    │
                    ▼
Leaf nodes:   [2018 | 2018 | 2019 | 2019 | 2019 | 2020 | 2020]
                              ▲
                    Database nhảy thẳng đến đây!
```

Database không cần đọc qua 2015, 2016, 2017, 2018... Nó nhảy thẳng đến khu vực 2018-2020 trong index summary, rồi tìm chính xác 2019 trong leaf nodes.

Hiểu nhầm phổ biến: **"Index càng lớn thì query càng chậm"**

Chưa chắc! Hãy nhớ index là cấu trúc cây (tree), không phải danh sách phẳng. Mỗi tầng internal node chia dữ liệu thành hàng trăm nhánh. Kết quả:


| Số rows       | Số bước nhảy (tree depth) |
| ------------- | ------------------------- |
| 1,000         | ~2                        |
| 1,000,000     | ~3                        |
| 1,000,000,000 | ~4                        |


Từ 1 nghìn lên 1 tỷ row, chỉ thêm 2 bước nhảy. Đó là sức mạnh của O(log n). Index đã được tối ưu hóa suốt hàng chục năm cho đúng use case này: đừng lo lắng về kích thước.

### Nguyên tắc 2: Quét theo một hướng

Sau khi nhảy đến một vị trí trong index (bằng Fast Lookup), database có thể tiếp tục đọc liên tục theo một hướng (ascending hoặc descending). Vì leaf nodes của B+ tree được liên kết với nhau (linked list), việc di chuyển sang entry tiếp theo là cực kỳ nhanh.

```sql
SELECT * FROM users WHERE age >= 35 ORDER BY age ASC LIMIT 3;
```

```
Index (age):
[18 | 22 | 25 | 28 | 30 | 35 | 37 | 42 | 48 | 55 | 61]
                      ▲
         Fast Lookup: age >= 35
                      │
                      ├──→ 35 ✅ (lấy)
                      ├──→ 37 ✅ (lấy)
                      ├──→ 42 ✅ (lấy, đủ 3 → DỪNG!)
                           48, 55, 61... (không cần đọc)
```

Tương tự với hướng ngược lại:

```sql
SELECT * FROM users WHERE age <= 35 ORDER BY age DESC LIMIT 3;
-- Fast lookup đến 35, scan ngược: 35 → 30 → 28 → DỪNG!
```



#### Sức mạnh thực sự khi kết hợp với LIMIT

Không có index, query `WHERE age >= 35 ORDER BY age ASC LIMIT 3` trên bảng 10 triệu row phải: scan toàn bộ 10M rows → filter → sort → lấy 3. Với index, chỉ cần: nhảy đến 35 → đọc 3 entries → xong. Chênh lệch có thể từ vài giây xuống dưới 1ms.

Nhưng nhớ: Scan chỉ đi một hướng. Không thể vừa scan ascending vừa scan descending cùng lúc trong một index traversal. Nếu query cần sort theo 2 cột với hướng khác nhau (vd: `ORDER BY score DESC, created_at ASC`), bạn cần tạo index với đúng thứ tự sort đó (xem thêm ở Phần 3 - ORDER BY).

### Nguyên tắc 3: Từ trái sang phải — Nguyên tắc "Phễu" cho index nhiều cột

Đây là nguyên tắc quan trọng nhất và cũng dễ hiểu sai nhất. Single-column index đơn giản, nhưng multi-column index (composite index) mới là nơi mang lại cải thiện performance lớn nhất — và cũng là nơi dễ sai nhất. Hãy hình dung multi-column index như một cái phễu (funnel) lọc dữ liệu từ trái sang phải.

Cách multi-column index được sắp xếp: index trên `(country, lastname, firstname)` sắp xếp dữ liệu theo nguyên tắc: sort by country trước, trong mỗi country sort by lastname, trong mỗi lastname sort by firstname.

```
Index (country, lastname, firstname):
┌──────────┬──────────┬──────────┐
│ country  │ lastname │ firstname│
├──────────┼──────────┼──────────┤
│ JP       │ Sato     │ Kenji    │
│ JP       │ Suzuki   │ Yuki     │
│ JP       │ Tanaka   │ Hiroshi  │
│ US       │ Johnson  │ Emily    │
│ US       │ Smith    │ Alice    │
│ US       │ Smith    │ Bob      │
│ VN       │ Le       │ Minh     │
│ VN       │ Nguyen   │ An       │  ← target
│ VN       │ Nguyen   │ Huy      │  ← target
│ VN       │ Tran     │ Duc      │
└──────────┴──────────┴──────────┘
```

Bạn có thể thấy: trong mỗi country, các lastname được sorted. Nhưng nhìn toàn bộ cột lastname, nó KHÔNG sorted (Sato, Suzuki, Tanaka, Johnson, Smith...). Đây chính là lý do tại sao bạn phải đi từ trái sang phải.

#### Phễu hoạt động thế nào

```sql
WHERE country = 'VN' AND lastname = 'Nguyen' AND firstname = 'Huy'
```

```
Bước 1: country = 'VN'         → Phễu thu hẹp: 4 entries (VN block)
Bước 2: lastname = 'Nguyen'    → Phễu thu hẹp: 2 entries (An, Huy)
Bước 3: firstname = 'Huy'      → Phễu thu hẹp: 1 entry (chính xác!)
```

Mỗi bước "thắt" phễu lại, giảm số entries phải xét. Rất hiệu quả!

Các query dùng được index này:

```sql
-- ✅ Dùng 3/3 bước phễuChỉ đánh index
WHERE country = 'VN' AND lastname = 'Nguyen' AND firstname = 'Huy';

-- ✅ Dùng 2/3 bước phễu (firstname bị bỏ → vẫn OK, chỉ kém tối ưu hơn)
WHERE country = 'VN' AND lastname = 'Nguyen';
Chỉ đánh index
-- ✅ Dùng 1/3 bước phễu
WHERE country = 'VN';

-- ❌ KHÔNG dùng được! (bỏ qua country: cột đầu tiên)
WHERE lastname = 'Nguyen';
-- Vì: lastname 'Nguyen' nằm rải rác ở JP, US, VN... không nằm liền nhau

-- ❌ KHÔNG dùng được! (bỏ qua cả country và lastname)
WHERE firstname = 'Huy';
```

Câu thần chú: **"Từ trái sang phải, không bỏ qua cột"** — hãy khắc câu này vào đầu.

Hiểu nhầm phổ biến: "Đặt cột selective nhất lên đầu"

Bạn sẽ thấy nhiều người khuyên rằng: "đặt cột có nhiều distinct values nhất (selective nhất) lên đầu index". Hãy xem tại sao nó chưa chắc đã luôn đúng:

Giả sử bạn đổi thứ tự index thành `(lastname, firstname, country)` vì lastname có nhiều giá trị nhất. Query `WHERE country = 'VN' AND lastname = 'Nguyen' AND firstname = 'Huy'` vẫn dùng được toàn bộ phễu: vì dùng đủ 3 cột. Số bước phễu giống nhau, selectivity ở đây không tạo ra khác biệt.

Nhưng bây giờ, query `WHERE country = 'VN'` (rất phổ biến trong app multi-tenant) hoàn toàn không dùng được index này! Vì country ở vị trí cuối.

Cách đúng: Thứ tự cột nên được quyết định bởi tập hợp các query mà app bạn chạy, nhằm tối đa hóa số query được phục vụ bởi cùng một index:

```sql
-- App chạy các query này:
-- Q1: WHERE country = 'VN'                                          (rất thường xuyên)
-- Q2: WHERE country = 'VN' AND lastname = 'Nguyen'                  (thường xuyên)
-- Q3: WHERE country = 'VN' AND lastname = 'Nguyen' AND firstname = 'Huy' (ít hơn)

-- Index (country, lastname, firstname) → phục vụ CẢ 3 query
-- Index (lastname, firstname, country) → chỉ phục vụ Q3 tốt, Q1 không dùng được
-- Index (firstname, lastname, country) → chẳng phục vụ tốt query nào ở trên
```



#### Bỏ qua cột giữa: Vẫn dùng được, nhưng kém hiệu quả

```sql
-- Index: (firstname, lastname, country)
-- Query: WHERE firstname = 'Huy' AND country = 'VN'
-- (bỏ qua lastname ở giữa)
```

Database vẫn dùng index này! Nhưng cách nó hoạt động kém tối ưu hơn:

```
Bước 1: firstname = 'Huy' → Fast Lookup, tìm được block entries cho 'Huy'
Bước 2: lastname bị bỏ qua → KHÔNG thể dùng phễu tiếp
Bước 3: Scan TOÀN BỘ entries có firstname = 'Huy',
        kiểm tra TỪNG entry xem country = 'VN' không

Index entries cho firstname = 'Huy':
[Huy | Le    | JP ] → country != 'VN' ❌ (bỏ)
[Huy | Nguyen| US ] → country != 'VN' ❌ (bỏ)
[Huy | Nguyen| VN ] → country  = 'VN' ✅ (giữ)
[Huy | Tran  | JP ] → country != 'VN' ❌ (bỏ)
[Huy | Tran  | VN ] → country  = 'VN' ✅ (giữ)
→ Scan 5 entries, giữ 2
```

So sánh: index hoàn hảo `(firstname, country)` chỉ cần scan 2 entries (đúng entries cần). Với bảng lớn, "firstname = 'Huy'" có thể match hàng trăm nghìn entries → scan tất cả để filter country là rất lãng phí.

Tuy nhiên, skipping a column vẫn tốt hơn không có index: vì ít nhất database giới hạn được phạm vi scan (chỉ entries có firstname = 'Huy'), và có thể filter trên country ngay trong index mà không cần load row từ table.

#### Index trùng lặp: Dọn dẹp index thừa

Mỗi index phải được cập nhật khi write, nên index thừa = tốn tài nguyên vô ích. Quy tắc:

- ✅ Index `(country, lastname, firstname)` BAO GỒM chức năng của:
  - `(country)`
  - `(country, lastname)`
  - → Xóa 2 index đơn/đôi này đi nếu tồn tại
- ❌ Nhưng KHÔNG BAO GỒM:
  - `(country, lastname, telephone)` → cột cuối khác
  - `(lastname, country)` → thứ tự cột khác
  - → Đây là các index độc lập, không thể thay thế

Ví dụ thực tế dọn dẹp:

```sql
-- Bảng users hiện có 4 index:
-- idx_1: (tenant_id)
-- idx_2: (tenant_id, email)
-- idx_3: (tenant_id, created_at)
-- idx_4: (email)

-- Phân tích:
-- idx_1 bị bao gồm bởi idx_2 (cùng prefix) → XÓA idx_1
-- idx_2 và idx_3 có prefix giống nhưng cột 2 khác → GIỮ cả hai
-- idx_4 phục vụ query WHERE email = '...' (không có tenant_id) → GIỮ
-- Kết quả: giữ idx_2, idx_3, idx_4. Xóa idx_1.
```



### Nguyên tắc 4: Quét khi gặp điều kiện phạm vi — Điều kiện phạm vi "phá vỡ" phễu

Đây là nguyên tắc hay bị bỏ sót nhất, nhưng lại ảnh hưởng performance rất lớn. Khi gặp range condition (`>`, `<`, `>=`, `<=`, `BETWEEN`), database chuyển sang chế độ scan: và từ lúc đó, phễu không thể thu hẹp thêm bằng các cột phía sau.

Tại sao range condition "phá vỡ" phễu? Hãy nhìn index `(country, age, married)` với query:

```sql
WHERE country = 'VN' AND age > 28 AND married = 'yes'
```

```
Index (country, age, married):
┌──────────┬─────┬─────────┐
│ country  │ age │ married │
├──────────┼─────┼─────────┤
│ VN       │ 25  │ no      │
│ VN       │ 27  │ yes     │
│ VN       │ 29  │ no      │  ← age > 28 bắt đầu scan từ đây
│ VN       │ 29  │ yes     │  ← married = 'yes' ✅
│ VN       │ 31  │ no      │  ← married = 'no'  ❌ (vẫn phải đọc!)
│ VN       │ 31  │ yes     │  ← married = 'yes' ✅
│ VN       │ 35  │ no      │  ← married = 'no'  ❌ (vẫn phải đọc!)
│ VN       │ 42  │ yes     │  ← married = 'yes' ✅
└──────────┴─────┴─────────┘
```

Sau khi `age > 28` bắt đầu scan, các entry có `married = 'no'` và `married = 'yes'` xen kẽ nhau. Database không thể "nhảy qua" các entry `married = 'no'`: nó phải đọc từng entry và check. Đọc 6 entries nhưng chỉ giữ 3.

Giờ đổi thứ tự: index `(country, married, age)`:

```
Index (country, married, age):
┌──────────┬─────────┬─────┐
│ country  │ married │ age │
├──────────┼─────────┼─────┤
│ VN       │ no      │ 25  │
│ VN       │ no      │ 29  │  ← married = 'no' → bỏ qua toàn bộ block!
│ VN       │ no      │ 31  │
│ VN       │ no      │ 35  │
│ VN       │ yes     │ 27  │
│ VN       │ yes     │ 29  │  ← Fast lookup: country='VN', married='yes', age>28
│ VN       │ yes     │ 31  │  ← scan
│ VN       │ yes     │ 42  │  ← scan
└──────────┴─────────┴─────┘
```

Bây giờ: Fast Lookup qua 2 bước phễu (country → married) → scan từ `age > 28`. Chỉ đọc 3 entries thay vì 6! Và không cần filter thêm gì.

Ảnh hưởng thực tế: Với bảng nhỏ, chênh lệch không đáng kể. Nhưng hãy tưởng tượng bảng 10 triệu users:

- Index sai `(country, age, married)`: scan 500,000 entries cho `age > 28`, filter ra 250,000 → đọc gấp đôi cần thiết
- Index đúng `(country, married, age)`: scan đúng 250,000 entries → không lãng phí

Khi có NHIỀU range condition:

```sql
WHERE country = 'VN' AND age > 25 AND salary > 20000000
```

Chỉ một cột range có thể hưởng lợi từ index scan. Cột range thứ hai chỉ dùng để filter:

```sql
-- Index: (country, age, salary)
-- country → phễu, age > 25 → scan, salary > 20M → filter (KHÔNG giới hạn scan)

-- Index: (country, salary, age)
-- country → phễu, salary > 20M → scan, age > 25 → filter

-- Chọn cột nào đặt trước? Cột nào filter được NHIỀU row hơn!
-- Nếu 90% users có age > 25 nhưng chỉ 10% có salary > 20M
-- → Đặt salary trước: (country, salary, age)
```



#### Ngoại lệ: Loose Index Scan / Skip Scan

Một số database có thể "nhảy qua" entries trong trường hợp đặc biệt (thường là GROUP BY min/max):

- MySQL: Loose Index Scan
- Oracle / SQL Server: Skip Scan (optimizer có thể nhảy giữa các nhóm giá trị của cột đầu)
- PostgreSQL: chưa hỗ trợ Skip Scan — thiếu cột đầu = gần như không dùng được composite index

Nhưng đây là tối ưu đặc biệt, không phải hành vi mặc định. Đừng thiết kế index dựa trên nó.

**Quy tắc vàng cần nhớ:**

1. Equality columns trước, Range columns sau
2. Nếu có nhiều range conditions, đặt cột filter được nhiều nhất trước
3. Sau cột range đầu tiên, các cột tiếp theo chỉ dùng để filter (vẫn hữu ích, nhưng không giới hạn scan range)



## Index hoạt động thế nào với từng thao tác SQL

Bốn nguyên tắc ở Phần 2 là nền tảng. Giờ chúng ta sẽ xem chúng hoạt động thế nào với các SQL operation phức tạp hơn. Một điều quan trọng cần nhớ: thứ tự thực thi SQL không giống thứ tự bạn viết. Database thực thi theo thứ tự:

```
FROM / JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

Điều này ảnh hưởng trực tiếp đến cách xây dựng index: cột WHERE phải nằm trước cột ORDER BY trong index, cột GROUP BY phải nằm trước cột SELECT aggregate, v.v.

### Phép so sánh không bằng (!=): Kẻ giết hiệu suất thầm lặng

Điều kiện `!=` là một trong những thứ tệ nhất cho index. Với index trên `(status)`, database phải scan toàn bộ index entries, kiểm tra từng cái xem có `!= 'open'` không: vì kết quả match nằm ở cả hai phía (trước và sau 'open'). Đây không phải range scan (chỉ scan một hướng): mà phải đọc tất cả.

```sql
SELECT * FROM payments WHERE status != 'open';
```

```
Index (status):
[cancelled | cancelled | open | open | open | paid | paid | refunded]
     ✅          ✅        ❌     ❌     ❌     ✅     ✅      ✅
→ Phải đọc 8 entries, bỏ 3, giữ 5
→ Chẳng khác gì full table scan!
```

Tệ hơn: database không thể dự đoán bao nhiêu row match `!= 'open'` (có thể 5% hoặc 95%), nên cost model không đánh giá được → thường fall back về full table scan luôn, hoàn toàn bỏ qua index.

#### Cách 1: Kết hợp thêm cột equality thu hẹp phạm vi

```sql
SELECT * FROM payments
WHERE shop_id = 42 AND status != 'open';
-- Index: (shop_id, status)

-- Workflow:
-- 1. Fast lookup: shop_id = 42 → thu hẹp xuống vài trăm entries
-- 2. Scan entries của shop_id = 42, filter status != 'open'
-- → Dù inequality vẫn phải scan, phạm vi scan đã nhỏ hơn RẤT nhiều!
```



#### Cách 2: Chuyển inequality thành IN(...) nếu biết trước giá trị

```sql
-- Nếu status chỉ có: 'open', 'paid', 'cancelled', 'refunded'
-- Thay vì: WHERE status != 'open'
-- Viết:    WHERE status IN ('paid', 'cancelled', 'refunded')
-- → Database tách thành 3 equality lookups, hiệu quả hơn nhiều!
```



#### Cách 3: Chuyển thành boolean column / functional index

```sql
-- PostgreSQL: expression / functional index
CREATE INDEX payments_not_open ON payments ((status <> 'open'));
-- WHERE (status <> 'open') = true  → equality

-- MySQL 8.0+: functional index
CREATE INDEX payments_not_open ON payments ((status != 'open'));
-- WHERE (status != 'open') = true

-- SQL Server: computed column + index (biểu thức phải deterministic, nên PERSISTED)
ALTER TABLE payments ADD is_not_open AS (CASE WHEN status <> 'open' THEN 1 ELSE 0 END) PERSISTED;
CREATE INDEX payments_not_open ON payments (is_not_open);
-- WHERE is_not_open = 1
```



### NULL: Giá trị đặc biệt cần đặc biệt chú ý

`NULL` trong SQL nghĩa là "không biết" (unknown), không phải "rỗng" hay "zero".

```
NULL = NULL    → NULL (không phải TRUE!)
NULL != NULL   → NULL (không phải TRUE!)
NULL > 5       → NULL
NULL + 10      → NULL
-- Bất kỳ phép tính nào với NULL đều trả về NULL
-- Trong context WHERE, NULL được coi như FALSE
```

- PostgreSQL: `ASC` mặc định **NULLS LAST**, `DESC` mặc định **NULLS FIRST**. Có thể ghi rõ `NULLS FIRST` / `NULLS LAST`.
- SQL Server: `NULL` được coi là giá trị **nhỏ nhất** (`ASC` → NULL lên đầu). Không có `NULLS FIRST` / `NULLS LAST`.
- MySQL: `NULL` đứng trước (nhỏ nhất), giống SQL Server.
- `IS NULL` hoạt động giống equality — dùng được index
- `IS NOT NULL` giống inequality — có thể match quá nhiều rows → optimizer bỏ qua index

Cái bẫy ngầm: `WHERE country <> 'VN'` / `!=` sẽ **bỏ sót** các row `country IS NULL`, vì `NULL != 'VN'` → NULL → coi như FALSE.

```sql
-- Cách dài (mọi hệ):
WHERE country != 'VN' OR country IS NULL

-- MySQL:
WHERE NOT (country <=> 'VN')

-- PostgreSQL, và SQL Server 2022+:
WHERE country IS DISTINCT FROM 'VN'
```

NULL trong ORDER BY:

```sql
-- PostgreSQL
SELECT * FROM customers ORDER BY country ASC NULLS FIRST;
SELECT * FROM customers ORDER BY country ASC NULLS LAST;

-- MySQL: muốn NULL ở cuối
SELECT * FROM customers ORDER BY country IS NULL, country ASC;

-- SQL Server: đưa NULL xuống cuối khi ASC
SELECT * FROM customers
ORDER BY CASE WHEN country IS NULL THEN 1 ELSE 0 END, country ASC;
```



### LIKE: Tìm kiếm mẫu chuỗi và "cái bẫy" ký tự đại diện ở đầu

`LIKE 'Nguyễn%'` được database chuyển thành range condition, nên tuân theo Nguyên tắc 4.

```sql
-- LIKE 'Nguyễn%' tương đương:
WHERE firstname >= 'Nguyễn' AND firstname < 'Nguyễo'

-- ✅ (type, firstname): type equality → phễu, firstname LIKE → range scan
-- ❌ (firstname, type): firstname LIKE → range scan ngay, type chỉ filter

SELECT * FROM contacts WHERE name LIKE '%Nguyễn%';
-- Database KHÔNG thể dùng B-tree index!
```

**PostgreSQL:** Trigram Index (`pg_trgm`)

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX trgm_idx ON contacts USING GIN (name gin_trgm_ops);
SELECT * FROM contacts WHERE name LIKE '%Nguyễn%';
```

**SQL Server:** B-tree không giúp `LIKE '%...%'`. Dùng Full-Text Search (`CONTAINS` / `FREETEXT`) hoặc search engine ngoài (Elasticsearch / MeiliSearch). `LIKE 'Nguyễn%'` (wildcard ở cuối) vẫn dùng được index.

**MySQL:** Không có trigram built-in. Cân nhắc Full-Text Search (`MATCH AGAINST`) hoặc Elasticsearch / MeiliSearch.

### ORDER BY: Tránh bước sort bổ sung bằng mọi giá

Nếu bạn thêm cột sort vào cuối index (sau các cột WHERE), database đọc từ index ra đã đúng thứ tự → không cần sort thêm.

```sql
SELECT * FROM issues
WHERE type = 'bug'
ORDER BY severity DESC, created_at DESC;

-- ✅ Index: (type, severity DESC, created_at DESC)
```

Disk-based sort có thể biến query 10ms thành 10 giây.

- **PostgreSQL:** tăng `work_mem` (mặc định 4MB → 32–64MB cho session/query nặng). Áp dụng **per sort/hash operation**, không phải per server.
- **SQL Server:** không có knob tương đương từng query. Memory grant do optimizer quyết. Theo dõi spill to `tempdb` trong plan (`Sort` / `Hash Match` có warning). Giảm spill bằng index đúng thứ tự sort, hoặc covering để khỏi sort.
- **MySQL:** tăng `sort_buffer_size` (mặc định 256KB → 4–8MB). Cẩn thận: setting này apply per-operation / per-thread.

`ORDER BY ... LIMIT 10` vẫn phải sort toàn bộ matching rows nếu không có index phù hợp. Index giúp scan 10 entries đầu tiên và dừng.

Khi sort nhiều cột với hướng khác nhau, tạo index matching:

```sql
SELECT * FROM highscores ORDER BY score DESC, created_at ASC LIMIT 10;
CREATE INDEX highscores_correct ON highscores (score DESC, created_at ASC);
```

Ví dụ e-commerce:

```sql
SELECT * FROM products
WHERE category_id = 5 AND in_stock = true
ORDER BY price ASC
LIMIT 20;

-- ✅ Index: (category_id, in_stock, price)
-- filter bằng phễu → scan price ascending → lấy 20 → DONE
```



### GROUP BY & DISTINCT: Thách thức lớn nhất

DISTINCT = GROUP BY về mặt execution. Khi có index phù hợp, database scan index đã sorted và đếm liên tục (loop-and-count) — không cần temporary table, không cần sort.

```sql
SELECT is_paying, COUNT(*) FROM users GROUP BY is_paying;
-- Index: (is_paying)
```

Quy tắc:

- Simple GROUP BY: index cùng cột, cùng thứ tự
- GROUP BY + WHERE: cột WHERE đặt trước (WHERE chạy trước GROUP BY)
- GROUP BY + WHERE range: range phá vỡ phễu → không loop-and-count được. Giải pháp: biến range thành equality (xem Phần 5)
- Aggregate (AVG, SUM, MAX): đặt cột aggregate cuối index để index-only

Khi GROUP BY trên primary key, không cần liệt kê các cột khác của cùng bảng:

```sql
GROUP BY actors.id;  -- Primary key là đủ!
```



### JOIN: Phân tách và kết hợp

Nested-loop join = hai query độc lập: một chạy 1 lần (driving), một chạy N lần trong loop (driven). Mỗi query cần index riêng.

```sql
SELECT employee.* FROM employee
JOIN department USING (department_id)
WHERE employee.salary > 100000 AND department.country = 'NR';

CREATE INDEX idx_emp_salary ON employee (salary);
CREATE INDEX idx_dept ON department (department_id, country);
```

Thứ tự JOIN trong SQL không phải thứ tự database thực thi. Luôn tạo index hỗ trợ join theo mọi thứ tự có thể. Nên giữ số bảng join dưới 6-8.

**Lấy N dòng cho mỗi nhóm** — PostgreSQL dùng `LATERAL`, SQL Server dùng `APPLY`:

```sql
-- PostgreSQL
SELECT customers.*, recent_sales.*
FROM customers
LEFT JOIN LATERAL (
  SELECT * FROM sales
  WHERE sales.customer_id = customers.id
  ORDER BY created_at DESC
  LIMIT 3
) AS recent_sales ON true;

-- SQL Server
SELECT customers.*, recent_sales.*
FROM customers
OUTER APPLY (
  SELECT TOP (3) *
  FROM sales
  WHERE sales.customer_id = customers.id
  ORDER BY created_at DESC
) AS recent_sales;

-- Index cần: (customer_id, created_at DESC) trên sales
```



### Subquery: Không chậm như bạn nghĩ

Subquery chậm vì thiếu index, không phải vì bản chất của subquery.

- **Independent subquery:** chạy 1 lần, kết quả thay thế vào query chính
- **Dependent subquery:** chạy lặp lại cho mỗi row, giống join. `EXISTS` tự dừng sau 1 row match

```sql
SELECT * FROM products
WHERE remaining = 0 AND EXISTS (
  SELECT * FROM sales
  WHERE created_at >= '2023-01-01' AND product_id = products.product_id
);
-- Index outer: (remaining) trên products
-- Index subquery: (product_id, created_at) trên sales
```



### UPDATE & DELETE: Đừng quên tối ưu cho chúng

Phần "tìm row" hoàn toàn giống SELECT. Viết lại thành SELECT để test:

```sql
-- Thay vì: DELETE FROM logs WHERE created_at < '2024-01-01';
SELECT * FROM logs WHERE created_at < '2024-01-01';
-- Nếu SELECT chậm → DELETE cũng chậm → cần index trên (created_at)
```



## Tại sao Database không dùng Index của tôi

Đây là câu hỏi gây bực bội nhất mà developer hay gặp. Index đã tạo, query rõ ràng match — nhưng database vẫn lờ tịt.

### Quy trình thực thi query: Bên trong "bộ não" của database

Mỗi query đi qua 4 bước:

```
1. PARSE         — Phân tích cú pháp SQL
2. INITIAL PLAN  — Tạo plan cơ bản (full table scan)
3. OPTIMIZE      — Tìm plan tốt hơn (dùng index?)
4. EXECUTE       — Chạy plan tốt nhất
```

Database luôn bắt đầu với plan "full table scan" (vì nó luôn chạy được). Optimizer chỉ dùng index nếu cost model tin rằng nó rẻ hơn.

```sql
-- PostgreSQL: luôn dùng ANALYZE khi đo thật (có chạy query)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'test@example.com';
-- Seq Scan          = full table scan
-- Index Scan        = dùng index rồi nhảy vào heap
-- Index Only Scan   = covering, không đụng heap (tốt)
-- Bitmap Heap Scan  = nhiều match, đọc heap theo batch

-- SQL Server: xem Actual Execution Plan, hoặc:
SET STATISTICS IO, TIME ON;
SELECT * FROM users WHERE email = 'test@example.com';
-- Index Seek              = nhảy đúng vị trí (tốt)
-- Index Scan              = đọc cả index
-- Clustered Index Seek    = tìm theo clustered key (tốt)
-- Clustered Index Scan    = gần như table scan
-- Key Lookup              = nonclustered → clustered (đắt nếu nhiều row)
-- Table Scan              = heap, không clustered index

-- MySQL — xem cột 'type':
--   ALL   = full table scan (tệ)
--   index = full index scan
--   range = range scan trên index (khá)
--   ref   = index lookup (tốt)
--   const = tìm đúng 1 row qua unique index (tuyệt vời)
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

Hãy tập thói quen đọc plan cho mọi query quan trọng trước khi deploy.

### Index không khớp với query: Lý do phổ biến nhất

Bất kỳ phép biến đổi nào áp dụng lên **cột** (không phải giá trị) đều khiến index bị "mù":

```sql
-- ❌ Hàm trên cột → index trên birthday VÔ DỤNG
-- PostgreSQL
SELECT * FROM contacts WHERE EXTRACT(YEAR FROM birthday) = 1988;
-- MySQL / SQL Server
SELECT * FROM contacts WHERE YEAR(birthday) = 1988;

-- ✅ Viết lại: range trên cột gốc
SELECT * FROM contacts
WHERE birthday >= '1988-01-01' AND birthday < '1989-01-01';
```

Các trường hợp hay gặp:

```sql
-- ❌ WHERE col + 5 < 20                   → ✅ WHERE col < 15
-- ❌ WHERE CONCAT(first, ' ', last) = ... → ✅ WHERE first = 'Huy' AND last = 'Nguyen'
-- ❌ WHERE first || ' ' || last = '...'   → ✅ tương tự (PostgreSQL)
-- ❌ WHERE varchar_col = 12345            → ✅ WHERE varchar_col = '12345'  (xem implicit conversion)
-- ❌ WHERE DATE(created_at) = '...'       → ✅ range trên created_at
-- ❌ WHERE LOWER(email) = '...'           → ✅ functional index / computed column
```

Index ẩn (MySQL): kiểm tra `IS_VISIBLE` trong `INFORMATION_SCHEMA.STATISTICS`.

Query phải **SARGable** (Search ARGument Able): điều kiện viết sao cho optimizer so sánh trực tiếp cột đã index, không biến đổi cột.

### Full table scan nhanh hơn: Khi database đúng mà bạn sai

Khi query match khoảng 10–30% row trở lên, full table scan thường nhanh hơn vì random I/O đắt hơn sequential I/O rất nhiều.

Tip PostgreSQL: `random_page_cost` mặc định 4.0 (tối ưu HDD). Nếu dùng SSD hoặc data fit trong RAM:

```sql
SET random_page_cost = 1.1;
```

Bảng nhỏ (100–200 rows): full scan là hành vi bình thường.

**Thống kê cũ:** sau bulk load / update / delete lớn, luôn làm mới statistics:

```sql
-- PostgreSQL
ANALYZE users;

-- SQL Server
UPDATE STATISTICS users WITH FULLSCAN;
-- Production thường đã bật AUTO_UPDATE_STATISTICS; vẫn nên cập nhật tay sau thao tác lớn.

-- MySQL
ANALYZE TABLE users;
```



### Database chọn index khác: Khi có nhiều lựa chọn

Khi có nhiều index đơn lẻ, database chọn index ước tính match ít row hơn. Giải pháp tốt nhất: composite index.

Khi query có cả WHERE và ORDER BY, index tốt nhất bao gồm cả hai: **cột filter trước, cột sort sau**.

```sql
SELECT * FROM issues
WHERE type = 'open' ORDER BY created_at DESC LIMIT 10;

-- ✅ Index: (type, created_at DESC)
```



### Parameter sniffing: Plan đúng với lần chạy đầu, sai với lần sau

SQL Server cache execution plan theo parameterized query. Lần đầu nhận `@status = 'rare'` → chọn Index Seek. Lần sau `@status = 'common'` (30% bảng) vẫn tái sử dụng Seek → thảm họa. Ngược lại cũng vậy.

```sql
-- SQL Server: xem plan đang cache; khi lệch thống kê / phân bố:
UPDATE STATISTICS orders WITH FULLSCAN;
-- Hoặc local, không cache plan cho query này:
SELECT * FROM orders WHERE status = @status OPTION (RECOMPILE);
-- Hoặc OPTIMIZE FOR UNKNOWN — dùng density trung bình, không sniff giá trị cụ thể
```

**PostgreSQL** có generic plan vs custom plan (prepared statement). Nếu thấy plan "lạ" sau `PREPARE`/`EXECUTE`, thử `DISCARD PLANS` hoặc không prepare query có selectivity thay đổi mạnh.

**MySQL:** prepared statement cũng cache plan; `ANALYZE TABLE` sau khi phân bố đổi.

### Tạo và bảo trì index mà không khóa bảng

Trên production, `CREATE INDEX` thường khóa write. Dùng bản online:

```sql
-- PostgreSQL
CREATE INDEX CONCURRENTLY idx_orders_status ON orders (status);
REINDEX INDEX CONCURRENTLY idx_orders_status;

-- SQL Server (Enterprise / một số edition)
CREATE INDEX idx_orders_status ON orders (status) WITH (ONLINE = ON);
ALTER INDEX idx_orders_status ON orders REBUILD WITH (ONLINE = ON);
-- Standard: REORGANIZE nhẹ hơn REBUILD
ALTER INDEX idx_orders_status ON orders REORGANIZE;

-- MySQL 8.0 InnoDB: DDL online mặc định cho nhiều trường hợp
CREATE INDEX idx_orders_status ON orders (status);
-- ALGORITHM=INPLACE, LOCK=NONE khi engine hỗ trợ
```



## Cạm bẫy và mẹo nâng cao về Indexing



### Index trên biểu thức: Khi không thể viết lại query

```sql
-- PostgreSQL: expression index — biểu thức trong WHERE phải khớp y chang
CREATE INDEX contacts_birthmonth ON contacts ((EXTRACT(MONTH FROM birthday)));
SELECT * FROM contacts WHERE EXTRACT(MONTH FROM birthday) = 5;

-- MySQL 8.0+: functional index
CREATE INDEX contacts_birthmonth ON contacts ((MONTH(birthday)));
SELECT * FROM contacts WHERE MONTH(birthday) = 5;

-- SQL Server / MariaDB: computed / virtual column rồi đánh index
ALTER TABLE contacts ADD birth_month AS (MONTH(birthday)) PERSISTED;  -- SQL Server
-- MariaDB: ADD COLUMN birth_month INT AS (MONTH(birthday)) VIRTUAL;
CREATE INDEX contacts_birthmonth ON contacts (birth_month);
SELECT * FROM contacts WHERE birth_month = 5;
```

Biểu thức trong index phải khớp chính xác với biểu thức trong `WHERE`. SQL Server / MariaDB thường đi qua cột ảo (computed / virtual / generated) rồi đánh index lên cột đó.

### Cột giá trị ít (boolean, trạng thái): Khi index trở nên vô nghĩa

Index trên boolean/`status` thường bị bỏ qua nếu giá trị khớp chiếm 10–30%+ tổng số dòng (random I/O đắt hơn sequential scan).

Cả hai hệ đều có index "chỉ một phần bảng" — rất hợp giá trị hiếm (`is_processed = false`, `status = 'error'`):

```sql
-- PostgreSQL: Partial Index
CREATE INDEX orders_unprocessed
ON orders (created_at)
WHERE is_processed = FALSE;

-- SQL Server: Filtered Index
CREATE INDEX orders_unprocessed
ON orders (created_at)
WHERE is_processed = 0;
```

Query phải chứa **đúng predicate** đó (hoặc predicate chặt hơn) thì optimizer mới dùng. Chỉ index boolean/status khi tìm giá trị hiếm (dưới vài phần trăm tổng số dòng).

### Biến đổi điều kiện phạm vi: Biến range thành so sánh bằng

Khi query có cả lọc range lẫn sắp xếp, biến range thành boolean để phễu không bị phá:

```sql
-- PostgreSQL
CREATE INDEX repos_search ON repos (
  language,
  ((stars > 1000)),
  sponsors
);

SELECT * FROM repos
WHERE language = 'TypeScript'
  AND (stars > 1000) = TRUE
ORDER BY sponsors ASC;

-- MySQL
CREATE INDEX repos_search ON repos (
  language,
  ((stars > 1000)),
  sponsors
);
-- hoặc IF():
-- CREATE INDEX repos_search ON repos (language, ((IF(stars > 1000, 1, 0))), sponsors);
SELECT * FROM repos
WHERE language = 'TypeScript'
  AND IF(stars > 1000, 1, 0) = 1
ORDER BY sponsors ASC;

-- SQL Server: computed column
ALTER TABLE repos ADD is_popular AS (CASE WHEN stars > 1000 THEN 1 ELSE 0 END) PERSISTED;
CREATE INDEX repos_search ON repos (language, is_popular, sponsors);

SELECT * FROM repos
WHERE language = 'TypeScript'
  AND is_popular = 1
ORDER BY sponsors ASC;
```



### Kiểu dữ liệu không khớp: Implicit conversion

Nếu hai bên so sánh khác kiểu, optimizer có thể convert **cột** → index không còn SARGable.

**PostgreSQL** thường từ chối so sánh khác kiểu (lỗi hoặc không dùng index), ít "âm thầm convert" hơn.

**MySQL:** nếu cột là `VARCHAR` nhưng so sánh với số, MySQL chuyển **cột** sang số → `CAST()` → index vô dụng.

```sql
-- ❌ chậm (MySQL)
SELECT * FROM orders WHERE payment_id = 57013925718;

-- ✅
SELECT * FROM orders WHERE payment_id = '57013925718';
```

Chiều ngược lại (cột số, giá trị chuỗi) thì MySQL thường an toàn hơn.

**SQL Server** rất hay implicit convert — đây là cạm bẫy số 1 trên production:

```sql
-- Cột payment_id là VARCHAR / NVARCHAR
-- ❌ số → convert cột → Index Scan
SELECT * FROM orders WHERE payment_id = 57013925718;

-- ✅ cùng kiểu với cột
SELECT * FROM orders WHERE payment_id = '57013925718';
```

Các cặp "giết Seek" phổ biến trên SQL Server:


| Cột                          | Tham số / literal                    | Kết quả               |
| ---------------------------- | ------------------------------------ | --------------------- |
| `VARCHAR`                    | `NVARCHAR` (ADO.NET string mặc định) | Convert cột → Scan    |
| `DATE` / `DATETIME2`         | `DATETIME`                           | Có thể không Seek     |
| `INT`                        | `VARCHAR`                            | Convert cột           |
| Collation khác nhau khi JOIN |                                      | Scan / Compute Scalar |


Trong plan, nhìn `CONVERT_IMPLICIT`. Fix: khớp kiểu cột, hoặc `NVARCHAR` parameter với cột `NVARCHAR`. Entity Framework / Dapper hay gửi `nvarchar` vào cột `varchar`.

### Truy vấn chỉ từ index: Không cần chạm vào bảng dữ liệu

Nếu mọi cột cần thiết nằm trong index, database bỏ qua bước load row (index-only query).

```sql
-- Bảng user_roles(user_id, role_id)
CREATE INDEX idx_user_roles ON user_roles (user_id, role_id);
CREATE INDEX idx_role_users ON user_roles (role_id, user_id);

SELECT role_id FROM user_roles WHERE user_id = 42;  -- index-only
SELECT user_id FROM user_roles WHERE role_id = 1;   -- index-only
```

`INCLUDE` thêm cột "đi kèm" mà không tham gia sort / UNIQUE:

```sql
-- PostgreSQL, SQL Server, MySQL 8.0.13+
CREATE INDEX invoices_covering
ON invoices (customer_id, year)
INCLUDE (price);
```

Dùng `INCLUDE` khi chỉ cần cột đó để tránh nhảy vào bảng (Index Only Scan / covering), không cần filter hay sort theo cột đó. Đặt cột filter/sort trong key; cột chỉ `SELECT` thì để `INCLUDE`.

### Lọc và sắp xếp khi JOIN: Bài toán không giải được bằng index

Khi thông tin lọc nằm ở 2 bảng khác nhau, database phải join hàng chục nghìn lần. Đó là tín hiệu **thiết kế lại schema** (denormalize), không phải thêm index.

```sql
-- Copy trạng thái project vào tasks, rồi query không cần JOIN
ALTER TABLE tasks ADD COLUMN project_status VARCHAR(20);
SELECT * FROM tasks
WHERE team_id = 4 AND status = 'open' AND project_status = 'open';
```



### Vượt giới hạn kích thước index

Ba kỹ thuật:

1. Index biểu thức / computed column trên tiền tố: `LEFT(title, 20)` / `SUBSTRING(title, 1, 20)`
2. Hash của chuỗi dài rồi index cột hash
3. **PostgreSQL** Hash Index: `USING HASH (uniqid)` (chỉ `=`). **SQL Server** hash index chỉ có trên memory-optimized table — với disk-based table cứ dùng B-tree. **MySQL:** prefix index `title(20)` vẫn hay dùng cho `VARCHAR`/`TEXT` dài.



### JSON: Đánh index trong thế giới phi cấu trúc

- 1–2 trường cố định: expression index (PostgreSQL), computed column (SQL Server), hoặc cột ảo (MySQL)
- Nhiều trường, tìm linh hoạt: **PostgreSQL GIN** trên `jsonb`

```sql
-- PostgreSQL
CREATE INDEX contacts_email ON contacts ((attributes->>'email'));
CREATE INDEX contacts_attrs ON contacts USING GIN (attributes);
SELECT * FROM contacts WHERE attributes @> '{"email": "admin@example.com"}';

-- MySQL: generated / virtual column
ALTER TABLE contacts ADD COLUMN email VARCHAR(255)
  GENERATED ALWAYS AS (attributes->>'$.email') STORED;
CREATE INDEX contacts_email ON contacts (email);

-- SQL Server: computed column từ JSON path
ALTER TABLE contacts ADD email AS (JSON_VALUE(attributes, '$.email')) PERSISTED;
CREATE INDEX contacts_email ON contacts (email);
SELECT * FROM contacts WHERE email = 'admin@example.com';
```



### Ràng buộc duy nhất và giá trị NULL: Lỗi bất ngờ

Trong SQL, `NULL ≠ NULL`, nên UNIQUE không chặn hai `(17, NULL)`.

```sql
-- PostgreSQL 15+
CREATE UNIQUE INDEX one_pending_order
ON orders (customer_id, shipment_id)
NULLS NOT DISTINCT;

-- SQL Server: UNIQUE cho phép nhiều NULL. Muốn "chỉ một NULL":
CREATE UNIQUE INDEX one_pending_order
ON orders (customer_id, shipment_id)
WHERE shipment_id IS NOT NULL;
-- (không chặn nhiều row (customer_id, NULL) — nếu cần chặn, dùng filtered
--  index riêng hoặc computed column thay NULL bằng sentinel)

-- Mọi hệ: thay NULL bằng giá trị đặc biệt trong biểu thức / computed column
CREATE UNIQUE INDEX one_pending_order ON orders (
  customer_id,
  (CASE WHEN shipment_id IS NULL THEN -1 ELSE shipment_id END)
);
```



### Tìm và dọn dẹp index không sử dụng

- Overlapping: `(country, lastname, firstname)` đã bao gồm `(country)` và `(country, lastname)`
- Unused — theo dõi ít nhất 1–2 chu kỳ nghiệp vụ (đừng xóa index "chưa dùng tuần này"):

```sql
-- PostgreSQL
SELECT indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan;

-- SQL Server
SELECT OBJECT_NAME(s.object_id) AS table_name,
       i.name AS index_name,
       s.user_seeks, s.user_scans, s.user_lookups, s.user_updates
FROM sys.dm_db_index_usage_stats s
JOIN sys.indexes i ON i.object_id = s.object_id AND i.index_id = s.index_id
WHERE s.database_id = DB_ID();

-- MySQL: performance_schema
SELECT object_schema, object_name, index_name, count_star
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'app'
ORDER BY count_star;
```

- Trước khi DROP:
  - **SQL Server:** `ALTER INDEX ... DISABLE` (bật lại bằng `REBUILD`)
  - **MySQL:** `ALTER TABLE ... ALTER INDEX idx INVISIBLE` — test rồi mới xóa
  - **PostgreSQL:** không có invisible/disable — thử trên staging hoặc đổi tên tạm



### Điều kiện "ma": Giúp database mà không thay đổi kết quả

Thêm điều kiện thừa theo quy tắc nghiệp vụ để database dùng thêm cột index. Ghi chú rõ trong code — nếu rule đổi, query sẽ lọc sai.

### Tìm kiếm theo vị trí: Khi hai điều kiện phạm vi đụng nhau

Longitude + latitude = hai range → index B-tree chỉ dùng được một. Dùng Spatial Index (R-tree / GIST).

```sql
-- PostgreSQL
CREATE INDEX search_idx ON businesses USING GIST (type, location);
SELECT * FROM businesses
WHERE type = 'restaurant'
  AND location && ST_MakeEnvelope(-74.0083, 40.7216, -73.9752, 40.7422, 4326);

-- SQL Server
CREATE SPATIAL INDEX search_idx ON businesses (location);
SELECT * FROM businesses
WHERE type = 'restaurant'
  AND location.STIntersects(geography::STGeomFromText(
    'POLYGON((...))', 4326)) = 1;

-- MySQL: spatial index chỉ 1 cột, tính khoảng cách trên mặt phẳng
CREATE SPATIAL INDEX search_idx ON businesses (location);
```

PostgreSQL hỗ trợ nhiều cột trên GiST + SRID 4326. MySQL spatial index chỉ 1 cột. SQL Server có `geography` / `geometry` + spatial index riêng.

### Tìm kiếm ký tự đại diện ở đầu: Trường hợp đặc biệt

`LIKE '%abc%'` không dùng B-tree. PostgreSQL: trigram index (`pg_trgm`). MySQL/SQL Server: dùng search engine chuyên biệt.

## Kỹ thuật thao tác dữ liệu hiệu quả



### Tranh chấp khóa: Khi bộ đếm bị "nghẽn cổ chai"

Phân tán counter thành nhiều row (fanout) để update song song:

```sql
-- MySQL
INSERT INTO post_statistics (post_id, fanout, likes_count)
VALUES (1475870220422107137, FLOOR(RAND() * 100), 1)
ON DUPLICATE KEY UPDATE likes_count = likes_count + VALUES(likes_count);

-- PostgreSQL
INSERT INTO post_statistics (post_id, fanout, likes_count)
VALUES (1475870220422107137, FLOOR(random() * 100), 1)
ON CONFLICT (post_id, fanout)
DO UPDATE SET likes_count = post_statistics.likes_count + EXCLUDED.likes_count;

-- SQL Server
MERGE post_statistics AS t
USING (SELECT 1475870220422107137 AS post_id, ABS(CHECKSUM(NEWID())) % 100 AS fanout, 1 AS likes_count) AS s
ON t.post_id = s.post_id AND t.fanout = s.fanout
WHEN MATCHED THEN UPDATE SET likes_count = t.likes_count + 1
WHEN NOT MATCHED THEN INSERT (post_id, fanout, likes_count) VALUES (s.post_id, s.fanout, s.likes_count);

SELECT SUM(likes_count) FROM post_statistics WHERE post_id = 1475870220422107137;
```



### Cập nhật dữ liệu từ bảng khác: JOIN trong UPDATE

```sql
-- MySQL
UPDATE products
JOIN categories USING (category_id)
SET price = price_base - price_base * categories.discount;

-- PostgreSQL
UPDATE products
SET price = price_base - price_base * categories.discount
FROM categories
WHERE products.category_id = categories.category_id;

-- SQL Server
UPDATE p
SET price = price_base - price_base * c.discount
FROM products p
JOIN categories c ON p.category_id = c.category_id;
```



### Lấy dữ liệu ngay sau khi thay đổi: RETURNING / OUTPUT

```sql
-- PostgreSQL
DELETE FROM sessions WHERE ip = '127.0.0.1'
RETURNING id, user_agent, last_access;

-- SQL Server
DELETE FROM sessions
OUTPUT deleted.id, deleted.user_agent, deleted.last_access
WHERE ip = '127.0.0.1';

-- MySQL 8.0.21+: RETURNING cho một số statement (MariaDB hỗ trợ rộng hơn)
-- DELETE FROM sessions WHERE ip = '127.0.0.1' RETURNING id, user_agent, last_access;
```



### Xóa dòng trùng lặp: Dùng CTE thay vì xử lý ở tầng ứng dụng

```sql
WITH duplicates AS (
  SELECT id, ROW_NUMBER() OVER (
    PARTITION BY firstname, lastname, email
    ORDER BY age DESC
  ) AS rownum
  FROM contacts
)
DELETE FROM contacts
USING duplicates
WHERE contacts.id = duplicates.id AND duplicates.rownum > 1;

-- SQL Server
-- DELETE c FROM contacts c
-- INNER JOIN duplicates d ON c.id = d.id
-- WHERE d.rownum > 1;
```



## Viết query như chuyên gia



### Phân trang đúng cách: Phân trang theo khóa

- Thêm PK vào `ORDER BY` để thứ tự ổn định
- Keyset pagination thay vì `OFFSET` lớn:

```sql
-- PostgreSQL / MySQL 8.0.13+ (row constructor)
SELECT * FROM users
WHERE (firstname, lastname, id) > ('Huy', 'Nguyen', 3150)
ORDER BY firstname, lastname, id
LIMIT 30;

-- SQL Server: không so sánh tuple — viết dạng bậc thang
SELECT TOP (30) *
FROM users
WHERE firstname > 'Huy'
   OR (firstname = 'Huy' AND lastname > 'Nguyen')
   OR (firstname = 'Huy' AND lastname = 'Nguyen' AND id > 3150)
ORDER BY firstname, lastname, id;

-- OFFSET/FETCH (SQL Server, cũng có trên PostgreSQL) — tránh OFFSET lớn
SELECT * FROM users
ORDER BY firstname, lastname, id
OFFSET 0 ROWS FETCH NEXT 30 ROWS ONLY;
```



### FOR UPDATE / UPDLOCK: Khóa dòng ở tầng database

```sql
-- PostgreSQL / MySQL
START TRANSACTION;
SELECT balance FROM account WHERE account_id = 7 FOR UPDATE;
UPDATE account SET balance = 540 WHERE account_id = 7;
COMMIT;

-- SQL Server
BEGIN TRAN;
SELECT balance FROM account WITH (UPDLOCK, HOLDLOCK)
WHERE account_id = 7;
UPDATE account SET balance = 540 WHERE account_id = 7;
COMMIT;
```



### Biểu thức bảng tạm (CTE): Xử lý query phức tạp

Chia query thành bước nhỏ, mỗi CTE test độc lập — dễ debug hơn nested subquery.

### Các tips query hữu ích khác

```sql
-- Tránh division by zero
SELECT visitors_today / NULLIF(visitors_yesterday, 0) FROM stats;

-- Gap-filling (PostgreSQL)
SELECT dates.day, COALESCE(SUM(stats.count), 0)
FROM generate_series(CURRENT_DATE - INTERVAL '14 days', CURRENT_DATE, '1 day') AS dates(day)
LEFT JOIN statistics stats ON stats.day = dates.day
GROUP BY dates.day;

-- SQL Server 2022+: GENERATE_SERIES; bản cũ dùng tally / recursive CTE
SELECT d.day, COALESCE(SUM(stats.count), 0)
FROM GENERATE_SERIES(CAST(DATEADD(DAY, -14, CAST(GETDATE() AS date)) AS date), CAST(GETDATE() AS date), 1) AS d(day)
LEFT JOIN statistics stats ON stats.day = d.day
GROUP BY d.day;

-- Multiple aggregates
-- PostgreSQL:
SELECT
  COUNT(*) FILTER (WHERE released_at = 2024) AS released_2024,
  COUNT(*) FILTER (WHERE director = 'Nolan') AS nolan_movies
FROM movies;
-- MySQL:
SELECT
  SUM(released_at = 2024) AS released_2024,
  SUM(director = 'Nolan') AS nolan_movies
FROM movies;
-- SQL Server:
SELECT
  SUM(CASE WHEN released_at = 2024 THEN 1 ELSE 0 END) AS released_2024,
  SUM(CASE WHEN director = 'Nolan' THEN 1 ELSE 0 END) AS nolan_movies
FROM movies;

-- DISTINCT ON (PostgreSQL): đơn đắt nhất mỗi customer
SELECT DISTINCT ON (customer_id) *
FROM orders
WHERE EXTRACT(YEAR FROM created_at) = 2024
ORDER BY customer_id ASC, price DESC;

-- SQL Server / MySQL: cùng ý tưởng bằng ROW_NUMBER()
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY price DESC) AS rn
  FROM orders
  WHERE YEAR(created_at) = 2024
) t WHERE rn = 1;
```



## Thiết kế Schema: Nền móng vững chắc



### UUID vs Auto-increment: Lựa chọn Primary Key


| Tiêu chí           | Auto-increment               | UUIDv4                 | UUIDv7/ULID         |
| ------------------ | ---------------------------- | ---------------------- | ------------------- |
| Insert speed       | Nhanh nhất                   | Chậm (random position) | Nhanh (time-sorted) |
| Size               | 4–8 bytes                    | 16 bytes               | 16 bytes            |
| Predictable        | Dễ đoán (enumeration attack) | Không đoán được        | Không đoán được     |
| Distributed ID gen | Không                        | Có                     | Có                  |


Khuyến nghị: auto-increment cho internal PK; thêm cột UUID riêng cho URL/API.

```sql
-- PostgreSQL
ALTER TABLE users ADD COLUMN uuid UUID NOT NULL DEFAULT gen_random_uuid();
CREATE UNIQUE INDEX users_uuid ON users (uuid);

-- SQL Server
ALTER TABLE users ADD uuid UNIQUEIDENTIFIER NOT NULL DEFAULT NEWSEQUENTIALID();
CREATE UNIQUE INDEX users_uuid ON users (uuid);

-- MySQL
ALTER TABLE users ADD COLUMN uuid BINARY(16) NOT NULL DEFAULT (UUID_TO_BIN(UUID(), 1));
CREATE UNIQUE INDEX users_uuid ON users (uuid);
```



### JSON Column: Khi NoSQL gặp SQL

Dùng JSON khi: metadata/settings ít query; thay EAV; giảm JOIN cho data seldom-used.

Vẫn dùng relational cho data chính. Tránh deeply nested JSON. Đừng lưu FK trong JSON.

```sql
-- MySQL 8.0.17+
ALTER TABLE products ADD CONSTRAINT CHECK (
  JSON_SCHEMA_VALID('{
    "type": "object",
    "properties": {
      "tags": {"type": "array", "items": {"type": "string"}}
    },
    "additionalProperties": false
  }', attributes)
);

-- SQL Server: ISJSON
ALTER TABLE products ADD CONSTRAINT attributes_is_json CHECK (ISJSON(attributes) = 1);

-- PostgreSQL: jsonb tự validate JSON; schema chặt hơn cần extension hoặc check tay
ALTER TABLE products ADD CONSTRAINT attributes_is_object CHECK (jsonb_typeof(attributes) = 'object');
```



### Constraint: Hàng rào bảo vệ cuối cùng

```sql
ALTER TABLE reservations
ADD CONSTRAINT start_before_end CHECK (checkin_at < checkout_at);

ALTER TABLE invoices
ADD CONSTRAINT eu_vat CHECK (NOT (is_eu) OR vatid IS NOT NULL);
```

Application validation có thể bị bypass. Database constraint luôn được enforce.

### Ràng buộc loại trừ: Chống chồng chéo (PostgreSQL)

```sql
CREATE TABLE bookings (
  room_number INT,
  reservation TSTZRANGE,
  EXCLUDE USING GIST (room_number WITH =, reservation WITH &&)
);
```



### Đường dẫn vật lý hóa: Lưu trữ cây đơn giản

```sql
-- PostgreSQL
CREATE EXTENSION ltree;
CREATE TABLE categories (path LTREE);
INSERT INTO categories VALUES ('Food'), ('Food.Fruit'), ('Food.Fruit.Cherry');
SELECT * FROM categories WHERE path ~ 'Food.Fruit.*{1,}';

-- SQL Server: hierarchyid
CREATE TABLE categories (path HIERARCHYID PRIMARY KEY);
-- GetDescendant / IsDescendantOf để duyệt cây
```



### Partition: Xóa data lớn trong tích tắc

```sql
-- MySQL
ALTER TABLE logs DROP PARTITION logs_2023_january;

-- PostgreSQL
DROP TABLE logs_2023_january;  -- hoặc DETACH PARTITION rồi DROP

-- SQL Server
TRUNCATE TABLE logs WITH (PARTITIONS (1));  -- hoặc SWITCH ra bảng staging rồi DROP
```



### Bảng sắp xếp trước: Tối ưu cho quét phạm vi

```sql
-- MySQL: composite PK để sắp xếp vật lý
CREATE TABLE product_comments (
  product_id BIGINT,
  comment_id BIGINT AUTO_INCREMENT UNIQUE KEY,
  message TEXT,
  PRIMARY KEY (product_id, comment_id)
);

-- PostgreSQL: CLUSTER một lần, không tự duy trì
CLUSTER product_comments USING product_comments_pkey;

-- SQL Server: clustered index *luôn* giữ thứ tự vật lý (đây là mặc định)
-- CREATE CLUSTERED INDEX ... ON product_comments (product_id, comment_id);
```



### Tính toán trước: Khi index cũng không đủ nhanh

Lưu sẵn giá trị đã aggregate:

```sql
CREATE TABLE articles_stats (
  user_id BIGINT,
  publish_year INT,
  total_likes BIGINT,
  PRIMARY KEY (user_id, publish_year)
);

SELECT total_likes FROM articles_stats
WHERE user_id = 1 AND publish_year = 2024;
```

