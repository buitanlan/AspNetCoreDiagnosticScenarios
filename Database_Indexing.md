# Database Indexing

## Mục lục

- [Phần 1. Nền tảng: Hiểu Index từ gốc rễ](#phần-1-nền-tảng-hiểu-index-từ-gốc-rễ)
  - [B+ Tree](#b-tree-không-cần-hiểu-chi-tiết-chỉ-cần-hiểu-ý-tưởng)
  - [Có sắp xếp hay không](#có-sắp-xếp-hay-không-chuyện-của-primary-key)
  - [Heap vs Clustered](#hai-cách-lưu-trữ-khác-nhau-heap-table-và-clustered-index)
  - [Có index chưa chắc nhanh](#có-index-chưa-chắc-query-nhanh-hiểu-lầm-phổ-biến-nhất)
- [Phần 2. Bốn nguyên tắc vàng](#phần-2-bốn-nguyên-tắc-vàng-khi-sử-dụng-index)
  - [1. Tra cứu nhanh](#nguyên-tắc-1-tra-cứu-nhanh--nhảy-thẳng-đến-vị-trí-cần-tìm)
  - [2. Quét một hướng](#nguyên-tắc-2-quét-theo-một-hướng)
  - [3. Phễu trái → phải](#nguyên-tắc-3-từ-trái-sang-phải--nguyên-tắc-phễu-cho-index-nhiều-cột)
  - [4. Range phá phễu](#nguyên-tắc-4-quét-khi-gặp-điều-kiện-phạm-vi--điều-kiện-phạm-vi-phá-vỡ-phễu)
- [Phần 3. Đọc execution plan trong 30 giây](#phần-3-đọc-execution-plan-trong-30-giây)
  - [Đỏ](#đỏ--xử-lý-ngay)
  - [Vàng](#vàng--chưa-chắc-sai)
  - [Xanh](#xanh--plan-tốt)
  - [Checklist 30s](#30-giây--checklist)
- [Phần 4. Index với từng thao tác SQL](#phần-4-index-hoạt-động-thế-nào-với-từng-thao-tác-sql)
  - [So sánh không bằng !=](#phép-so-sánh-không-bằng--kẻ-giết-hiệu-suất-thầm-lặng)
  - [NULL](#null-giá-trị-đặc-biệt-cần-đặc-biệt-chú-ý)
  - [LIKE](#like-tìm-kiếm-mẫu-chuỗi-và-cái-bẫy-ký-tự-đại-diện-ở-đầu)
  - [ORDER BY](#order-by-tránh-bước-sort-bổ-sung-bằng-mọi-giá)
  - [GROUP BY & DISTINCT](#group-by--distinct-thách-thức-lớn-nhất)
  - [JOIN](#join-phân-tách-và-kết-hợp)
  - [Subquery](#subquery-không-chậm-như-bạn-nghĩ)
  - [UPDATE & DELETE](#update--delete-đừng-quên-tối-ưu-cho-chúng)
  - [IN](#in-nhiều-equality-không-phải-range)
  - [HAVING](#having-không-thay-được-where)
  - [UNION / UNION ALL](#union-và-union-all-mỗi-nhánh-một-index)
  - [Window function](#window-function-partition--order-cũng-cần-index)
- [Phần 5. Tại sao Database không dùng Index](#phần-5-tại-sao-database-không-dùng-index-của-tôi)
  - [Quy trình thực thi](#quy-trình-thực-thi-query-bên-trong-bộ-não-của-database)
  - [Index không khớp](#index-không-khớp-với-query-lý-do-phổ-biến-nhất)
  - [Full scan nhanh hơn](#full-table-scan-nhanh-hơn-khi-database-đúng-mà-bạn-sai)
  - [Chọn index khác](#database-chọn-index-khác-khi-có-nhiều-lựa-chọn)
  - [Parameter sniffing](#parameter-sniffing-plan-đúng-với-lần-chạy-đầu-sai-với-lần-sau)
  - [Tạo index không khóa bảng](#tạo-và-bảo-trì-index-mà-không-khóa-bảng)
  - [Seek rồi Filter](#seek-rồi-filter-index-dùng-mà-vẫn-chậm)
  - [Key Lookup](#key-lookup--heap-fetch-khi-seek-thua-scan)
  - [Partial / filtered index](#partial--filtered-index-không-khớp-predicate)
  - [Thống kê lệch](#thống-kê-lệch-và-cột-tương-quan)
  - [View bọc cột](#view-và-hàm-bọc-cột-index-có-mà-optimizer-mù)
- [Phần 6. Cạm bẫy và mẹo nâng cao](#phần-6-cạm-bẫy-và-mẹo-nâng-cao-về-indexing)
  - [Index trên biểu thức](#index-trên-biểu-thức-khi-không-thể-viết-lại-query)
  - [Cột ít giá trị](#cột-giá-trị-ít-boolean-trạng-thái-khi-index-trở-nên-vô-nghĩa)
  - [Biến range thành =](#biến-đổi-điều-kiện-phạm-vi-biến-range-thành-so-sánh-bằng)
  - [Implicit conversion](#kiểu-dữ-liệu-không-khớp-implicit-conversion)
  - [Covering / index-only](#truy-vấn-chỉ-từ-index-không-cần-chạm-vào-bảng-dữ-liệu)
  - [Lọc + sort khi JOIN](#lọc-và-sắp-xếp-khi-join-bài-toán-không-giải-được-bằng-index)
  - [Giới hạn kích thước](#vượt-giới-hạn-kích-thước-index)
  - [JSON](#json-đánh-index-trong-thế-giới-phi-cấu-trúc)
  - [UNIQUE và NULL](#ràng-buộc-duy-nhất-và-giá-trị-null-lỗi-bất-ngờ)
  - [Index không dùng](#tìm-và-dọn-dẹp-index-không-sử-dụng)
  - [Điều kiện “ma”](#điều-kiện-ma-giúp-database-mà-không-thay-đổi-kết-quả)
  - [Tìm theo vị trí](#tìm-kiếm-theo-vị-trí-khi-hai-điều-kiện-phạm-vi-đụng-nhau)
  - [Wildcard ở đầu](#tìm-kiếm-ký-tự-đại-diện-ở-đầu-trường-hợp-đặc-biệt)
  - [OR trên hai cột](#or-trên-hai-cột-một-index-không-đủ-phễu)
  - [Index phình to](#index-phình-to-bloat-fragmentation-reindex)
- [Phần 7. Thao tác dữ liệu hiệu quả](#phần-7-kỹ-thuật-thao-tác-dữ-liệu-hiệu-quả)
  - [Tranh chấp khóa](#tranh-chấp-khóa-khi-bộ-đếm-bị-nghẽn-cổ-chai)
  - [UPDATE … JOIN](#cập-nhật-dữ-liệu-từ-bảng-khác-join-trong-update)
  - [RETURNING / OUTPUT](#lấy-dữ-liệu-ngay-sau-khi-thay-đổi-returning--output)
  - [Xóa dòng trùng](#xóa-dòng-trùng-lặp-dùng-cte-thay-vì-xử-lý-ở-tầng-ứng-dụng)
  - [UPSERT](#upsert-insert-hoặc-update-một-lần)
  - [Xóa / sửa theo lô](#xóa--sửa-theo-lô-đừng-nuốt-cả-bảng-trong-một-transaction)
  - [Nạp hàng loạt](#nạp-dữ-liệu-hàng-loạt-copy-bulk-insert-sqlbulkcopy)
- [Phần 8. Viết query như chuyên gia](#phần-8-viết-query-như-chuyên-gia)
  - [Phân trang theo khóa](#phân-trang-đúng-cách-phân-trang-theo-khóa)
  - [FOR UPDATE / UPDLOCK](#for-update--updlock-khóa-dòng-ở-tầng-database)
  - [SKIP LOCKED](#skip-locked-hàng-đợi-không-tranh-chấp)
  - [EXISTS / NOT EXISTS](#exists-và-not-exists-semi-join--anti-join)
  - [LATERAL / APPLY](#lateral--cross-apply-top-n-mỗi-nhóm)
  - [CTE](#biểu-thức-bảng-tạm-cte-xử-lý-query-phức-tạp)
  - [Tips khác](#các-tips-query-hữu-ích-khác)
  - [N+1](#n1-vòng-lặp-query-vs-một-câu-sql)
  - [Optional filter](#optional-filter-or-is-null-phá-index)
  - [LEFT JOIN IS NULL](#left-join--is-null-anti-join-dễ-viết-sai)
  - [COUNT lớn](#count-lớn-đừng-đếm-cả-bảng-mỗi-request)
  - [Khoảng nửa-mở](#khoảng-thời-gian-nửa-mở)
- [Phần 9. Thiết kế Schema](#phần-9-thiết-kế-schema-nền-móng-vững-chắc)
  - [UUID vs Auto-increment](#uuid-vs-auto-increment-lựa-chọn-primary-key)
  - [JSON column](#json-column-khi-nosql-gặp-sql)
  - [Constraint](#constraint-hàng-rào-bảo-vệ-cuối-cùng)
  - [Exclusion](#ràng-buộc-loại-trừ-chống-chồng-chéo-postgresql)
  - [Materialized path](#đường-dẫn-vật-lý-hóa-lưu-trữ-cây-đơn-giản)
  - [Partition](#partition-xóa-data-lớn-trong-tích-tắc)
  - [Bảng sắp sẵn](#bảng-sắp-xếp-trước-tối-ưu-cho-quét-phạm-vi)
  - [Tính toán trước](#tính-toán-trước-khi-index-cũng-không-đủ-nhanh)
  - [Soft delete](#soft-delete-deleted_at-và-index)
  - [Clustered key hẹp](#khóa-clustered-hẹp-đừng-nhét-uuid-vào-mọi-secondary)
  - [Multi-tenant](#multi-tenant-tenant_id-đứng-đầu-schema)
  - [Status / ENUM](#status-enum-lookup-hay-varchar)
  - [Bảng nối N–N](#bảng-nối-nhiều-nhiều-pk-kép-thay-id-thừa)
  - [Thời gian](#thời-gian-timestamptz-không-timestamp-without-time-zone)
- [Phần 10. Checklist](#phần-10-checklist--lỗ-hổng-hay-quên)
  - [Index khóa ngoại](#luôn-index-khóa-ngoại)
  - [Đừng SELECT *](#đừng-select--nếu-muốn-covering)
  - [HOT update](#hot-update-postgresql-và-cột-không-nằm-trong-index)
  - [BRIN / columnstore](#brin--columnstore-cho-dữ-liệu-append-only)
  - [Thống kê](#thống-kê-histogram-và-index-có-mà-không-seek)
  - [Invisible / disable](#invisible--disable-trước-khi-drop)
  - [Trước khi CREATE INDEX](#trước-khi-create-index)
- [Bonus. Hiểu sâu B+ Tree trong RDBMS](#bonus-hiểu-sâu-b-tree-trong-rdbms)
  - [B-tree vs B+ Tree](#b-tree-vs-b-tree-data-chỉ-nằm-ở-lá)
  - [Page, fanout, chiều cao](#page-fanout-chiều-cao-cây)
  - [Lá nối nhau](#lá-nối-nhau-vì-sao-range-chỉ-đi-một-hướng)
  - [Page split](#page-split-random-insert-đắt-hơn-sequential)
  - [Leaf chứa gì](#leaf-chứa-gì-tid-heap-vs-clustered-key)
  - [Fillfactor](#fillfactor-chỗ-trống-cố-ý)
  - [Khi nào không dùng B+ Tree](#khi-nào-không-dùng-b-tree)

> **Quy ước trong ebook:** Ví dụ lấy **PostgreSQL** và **SQL Server (MSSQL)** làm hệ chính. Cú pháp MySQL, Oracle, MariaDB được **giữ nguyên** khi khác biệt — không thay thế, chỉ bổ sung bên cạnh.



## Phần 1. Nền tảng: Hiểu Index từ gốc rễ



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

Muốn hiểu *vì sao* random UUID làm page split, leaf chứa TID hay clustered key, fanout ~ vài trăm, đọc [Bonus — B+ Tree trong RDBMS](#bonus-hiểu-sâu-b-tree-trong-rdbms) sau khi xong bốn nguyên tắc. Bốn nguyên tắc không phụ thuộc chi tiết đó.

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

Câu hỏi hay gặp: "Vậy tạo bao nhiêu index là quá nhiều?" Không có con số cố định, nhưng nếu bảng có hơn 10 index và bạn thấy write performance giảm đáng kể, đó là lúc nên review lại. Dùng query kiểm tra unused index (sẽ học ở Phần 6) để tìm và xóa index thừa.

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

Điểm quan trọng: mọi index (kể cả PK) đều bình đẳng — đều trỏ `ctid`. Lookup PK hay secondary đều là: Seek trên index rồi **fetch heap** (2 bước logic). Khác clustered: PK Seek là xong, không fetch thêm.

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

Bù lại, secondary lookup phải nhảy thêm một bước: **Key Lookup** (SQL Server) hoặc PK lookup (MySQL). Nếu query lấy nhiều cột ngoài index, bước này hàng loạt có thể đắt hơn table scan. Giải pháp: covering index — PostgreSQL / SQL Server dùng `INCLUDE` (cột đi kèm, không tham gia sort). **MySQL không có `INCLUDE`**: InnoDB secondary index đã chứa PK, muốn covering thì phải đưa các cột `SELECT` vào *key* của index (index sẽ rộng hơn).

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

## Phần 2. Bốn nguyên tắc vàng khi sử dụng Index

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

Nhưng nhớ: Scan chỉ đi một hướng. Không thể vừa scan ascending vừa scan descending cùng lúc trong một index traversal. Nếu query cần sort theo 2 cột với hướng khác nhau (vd: `ORDER BY score DESC, created_at ASC`), bạn cần tạo index với đúng thứ tự sort đó (xem thêm ở Phần 4 - ORDER BY).

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
-- ✅ Dùng 3/3 bước phễu
WHERE country = 'VN' AND lastname = 'Nguyen' AND firstname = 'Huy';

-- ✅ Dùng 2/3 bước phễu (firstname bị bỏ → vẫn OK, chỉ kém tối ưu hơn)
WHERE country = 'VN' AND lastname = 'Nguyen';

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

Tuy nhiên, bỏ cột giữa vẫn tốt hơn không có index: vì ít nhất database giới hạn được phạm vi scan (chỉ entries có firstname = 'Huy'), và có thể filter trên country ngay trong index mà không cần load row từ table.

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

- MySQL 8.0.13+: Index Skip Scan (và Loose Index Scan cho một số `GROUP BY`)
- Oracle: Index Skip Scan
- SQL Server: **không có** Skip Scan kiểu Oracle. Thiếu cột đầu thường = không Seek được composite index (có thể Index Intersection / Scan, đừng thiết kế dựa vào đó)
- PostgreSQL 18+: B-tree Skip Scan (bản cũ: thiếu cột đầu ≈ không dùng được composite index)

Nhưng đây là tối ưu đặc biệt, không phải hành vi mặc định. Đừng thiết kế index dựa trên nó.

**Quy tắc vàng cần nhớ:**

1. Equality columns trước, Range columns sau
2. Nếu có nhiều range conditions, đặt cột filter được nhiều nhất trước
3. Sau cột range đầu tiên, các cột tiếp theo chỉ dùng để filter (vẫn hữu ích, nhưng không giới hạn scan range)

`IN (...)` và `IS NULL` tính như **equality** (nhiều seek / một khối NULL). `LIKE 'Nguyen%'` tính như **range**. `!=` / `NOT IN` không đi phễu — xem Phần 4.

#### Bốn nguyên tắc đã đủ chưa?

Đủ cho **cách đi trên index** (seek → scan một hướng → phễu trái→phải → range cắt phễu). Đó đúng 4 thao tác B-tree làm được.

Không nhét thành “nguyên tắc 5”: **covering** (`INCLUDE` / đủ cột trong key) — đó là *sau khi* phễu đúng, đừng nhảy table. **OR hai cột khác nhau** thường không nhét một phễu; cần hai index hoặc viết `UNION ALL`.

Công thức đóng một dòng — vẽ query ra rồi điền:

```text
INDEX ( equality, equality, … , range_hoặc_ORDER_BY )  INCLUDE ( cột chỉ SELECT )
```

Xong thì mở plan: seek đúng prefix đó chưa? Lookup/sort còn không? — [30 giây](#phần-3-đọc-execution-plan-trong-30-giây).

## Phần 3. Đọc execution plan trong 30 giây

Không cần thuộc hết operator. Mở **plan thật** (có chạy query), tìm chỗ **đắt nhất**, đối chiếu bảng đỏ dưới đây.

```sql
-- PostgreSQL (chạy thật + IO)
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT ...;

-- SQL Server: Actual Plan (Ctrl+M) — đừng dừng ở Estimated
SET STATISTICS IO, TIME ON;
SELECT ...;

-- MySQL 8.0.18+
EXPLAIN ANALYZE SELECT ...;
-- hoặc EXPLAIN FORMAT=TREE / EXPLAIN (cột type, Extra, rows)
```

Cách nhìn 10 giây đầu:

| Hệ | Nhìn gì trước |
| --- | ------------- |
| SQL Server | Node **% Cost** cao nhất; mũi tên **dày** = nhiều row. Đọc **phải → trái** |
| PostgreSQL | `actual time` / `Buffers` lớn nhất; so `rows` ước lượng vs `actual rows` |
| MySQL | Cột `type` + `Extra`; `rows` × số vòng join |

### Đỏ — xử lý ngay

| Operator (SQL Server) | PostgreSQL | MySQL `type` / Extra | Nghĩa |
| --------------------- | ---------- | -------------------- | ----- |
| **Table Scan** / **Clustered Index Scan** trên bảng lớn, filter hẹp | **Seq Scan** + `Filter` trên bảng lớn | `ALL` | Không dùng index, hoặc index vô dụng (hàm trên cột, sai kiểu, selectivity cao) |
| **Index Scan** hết index rồi filter | Seq/Index Scan + Filter | `index` (full index scan) | Đi cả lá index — gần đắt như table scan |
| **Key Lookup** / **RID Lookup** × hàng chục nghìn | **Index Scan** + heap fetch; Index Only Scan với **Heap Fetches** lớn | (InnoDB: PK lookup sau secondary) | Index không covering → `INCLUDE` / thêm cột vào key |
| **Sort** có warning **spill to tempdb** | Sort **external merge Disk**; `BUFFERS` đọc file | `Using filesort` | Thiếu index khớp `ORDER BY` (nguyên tắc 2 + 4) |
| **Hash Match** spill; memory grant khổng lồ | Hash Join / HashAggregate **Batches: Disk** | `Using temporary` | Join/GROUP BY không được index; hoặc thống kê sai |
| **Nested Loops** outer lớn × inner Scan | Nested Loop `loops` lớn × inner Seq Scan | `type=ALL` ở bảng trong join | Inner chưa Seek được — thiếu index join/FK |
| **Compute Scalar** `CONVERT_IMPLICIT` | (cast cột) | `type=ALL` dù có index | Sai kiểu dữ liệu (Phần 6 — implicit conversion) |

Mũi tên / row count: Seek 5 row rồi Lookup 200.000 lần = đỏ, dù operator tên “Seek”.

### Vàng — chưa chắc sai

- **Bitmap Heap Scan** / **Bitmap Index Scan** (PG): nhiều match, đọc heap theo block — ổn ở độ chọn lọc trung bình.
- **Index Scan** / Clustered Scan khi bạn **cần** phần lớn bảng (report, `WHERE 1=1`).
- **Nested Loops** khi phía ngoài vài chục row và inner là **Seek**.
- **Hash Join** in-memory trên hai tập lớn: thường *đúng*, đừng đổi thành Nested Loop.

### Xanh — plan tốt

- **Index Seek** / **Index Only Scan** / `type=ref` hoặc `const`
- Seek đúng prefix phễu, không Lookup (covering), không Sort
- `actual rows` ≈ `estimated rows` (lệch < ~10×)

### 30 giây — checklist

1. Plan **Actual**, không phải Estimated.
2. Node / dòng đắt nhất là gì? Scan bảng lớn? Lookup? Sort disk? Nested Loop nổ?
3. Map về 4 nguyên tắc: thiếu equality trái? Range đứng trước equality? `ORDER BY` lệch hướng? Chưa covering?
4. `estimated` lệch `actual` một hoặc hai bậc 10 → **thống kê / parameter sniffing** (Phần 5), chưa chắc thiếu index.
5. Sửa **một** thứ (index hoặc viết lại SARGable), chạy lại plan — đừng thêm 4 index một lúc.

```
Seq Scan / ALL / Clustered Scan (bảng lớn, filter hẹp)  →  index hoặc viết lại WHERE
Key Lookup × N lớn                                      →  covering
Sort / filesort / spill                                 →  index đúng ORDER BY
Nested Loop × inner Scan                                →  index cho join / FK
estimated ≪ actual                                      →  ANALYZE / UPDATE STATISTICS / sniffing
```

## Phần 4. Index hoạt động thế nào với từng thao tác SQL

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

- **PostgreSQL:** tăng `work_mem` cho *session/query nặng* (mặc định 4MB → 16–64MB). Áp dụng **per sort/hash operation**, không phải per server. Cảnh báo: `work_mem` × số operation × số connection có thể ăn hết RAM — đừng set 64MB global trên server 500 connection.
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
- GROUP BY + WHERE range: range phá vỡ phễu → không loop-and-count được. Giải pháp: biến range thành equality (xem Phần 6)
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

`UPDATE` cột nằm trong index = xóa entry cũ + chèn entry mới (page split, giết HOT — Phần 10). `UPDATE` 1 triệu row theo điều kiện hẹp: index cho **mệnh đề WHERE**, không phải cho cột đang gán.

### IN: Nhiều equality, không phải range

`IN ('a','b','c')` = vài nhánh **equality** (nguyên tắc 1), không phải range (nguyên tắc 4). Optimizer thường Bitmap/Index Union vài Seek.

```sql
-- ✅ Index (status) hoặc (shop_id, status)
WHERE shop_id = 42 AND status IN ('paid', 'refunded', 'cancelled');
```

- `NOT IN (...)` giống `!=`: hai phía + bẫy `NULL` (Phần 4 NULL). `NOT EXISTS` an toàn hơn (Phần 8).
- `IN` vài nghìn literal: parse/plan phình, cardinality đoán mò. Nhét staging table + `JOIN` (có index) thường ổn hơn.
- `IN (SELECT …)`: index bảng trong subquery (cột so sánh). Không phải lúc nào cũng kém JOIN — xem plan.

`= ANY(ARRAY[...])` (PostgreSQL) cùng họ với `IN`.

### HAVING: Không thay được WHERE

`WHERE` lọc **row** (trước GROUP BY) → index theo nguyên tắc 3–4. `HAVING` lọc **nhóm đã gộp** — không Seek vào từng row.

```sql
-- ❌ filter row nhét HAVING: GROUP BY xong mới bỏ, index WHERE không giúp
SELECT shop_id, COUNT(*)
FROM orders
GROUP BY shop_id
HAVING shop_id = 42;

-- ✅ WHERE trước, HAVING chỉ cho aggregate
SELECT shop_id, COUNT(*)
FROM orders
WHERE shop_id = 42
GROUP BY shop_id
HAVING COUNT(*) > 100;
```

Index cho câu dưới: `(shop_id)` đủ đếm; list nhóm “nặng” toàn bảng: `(shop_id)` vẫn scan index — đừng kỳ vọng Seek 1 shop nếu không có `WHERE`.

### UNION và UNION ALL: Mỗi nhánh một index

Mỗi `SELECT` trong `UNION` là query riêng — index **từng nhánh**, không có “một index cho cả UNION”.

```sql
SELECT id FROM users WHERE email = @e
UNION ALL
SELECT id FROM users WHERE phone = @p;
-- Index (email), index (phone) — xem Phần 6 (OR hai cột)
```

`UNION` (không `ALL`) = cộng rồi **loại trùng** (sort/hash) → đắt. Trùng không thể xảy ra thì `UNION ALL`. `OR` một câu vs `UNION ALL` hai Seek: khi hai cột khác index, `UNION ALL` thường thắng (Phần 6).

### Window function: Partition + Order cũng cần index

`ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC)` cần sort/partition giống `GROUP BY` + `ORDER BY`. Index `(customer_id, created_at DESC)` cho phép đọc đã thứ tự — tránh WindowAggregate + Sort.

```sql
-- Top 1 đơn mỗi khách: window hoặc LATERAL (Phần 8)
SELECT *
FROM (
  SELECT *, ROW_NUMBER() OVER (
    PARTITION BY customer_id ORDER BY created_at DESC
  ) AS rn
  FROM orders
) t
WHERE rn = 1;
```

Filter `WHERE created_at >= …` **trước** window thì đưa `created_at` vào index đúng chỗ (equality/range trái). `WHERE rn = 1` **không** đẩy xuống index — đó là filter sau cửa sổ. Top-N mỗi nhóm trên nhiều khách: `LATERAL` / `APPLY` + Seek từng nhóm đôi khi rẻ hơn window sort cả bảng (Phần 8).

## Phần 5. Tại sao Database không dùng Index của tôi

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

**PostgreSQL** prepared statement: generic plan vs custom plan. Query selectivity đổi mạnh → `SET plan_cache_mode = force_custom_plan` (PG 12+) hoặc đừng prepare. SQL Server 2022+ có **Parameter Sensitive Plan (PSP)** — optimizer có thể cache nhiều plan theo dải tần suất; vẫn nên `UPDATE STATISTICS` và cân `OPTION (RECOMPILE)` cho outlier.

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

### Seek rồi Filter: Index dùng mà vẫn chậm

Plan có **Seek** không có nghĩa là predicate đã được đẩy hết vào index. Optimizer Seek theo prefix khớp, rồi **Filter** phần còn lại trên từng row (residual predicate).

```sql
-- Index (shop_id, created_at)
WHERE shop_id = 42
  AND EXTRACT(YEAR FROM created_at) = 2024;  -- không SARGable
```

Seek `shop_id = 42` (đúng), rồi lọc từng đơn 2024 bằng hàm — shop lớn = đọc hàng trăm nghìn leaf. Plan: Index Seek + Filter. Viết lại range trên cột (mục trên) hoặc expression index (Phần 6).

Nhìn plan: **Filter** / `Recheck Cond` / predicate không nằm trong Seek keys. SQL Server: Seek keys vs Residual. PostgreSQL: `Index Cond` vs `Filter`.

### Key Lookup / heap fetch: Khi Seek thua Scan

Nonclustered Seek rồi **Key Lookup** (SQL Server) / heap fetch (PostgreSQL) cho mỗi row vì `SELECT *` hoặc cột ngoài index. Vài chục lookup ổn; vài trăm nghìn lookup = random I/O thảm — optimizer **đúng** khi chọn Clustered/Seq Scan.

```sql
-- Index (status) không covering
SELECT * FROM orders WHERE status = 'pending';
```

`pending` 2% bảng: Seek + lookup. `pending` 40%: scan. Sửa: covering / `INCLUDE` (Phần 6), hoặc bớt cột SELECT (Phần 10). Đừng `WITH (INDEX=…)` / `pg_hint_plan` ép Seek khi lookup đắt hơn scan.

### Partial / filtered index không khớp predicate

Index “một phần bảng” chỉ dùng khi query có **đúng** (hoặc chặt hơn) predicate lúc tạo:

```sql
CREATE INDEX orders_open_created ON orders (created_at)
WHERE status = 'open';   -- PostgreSQL partial / SQL Server filtered
```

| Query | Dùng được? |
| ----- | ---------- |
| `WHERE status = 'open' AND created_at > …` | Có |
| `WHERE created_at > …` (không ghi status) | Không |
| `WHERE status = @status` (SQL Server parameter) | **Thường không** — optimizer không dám giả định `@status` luôn `'open'` |

SQL Server: literal `'open'` hoặc `OPTION (RECOMPILE)` / dynamic SQL với giá trị cụ thể. PostgreSQL custom plan thường ổn hơn với parameter. Predicate lệch một chút (`status IN ('open','paused')`) cũng **không** dùng index `WHERE status = 'open'`.

### Thống kê lệch và cột tương quan

Optimizer nhân selectivity **độc lập**: `city = 'HCM' AND zip = '700000'` tưởng 1% × 1% = 0.01%, thực tế zip gần như suy ra city → ước lượng sai → chọn index/scan ngược.

Sau bulk load, histogram cũ = “index có mà không Seek” (Phần 10).

```sql
-- PostgreSQL 10+: extended statistics
CREATE STATISTICS orders_city_zip ON city, zip FROM orders;
ANALYZE orders;

-- SQL Server: stats nhiều cột (tạo index composite cũng tạo stats)
CREATE STATISTICS orders_city_zip ON orders (city, zip);
UPDATE STATISTICS orders WITH FULLSCAN;
```

`EXPLAIN` lệch `rows` ước lượng vs `actual` một–hai bậc 10 → thống kê / correlation / sniffing, chưa chắc thiếu index (Phần 3).

### View và hàm bọc cột: Index “có” mà optimizer mù

Index nằm trên **bảng**, không phải trên tên view. View `CREATE VIEW v AS SELECT *, LOWER(email) AS email_lc` rồi `WHERE email_lc = …` = hàm trên cột (mục Index không khớp). Scalar UDF trong `WHERE` (SQL Server) thường **chặn** Seek — inline / viết lại.

```sql
-- ❌ view giấu hàm
CREATE VIEW v_users AS
SELECT id, LOWER(email) AS email FROM users;
SELECT * FROM v_users WHERE email = 'a@b.com';  -- không dùng index (email)

-- ✅ predicate trên cột gốc, hoặc expression index (Phần 6)
SELECT * FROM users WHERE email = 'a@b.com';
-- hoặc WHERE LOWER(email) = ... + index (LOWER(email))
```

Indexed view / materialized view là chuyện khác (SQL Server indexed view, PostgreSQL `MATERIALIZED VIEW`) — phải `REFRESH` / maintenance, không phải SELECT view thường.

## Phần 6. Cạm bẫy và mẹo nâng cao về Indexing



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
-- PostgreSQL, SQL Server (không có trên MySQL)
CREATE INDEX invoices_covering
ON invoices (customer_id, year)
INCLUDE (price);

-- MySQL: covering = đưa cột SELECT vào key (InnoDB secondary đã chứa PK)
-- CREATE INDEX invoices_covering ON invoices (customer_id, year, price);
```

Dùng `INCLUDE` khi chỉ cần cột đó để tránh nhảy vào bảng (Index Only Scan / covering), không cần filter hay sort theo cột đó. Đặt cột filter/sort trong key; cột chỉ `SELECT` thì để `INCLUDE`. MySQL: nhét vào key, chấp nhận index to hơn.

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

-- PostgreSQL: expression index (SQL Server / MySQL: computed / generated column rồi UNIQUE)
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

Thêm predicate **thừa về mặt nghiệp vụ** (không đổi kết quả) để optimizer bám đúng index. Ghi chú cạnh query: nếu rule đổi, điều kiện “ma” sẽ lọc sai.

Ví dụ: mỗi user thuộc đúng một `org_id`. Index hay dùng là `(org_id, created_at)` vì hầu hết query đa tenant đều lọc org trước. Query “bài viết của tôi” chỉ cần `user_id` vẫn **đúng**, nhưng Seek vào `(org_id, created_at)` thì không.

```sql
-- Index: posts (org_id, created_at)  — user_id không đứng đầu phễu
-- Nghiệp vụ: user 17 luôn thuộc org 42

-- ❌ đúng kết quả, dễ Seq Scan / lookup theo user_id (nếu có)
SELECT * FROM posts
WHERE user_id = 17
ORDER BY created_at DESC
LIMIT 20;

-- ✅ cùng kết quả, Seek (org_id) rồi scan created_at
SELECT * FROM posts
WHERE org_id = 42          -- "ma": suy ra từ session, không đổi kết quả
  AND user_id = 17
ORDER BY created_at DESC
LIMIT 20;
-- Index tốt hơn: (org_id, user_id, created_at) hoặc (user_id, created_at)
```

Chỉ thêm “ma” khi bạn **chắc** invariant (tenant, `deleted_at IS NULL`, `status IN (...)` luôn đúng). Không dùng để “đánh lừa” optimizer khi dữ liệu không thật sự thoả.

### Tìm kiếm theo vị trí: Khi hai điều kiện phạm vi đụng nhau

Longitude + latitude = hai range → index B-tree chỉ dùng được một. Dùng Spatial Index (R-tree / GIST).

```sql
-- PostgreSQL (cần extension btree_gist nếu `type` không phải geometry)
CREATE EXTENSION IF NOT EXISTS btree_gist;
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

PostgreSQL GiST nhiều cột: scalar + geometry cần `btree_gist`. MySQL spatial index chỉ 1 cột. SQL Server: `geography` / `geometry` + spatial index riêng; filter `type` thường cần index B-tree riêng (spatial index không “gói” equality như GiST đa cột).

### Tìm kiếm ký tự đại diện ở đầu: Trường hợp đặc biệt

`LIKE 'abc%'` là range (nguyên tắc 4) → B-tree dùng được. `LIKE '%abc'` / `'%abc%'` **không** đi từ trái lá index → B-tree vô dụng.

```sql
-- PostgreSQL: trigram (substring, ILIKE)
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX users_name_trgm ON users USING GIN (name gin_trgm_ops);
SELECT * FROM users WHERE name ILIKE '%nguyen%';

-- PostgreSQL: full-text (từ, ngôn ngữ, ranking — không phải substring thuần)
CREATE INDEX articles_fts ON articles USING GIN (to_tsvector('simple', title || ' ' || body));
SELECT * FROM articles
WHERE to_tsvector('simple', title || ' ' || body) @@ plainto_tsquery('simple', 'indexing btree');

-- SQL Server: Full-Text (không phải B-tree)
-- CREATE FULLTEXT CATALOG ft AS DEFAULT;
-- CREATE FULLTEXT INDEX ON users(name) KEY INDEX pk_users;
SELECT * FROM users WHERE CONTAINS(name, '"nguyen"');

-- MySQL: FULLTEXT (InnoDB) — ngram parser cho CJK / substring ngắn
ALTER TABLE users ADD FULLTEXT INDEX users_name_ft (name);
SELECT * FROM users WHERE MATCH(name) AGAINST ('nguyen' IN BOOLEAN MODE);
```

`pg_trgm` / FULLTEXT **không** thay B-tree cho equality, FK, `ORDER BY` theo thời gian. Một cột search + một cột sort: thường **hai** index, hoặc GIN + sort ngoài — xem plan (Sort / Bitmap).

### OR trên hai cột: Một index không đủ phễu

`WHERE email = @e OR phone = @p` không đi một phễu `(email, phone)`. Optimizer hay Bitmap OR hai index, hoặc **bỏ index**. Tách thành hai Seek rồi `UNION ALL`:

```sql
-- ❌ một cây, hai nhánh OR
SELECT * FROM users WHERE email = @e OR phone = @p;

-- ✅ mỗi nhánh một index: (email), (phone)
SELECT * FROM users WHERE email = @e
UNION ALL
SELECT * FROM users WHERE phone = @p
  AND (email IS DISTINCT FROM @e);  -- tránh trùng nếu cả hai khớp
-- SQL Server: AND (email <> @e OR email IS NULL)
```

`OR` trên **cùng** cột (`status IN (...)` / `status = 'a' OR status = 'b'`) vẫn là equality — nguyên tắc 1, không phải pattern này.

### Index phình to: Bloat, fragmentation, REINDEX

Index không co lại khi `DELETE` / `UPDATE` cột key. Chỗ trống nằm rải — scan cùng số row nhưng **nhiều page hơn**.

```sql
-- PostgreSQL: bloat ước lượng (cần pgstattuple cho số chính xác)
SELECT relname, pg_size_pretty(pg_relation_size(indexrelid)) AS index_size, idx_scan
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- REINDEX không khóa đọc/ghi lâu (PG 12+)
REINDEX INDEX CONCURRENTLY users_email_idx;
-- VACUUM (FULL) không phải cách dọn index thường ngày

-- SQL Server
SELECT OBJECT_NAME(ips.object_id), i.name, ips.avg_fragmentation_in_percent, ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON i.object_id = ips.object_id AND i.index_id = ips.index_id
WHERE ips.page_count > 1000
ORDER BY ips.avg_fragmentation_in_percent DESC;

ALTER INDEX users_email_idx ON users REORGANIZE;           -- nhẹ
ALTER INDEX users_email_idx ON users REBUILD WITH (ONLINE = ON);  -- Enterprise / một số edition
```

Rebuild khi fragmentation / bloat **cao và index hay được scan**. Seek vào 1–2 leaf không đau vì index “xốp”. Fillfactor / `FILLFACTOR` thấp hơn 100 trên bảng update-nhiều giảm page split — đổi khi tạo index, không phải query.

## Phần 7. Kỹ thuật thao tác dữ liệu hiệu quả



### Tranh chấp khóa: Khi bộ đếm bị "nghẽn cổ chai"

Phân tán counter thành nhiều row (fanout) để update song song:

```sql
-- MySQL 8.0.20+: alias thay cho VALUES() (đã deprecated)
INSERT INTO post_statistics (post_id, fanout, likes_count)
VALUES (1475870220422107137, FLOOR(RAND() * 100), 1) AS new_row
ON DUPLICATE KEY UPDATE likes_count = likes_count + new_row.likes_count;

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

-- MariaDB: RETURNING (INSERT/DELETE/REPLACE). MySQL 8.x **không** có RETURNING.
```



### Xóa dòng trùng lặp: Dùng CTE thay vì xử lý ở tầng ứng dụng

```sql
-- PostgreSQL
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
WITH duplicates AS (
  SELECT id, ROW_NUMBER() OVER (
    PARTITION BY firstname, lastname, email
    ORDER BY age DESC
  ) AS rownum
  FROM contacts
)
DELETE c
FROM contacts c
INNER JOIN duplicates d ON c.id = d.id
WHERE d.rownum > 1;
```

### UPSERT: INSERT hoặc UPDATE một lần

Race “SELECT rồi INSERT/UPDATE” ở app = hai request cùng lúc, mất unique. Một statement, unique key làm trọng tài.

```sql
-- PostgreSQL
INSERT INTO settings (user_id, theme)
VALUES (42, 'dark')
ON CONFLICT (user_id)
DO UPDATE SET theme = EXCLUDED.theme, updated_at = now()
RETURNING *;

-- SQL Server 2008+: MERGE (khóa range nếu không có unique đúng cột)
MERGE settings WITH (HOLDLOCK) AS t
USING (SELECT 42 AS user_id, 'dark' AS theme) AS s
ON t.user_id = s.user_id
WHEN MATCHED THEN UPDATE SET theme = s.theme, updated_at = SYSUTCDATETIME()
WHEN NOT MATCHED THEN INSERT (user_id, theme) VALUES (s.user_id, s.theme);

-- SQL Server 2022+: có thể INSERT … ON. (tùy edition); pattern cũ hay dùng:
-- UPDATE … IF @@ROWCOUNT = 0 INSERT …  trong một transaction + unique index

-- MySQL 8.0.20+: alias, đừng dùng VALUES() deprecated
INSERT INTO settings (user_id, theme)
VALUES (42, 'dark') AS new_row
ON DUPLICATE KEY UPDATE theme = new_row.theme;
```

Cần **UNIQUE** (hoặc PK) trên khóa xung đột. `MERGE` SQL Server dễ deadlock / race nếu thiếu unique và `HOLDLOCK`. PostgreSQL `ON CONFLICT ON CONSTRAINT …` rõ hơn “cột nào thắng”.

### Xóa / sửa theo lô: Đừng nuốt cả bảng trong một transaction

`DELETE FROM logs WHERE created_at < …` mười triệu row = WAL/log khổng lồ, lock lâu, replica tụt, `VACUUM` / index phình (Phần 6). Cắt lô nhỏ, commit giữa chừng:

```sql
-- PostgreSQL: lặp đến khi 0 row
DELETE FROM logs
WHERE ctid IN (
  SELECT ctid FROM logs
  WHERE created_at < DATE '2024-01-01'
  LIMIT 5000
);

-- SQL Server
DELETE TOP (5000)
FROM logs
WHERE created_at < '20240101';

-- MySQL
DELETE FROM logs
WHERE created_at < '2024-01-01'
LIMIT 5000;
```

App loop + sleep ngắn nếu replication lag. Xóa theo partition (`DROP` / `TRUNCATE` partition) rẻ hơn DELETE khi dữ liệu theo tháng — Phần 9.

`UPDATE` hàng loạt cùng bài: đừng set 5 triệu row một phát (lock + WAL + index maintain). Cột nằm trong index: mỗi lô còn đắt hơn (Phần 10, HOT).

### Nạp dữ liệu hàng loạt: COPY, BULK INSERT, SqlBulkCopy

`INSERT` từng row từ app = parse + plan + WAL + index cho mỗi câu. Nạp file / stream:

```sql
-- PostgreSQL (file trên server; từ client: \copy trong psql, hoặc Npgsql COPY)
COPY staging_users (email, name)
FROM '/tmp/users.csv' WITH (FORMAT csv, HEADER true);
-- UNLOGGED TABLE / disable index rồi rebuild: chỉ staging, không phải bảng live có replica

-- SQL Server
BULK INSERT staging_users
FROM 'C:\tmp\users.csv'
WITH (FIRSTROW = 2, FIELDTERMINATOR = ',', ROWTERMINATOR = '\n', TABLOCK);

-- MySQL
LOAD DATA LOCAL INFILE '/tmp/users.csv'
INTO TABLE staging_users
FIELDS TERMINATED BY ',' IGNORE 1 LINES (email, name);
```

App: PostgreSQL `COPY FROM STDIN`; SQL Server `SqlBulkCopy` + `TABLOCK` khi được. Load xong `ANALYZE` / `UPDATE STATISTICS` — không thì Phần 5 (plan sai vì histogram cũ).

## Phần 8. Viết query như chuyên gia



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

`FOR UPDATE` / `UPDLOCK` giữ khóa đến hết transaction. Thiếu index trên `account_id` → khóa **scan**, không phải một row. Timeout / deadlock: khóa cùng thứ tự bảng (users rồi orders, không ngược).

### SKIP LOCKED: Hàng đợi không tranh chấp

Nhiều worker `SELECT … FOR UPDATE` cùng hàng đợi → xếp hàng, hoặc deadlock. Bỏ qua row đang khóa, lấy việc còn trống:

```sql
-- PostgreSQL 9.5+ / MySQL 8.0.1+
BEGIN;
SELECT id, payload
FROM job_queue
WHERE status = 'pending'
ORDER BY id
LIMIT 1
FOR UPDATE SKIP LOCKED;
-- xử lý rồi UPDATE status = 'done'
COMMIT;

-- SQL Server (READPAST: bỏ row bị khóa; UPDLOCK: mình khóa row lấy được)
BEGIN TRAN;
SELECT TOP (1) id, payload
FROM job_queue WITH (UPDLOCK, READPAST, ROWLOCK)
WHERE status = 'pending'
ORDER BY id;
COMMIT;
```

Index `(status, id)` (hoặc filtered `WHERE status = 'pending'`). `SKIP LOCKED` **không** phải isolation “không bao giờ miss”: row đang khóa sẽ do worker khác làm. Không dùng cho “đọc đủ mọi row đúng một lần trong cùng statement” — dùng cho queue.

### EXISTS và NOT EXISTS: Semi-join / anti-join

`EXISTS` chỉ cần **biết có** row khớp, không kéo cả tập con. `IN (subquery)` dễ gãy với `NULL` (`NOT IN` + NULL = không ra row). Anti-join chuẩn: `NOT EXISTS`.

```sql
-- Khách có ít nhất một đơn 2024 — semi-join, index orders (customer_id, created_at)
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o
  WHERE o.customer_id = c.id
    AND o.created_at >= TIMESTAMP '2024-01-01'
    AND o.created_at <  TIMESTAMP '2025-01-01'
);

-- Khách không có đơn — đừng NOT IN (SELECT customer_id …) khi cột nullable
SELECT c.id
FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

`COUNT(*) > 0` trong subquery = làm hết việc rồi mới so sánh. `EXISTS` dừng ở row đầu. Plan: Nested Loop Semi Join / Anti Join + Seek, không Hash cả bảng orders nếu selectivity tốt.

### LATERAL / CROSS APPLY: Top-N mỗi nhóm

“3 đơn mới nhất mỗi khách” — `ROW_NUMBER()` rồi filter `rn <= 3` đúng nhưng sort **cả** orders. `LATERAL` / `APPLY` = với mỗi khách, Seek 3 row trên `(customer_id, created_at DESC)`.

```sql
-- PostgreSQL
SELECT c.id, o.id AS order_id, o.created_at
FROM customers c
CROSS JOIN LATERAL (
  SELECT id, created_at
  FROM orders
  WHERE customer_id = c.id
  ORDER BY created_at DESC
  LIMIT 3
) o;

-- SQL Server
SELECT c.id, o.id AS order_id, o.created_at
FROM customers c
CROSS APPLY (
  SELECT TOP (3) id, created_at
  FROM orders
  WHERE customer_id = c.id
  ORDER BY created_at DESC
) o;
```

Chỉ rẻ khi **số nhóm nhỏ** (hoặc đã filter khách) và index `(customer_id, created_at DESC)` — covering nếu SELECT đủ cột. Hàng triệu khách × LATERAL = Nested Loop nổ; lúc đó window + filter có thể thắng. Đọc plan (Phần 3).

### Biểu thức bảng tạm (CTE): Xử lý query phức tạp

Chia query thành bước nhỏ, mỗi CTE test độc lập — dễ debug hơn subquery lồng.

CTE **không** phải bảng tạm trên đĩa. PostgreSQL (đến 11 inlined; 12+ có thể materialize): CTE bị đọc nhiều lần hoặc tối ưu kém hơn subquery. Ép: `AS MATERIALIZED` / `NOT MATERIALIZED` (PG 12+). SQL Server: CTE thường inlined; spool khi recursive hoặc optimizer chọn spool.

```sql
-- Recursive: cây category (cần index parent_id)
WITH RECURSIVE tree AS (
  SELECT id, parent_id, name, 0 AS depth
  FROM categories
  WHERE id = 10
  UNION ALL
  SELECT c.id, c.parent_id, c.name, tree.depth + 1
  FROM categories c
  JOIN tree ON c.parent_id = tree.id
  WHERE tree.depth < 20          -- chặn chu trình / sâu vô hạn
)
SELECT * FROM tree;

-- SQL Server: WITH tree AS (anchor UNION ALL recursive) — không ghi RECURSIVE
```

Cây sâu / graph: materialized path (Phần 9) thường rẻ hơn recursive trên mọi request.

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
WHERE created_at >= TIMESTAMP '2024-01-01'
  AND created_at <  TIMESTAMP '2025-01-01'
ORDER BY customer_id ASC, price DESC;
-- (tránh EXTRACT(YEAR FROM created_at) / YEAR(created_at) — hàm trên cột = không SARGable)

-- SQL Server / MySQL: cùng ý tưởng bằng ROW_NUMBER()
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY price DESC) AS rn
  FROM orders
  WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
) t WHERE rn = 1;
```

### N+1: Vòng lặp query vs một câu SQL

ORM/`foreach` khách rồi `SELECT * FROM orders WHERE customer_id = @id` = N+1 round-trip. Index `(customer_id)` **không** cứu latency mạng.

```sql
-- 1 round-trip: IN / JOIN / LATERAL (Phần 8 trên)
SELECT * FROM orders
WHERE customer_id = ANY(@ids);          -- PostgreSQL
-- SQL Server: WHERE customer_id IN (SELECT id FROM @ids)

-- Chi tiết sau khi có id: hai bước (covering hẹp rồi hydrate)
SELECT id FROM orders WHERE customer_id = @c AND created_at >= @from ORDER BY created_at DESC LIMIT 50;
SELECT * FROM orders WHERE id = ANY(@page_ids);
```

Bước 1: index hẹp `(customer_id, created_at DESC) INCLUDE (id)` — Index Only. Bước 2: PK Seek 50 row. `SELECT *` một phát trên 50k đơn = lookup nổ (Phần 5).

### Optional filter: `OR col IS NULL` / `@p IS NULL OR col = @p`

Form search “mọi field tùy chọn”:

```sql
-- ❌ phá Seek: optimizer không đẩy @status vào index (status)
WHERE (@status IS NULL OR status = @status)
  AND (@from  IS NULL OR created_at >= @from);
```

SQL Server hay scan; PostgreSQL generic plan cũng dễ bỏ index. Cách làm:

1. **Dynamic SQL** / query builder: chỉ `AND` khi user chọn filter — mỗi combo một plan, SARGable.
2. SQL Server: `OPTION (RECOMPILE)` cho báo cáo thưa (compile mỗi lần).
3. Hai query: “có filter” vs “không filter”, không nhét một câu catch-all.

`COALESCE(@status, status) = status` cùng họ — hàm/`OR` trên cột, không Seek.

### LEFT JOIN … IS NULL: Anti-join dễ viết sai

`NOT EXISTS` (trên) là anti-join chuẩn. Người ta hay viết:

```sql
SELECT c.id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;
```

Đúng nghĩa “không có đơn” **chỉ khi** `ON` đủ khóa, `WHERE` lọc phía phải `IS NULL`. Lỡ `AND o.created_at >= @d` viết vào `WHERE` thay vì `ON` → mọi khách không đơn 2024 bị loại luôn (inner join ngầm). `COUNT(o.id) = 0` sau `GROUP BY` cũng anti-join — thường nặng hơn `NOT EXISTS`. Plan: Anti Join + Seek `(customer_id)` trên orders; nếu Hash cả bảng, thiếu index FK.

### COUNT lớn: Đừng đếm cả bảng mỗi request

`SELECT COUNT(*) FROM orders` = scan (index-only nếu có index hẹp, vẫn O(n)). Dashboard mỗi request = đắt.

- Ước lượng: PostgreSQL `reltuples` (`pg_class`); SQL Server `sys.dm_db_partition_stats` / `sp_spaceused`.
- Đếm theo điều kiện hẹp: index khớp phễu, covering không cần `*`.
- Số liệu “đủ gần”: counter table (Phần 7 fan-out) hoặc materialized (Phần 9).
- `COUNT(col)` bỏ NULL — không nhanh hơn `COUNT(*)` nếu không covering cột đó.

UI “1.2 triệu đơn” không cần đúng từng row: làm tròn + cache.

### Khoảng thời gian nửa-mở

`BETWEEN @from AND @to` **bao gồm** hai đầu. `DATETIME`/`timestamptz` cuối ngày `23:59:59` bỏ sót hoặc trùng trang.

```sql
-- ✅ [from, to)
WHERE created_at >= TIMESTAMPTZ '2024-01-01 00:00+07'
  AND created_at <  TIMESTAMPTZ '2024-02-01 00:00+07'
```

Keyset pagination cùng cột thời gian: cursor `created_at, id`, so sánh bậc thang (đầu Phần 8) — `BETWEEN` + `OFFSET` vừa trùng vừa nhảy trang. Không `DATE(created_at)` / `CAST(created_at AS date)` (Phần 5). Timezone: Phần 9.

## Phần 9. Thiết kế Schema: Nền móng vững chắc



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

### Soft delete: `deleted_at` và index

`deleted_at IS NULL` = “còn sống” xuất hiện gần như mọi query. Index thường `(user_id, created_at)` **không** biết soft-delete → Seek xong vẫn lọc xác, hoặc bitmap cả đống row đã xóa.

```sql
-- PostgreSQL: partial — chỉ lá “còn sống”
CREATE INDEX orders_user_live
ON orders (user_id, created_at DESC)
WHERE deleted_at IS NULL;

SELECT * FROM orders
WHERE user_id = 42
  AND deleted_at IS NULL          -- phải có đúng predicate này
ORDER BY created_at DESC
LIMIT 20;

-- SQL Server: filtered index
CREATE INDEX orders_user_live
ON orders (user_id, created_at DESC)
WHERE deleted_at IS NULL;
```

Query **phải** ghi `deleted_at IS NULL` (hoặc predicate chặt hơn) thì optimizer mới dùng. Unique “một email active”: `UNIQUE (email) WHERE deleted_at IS NULL` — cùng email xóa mềm được insert lại. Đừng biến `deleted_at` thành cột hay sửa trên index “nóng” nếu bảng update dày (HOT / page split — Phần 10, Bonus).

### Khóa clustered hẹp: Đừng nhét UUID vào mọi secondary

SQL Server và InnoDB: mọi nonclustered index **mang theo clustered key** (thường là PK) ở leaf. PK `UNIQUEIDENTIFIER` / `CHAR(36)` 16–36 byte → nhân với số index → leaf phình, cache miss.

| PK clustered | Secondary `(email)` leaf chứa |
| ------------ | ----------------------------- |
| `INT` / `BIGINT` identity | email + 4–8 byte |
| UUIDv4 | email + 16 byte (+ fragmentation, Phần 1) |

Công thức: **PK nội bộ hẹp, tăng dần**; UUID/ULID là cột unique riêng cho API (đầu Phần 9). PostgreSQL heap: PK không “dính” mọi secondary như clustered — vẫn nên BIGINT/`IDENTITY` cho join; UUID random chỉ làm **index UUID** phân mảnh, không xé cả bảng.

### Multi-tenant: `tenant_id` đứng đầu schema

SaaS: gần như mọi câu đều `WHERE tenant_id = @t`. Cột đó phải **equality trái** (nguyên tắc 3), trên PK/index và thường trên FK.

```sql
CREATE TABLE orders (
  tenant_id   INT NOT NULL,
  id          BIGINT GENERATED ALWAYS AS IDENTITY,
  customer_id BIGINT NOT NULL,
  PRIMARY KEY (tenant_id, id)
);

CREATE INDEX orders_customer
ON orders (tenant_id, customer_id, created_at DESC);
```

Thiếu `tenant_id` trên index = Seek theo `customer_id` xuyên tenant, hoặc RLS/filter thêm sau. PostgreSQL: `ENABLE ROW LEVEL SECURITY` + policy `tenant_id = current_setting(...)` — **vẫn** cần index `(tenant_id, …)`; RLS không thay Seek. SQL Server: tương tự predicate trong view/proc, không phải “máy tự hiểu tenant”.

### Status: ENUM, lookup, hay VARCHAR

- **VARCHAR + `CHECK`**: đổi giá trị = migration data, không lock catalog kiểu enum. Index B-tree bình thường. Selectivity thấp → partial/filtered (Phần 6).
- **Lookup table** (`status_id` FK): thêm trạng thái không `ALTER TYPE`. Index FK (Phần 10). JOIN nhỏ, covering được.
- **ENUM (PostgreSQL)**: compact, nhưng `ALTER TYPE … ADD VALUE` hạn chế trong transaction; xóa/đổi label đau. MySQL enum = string ngầm, reorder giá trị dễ vỡ sort.

Không tạo index **chỉ** trên `status` nếu `open` chiếm 40% bảng. Kết hợp `(tenant_id, status, created_at)` hoặc `WHERE status = 'open'`.

### Bảng nối nhiều-nhiều: PK kép thay `id` thừa

```sql
-- ❌ id vô nghĩa + unique muộn: hai row (1,2) lọt nếu quên unique
CREATE TABLE user_roles (
  id SERIAL PRIMARY KEY,
  user_id INT NOT NULL,
  role_id INT NOT NULL
);

-- ✅ PK = chính cặp; Seek theo user
CREATE TABLE user_roles (
  user_id INT NOT NULL,
  role_id INT NOT NULL,
  PRIMARY KEY (user_id, role_id),
  FOREIGN KEY (user_id) REFERENCES users (id),
  FOREIGN KEY (role_id) REFERENCES roles (id)
);
CREATE INDEX user_roles_role ON user_roles (role_id, user_id);  -- chiều ngược + covering
```

`id` trên bảng nối chỉ cần khi chính row đó bị FK từ chỗ khác. Index một chiều không đủ cho “mọi user của role X”.

### Thời gian: `timestamptz`, không `TIMESTAMP without time zone`

`TIMESTAMP` / `DATETIME` không timezone = “3 giờ” không biết 3 giờ ở đâu. App UTC, DB local, DST → lệch index range (`created_at >= …`).

| Hệ | Kiểu nên dùng cho “mốc thời điểm” |
| -- | --------------------------------- |
| PostgreSQL | `TIMESTAMPTZ` (`timestamp with time zone`) — lưu UTC, hiện theo session |
| SQL Server | `DATETIME2` + quy ước UTC, hoặc `DATETIMEOFFSET` nếu cần offset gốc |
| MySQL | `DATETIME` (naive) hoặc `TIMESTAMP` (convert theo `time_zone`) — **đừng trộn** |

Range luôn nửa-mở, không hàm trên cột (Phần 4):

```sql
WHERE created_at >= TIMESTAMPTZ '2024-01-01 00:00+07'
  AND created_at <  TIMESTAMPTZ '2025-01-01 00:00+07'
```

Cột “ngày sinh / ngày hóa đơn theo lịch” (không phải mốc tuyệt đối) mới dùng `DATE` / naive datetime.

## Phần 10. Checklist & lỗ hổng hay quên

Những điểm dưới đây hay bị bỏ khi tối ưu index — không thay thế các phần trên, chỉ là lưới an toàn trước khi merge.

### Luôn index khóa ngoại

`JOIN` / `ON DELETE` / `ON UPDATE` đi theo FK. Không index cột FK → Nested Loop / Key Lookup hàng loạt, hoặc lock scan cả bảng khi xóa parent.

```sql
-- orders.customer_id REFERENCES customers(id)
CREATE INDEX orders_customer_id ON orders (customer_id);
```

Composite hay dùng: `(customer_id, created_at DESC)` nếu list đơn theo khách + thời gian.

### Đừng `SELECT *` nếu muốn covering

Covering / Index Only Scan chỉ xảy ra khi **mọi cột** trong query nằm trong index (key + `INCLUDE` / PK InnoDB). `SELECT *` phá covering ngay khi bảng có cột “mỡ”. Lấy đúng cột cần.

### HOT update (PostgreSQL) và “cột không nằm trong index”

Phần 1 nói: update cột không thuộc index thì index không đổi. Trên PostgreSQL, nếu **không** đụng cột indexed và còn chỗ trên page, update có thể là **HOT** (Heap-Only Tuple) — không đụng secondary index. Index “thừa” trên cột hay bị sửa (status, updated_at, counter) giết HOT → write chậm hơn bạn nghĩ.

### BRIN / columnstore cho dữ liệu append-only

B-tree không phải câu trả lời duy nhất:

| Tình huống | Công cụ |
| ---------- | ------- |
| Time-series / log, insert cuối bảng, query theo khoảng thời gian | PostgreSQL **BRIN** trên `created_at` (rẻ, nhỏ) |
| Analytics, scan vài cột trên bảng rất rộng | SQL Server **columnstore** (clustered/nonclustered) |
| Equality thuần, không range | PostgreSQL `USING HASH` (PG 10+ đã WAL-safe) — vẫn không thay B-tree cho `ORDER BY` / range |

### Thống kê, histogram, và “index có mà không Seek”

Optimizer **đoán** cardinality. Thống kê cũ sau bulk load = plan sai (Phần 5). SQL Server: xem histogram `DBCC SHOW_STATISTICS`. PostgreSQL: `pg_stats`, `ANALYZE`. MySQL: histogram 8.0 (`ANALYZE TABLE ... UPDATE HISTOGRAM`).

### Invisible / disable trước khi DROP

Nhắc lại: MySQL `INVISIBLE`, SQL Server `DISABLE`, PostgreSQL không có — test staging. `idx_scan = 0` sau restart không có nghĩa là index vô dụng (PG stat reset; SQL Server `dm_db_index_usage_stats` mất khi instance restart nếu không persist).

### Trước khi `CREATE INDEX`

1. Query chậm thật sự đọc bao nhiêu row? (`EXPLAIN ANALYZE` / Actual Plan)
2. Cột equality trái → range → sort/`INCLUDE` (bốn nguyên tắc)
3. Đã có prefix trùng chưa? (index thừa)
4. Online / `CONCURRENTLY` trên production
5. FK đã có index chưa?
6. Write path: bảng này insert/update thế nào? thêm index có giết HOT / page split PK random không?

Xong checklist này rồi hãy deploy index.

## Bonus. Hiểu sâu B+ Tree trong RDBMS

Phần 1 đủ để thiết kế index. Phần này giải thích *cơ chế* — đọc khi cần hiểu page split, random UUID, covering, và vì sao range chỉ đi một hướng (nguyên tắc 2).

Engine B-tree của PostgreSQL / SQL Server / InnoDB đều là **B+ Tree** (hoặc biến thể): dữ liệu hàng nằm ở **lá**, tầng trên chỉ là chìa khóa dẫn đường. (PostgreSQL gọi access method `btree`; bên trong vẫn là B+.)

### B-tree vs B+ Tree: data chỉ nằm ở lá

B-tree cổ điển có thể cất record ở **mọi** tầng. B+ Tree:

- **Internal (nhánh):** separator keys + pointer xuống con. Không đủ cột để trả query.
- **Leaf (lá):** mọi key, theo thứ tự, trỏ ra row (hoặc chứa luôn clustered row).

```
          [  50  |  120  ]                    ← internal (chỉ “ngã rẽ”)
          /      |       \
    [..50)   [50..120)   [120..]              ← lá: key đã sort + con trỏ row
     ←prev───────────────next→                ← lá móc đôi
```

Hệ quả: Seek = vài lần đọc nhánh (thường trong buffer) rồi **một** lần đọc lá. Range = Seek lá trái + đi `next` — không nhảy lung tung trên cây. Đó là nguyên tắc 2.

### Page, fanout, chiều cao cây

Index không lưu từng byte một file phẳng. Đơn vị I/O là **page** (PostgreSQL mặc định 8 KB, SQL Server 8 KB, InnoDB 16 KB thường gặp).

Một page nhánh chứa hàng trăm separator → **fanout** (số con mỗi node) ~ 100–500 tùy độ rộng key. Chiều cao:

| Số row (lá ~300 key/page) | Chiều cao (xấp xỉ) |
| ------------------------- | ------------------ |
| 300                       | 1 (gần như chỉ lá) |
| 90.000                    | 2                  |
| 27 triệu                  | 3                  |
| 8 tỷ                      | 4                  |

Tỷ row vẫn 3–4 hop. Seek đắt vì **lá + heap/clustered lookup**, không vì “cây quá cao”. Key càng rộng (UUID, `VARCHAR(500)`) fanout càng nhỏ → cao hơn một chút, cache kém hơn. **Thu hẹp key** (Phần 9) chính là tăng fanout.

Root + tầng trên gần như luôn trong shared buffers / buffer pool. Random insert đau ở **lá** (và data page), không ở việc “đi hết chiều cao”.

### Lá nối nhau: Vì sao range chỉ đi một hướng

Lá là danh sách móc **hai chiều** (`prev` / `next`). `BETWEEN` / `LIKE 'abc%'` / `ORDER BY key`:

1. Xuống cây tới lá chứa cận trái  
2. Đọc tuần tự lá → lá kế, **một hướng**

Không có “đi tới vừa lùi” trong một lần duyệt. `ORDER BY score DESC, created_at ASC` trên một index `(score, created_at)` cùng chiều → nguyên tắc 2 gãy. `!=` nằm hai phía key → không phải một range (Phần 4).

InnoDB / SQL Server clustered: **bảng** cũng là B+ Tree — range trên PK là scan lá clustered, sequential theo key, không theo thời điểm insert (trừ khi PK tăng dần).

### Page split: Random insert đắt hơn sequential

Lá đầy mà phải chèn key nằm giữa → **split**: một page thành hai, ~một nửa key sang page mới, sửa pointer tầng trên (đôi khi split lan lên root).

```
Lá đầy:  [10|20|30|40|50|60|70|80]
Insert 45 → split
         [10|20|30|40|45]  [50|60|70|80]
                ↑ page mới, I/O + fragment
```

- **Sequential** (`IDENTITY`, UUIDv7): insert cuối lá phải, split ít, page mới append — cache thân thiện.  
- **Random** (UUIDv4, hash): split giữa, page nửa đầy rải disk → fragmentation (Phần 6 REINDEX / `REBUILD`).

SQL Server: `page_split` trong `sys.dm_db_index_operational_stats`. PostgreSQL: ít counter “split” lộ; nhìn bloat / `pg_stat_all_indexes`. Fillfactor (dưới) giảm split cho update-at-the-key.

### Leaf chứa gì: TID (heap) vs clustered key

Sau khi Seek tới lá, engine phải ra **row**:

| | PostgreSQL (heap mặc định) | SQL Server clustered / InnoDB |
| --- | --- | --- |
| Secondary leaf | key + **TID** (`ctid`: file/block/offset) | key + **clustered key** (thường PK) |
| Bước tiếp | Heap fetch theo TID (có thể random I/O) | Seek thêm trên cây clustered bằng PK |

Covering / Index Only Scan: đủ cột trên lá → **bỏ** heap fetch. Visibility map (PG) / không lookup (SQL Server) mới “only”. `SELECT *` phá (Phần 10).

Heap: update không đổi cột index → TID có thể giữ (HOT). Clustered: đổi PK = di chuyển row + sửa **mọi** secondary (vì chúng lưu PK). Đổi PK production = thảm họa có chủ đích.

Nonclustered SQL Server / InnoDB secondary: leaf **không** có toàn bộ cột bảng — trừ `INCLUDE` / cột nằm trong clustered key. Đó là lý do clustered key hẹp (Phần 9).

### Fillfactor: Chỗ trống cố ý

Tạo index có thể để **trống cố ý** trên lá, dành cho insert/update sau này, giảm split.

```sql
-- PostgreSQL: 100 = đầy lá (tốt cho append-only / read-mostly)
CREATE INDEX orders_created_at ON orders (created_at)
WITH (fillfactor = 90);

-- SQL Server
CREATE INDEX orders_created_at ON orders (created_at)
WITH (FILLFACTOR = 90);
```

- **100 / 100:** bảng log, PK tăng, ít update key → mật độ tối đa, scan ít page.  
- **70–90:** update tại chỗ / insert “gần random” trên secondary.  
- PostgreSQL `fillfactor` cũng ảnh hưởng **heap** (`ALTER TABLE … SET (fillfactor = 80)`) — chỗ cho HOT. SQL Server fillfactor chủ yếu **index**; heap khác (forwarding records).

Không phải núm vặn query. Sai fillfactor = lãng phí RAM hoặc split như random UUID. Đo fragmentation / bloat rồi mới hạ.

### Khi nào không dùng B+ Tree

B+ Tree thắng khi có **thứ tự**: equality, range, `ORDER BY`, prefix `LIKE`. Không phải mọi index đều B+ Tree:

| Nhu cầu | Cấu trúc | Hệ |
| ------- | -------- | --- |
| Equality thuần, không range / sort | Hash | PostgreSQL `USING HASH`; SQL Server hash chỉ memory-optimized |
| Full-text, JSON containment, array | GIN | PostgreSQL |
| Hình học, range overlap, exclusion | GiST / SP-GiST | PostgreSQL (Phần 6, 9) |
| Time-series append, query theo khoảng thô | BRIN | PostgreSQL (Phần 10) |
| Scan vài cột, bảng rất rộng | Columnstore | SQL Server (Phần 10) |
| `LIKE '%abc%'` | Trigram GIN / FULLTEXT | Phần 6 |

Default `CREATE INDEX` = B+ Tree. Đừng đổi sang hash/GIN vì “nghe nhanh hơn” khi query vẫn là `WHERE ts BETWEEN … ORDER BY ts`.

---

Bốn nguyên tắc (Phần 2) là hệ quả của B+ Tree: Seek xuống lá, scan một hướng, phễu trái→phải vì key ghép sort như từ điển, range cắt phễu vì lá không “nhảy cóc” cột sau. Hiểu page và leaf xong, quay lại thiết kế index — đừng tối ưu fanout trước khi query đúng nguyên tắc.
