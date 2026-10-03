# Database Indexing

## Mục lục

- [Phần 1. Nền tảng: Hiểu Index từ gốc rễ](#phần-1-nền-tảng-hiểu-index-từ-gốc-rễ)
  - [Các thuật ngữ cần phân biệt](#các-thuật-ngữ-cần-phân-biệt)
  - [B+ Tree](#b-tree-không-cần-hiểu-chi-tiết-chỉ-cần-hiểu-ý-tưởng)
  - [Có sắp xếp hay không](#có-sắp-xếp-hay-không-chuyện-của-primary-key)
  - [Heap vs Clustered](#hai-cách-lưu-trữ-khác-nhau-heap-table-và-clustered-index)
  - [Có index chưa chắc nhanh](#có-index-chưa-chắc-query-nhanh-hiểu-lầm-phổ-biến-nhất)
- [Phần 2. Bốn nguyên tắc vàng khi sử dụng Index](#phần-2-bốn-nguyên-tắc-vàng-khi-sử-dụng-index)
  - [1. Tra cứu nhanh](#nguyên-tắc-1-tra-cứu-nhanh--nhảy-thẳng-đến-vị-trí-cần-tìm)
  - [2. Quét một hướng](#nguyên-tắc-2-quét-theo-một-hướng)
  - [3. Phễu trái → phải](#nguyên-tắc-3-từ-trái-sang-phải--nguyên-tắc-phễu-cho-index-nhiều-cột)
  - [4. Range phá phễu](#nguyên-tắc-4-quét-khi-gặp-điều-kiện-phạm-vi--điều-kiện-phạm-vi-phá-vỡ-phễu)
- [Phần 3. Đọc execution plan trong 30 giây](#phần-3-đọc-execution-plan-trong-30-giây)
  - [Đỏ — cần điều tra bằng số liệu](#đỏ--cần-điều-tra-bằng-số-liệu)
  - [Vàng](#vàng--chưa-chắc-sai)
  - [Xanh — kết quả đã được kiểm chứng](#xanh--kết-quả-đã-được-kiểm-chứng)
  - [Checklist 30s](#30-giây--checklist)
- [Phần 4. Index hoạt động thế nào với từng thao tác SQL](#phần-4-index-hoạt-động-thế-nào-với-từng-thao-tác-sql)
  - [Phép so sánh không bằng (!=)](#phép-so-sánh-không-bằng--xem-phân-bố-trước-khi-kết-luận)
  - [NULL](#null-giá-trị-đặc-biệt-cần-đặc-biệt-chú-ý)
  - [LIKE](#like-prefix-collation-và-ký-tự-đại-diện-ở-đầu)
  - [ORDER BY](#order-by-tận-dụng-thứ-tự-index-khi-có-lợi)
  - [GROUP BY & DISTINCT](#group-by--distinct-thách-thức-lớn-nhất)
  - [JOIN](#join-phân-tách-và-kết-hợp)
  - [Subquery](#subquery-không-chậm-như-bạn-nghĩ)
  - [UPDATE & DELETE](#update--delete-đừng-quên-tối-ưu-cho-chúng)
  - [IN](#in-nhiều-equality-không-phải-range)
  - [HAVING](#having-không-thay-được-where)
  - [UNION / UNION ALL](#union-và-union-all-mỗi-nhánh-một-index)
  - [Window function](#window-function-partition--order-cũng-cần-index)
- [Phần 5. Tại sao Database không dùng Index của tôi](#phần-5-tại-sao-database-không-dùng-index-của-tôi)
  - [Quy trình thực thi](#quy-trình-thực-thi-query-bên-trong-bộ-não-của-database)
  - [Index không khớp](#index-không-khớp-với-query-lý-do-phổ-biến-nhất)
  - [Full scan nhanh hơn](#full-table-scan-nhanh-hơn-khi-database-đúng-mà-bạn-sai)
  - [Chọn index khác](#database-chọn-index-khác-khi-có-nhiều-lựa-chọn)
  - [Parameter sniffing](#parameter-sniffing-plan-đúng-với-lần-chạy-đầu-sai-với-lần-sau)
  - [Tạo và bảo trì index](#tạo-và-bảo-trì-index-giảm-thời-gian-chặn-ghi)
  - [Seek rồi Filter](#seek-rồi-filter-index-dùng-mà-vẫn-chậm)
  - [Key Lookup](#key-lookup--heap-fetch-khi-seek-thua-scan)
  - [Partial / filtered index](#partial--filtered-index-không-khớp-predicate)
  - [Thống kê lệch](#thống-kê-lệch-và-cột-tương-quan)
  - [View bọc cột](#view-và-hàm-bọc-cột-index-có-mà-optimizer-mù)
- [Phần 6. Cạm bẫy và mẹo nâng cao về Indexing](#phần-6-cạm-bẫy-và-mẹo-nâng-cao-về-indexing)
  - [Index trên biểu thức](#index-trên-biểu-thức-khi-không-thể-viết-lại-query)
  - [Cột giá trị ít](#cột-giá-trị-ít-đo-selectivity-của-giá-trị-đang-tìm)
  - [Biến range thành =](#biến-đổi-điều-kiện-phạm-vi-biến-range-thành-so-sánh-bằng)
  - [Implicit conversion](#kiểu-dữ-liệu-không-khớp-implicit-conversion)
  - [Covering / index-only](#truy-vấn-chỉ-từ-index-không-cần-chạm-vào-bảng-dữ-liệu)
  - [Lọc và sắp xếp khi JOIN](#lọc-và-sắp-xếp-khi-join-tối-ưu-trước-khi-denormalize)
  - [Giới hạn kích thước](#vượt-giới-hạn-kích-thước-index)
  - [JSON](#json-đánh-index-trong-thế-giới-phi-cấu-trúc)
  - [Ràng buộc duy nhất và giá trị NULL](#ràng-buộc-duy-nhất-và-giá-trị-null-khác-nhau-giữa-các-hệ)
  - [Index không dùng](#tìm-và-dọn-dẹp-index-không-sử-dụng)
  - [Điều kiện “ma”](#điều-kiện-ma-giúp-database-mà-không-thay-đổi-kết-quả)
  - [Tìm theo vị trí](#tìm-kiếm-theo-vị-trí-khi-hai-điều-kiện-phạm-vi-đụng-nhau)
  - [Wildcard ở đầu](#tìm-kiếm-ký-tự-đại-diện-ở-đầu-trường-hợp-đặc-biệt)
  - [OR trên hai cột](#or-trên-hai-cột-một-index-không-đủ-phễu)
  - [Index phình to](#index-phình-to-bloat-fragmentation-reindex)
- [Phần 7. Kỹ thuật thao tác dữ liệu hiệu quả](#phần-7-kỹ-thuật-thao-tác-dữ-liệu-hiệu-quả)
  - [Tranh chấp khóa](#tranh-chấp-khóa-khi-bộ-đếm-bị-nghẽn-cổ-chai)
  - [UPDATE … JOIN](#cập-nhật-dữ-liệu-từ-bảng-khác-join-trong-update)
  - [RETURNING / OUTPUT](#lấy-dữ-liệu-ngay-sau-khi-thay-đổi-returning--output)
  - [Xóa dòng trùng](#xóa-dòng-trùng-lặp-dùng-cte-thay-vì-xử-lý-ở-tầng-ứng-dụng)
  - [UPSERT](#upsert-giữ-đúng-dữ-liệu-khi-có-concurrency)
  - [Xóa / sửa theo lô](#xóa--sửa-theo-lô-đừng-nuốt-cả-bảng-trong-một-transaction)
  - [Nạp hàng loạt](#nạp-dữ-liệu-hàng-loạt-copy-bulk-insert-sqlbulkcopy)
- [Phần 8. Viết query như chuyên gia](#phần-8-viết-query-như-chuyên-gia)
  - [Phân trang theo khóa](#phân-trang-đúng-cách-phân-trang-theo-khóa)
  - [FOR UPDATE / UPDLOCK](#for-update--updlock-khóa-dòng-ở-tầng-database)
  - [SKIP LOCKED](#skip-locked-claim-công-việc-trong-transaction-ngắn)
  - [EXISTS / NOT EXISTS](#exists-và-not-exists-semi-join--anti-join)
  - [LATERAL / APPLY](#lateral--cross-apply-top-n-mỗi-nhóm)
  - [CTE](#biểu-thức-bảng-tạm-cte-xử-lý-query-phức-tạp)
  - [Bảng tạm và bảng biến](#bảng-tạm-temp-và-bảng-biến-table-sql-server)
  - [Tips khác](#các-tips-query-hữu-ích-khác)
  - [N+1](#n1-vòng-lặp-query-vs-một-câu-sql)
  - [Optional filter](#optional-filter-or-col-is-null--p-is-null-or-col--p)
  - [LEFT JOIN IS NULL](#left-join--is-null-anti-join-dễ-viết-sai)
  - [COUNT lớn](#count-lớn-đừng-đếm-cả-bảng-mỗi-request)
  - [Khoảng nửa-mở](#khoảng-thời-gian-nửa-mở)
- [Phần 9. Thiết kế Schema: Nền móng vững chắc](#phần-9-thiết-kế-schema-nền-móng-vững-chắc)
  - [UUID vs Auto-increment](#uuid-vs-auto-increment-lựa-chọn-primary-key)
  - [JSON column](#json-column-khi-nosql-gặp-sql)
  - [Constraint](#constraint-hàng-rào-bảo-vệ-cuối-cùng)
  - [Exclusion](#ràng-buộc-loại-trừ-chống-chồng-chéo-postgresql)
  - [Materialized path](#đường-dẫn-vật-lý-hóa-lưu-trữ-cây-đơn-giản)
  - [Partition](#partition-retention-và-pruning-có-điều-kiện)
  - [Bảng sắp sẵn](#bảng-sắp-xếp-trước-tối-ưu-cho-quét-phạm-vi)
  - [Tính toán trước](#tính-toán-trước-khi-index-cũng-không-đủ-nhanh)
  - [Soft delete](#soft-delete-deleted_at-và-index)
  - [Clustered key hẹp](#khóa-clustered-hẹp-đừng-nhét-uuid-vào-mọi-secondary)
  - [Multi-tenant](#multi-tenant-tenant_id-đứng-đầu-schema)
  - [Status / ENUM](#status-enum-lookup-hay-varchar)
  - [Bảng nối N–N](#bảng-nối-nhiều-nhiều-pk-kép-thay-id-thừa)
  - [Thời gian](#thời-gian-chọn-kiểu-theo-mốc-thời-điểm-và-lịch-địa-phương)
- [Phần 10. Checklist & lỗ hổng hay quên](#phần-10-checklist--lỗ-hổng-hay-quên)
  - [Kiểm tra index phía tham chiếu của khóa ngoại](#kiểm-tra-index-phía-tham-chiếu-của-khóa-ngoại)
  - [Đừng SELECT *](#đừng-select--nếu-muốn-covering)
  - [HOT update](#hot-update-postgresql-và-cột-không-nằm-trong-index)
  - [BRIN / columnstore](#brin--columnstore-chọn-theo-workload)
  - [Thống kê](#thống-kê-histogram-và-index-có-mà-không-seek)
  - [Invisible / disable](#invisible--disable-trước-khi-drop)
  - [Trước khi CREATE INDEX](#trước-khi-create-index)
- [Phần 11. Quy trình tối ưu và thực hành](#phần-11-quy-trình-tối-ưu-và-thực-hành)
  - [Đo trước và sau theo workload](#đo-trước-và-sau-theo-workload)
  - [Bài thực hành PostgreSQL](#bài-thực-hành-postgresql-equality-range-order-và-covering)
  - [Các thông tin cần giữ khi bàn giao một index](#các-thông-tin-cần-giữ-khi-bàn-giao-một-index)
- [Bonus. Hiểu sâu B+ Tree trong RDBMS](#bonus-hiểu-sâu-b-tree-trong-rdbms)
  - [B-tree vs B+ Tree](#b-tree-vs-b-tree-data-chỉ-nằm-ở-lá)
  - [Page, fanout, chiều cao](#page-fanout-chiều-cao-cây)
  - [Lá nối nhau](#lá-nối-nhau-vì-sao-range-chỉ-đi-một-hướng)
  - [Page split](#page-split-random-insert-đắt-hơn-sequential)
  - [Leaf chứa gì](#leaf-chứa-gì-tid-heap-vs-clustered-key)
  - [Fillfactor](#fillfactor-chỗ-trống-cố-ý)
  - [Khi nào không dùng B+ Tree](#khi-nào-không-dùng-b-tree)

> **Phạm vi:** Tài liệu tập trung vào B-tree/B+ tree cho bảng rowstore của **PostgreSQL**, **SQL Server** và **MySQL/InnoDB**. GIN, GiST, BRIN, spatial và columnstore có quy tắc riêng. Ví dụ dành cho các hệ khác được ghi rõ khi cần.
>
> **Cách đọc ví dụ:** Mỗi đoạn SQL là một ví dụ độc lập, giả định bảng/cột đã tồn tại trừ đoạn có `CREATE TABLE`. Các khối ghi nhiều hệ là những phương án thay thế, không chạy liên tiếp. `LIMIT`, boolean `TRUE/FALSE`, `TIMESTAMPTZ` và `EXTRACT` chủ yếu theo PostgreSQL; SQL Server dùng `TOP`/`OFFSET FETCH`, `BIT` và kiểu thời gian tương ứng. Tham số `@p` là placeholder của ứng dụng; cú pháp native PostgreSQL là `$1`, MySQL là `?`.
>
> **Giới hạn của quy tắc:** “Equality trước, range sau” là điểm xuất phát, không phải bảo đảm plan tối ưu. Kết luận phải dựa trên workload, phân bố dữ liệu và execution plan. Các số về row, độ sâu cây và thời gian chỉ để minh họa, không phải benchmark.
>
> **Mốc đối chiếu:** 03/10/2026; tham chiếu PostgreSQL 18, SQL Server 2022/2025 và MySQL 8.4. Tính năng mới cần đúng phiên bản, edition và compatibility level. Nguồn chính thức được liên kết ngay tại các mục liên quan.


## Phần 1. Nền tảng: Hiểu Index từ gốc rễ

### Các thuật ngữ cần phân biệt

| Thuật ngữ | Nghĩa trong tài liệu |
| --- | --- |
| Key column | Cột quyết định thứ tự và khả năng định vị trên index |
| INCLUDE / payload | Dữ liệu đi kèm ở leaf, giúp tránh lookup nhưng không cung cấp seek/order |
| Selectivity | Tỷ lệ row match predicate; tỷ lệ thấp hơn thường có nghĩa predicate chọn lọc hơn |
| Cardinality | Số row ở một bước plan; “distinct cardinality” là số giá trị khác nhau của cột |
| Seek / range scan | Tìm vị trí bắt đầu rồi đọc khoảng liên quan; vẫn có thể đọc nhiều row |
| Residual predicate | Điều kiện còn phải kiểm tra sau khi đã định vị bằng key |
| Covering | Access path có dữ liệu cần trả/lọc; PostgreSQL còn phụ thuộc MVCC visibility để tránh heap fetch |
| Logical / physical read | Truy cập page qua cache / phải đọc từ storage; logical reads cao vẫn tốn CPU dù cache hit |

Index tốt giảm **tổng công việc** của workload: định vị, đọc entry, lấy row, join/sort và duy trì khi ghi. Nó không chỉ đổi tên operator trong plan.


### B+ Tree: Không cần hiểu chi tiết, chỉ cần hiểu ý tưởng

Để thiết kế index, trước hết cần hiểu thứ tự của key, phạm vi phải đọc và chi phí lấy dữ liệu từ bảng. Chi tiết insert, rebalance và page split sẽ giúp giải thích các vấn đề sâu hơn; bạn có thể đọc phần Bonus sau khi nắm mô hình cơ bản.

Thay vào đó, hãy nhớ hai điều cốt lõi về B+ tree:

#### Index là một danh sách đã được sắp xếp (sorted list) + bảng tóm tắt phân cấp

Hãy tưởng tượng bạn có một cuốn từ điển dày 2000 trang. Bạn muốn tìm từ "performance". Bạn sẽ:

- Nhìn vào phần gáy sách → thấy P nằm khoảng trang 1200
- Lật đến trang 1200, nhìn header → thấy "PER" bắt đầu từ trang 1245
- Lật đến 1245, scan vài trang → tìm thấy "performance"

Bạn chỉ cần 3 bước thay vì đọc 2000 trang. Index trong database hoạt động y hệt:

- Leaf nodes (tầng dưới cùng) = các trang từ điển, chứa danh sách giá trị đã sorted
- Internal nodes (các tầng trên) = phần gáy/header sách, chứa "tóm tắt phạm vi" để nhảy nhanh
- Cây thường khá thấp vì một page nhánh chứa nhiều key. Độ sâu cụ thể phụ thuộc kích thước key, page, mật độ và số entry; không suy ra một con số cố định chỉ từ số row.

Từ đây bạn có thể hình dung đơn giản: index = sorted list + bảng tóm tắt giúp nhảy nhanh. Không cần phức tạp hơn thế.

Muốn hiểu *vì sao* random UUID làm page split, leaf chứa TID hay clustered key, fanout ~ vài trăm, đọc [Bonus — B+ Tree trong RDBMS](#bonus-hiểu-sâu-b-tree-trong-rdbms) sau khi xong bốn nguyên tắc. Bốn nguyên tắc không phụ thuộc chi tiết đó.

#### Database tự duy trì index, ứng dụng vẫn phải chọn index phù hợp

Database duy trì tính đúng đắn của index khi `INSERT`, `UPDATE`, `DELETE`. Chi phí phụ thuộc loại index và engine:

- `INSERT`: thêm entry vào index áp dụng cho row; partial/filtered index chỉ chứa row thỏa predicate.
- `DELETE`: row/entry thường được đánh dấu để dọn sau theo cơ chế MVCC hoặc ghost cleanup, không nhất thiết biến mất vật lý ngay.
- `UPDATE`: thay đổi key, cột `INCLUDE`, biểu thức hoặc predicate của index có thể cần duy trì index. Đổi clustered key còn ảnh hưởng locator trong nonclustered index.
- PostgreSQL tạo phiên bản tuple mới. Ngay cả khi sửa cột không được index, vẫn có thể cần entry mới ở các index nếu không đủ điều kiện HOT. Xem [HOT update](#hot-update-postgresql-và-cột-không-nằm-trong-index).

Nhiều index làm tăng dung lượng, WAL/transaction log, CPU và chi phí ghi. Không có tỷ lệ read/write hay số index “chuẩn” cho mọi bảng. Đánh giá từng index bằng query nó phục vụ, mức cải thiện đọc và tác động đến ghi; index dùng cho constraint hoặc tác vụ hiếm vẫn có thể cần giữ.

### Có sắp xếp hay không: Chuyện của Primary Key

Key tăng dần thường giúp insert tập trung vào một số page cuối, còn key ngẫu nhiên phân tán truy cập và có thể gây nhiều page split. Tuy nhiên, insert đồng thời vào cuối cây cũng có thể gặp **last-page latch contention**; key tăng dần không luôn thắng trong mọi workload.

| Loại key | Đặc điểm | Điểm cần cân nhắc |
| --- | --- | --- |
| Identity / auto-increment | Hẹp, thường tăng dần | Có gap, thứ tự cấp ID không bảo đảm thứ tự commit; có thể tranh chấp page cuối |
| UUIDv4 | Ngẫu nhiên, 16 byte ở kiểu native/binary | Insert phân tán, index rộng hơn BIGINT |
| UUIDv7 / ULID | Có thành phần thời gian | Gần thứ tự thời gian khi cách lưu và comparator phù hợp; clock và concurrency ảnh hưởng thứ tự |
| Snowflake ID | Thường là số 64 bit có timestamp + worker + sequence | Cần phối hợp worker ID và xử lý clock rollback |

Ảnh hưởng của random key lớn hơn khi key đó cũng là **clustered key** của SQL Server hoặc PK của InnoDB. PostgreSQL lưu heap riêng, nên random PK không trực tiếp quyết định vị trí heap; index PK vẫn chịu chi phí insert phân tán. Không gán hệ số chậm hơn 3–10 lần khi chưa có benchmark cùng schema, dữ liệu và mức concurrency.

**SQL Server:** UUIDv7 ghi vào `UNIQUEIDENTIFIER` không mặc nhiên được sắp theo timestamp RFC, vì comparator của kiểu này không theo thứ tự byte thông thường. Nếu cần insert tuần tự, kiểm chứng cách sinh/lưu GUID hoặc dùng `BIGINT IDENTITY` làm clustered key và UUID làm unique key riêng. [Quy tắc so sánh uniqueidentifier](https://learn.microsoft.com/en-us/sql/t-sql/data-types/uniqueidentifier-transact-sql?view=sql-server-ver17).

### Hai cách lưu trữ khác nhau: Heap Table và Clustered Index

Đây là kiến thức nền quan trọng mà nhiều developer bỏ qua, nhưng nó ảnh hưởng trực tiếp đến cách bạn thiết kế schema và chọn primary key. Có hai cách database lưu trữ dữ liệu trên disk:

#### Heap Table (PostgreSQL mặc định)

Heap không duy trì thứ tự theo PK. Engine có thể tái sử dụng page còn chỗ trống, nên không bảo đảm row luôn được append theo thứ tự insert. B-tree index của PostgreSQL (kể cả PK) lưu TID gồm số block và vị trí tuple trong block. `ctid` là địa chỉ của phiên bản tuple hiện tại; nó có thể đổi khi UPDATE hoặc khi bảng được viết lại, nên không dùng làm ID lâu dài.

```
┌─────────────────────────┐      ┌─────────────────────────┐
│   INDEX (email)         │      │     TABLE (heap)        │
│                         │      │                         │
│ alice@... → row ở page 5│──────│ Page 5: [alice, 28, ...]│
│ bob@...   → row ở page 2│──────│ Page 2: [bob, 35, ...]  │
│ charlie@..→ row ở page 7│──────│ Page 7: [charlie, 22,..]│
└─────────────────────────┘      └─────────────────────────┘
```

Primary key được bảo vệ bằng unique index và yêu cầu các cột khóa `NOT NULL`.

Lookup qua B-tree thông thường gồm tìm entry rồi fetch heap để lấy dữ liệu và kiểm tra visibility. Index-only scan có thể bỏ heap fetch khi đủ cột và page all-visible. Đây là mô hình B-tree; GIN/BRIN có cấu trúc và cách truy cập khác.

#### Clustered Index (SQL Server và MySQL/InnoDB mặc định)

Leaf của clustered rowstore index chứa dữ liệu hàng, theo **thứ tự logic của clustered key**; page không nhất thiết liền nhau trên đĩa. Với InnoDB, clustered key là PK nếu có. SQL Server cho phép PK nonclustered và clustered index trên cột khác, hoặc một heap không có clustered index. Sơ đồ dưới giả định PK chính là clustered key; cột lớn/off-row có thể cần thêm I/O.

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

Với InnoDB, secondary index mang theo PK. Với bảng SQL Server có clustered index, nonclustered index dùng clustered key làm row locator; bảng SQL Server dạng heap dùng RID. Sơ đồ dưới minh họa trường hợp PK là clustered key:

```
┌──────────────────────────┐      ┌──────────────────────────┐
│ SECONDARY INDEX (email)  │      │ PRIMARY KEY INDEX = TABLE│
│                          │      │                          │
│ alice@... → PK=1         │──┐   │ PK=1 → [alice, ...]      │
│ bob@...   → PK=2         │──┤   │ PK=2 → [bob, ...]        │
│ charlie@..→ PK=3         │──┘   │ PK=3 → [charlie, ...]    │
└──────────────────────────┘      └──────────────────────────┘
```

Nếu cần cột ngoài index, secondary/nonclustered lookup tìm locator rồi lấy row từ clustered index (SQL Server gọi `Key Lookup`). Nếu đã covering, bước lookup này có thể được bỏ qua.

So sánh hai cách tiếp cận:


|                                 | Heap Table (PostgreSQL mặc định) | Clustered Index (SQL Server, MySQL/InnoDB)                        |
| ------------------------------- | -------------------------------- | ----------------------------------------------------------------- |
| PK lookup                       | 2 bước: PK index → heap table    | 1 bước: clustered / PK index = table                              |
| Secondary / nonclustered lookup | 2 bước: index → heap (`ctid`)    | 2 bước: secondary → PK (SQL Server gọi **Key Lookup**)            |
| Insert random PK                | Heap không sorted theo PK; PK index vẫn chịu random insert | Ảnh hưởng cả clustered leaf nếu PK là clustered key |
| Độ rộng key ảnh hưởng? | Cột PK chiếm chỗ trong heap/PK index; không tự trở thành locator của index khác | Clustered key/PK còn là locator ở secondary index |


SQL Server thường tạo PK clustered nếu chưa có clustered index và không chỉ định `NONCLUSTERED`; đây là mặc định DDL, không phải quy tắc bắt buộc. InnoDB chọn PK, nếu thiếu thì chọn unique index đầu tiên có mọi cột `NOT NULL`, cuối cùng mới tạo hidden row ID. PostgreSQL `CLUSTER` sắp xếp heap một lần và không tự giữ thứ tự sau đó. Không hệ nào bảo đảm thứ tự kết quả nếu thiếu `ORDER BY`.

#### Hệ quả thực tế cho Clustered Index (SQL Server, MySQL/InnoDB)


##### Hệ quả 1: Cân nhắc kỹ UUIDv4 làm clustered key

UUIDv4 là lựa chọn hợp lệ khi cần sinh ID phân tán. Nếu dùng làm clustered key, cần đo page split, dung lượng và độ trễ ghi. SQL Server có thể dùng PK UUID nonclustered với clustered key hẹp riêng; InnoDB có thể dùng BIGINT PK và unique UUID. Cách này thêm một index và chi phí ghi, nên cũng cần cân nhắc.

##### Hệ quả 2: Độ rộng clustered key ảnh hưởng các secondary index

Clustered key/PK được mang theo làm locator của secondary index. Với SQL Server, clustered key nonunique có thể thêm uniquifier. Ví dụ dưới chỉ ước lượng payload của locator, chưa gồm overhead, compression hay key đã có sẵn trong index:

```
Bảng: 1 triệu row, 5 secondary index
PK = BIGINT (8 bytes):       5 index × 1M × 8 bytes  =  40 MB overhead
PK = ULID string (26 bytes): 5 index × 1M × 26 bytes = 130 MB overhead
Chênh lệch: 90 MB — chỉ cho MỘT bảng!
Với 50 bảng tương tự: chênh lệch 4.5 GB
→ Có thể là khác biệt giữa "fit trong RAM" và "phải đọc disk"
```

Ưu tiên kiểu native/binary và key đủ hẹp; chọn BIGINT hay UUID theo yêu cầu sinh ID và workload.

- **PostgreSQL:** `BIGINT GENERATED ALWAYS AS IDENTITY`, hoặc UUIDv7 (`uuidv7()`, PostgreSQL 18+, 16 bytes). `gen_random_uuid()` là v4 — random, làm page split. Không lưu `CHAR(36)`.
- **SQL Server:** `BIGINT IDENTITY` hoặc `UNIQUEIDENTIFIER` + `NEWSEQUENTIALID()` trong DEFAULT. GUID này không phải RFC UUIDv7, không bảo đảm tăng toàn cục qua restart/failover, và có thể dễ suy đoán. UUIDv7 từ app cần lưu ý comparator ở mục trên.
- **MySQL:** `BIGINT AUTO_INCREMENT` hoặc ULID/UUIDv7 dạng `BINARY(16)` (sinh ở app). `UUID()` là v1; `UUID_TO_BIN(UUID(), 1)` chỉ đảo bit cho gần tuần tự hơn, không phải v7. Không dùng `CHAR(36)`.


##### Hệ quả 3: PK lookup cực nhanh — tận dụng cho CRUD apps

Nếu lookup dùng clustered key, leaf đã chứa dữ liệu hàng nên thường tránh được một bước lookup. Hiệu quả còn phụ thuộc cột lấy ra, cache, locking và số row; đặc điểm này không đủ để kết luận engine nào phù hợp nhất với CRUD.

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
-- → SQL Server có thể tránh Key Lookup; PostgreSQL còn phụ thuộc visibility map.
```

Quy tắc thực hành: Sau khi tạo index, luôn xem execution plan trước khi deploy.

- **PostgreSQL:** `EXPLAIN (ANALYZE, BUFFERS)` — xem `Index Scan` / `Index Only Scan` / `Seq Scan`, và `rows` ước lượng so với thực tế.
- **SQL Server:** lấy Actual Execution Plan trong SSMS (Ctrl+M) và `SET STATISTICS IO, TIME ON` để đo reads/CPU. Lệnh STATISTICS không tự cung cấp plan. Xem rows read/returned, số lần Lookup và spill; tên Seek hay Scan chưa nói lên plan tốt hay xấu.
- **MySQL:** `EXPLAIN ANALYZE` (8.0.18+) hoặc `EXPLAIN`. Cột `type`: `ALL` = full table scan, `index` = full index scan, `range` = range scan, `ref` = index lookup, `const` = đúng 1 row qua unique index.

Với truy vấn lọc hẹp, số row đọc lớn hơn nhiều số row trả về là dấu hiệu cần kiểm tra residual filter hoặc thứ tự key. Với aggregate, trả một row nhưng đọc nhiều row có thể là công việc bắt buộc.

## Phần 2. Bốn nguyên tắc vàng khi sử dụng Index

Bốn nguyên tắc sau giúp hình dung cách B-tree truy cập dữ liệu. Chúng là nền tảng để đề xuất index, rồi cần kiểm chứng bằng plan, thống kê và workload thực tế. Join, aggregate, MVCC và các loại index khác còn có những điều kiện riêng.

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


Bảng trên minh họa cây có fanout lớn, không phải cam kết về độ sâu. Chi phí tìm vị trí gần O(log n), nhưng lấy k kết quả còn cần đọc entry và có thể fetch k row. Dung lượng index vẫn quan trọng vì ảnh hưởng cache, I/O và write amplification.

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

Không có access path phù hợp, query có thể phải đọc nhiều row, lọc rồi dùng top-N sort. Với index trên age và không có filter bổ sung, engine có thể tìm cận rồi dừng sau ba kết quả nhìn thấy được. Các row bị loại bởi residual filter hoặc MVCC vẫn làm tăng số entry phải đọc.

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

-- ✅ Dùng prefix hai cột, phù hợp query không cần filter firstname
WHERE country = 'VN' AND lastname = 'Nguyen';

-- ✅ Dùng 1/3 bước phễu
WHERE country = 'VN';

-- ⚠ Không có prefix equality; thường không Seek thành một khoảng hẹp
WHERE lastname = 'Nguyen';
-- Vì: lastname 'Nguyen' nằm rải rác ở JP, US, VN... không nằm liền nhau

-- ⚠ Có thể scan index hoặc dùng tối ưu đặc biệt, nhưng không có prefix hẹp
WHERE firstname = 'Huy';
```

Quy tắc thiết kế thông thường: **ràng buộc các cột đầu giúp thu hẹp khoảng đọc**. Thiếu prefix không có nghĩa index hoàn toàn vô dụng: engine vẫn có thể scan index, lọc trên index, hoặc skip scan nếu hỗ trợ và có lợi.

Hiểu nhầm phổ biến: "Đặt cột selective nhất lên đầu"

Bạn sẽ thấy nhiều người khuyên rằng: "đặt cột có nhiều distinct values nhất (selective nhất) lên đầu index". Hãy xem tại sao nó chưa chắc đã luôn đúng:

Giả sử bạn đổi thứ tự index thành `(lastname, firstname, country)` vì lastname có nhiều giá trị nhất. Query `WHERE country = 'VN' AND lastname = 'Nguyen' AND firstname = 'Huy'` vẫn dùng được toàn bộ phễu: vì dùng đủ 3 cột. Số bước phễu giống nhau, selectivity ở đây không tạo ra khác biệt.

Nhưng query `WHERE country = 'VN'` không có prefix hẹp trên index này. Index đứng đầu bằng country thường hỗ trợ query đó tốt hơn; scan/skip scan vẫn là khả năng optimizer có thể cân nhắc.

Cách đúng: Thứ tự cột nên được quyết định bởi tập hợp các query mà app bạn chạy, nhằm tối đa hóa số query được phục vụ bởi cùng một index:

```sql
-- App chạy các query này:
-- Q1: WHERE country = 'VN'                                          (rất thường xuyên)
-- Q2: WHERE country = 'VN' AND lastname = 'Nguyen'                  (thường xuyên)
-- Q3: WHERE country = 'VN' AND lastname = 'Nguyen' AND firstname = 'Huy' (ít hơn)

-- Index (country, lastname, firstname) → phục vụ CẢ 3 query
-- Index (lastname, firstname, country) → Q3 dùng đủ key; Q1 thiếu prefix
-- Index (firstname, lastname, country) → Q3 cũng dùng đủ key; Q1/Q2 thiếu prefix
```


#### Bỏ qua cột giữa: Vẫn dùng được, nhưng kém hiệu quả

```sql
-- Index: (firstname, lastname, country)
-- Query: WHERE firstname = 'Huy' AND country = 'VN'
-- (bỏ qua lastname ở giữa)
```

Database có thể dùng prefix firstname. Ví dụ sau mô tả một lượt scan thông thường không có skip scan:

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

Index này vẫn có thể giới hạn phạm vi vào firstname='Huy' và filter country trước khi lấy row từ bảng. Có lợi bao nhiêu tùy phân bố và tổng cost; không bảo đảm luôn tốt hơn scan bảng.

#### Index trùng lặp: Dọn dẹp index thừa

Mỗi index phải được cập nhật khi write, nên index thừa = tốn tài nguyên vô ích. Quy tắc:

- Index `(country, lastname, firstname)` có thể phục vụ các access pattern prefix của:
  - `(country)`
  - `(country, lastname)`
  - → Đây là ứng viên gộp, cần đo trước khi xóa.
- Các index sau có key/order khác, cần đánh giá riêng:
  - `(country, lastname, telephone)` → cột cuối khác
  - `(lastname, country)` → thứ tự cột khác
  - → Không kết luận thừa chỉ từ prefix chung.

Index ngắn hơn có thể rẻ hơn cho scan/count vì ít page. Trước khi gộp, so sánh UNIQUE/constraint, predicate, ASC/DESC, collation/operator class, INCLUDE, partition và dung lượng; cùng prefix không bảo đảm cùng ngữ nghĩa.

Ví dụ đánh giá dọn dẹp:

```sql
-- Bảng users hiện có 4 index:
-- idx_1: (tenant_id)
-- idx_2: (tenant_id, email)
-- idx_3: (tenant_id, created_at)
-- idx_4: (email)

-- Phân tích:
-- idx_1 có prefix trùng idx_2 → ứng viên bỏ sau khi kiểm chứng
-- idx_2 và idx_3 có prefix giống nhưng cột 2 khác → GIỮ cả hai
-- idx_4 phục vụ query WHERE email = '...' (không có tenant_id) → GIỮ
-- Chỉ bỏ idx_1 nếu không phục vụ constraint/workload riêng và reads không tăng bất lợi.
```


### Nguyên tắc 4: Quét khi gặp điều kiện phạm vi — Điều kiện phạm vi "phá vỡ" phễu

Trong một lượt scan B-tree thông thường, equality trên prefix và range ở cột kế tiếp xác định khoảng đọc chủ yếu. Cột sau range vẫn có thể được kiểm tra trong index để giảm heap/Key Lookup, nhưng thường không tạo một khoảng hẹp liên tục như equality prefix.

Ví dụ `WHERE country = 'VN' AND age > 28 AND married = 'yes'`:

| Index | Khoảng đọc thông thường | Điều kiện còn lại |
| --- | --- | --- |
| `(country, age, married)` | Các entry country=VN, age>28 | Lọc married trong khoảng đó |
| `(country, married, age)` | Entry country=VN, married=yes, age>28 | Không còn filter này |

Với hai range `age > 25 AND salary > 20000000`, so sánh index `(country, age, salary)` và `(country, salary, age)` bằng **phân bố trong country đang query**, không chỉ selectivity toàn bảng. Còn phải xét ORDER BY, LIMIT và query khác dùng chung index.

#### Ngoại lệ: Skip Scan, nhiều range và index combination

- PostgreSQL 18 hỗ trợ B-tree skip scan: có thể chạy nhiều lần tìm với các giá trị prefix suy ra, hữu ích khi số giá trị prefix ít. Trước 18, index thiếu cột đầu vẫn có thể được scan; không đồng nghĩa “không dùng được”.
- MySQL có Skip Scan cho một số query với điều kiện cụ thể. Loose Index Scan cho GROUP BY là tối ưu khác; không dùng hai tên thay thế nhau.
- Oracle có Index Skip Scan. SQL Server không có access path skip scan tổng quát như Oracle; có thể scan hoặc kết hợp index.
- Bitmap AND/OR, Index Merge/Intersection hoặc nhiều seek cũng có thể tận dụng nhiều điều kiện. Không giả định mọi query chỉ có một lần seek và một range.

Vì vậy, “range cắt phễu” là mô hình cơ bản, không phải tuyên bố mọi cột phía sau chỉ có thể filter. [PostgreSQL: multicolumn B-tree và skip scan](https://www.postgresql.org/docs/18/indexes-multicolumn.html), [MySQL: range optimization](https://dev.mysql.com/doc/refman/8.4/en/range-optimization.html).

#### Công thức ban đầu để thử

`IS NULL` có thể đóng vai trò equality; `IN (...)` thường tạo nhiều giá trị equality; prefix LIKE có thể tạo range khi operator/collation phù hợp. Có nhiều giá trị IN ở cột đầu thì thứ tự toàn cục của cột sau chưa chắc đáp ứng ORDER BY.

```text
INDEX (equality_prefix, range_hoặc_ORDER_BY, ...)
INCLUDE (payload cần để covering)
```

Đây là đề xuất để đo. Nếu range ở cột A nhưng ORDER BY ở cột B, phải cân nhắc **lọc trước rồi sort** hoặc **đọc theo B rồi filter A và dừng theo LIMIT**. Không có một công thức key duy nhất thắng cả hai. Cột INCLUDE có thể hỗ trợ residual filter, nhưng không định vị seek hay cung cấp thứ tự.

Sau cùng kiểm tra rows read, lookup, sort và chi phí ghi; xem [Phần 3](#phần-3-đọc-execution-plan-trong-30-giây).

## Phần 3. Đọc execution plan trong 30 giây

Plan giải thích engine đọc bao nhiêu dữ liệu, qua những bước nào và ước lượng có sát thực tế không. **Tên operator không đủ để đánh giá tốc độ.**

```sql
-- PostgreSQL: chạy thật, có thời gian và buffer counters
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT id FROM orders WHERE customer_id = 42;

-- SQL Server: bật Actual Execution Plan trong SSMS (Ctrl+M), rồi chạy
SET STATISTICS IO, TIME ON;
SELECT id FROM orders WHERE customer_id = 42;
SET STATISTICS IO, TIME OFF;

-- MySQL 8.0.18+: chạy thật, trả TREE với actual rows/time/loops
EXPLAIN ANALYZE SELECT id FROM orders WHERE customer_id = 42;
-- EXPLAIN truyền thống có type/Extra/rows; các số rows là ước lượng.
```

Actual plan / EXPLAIN ANALYZE **thực thi câu lệnh**. Với DML, dùng dữ liệu thử hoặc môi trường kiểm chứng thích hợp; ROLLBACK không hoàn tác mọi side effect như sequence hoặc tác động bên ngoài của trigger. EXPLAIN không ANALYZE phù hợp khi chỉ cần xem ước lượng.

| Hệ | Nhìn gì trước |
| --- | --- |
| SQL Server | Actual Rows / Rows Read, Number of Executions, logical reads, CPU, spill và warnings. % Cost vẫn là **ước lượng**, kể cả trong Actual Plan |
| PostgreSQL | actual rows, loops, Rows Removed by Filter, Buffers, Heap Fetches. rows/time của node lặp là trung bình mỗi lượt; nhân loops để hiểu tổng công việc |
| MySQL | actual rows × loops và time trong TREE; dùng EXPLAIN truyền thống để xem type/Extra/key |

Không cộng thời gian/cost/buffers của mọi node thành tổng: node cha có thể bao gồm công việc của con; parallelism cũng làm elapsed time khác tổng CPU. [PostgreSQL: Using EXPLAIN](https://www.postgresql.org/docs/18/using-explain.html), [SQL Server: Actual Execution Plan](https://learn.microsoft.com/en-us/sql/relational-databases/performance/display-an-actual-execution-plan?view=sql-server-ver17).

### Đỏ — cần điều tra bằng số liệu

| Dấu hiệu | Giả thuyết cần kiểm tra |
| --- | --- |
| Lọc hẹp nhưng đọc rất nhiều row/page | Sai prefix, residual filter, biểu thức/cast, statistics hoặc parameter khác lúc compile |
| Key/RID Lookup lặp nhiều, logical reads cao | Index thiếu payload cần lấy; cân nhắc covering hoặc projection hẹp |
| Sort/Hash spill ra tempdb/disk | Ước lượng sai, row rộng, memory grant/work_mem hoặc lượng dữ liệu lớn; index có thể giúp nhưng không luôn là đáp án |
| Nested Loop có inner scan lặp nhiều | Access path inner, cardinality outer, spool hoặc khả năng dùng hash/merge join |
| Estimated rows lệch actual tại node quyết định plan | Statistics, correlation, parameter sensitivity hoặc predicate khó ước lượng |

MySQL `Using filesort` chỉ cho biết sort ngoài thứ tự index, **không bảo đảm ghi ra đĩa**; `Using temporary` cũng không đồng nghĩa disk spill. PostgreSQL `Recheck Cond` thuộc cơ chế bitmap/lossy index, không tự chứng minh filter bị bỏ khỏi index.

### Vàng — chưa chắc sai

- Scan bảng/index lớn có thể đúng khi cần nhiều row; scan covering index hẹp có thể rất rẻ.
- Bitmap Heap Scan đọc heap theo block; nó thường hợp lý với lượng match trung bình nhưng làm mất thứ tự index.
- Hash Join/HashAggregate có thể thắng các lookup lặp lại. Sort nhỏ trong RAM thường chấp nhận được.
- `CONVERT_IMPLICIT` cần xem nằm ở cột hay tham số và có warning/reads cao không; không phải conversion nào cũng chặn seek.

### Xanh — kết quả đã được kiểm chứng

- Query trả đúng dữ liệu và đạt mục tiêu latency/CPU/reads trên tham số đại diện.
- Predicate và key order giảm công việc đọc; lookup/sort nếu còn có chi phí chấp nhận được.
- Không có regression đáng kể ở write path hoặc query khác.
- Cardinality đủ sát ở các node ảnh hưởng quyết định. Không dùng ngưỡng “lệch dưới 10 lần” như chứng nhận plan tốt.

### 30 giây — checklist

1. Xác định query, tham số, số kết quả và mức độ chậm; có phải lock wait/I/O/network thay vì thiếu index?
2. Đo actual rows/read/loops và logical reads; tìm nơi đọc nhiều rồi loại bỏ nhiều.
3. Kiểm tra key prefix, residual filter, lookup, sort/spill và statistics.
4. Thử một thay đổi, đối chiếu kết quả và số liệu trước/sau với cả giá trị hiếm lẫn phổ biến.
5. Đo tác động dung lượng/ghi và giữ cách rollback. Quy trình chi tiết ở [Phần 11](#phần-11-quy-trình-tối-ưu-và-thực-hành).

## Phần 4. Index hoạt động thế nào với từng thao tác SQL

Bốn nguyên tắc ở Phần 2 là nền tảng. Khi đọc SQL, dùng thứ tự xử lý **logic** sau để hiểu ngữ nghĩa; đây không phải thứ tự thực thi vật lý bắt buộc của optimizer:

```
FROM / JOIN → WHERE → GROUP BY → HAVING → SELECT / WINDOW → DISTINCT → ORDER BY → LIMIT
```

Optimizer có thể đẩy filter xuống, đổi thứ tự join hoặc chọn hash/merge/loops miễn giữ đúng ngữ nghĩa. Thứ tự key của index phụ thuộc access pattern; không suy ra “mọi cột WHERE phải đứng trước ORDER BY” từ sơ đồ logic này.

### Phép so sánh không bằng (!=): Xem phân bố trước khi kết luận

`status <> 'open'` tương ứng các giá trị nhỏ hơn hoặc lớn hơn 'open', và loại NULL. Engine có thể dùng hai range, nhiều seek hoặc scan; **không bắt buộc đọc toàn bộ index**. Statistics/histogram giúp ước lượng số row match, dù ước lượng vẫn có thể sai.

```sql
SELECT * FROM payments
WHERE shop_id = 42 AND status <> 'open';
-- (shop_id, status) giới hạn công việc trong một shop trước.
```

Nếu 'open' chiếm gần toàn bộ bảng thì phần còn lại có thể rất selective. Nếu phần còn lại chiếm đa số, scan thường hợp lý. Không có phép so sánh nào “luôn giết index”.

Có thể thử `IN ('paid', 'cancelled', 'refunded')` **chỉ khi tập trạng thái đầy đủ được ràng buộc và ngữ nghĩa đúng**; thêm trạng thái mới có thể làm kết quả khác `<> 'open'`. Viết IN không tự làm cùng tập kết quả rẻ hơn.

Với query cố định, partial/filtered index vừa thu hẹp dữ liệu vừa giữ thứ tự cần dùng:

```sql
-- PostgreSQL / SQL Server: giả định status là kiểu chuỗi phù hợp
CREATE INDEX payments_not_open
ON payments (shop_id, created_at)
WHERE status <> 'open';
```

MySQL có thể dùng generated/functional column cho biểu thức boolean từ 8.0.13+, nhưng index boolean không tự tăng selectivity; nên kết hợp shop/order và đo plan. [MySQL: các toán tử hỗ trợ range](https://dev.mysql.com/doc/refman/8.4/en/range-optimization.html).

### NULL: Giá trị đặc biệt cần đặc biệt chú ý

`NULL` trong SQL nghĩa là "không biết" (unknown), không phải "rỗng" hay "zero".

```
NULL = NULL    → NULL (không phải TRUE!)
NULL != NULL   → NULL (không phải TRUE!)
NULL > 5       → NULL
NULL + 10      → NULL
-- Nhiều toán tử thông thường truyền NULL; COALESCE/IS NULL là các ngoại lệ.
-- WHERE chỉ giữ TRUE; FALSE và UNKNOWN đều bị loại.
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


### LIKE: Prefix, collation và ký tự đại diện ở đầu

`LIKE 'Nguyễn%'` có thể được tối ưu thành range từ tiền tố, nhưng phụ thuộc collation, operator class, kiểu dữ liệu và khả năng xác định pattern khi lập plan. **Không tự viết cận trên bằng cách tăng ký tự cuối**: cách đó không đúng tổng quát với Unicode/collation.

```sql
-- PostgreSQL, name kiểu text với deterministic collation:
-- text_pattern_ops hỗ trợ prefix LIKE khi collation mặc định không phù hợp.
CREATE INDEX contacts_name_prefix ON contacts (name text_pattern_ops);
SELECT * FROM contacts WHERE name LIKE 'Nguyễn%';

-- Substring / ILIKE: pg_trgm
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX contacts_name_trgm ON contacts USING GIN (name gin_trgm_ops);
SELECT * FROM contacts WHERE name ILIKE '%nguyễn%';
```

`text_pattern_ops` không thay index với operator class mặc định cho mọi ORDER BY/range theo collation. ILIKE có thể cần trigram hoặc index LOWER(name) cùng query LOWER(name). Pattern quá ngắn/ít trigram có thể vẫn đọc nhiều. [PostgreSQL: operator classes](https://www.postgresql.org/docs/18/indexes-opclass.html).

SQL Server/MySQL cũng có thể tận dụng B-tree cho prefix LIKE; `LIKE '%abc%'` thường không tạo seek theo prefix, dù engine vẫn có thể scan một covering index hoặc dùng index cho filter khác.

Full-Text Search (CONTAINS/FREETEXT, MATCH AGAINST) tìm theo token/ngôn ngữ, **không tương đương substring LIKE**. Chọn trigram, ngram hoặc search engine theo semantics cần giữ; không đổi query chỉ vì tên tính năng “full-text”.

### ORDER BY: Tận dụng thứ tự index khi có lợi

Nếu các cột đứng trước sort key được cố định bằng equality và phần còn lại khớp ORDER BY, index có thể cung cấp thứ tự. Range hoặc nhiều giá trị IN trước sort key có thể làm mất thứ tự toàn cục cần dùng.

```sql
SELECT * FROM issues
WHERE type = 'bug'
ORDER BY severity DESC, created_at DESC;

-- ✅ Index: (type, severity DESC, created_at DESC)
```

Sort có thể tốn CPU/RAM và spill khi dữ liệu lớn; sort nhỏ có thể rẻ hơn duy trì thêm một index. Không tránh sort bằng mọi giá.

- **PostgreSQL:** `work_mem` là mức cơ sở cho mỗi operation; hash còn chịu `hash_mem_multiplier`, parallel worker cũng dùng memory. Chỉ điều chỉnh theo số operation/concurrency thực tế, ưu tiên phạm vi session/transaction đã đo.
- **SQL Server:** memory grant dựa trên plan và ước lượng, có memory grant feedback ở phiên bản hỗ trợ. Kiểm tra statistics, độ rộng row, grant và spill; index theo order có thể bỏ Sort. Covering riêng lẻ không bảo đảm thứ tự. Grant hints chỉ dùng sau khi đo.
- **MySQL:** `sort_buffer_size` ảnh hưởng session thực hiện sort. Đo memory và filesort trước khi tăng; tránh áp một mức lớn toàn server khi nhiều session cùng sort.

`ORDER BY ... LIMIT 10` không có index phù hợp thường phải xét mọi matching row, nhưng có thể dùng top-N sort thay vì sắp xếp đầy đủ. Index phù hợp cho phép dừng khi có đủ 10 row thỏa mọi filter. Backward scan đảo **tất cả** chiều key; các chiều mixed cần khớp hoặc đảo toàn bộ. Thứ tự NULL và collation cũng phải khớp. [PostgreSQL: Indexes and ORDER BY](https://www.postgresql.org/docs/18/indexes-ordering.html).

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

DISTINCT và GROUP BY có thể dùng sort/stream aggregate hoặc hash, nhưng không luôn có cùng execution plan. Input đã có thứ tự có thể giúp stream/group aggregate tránh sort; hash vẫn có thể rẻ hơn.

```sql
SELECT is_paying, COUNT(*) FROM users GROUP BY is_paying;
-- Index: (is_paying)
```

Quy tắc:

- GROUP BY đơn giản: thử index có grouping keys liền nhau; optimizer có thể đổi thứ tự grouping keys.
- GROUP BY + WHERE: equality prefix có thể đặt trước grouping keys mà vẫn giữ thứ tự trong prefix đó.
- Range trước grouping keys khác thường không cung cấp thứ tự nhóm toàn cục; nếu GROUP BY chính range key thì thứ tự vẫn có thể hữu ích. So sánh range scan + hash/sort với scan theo nhóm.
- Cột dùng cho SUM/AVG có thể là payload INCLUDE để covering trên PostgreSQL/SQL Server. MIN/MAX và loose scan còn tùy query/engine.

Khi `GROUP BY` đúng primary key, **PostgreSQL** và **MySQL** (`ONLY_FULL_GROUP_BY`) cho phép `SELECT` cột khác của cùng bảng (functional dependency):

```sql
SELECT actors.id, actors.name, COUNT(roles.role_id)
FROM actors
LEFT JOIN roles ON roles.actor_id = actors.id
GROUP BY actors.id;  -- PG / MySQL: name suy ra từ PK
```

**SQL Server không làm vậy.** Cột không aggregate phải nằm trong `GROUP BY`, kể cả khi đã group theo PK — nếu không sẽ lỗi 8120.


### JOIN: Phân tách và kết hợp

Nested-loop join thường lấy row từ outer rồi truy cập inner theo khóa của row đó; optimizer có thể dùng index, spool/materialize hoặc scan. Hash Join và Merge Join là những phương án khác, thường hữu ích cho tập lớn. Đánh giá index trên từng bảng theo filter/join thực tế.

```sql
SELECT employee.* FROM employee
JOIN department USING (department_id)
WHERE employee.salary > 100000 AND department.country = 'NR';

-- Lọc country trước (ít phòng ban): Seek country, rồi Seek nhân viên
CREATE INDEX idx_dept_country ON department (country);
CREATE INDEX idx_emp_dept_salary ON employee (department_id, salary);
-- PK department(department_id) đã đủ cho lookup theo id

-- Lọc salary trước: range trên employee, rồi lookup PK department để lọc country
-- CREATE INDEX idx_emp_salary ON employee (salary) INCLUDE (department_id);
```

`(department_id, country)` không có prefix equality cho `country = 'NR'`; scan/skip scan tùy hệ vẫn có thể xảy ra, nhưng `(country)` thường phù hợp hơn cho driving lookup theo country.

Optimizer có thể đổi thứ tự inner join; outer join và các dependency hạn chế lựa chọn này. Không tạo index cho mọi thứ tự join hay áp giới hạn 6–8 bảng cứng. Theo dõi plan, cardinality và chi phí compile của những query thực sự quan trọng.

**Lấy N dòng cho mỗi nhóm** — PostgreSQL dùng `LATERAL`, SQL Server dùng `APPLY`:

```sql
-- PostgreSQL
SELECT customers.*, recent_sales.*
FROM customers
LEFT JOIN LATERAL (
  SELECT * FROM sales
  WHERE sales.customer_id = customers.id
  ORDER BY created_at DESC, id DESC
  LIMIT 3
) AS recent_sales ON true;

-- SQL Server
SELECT customers.*, recent_sales.*
FROM customers
OUTER APPLY (
  SELECT TOP (3) *
  FROM sales
  WHERE sales.customer_id = customers.id
  ORDER BY created_at DESC, id DESC
) AS recent_sales;

-- Index cần: (customer_id, created_at DESC) trên sales
```


### Subquery: Không chậm như bạn nghĩ

Subquery không mặc nhiên chậm. Optimizer có thể decorrelate thành join, materialize hoặc đánh giá lặp lại; thiếu index chỉ là một trong các nguyên nhân.

- **Independent subquery:** thường có thể tính một lần, tùy biểu thức và plan.
- **Correlated subquery:** có thể chạy theo outer row hoặc được chuyển thành semi/anti join. EXISTS chỉ yêu cầu biết có match; không bảo đảm engine luôn dùng nested loop rồi dừng đúng một lần đọc.

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

SELECT với cùng predicate giúp khảo sát phần tìm row, nhưng DML còn có locking, constraint/trigger, logging, index maintenance và có thể có operator bảo vệ Halloween. Xem cả plan DML khi cần; SELECT nhanh chưa bảo đảm DELETE/UPDATE nhanh.

```sql
-- Thay vì: DELETE FROM logs WHERE created_at < '2024-01-01';
SELECT * FROM logs WHERE created_at < '2024-01-01';
-- Nếu đọc nhiều để tìm ít row, thử index created_at; nếu xóa phần lớn bảng, scan có thể đúng.
```

`UPDATE` cột nằm trong index = xóa entry cũ + chèn entry mới (page split, giết HOT — Phần 10). `UPDATE` 1 triệu row theo điều kiện hẹp: index cho **mệnh đề WHERE**, không phải cho cột đang gán.

### IN: Nhiều equality, không phải range

`IN ('a','b','c')` có ngữ nghĩa nhiều equality, nhưng cách thực hiện có thể là nhiều seek, range, bitmap hoặc scan. Danh sách lớn có thể khiến scan rẻ hơn; thứ tự cột sau IN không mặc nhiên là thứ tự toàn cục.

```sql
-- ✅ Index (status) hoặc (shop_id, status)
WHERE shop_id = 42 AND status IN ('paid', 'refunded', 'cancelled');
```

- `NOT IN (...)` giống `!=`: hai phía + bẫy `NULL` (Phần 4 NULL). `NOT EXISTS` an toàn hơn (Phần 8).
- `IN` vài nghìn literal: parse/plan phình, cardinality đoán mò. Nhét staging table + `JOIN` (có index) thường ổn hơn.
- `IN (SELECT …)`: index bảng trong subquery (cột so sánh). Không phải lúc nào cũng kém JOIN — xem plan.

`= ANY(ARRAY[...])` (PostgreSQL) cùng họ với `IN`.

### HAVING: Không thay được WHERE

`WHERE` lọc row, `HAVING` lọc nhóm theo ngữ nghĩa logic. Optimizer có thể đẩy điều kiện HAVING chỉ tham chiếu grouping key xuống trước aggregate khi hợp lệ. Vì vậy HAVING không mặc nhiên ngăn dùng index, nhưng nên dùng WHERE để thể hiện rõ điều kiện row.

```sql
-- HAVING trên grouping key: optimizer có thể đẩy xuống; WHERE diễn đạt rõ hơn
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

`UNION` phải loại trùng, thường dùng sort/hash hoặc tận dụng thứ tự/uniqueness. UNION ALL phù hợp khi muốn giữ trùng hoặc bảo đảm không trùng. Viết lại OR thành UNION ALL cần guard chống trùng và xử lý NULL đúng; so plan trước khi kết luận nhanh hơn (Phần 6).

### Window function: Partition + Order cũng cần index

`ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC, id DESC)` cần input theo partition/order. Index `(customer_id, created_at DESC, id DESC)` có thể tránh Sort; window operator vẫn cần thực hiện. Thêm id để chọn ổn định khi thời gian trùng.

```sql
-- Top 1 đơn mỗi khách: window hoặc LATERAL (Phần 8)
SELECT *
FROM (
  SELECT *, ROW_NUMBER() OVER (
    PARTITION BY customer_id ORDER BY created_at DESC, id DESC
  ) AS rn
  FROM orders
) t
WHERE rn = 1;
```

Filter trước window có thể đổi ngữ nghĩa: “đơn mới nhất trong năm” khác “đơn mới nhất toàn lịch sử rồi chỉ giữ đơn của năm đó”. Index đứng đầu created_at có thể lọc tốt nhưng không cung cấp thứ tự theo customer_id. Filter rn=1 không phải seek key thông thường, dù engine có thể có tối ưu top-N/window. So sánh window với LATERAL/APPLY bằng tập khách và số row thực tế.

## Phần 5. Tại sao Database không dùng Index của tôi

Đây là câu hỏi gây bực bội nhất mà developer hay gặp. Index đã tạo, query rõ ràng match — nhưng database vẫn lờ tịt.

### Quy trình thực thi query: Bên trong "bộ não" của database

Mỗi query đi qua 4 bước:

```
1. PARSE         — Phân tích cú pháp SQL
2. BIND / REWRITE — Giải quyết tên/kiểu và biến đổi biểu thức
3. OPTIMIZE       — Cân nhắc access path, join order, cardinality và cost
4. EXECUTE        — Thực thi plan được chọn (có thể tái dùng plan cache)
```

Đây là sơ đồ khái quát; engine không bắt buộc bắt đầu bằng full table scan hay khảo sát mọi plan. Optimizer tìm plan có cost ước lượng tốt trong phạm vi tìm kiếm của nó; plan được chọn không bảo đảm nhanh nhất trong thực tế.

```sql
-- PostgreSQL: luôn dùng ANALYZE khi đo thật (có chạy query)
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'test@example.com';
-- Seq Scan          = full table scan
-- Index Scan        = dùng index rồi nhảy vào heap
-- Index Only Scan   = có thể trả từ index; kiểm tra Heap Fetches
-- Bitmap Heap Scan  = nhiều match, đọc heap theo batch

-- SQL Server: bật Actual Execution Plan và đo I/O/CPU:
SET STATISTICS IO, TIME ON;
SELECT * FROM users WHERE email = 'test@example.com';
-- Index Seek              = định vị theo key rồi đọc range; vẫn có thể đọc nhiều
-- Index Scan              = scan index, có thể dừng sớm theo TOP
-- Clustered Index Seek    = tìm theo clustered key
-- Clustered Index Scan    = gần như table scan
-- Key Lookup              = nonclustered → clustered (đắt nếu nhiều row)
-- Table Scan              = heap, không clustered index

-- MySQL — xem cột 'type':
--   ALL   = full table scan; có thể phù hợp khi cần nhiều row
--   index = full index scan
--   range = range scan trên index
--   ref   = index lookup theo key không nhất thiết unique
--   const = tối đa một row qua PK/UNIQUE với key cố định
EXPLAIN SELECT * FROM users WHERE email = 'test@example.com';
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

Hãy tập thói quen đọc plan cho mọi query quan trọng trước khi deploy.

### Index không khớp với query: Lý do phổ biến nhất

Hàm/cast trên cột thường khiến index trên giá trị gốc không cung cấp seek/range trực tiếp. Có ngoại lệ: expression/computed-column index khớp, optimizer biến đổi được predicate, hoặc dynamic seek với một số cast trên SQL Server. Hãy kiểm tra plan thay vì coi mọi hàm đều vô hiệu hóa index.

```sql
-- Hàm trên cột: index birthday thường không tạo range hẹp trực tiếp
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
-- WHERE col + 5 < 20 → col < 15 chỉ khi giữ đúng kiểu, overflow và NULL semantics.
-- CONCAT(first, ' ', last): dùng index biểu thức/cột computed nếu cần đúng phép ghép.
-- Tách thành first/last chỉ hợp lệ khi không có mơ hồ về dấu cách, NULL và collation.
-- ❌ WHERE varchar_col = 12345            → ✅ WHERE varchar_col = '12345'  (xem implicit conversion)
-- ❌ WHERE DATE(created_at) = '...'       → ✅ range trên created_at
-- ❌ WHERE LOWER(email) = '...'           → ✅ functional index / computed column
```

Index ẩn (MySQL): kiểm tra `IS_VISIBLE` trong `INFORMATION_SCHEMA.STATISTICS`.

Predicate **SARGable** (Search ARGument Able) là predicate có thể chuyển thành điều kiện tìm theo key của access path. So sánh trực tiếp cột với tham số đúng kiểu thường dễ tối ưu; biểu thức đã được index cũng có thể SARGable trên expression index.

### Full table scan nhanh hơn: Khi database đúng mà bạn sai

Scan có thể rẻ hơn khi phải lấy nhiều row qua lookup. Không có ngưỡng 10–30% áp dụng chung: tipping point phụ thuộc row/index width, covering, heap correlation, cache, storage và parallelism.

PostgreSQL: random_page_cost mô hình hóa chi phí I/O, không phải công tắc ép dùng index. Có thể thử trong session đã benchmark; SSD không tự chứng minh giá trị 1.1 là đúng:

```sql
SET random_page_cost = 1.1;
```

Bảng nhỏ (100–200 rows): full scan là hành vi bình thường.

**Thống kê cũ:** sau bulk load / update / delete lớn, luôn làm mới statistics:

```sql
-- PostgreSQL
ANALYZE users;

-- SQL Server
UPDATE STATISTICS users;
-- FULLSCAN có thể đắt; chỉ dùng khi sampling không đủ cho phân bố cần ước lượng.

-- MySQL
ANALYZE TABLE users;
```


### Database chọn index khác: Khi có nhiều lựa chọn

Khi có nhiều index, optimizer xét tổng cost của đọc index, lookup, sort và join; có thể chọn một index, kết hợp nhiều index hoặc scan. Composite index khớp query là ứng viên, không luôn là giải pháp tốt nhất.

Với equality filter, thử key filter trước rồi sort. Với range filter trên cột khác sort, phải cân nhắc độ chọn lọc và LIMIT (Phần 2, 4).

```sql
SELECT * FROM issues
WHERE type = 'open' ORDER BY created_at DESC LIMIT 10;

-- ✅ Index: (type, created_at DESC)
```


### Parameter sniffing: Plan đúng với lần chạy đầu, sai với lần sau

SQL Server có thể cache plan được compile với giá trị parameter ban đầu. Một plan seek + nhiều lookup phù hợp với giá trị hiếm có thể tốn hơn scan khi tái dùng cho giá trị phổ biến, hoặc ngược lại. Không phải mọi plan tái dùng đều gặp vấn đề; cần đối chiếu compiled/runtime parameters và phân bố dữ liệu.

```sql
-- SQL Server: xem plan đang cache; khi lệch thống kê / phân bố:
UPDATE STATISTICS orders WITH FULLSCAN;
-- Hoặc local, không cache plan cho query này:
SELECT * FROM orders WHERE status = @status OPTION (RECOMPILE);
-- Hoặc OPTIMIZE FOR UNKNOWN — dùng density trung bình, không sniff giá trị cụ thể
```

**PostgreSQL:** prepared statement có custom/generic plan. So sánh bằng EXPLAIN EXECUTE với tham số đại diện; force_custom_plan (PG 12+) chỉ là lựa chọn có chi phí planning, không cần bỏ prepared statement mặc định.

**SQL Server:** PSP từ 2022 với compatibility level 160 và cấu hình phù hợp có thể giữ nhiều plan cho predicate đủ điều kiện. OPTION(RECOMPILE) đánh đổi CPU compile; OPTIMIZE FOR UNKNOWN dùng ước lượng trung bình nên vẫn có thể kém cho cả hai cực. Statistics mới không tự giải quyết mọi skew. [PSP](https://learn.microsoft.com/en-us/sql/relational-databases/performance/parameter-sensitive-plan-optimization?view=sql-server-ver17).

**MySQL:** cache cấu trúc prepared statement không đồng nghĩa tái sử dụng plan theo kiểu parameter sniffing của SQL Server. MySQL lặp bước optimize khi execute; cập nhật statistics khi cần, nhưng chẩn đoán theo engine thực tế. [Prepared statement cache](https://dev.mysql.com/doc/refman/8.4/en/statement-caching.html), [MySQL: optimization khi EXECUTE](https://dev.mysql.com/blog-archive/re-factoring-some-internals-of-prepared-statements-in-5-7/).

### Tạo và bảo trì index: Giảm thời gian chặn ghi

Online/concurrent không có nghĩa “không khóa”: DDL vẫn có thể cần schema/metadata lock và chờ transaction dài. Ngoài blocking, tính cả CPU, I/O, dung lượng tạm, transaction log/WAL và replication lag.

```sql
-- PostgreSQL: ngoài transaction block; REINDEX CONCURRENTLY từ PG 12
CREATE INDEX CONCURRENTLY idx_orders_status ON orders (status);
REINDEX INDEX CONCURRENTLY idx_orders_status;

-- SQL Server: xác minh edition/version, loại index và kiểu cột được hỗ trợ
CREATE INDEX idx_orders_status ON orders (status) WITH (ONLINE = ON);
ALTER INDEX idx_orders_status ON orders REBUILD WITH (ONLINE = ON);
-- REORGANIZE online nhưng vẫn dùng tài nguyên/khóa; không thay mọi trường hợp REBUILD
ALTER INDEX idx_orders_status ON orders REORGANIZE;

-- MySQL/InnoDB: yêu cầu rõ mức concurrency, lỗi nếu không được hỗ trợ
ALTER TABLE orders ADD INDEX idx_orders_status (status),
  ALGORITHM=INPLACE, LOCK=NONE;
```

PostgreSQL concurrent build thất bại có thể để lại **INVALID index** vẫn tốn chi phí ghi. Kiểm tra pg_index.indisvalid/indisready, xử lý index lỗi rồi mới coi migration hoàn tất. Không chạy CREATE INDEX CONCURRENTLY trong transaction tự động của migration framework; bảng partitioned cần cách build từng partition phù hợp.

SQL Server ONLINE vẫn có schema lock ngắn ở các giai đoạn; WAIT_AT_LOW_PRIORITY/resumable tùy tính năng/phiên bản có thể giúp quản lý rollout. MySQL LOCK=NONE vẫn có metadata lock, và không phải mọi DDL dùng được INPLACE.

[PostgreSQL CREATE INDEX](https://www.postgresql.org/docs/18/sql-createindex.html), [SQL Server CREATE INDEX](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-index-transact-sql?view=sql-server-ver17), [InnoDB online DDL](https://dev.mysql.com/doc/refman/8.4/en/innodb-online-ddl-operations.html).

### Seek rồi Filter: Index dùng mà vẫn chậm

Plan có **Seek** không có nghĩa là predicate đã được đẩy hết vào index. Optimizer Seek theo prefix khớp, rồi **Filter** phần còn lại trên từng row (residual predicate).

```sql
-- Index (shop_id, created_at)
WHERE shop_id = 42
  AND EXTRACT(YEAR FROM created_at) = 2024;  -- không SARGable
```

Access path có thể định vị shop_id=42 rồi lọc năm trên từng entry/row, khiến shop lớn đọc rất nhiều dù trả ít. PostgreSQL hiển thị Index Scan, SQL Server tương tự là Index Seek với YEAR(created_at) ở residual filter. Viết lại range hoặc dùng expression/computed index phù hợp.

SQL Server: phân biệt Seek Predicates và residual Predicate, đối chiếu Rows Read. PostgreSQL: phân biệt Index Cond, Filter, Rows Removed by Filter; điều kiện ở cột sau range có thể thuộc Index Cond nhưng vẫn đọc nhiều entry. Recheck Cond còn phục vụ bitmap/lossy recheck, không đồng nghĩa residual filter.

### Key Lookup / heap fetch: Khi Seek thua Scan

Nonclustered Seek rồi **Key Lookup** (SQL Server) / heap fetch (PostgreSQL) cho mỗi row vì `SELECT *` hoặc cột ngoài index. Vài chục lookup ổn; vài trăm nghìn lookup = random I/O thảm — optimizer **đúng** khi chọn Clustered/Seq Scan.

```sql
-- Index (status) không covering
SELECT * FROM orders WHERE status = 'pending';
```

Ví dụ giá trị hiếm có thể dùng seek + lookup, giá trị phổ biến có thể dùng scan; phần trăm cụ thể không quyết định plan một mình. Thử covering/INCLUDE hoặc projection hẹp khi đo thấy lookup đắt. Không ép index chỉ vì muốn thấy Seek.

### Partial / filtered index không khớp predicate

Index một phần bảng chỉ là ứng viên khi optimizer chứng minh predicate query suy ra predicate index; viết “chặt hơn” theo ý định nghiệp vụ chưa đủ nếu optimizer không nhận ra:

```sql
CREATE INDEX orders_open_created ON orders (created_at)
WHERE status = 'open';   -- PostgreSQL partial / SQL Server filtered
```

| Query | Dùng được? |
| ----- | ---------- |
| `WHERE status = 'open' AND created_at > …` | Có |
| `WHERE created_at > …` (không ghi status) | Không |
| `WHERE status = @status` (SQL Server parameter) | **Thường không** — optimizer không dám giả định `@status` luôn `'open'` |

SQL Server có thể thử literal nghiệp vụ cố định hoặc OPTION(RECOMPILE); dynamic SQL vẫn parameterize input người dùng. PostgreSQL custom plan có thể nhận ra parameter đã biết, generic plan không được giả định giá trị đó. Predicate status IN('open','paused') không thể được phục vụ đầy đủ bằng riêng index chỉ chứa open.

### Thống kê lệch và cột tương quan

Nếu mô hình ước lượng không nắm tương quan giữa city và zip, cardinality có thể sai đáng kể. Ví dụ minh họa độc lập 1% × 1% không phải công thức bắt buộc của mọi optimizer/CE version.

Sau bulk load, histogram cũ = “index có mà không Seek” (Phần 10).

```sql
-- PostgreSQL 12+: ví dụ có MCV; dependencies/ndistinct đã có từ 10
CREATE STATISTICS orders_city_zip (dependencies, mcv, ndistinct)
ON city, zip FROM orders;
ANALYZE orders;

-- SQL Server: stats nhiều cột (tạo index composite cũng tạo stats)
CREATE STATISTICS orders_city_zip ON orders (city, zip);
UPDATE STATISTICS orders WITH FULLSCAN;
```

PostgreSQL extended statistics giúp một số điều kiện trên cùng bảng và GROUP BY, không giải quyết mọi join estimate. SQL Server statistics nhiều cột có histogram **chỉ cho cột đầu**, density cho các prefix; không phải histogram đa chiều tương đương PostgreSQL MCV. [PostgreSQL CREATE STATISTICS](https://www.postgresql.org/docs/18/sql-createstatistics.html), [SQL Server statistics](https://learn.microsoft.com/en-us/sql/relational-databases/statistics/statistics?view=sql-server-ver17).

Ước lượng lệch nhiều cần kiểm tra tại node quyết định join/access path, chưa chắc thiếu index (Phần 3).

### View và hàm bọc cột: Index “có” mà optimizer mù

Index nằm trên **bảng**, không phải trên tên view. View `CREATE VIEW v AS SELECT *, LOWER(email) AS email_lc` rồi `WHERE email_lc = …` = hàm trên cột (mục Index không khớp). Scalar UDF trong `WHERE` (SQL Server) thường **chặn** Seek — inline / viết lại.

```sql
-- ❌ view giấu hàm
CREATE VIEW v_users AS
SELECT id, LOWER(email) AS email FROM users;
SELECT * FROM v_users WHERE email = 'a@b.com';  -- không dùng index (email)

-- Predicate trên cột gốc chỉ cùng ngữ nghĩa nếu collation/chuẩn hóa email phù hợp.
-- Muốn tìm không phân biệt hoa thường, giữ LOWER(email) + index khớp.
SELECT * FROM users WHERE LOWER(email) = 'a@b.com';
-- hoặc WHERE LOWER(email) = ... + index (LOWER(email))
```

SQL Server indexed view được engine duy trì khi DML và có yêu cầu schema/SET options. PostgreSQL MATERIALIZED VIEW lưu kết quả và cần REFRESH theo chiến lược ứng dụng. Chúng khác view thường và khác nhau về độ mới/chi phí ghi.

## Phần 6. Cạm bẫy và mẹo nâng cao về Indexing


### Index trên biểu thức: Khi không thể viết lại query

```sql
-- PostgreSQL: giả định birthday kiểu DATE; expression phải IMMUTABLE
CREATE INDEX contacts_birthmonth ON contacts ((EXTRACT(MONTH FROM birthday)));
SELECT * FROM contacts WHERE EXTRACT(MONTH FROM birthday) = 5;

-- MySQL 8.0.13+: functional index
CREATE INDEX contacts_birthmonth ON contacts ((MONTH(birthday)));
SELECT * FROM contacts WHERE MONTH(birthday) = 5;

-- SQL Server / MariaDB: computed / virtual column rồi đánh index
ALTER TABLE contacts ADD birth_month AS (MONTH(birthday)) PERSISTED;  -- SQL Server
-- MariaDB: ADD COLUMN birth_month INT AS (MONTH(birthday)) VIRTUAL;
CREATE INDEX contacts_birthmonth ON contacts (birth_month);
SELECT * FROM contacts WHERE birth_month = 5;
```

Biểu thức, kiểu và collation của query phải tương thích với index; optimizer có thể nhận ra một số dạng tương đương, nhưng không phải mọi biến đổi đại số. PostgreSQL yêu cầu function/operator trong index IMMUTABLE: EXTRACT từ DATE khác EXTRACT từ TIMESTAMPTZ phụ thuộc timezone. SQL Server computed-column index cần deterministic/precision và SET options phù hợp; PERSISTED không phải điều kiện bắt buộc cho mọi biểu thức. [Expression index](https://www.postgresql.org/docs/18/indexes-expressional.html), [SQL Server computed-column index](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/indexes-on-computed-columns?view=sql-server-ver17).

### Cột giá trị ít: Đo selectivity của giá trị đang tìm

Ít distinct values không đồng nghĩa index vô dụng. Một trạng thái hiếm có thể rất selective; index hẹp covering COUNT/GROUP BY hoặc kết hợp tenant/order cũng có thể hữu ích dù giá trị phổ biến. Không có ngưỡng phần trăm cố định để bỏ index.

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

Optimizer phải chứng minh predicate của query suy ra predicate của index. Biểu thức tương đương, parameterization và kiểu dữ liệu có thể ảnh hưởng khả năng chứng minh. Đo index theo cả đọc và ghi, không chỉ số distinct values.

### Biến đổi điều kiện phạm vi: Biến range thành so sánh bằng

Khi ngưỡng là hằng số nghiệp vụ cố định, expression/computed boolean có thể tạo equality prefix trước sort key. Nếu ngưỡng thay đổi theo request, index này không thay được range tổng quát. PostgreSQL/SQL Server còn có thể dùng partial/filtered index theo ngưỡng.

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
-- Nếu chọn IF() thì query cũng dùng IF(stars > 1000, 1, 0) = 1:
-- CREATE INDEX repos_search ON repos (language, ((IF(stars > 1000, 1, 0))), sponsors);
SELECT * FROM repos
WHERE language = 'TypeScript'
  AND (stars > 1000) = TRUE
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

**PostgreSQL:** lựa chọn operator và cast phụ thuộc kiểu tham số. Một literal chuỗi chưa định kiểu có thể được suy ra theo cột, còn tham số TEXT so với UUID/INTEGER có thể báo lỗi hoặc cần cast. Bind đúng kiểu là cách dễ kiểm soát.

**MySQL:** nếu cột là `VARCHAR` nhưng so sánh với số, MySQL chuyển **cột** sang số → `CAST()` → index vô dụng.

```sql
-- ❌ chậm (MySQL)
SELECT * FROM orders WHERE payment_id = 57013925718;

-- ✅
SELECT * FROM orders WHERE payment_id = '57013925718';
```

Chiều ngược lại có thể convert giá trị thay vì cột, nhưng quy tắc so sánh số/chuỗi vẫn có thể gây kết quả bất ngờ. Bind tham số cùng kiểu và kích thước phù hợp.

**SQL Server:** implicit conversion trong parameter mapping là một nguyên nhân thường gặp cần kiểm tra:

```sql
-- Cột payment_id là VARCHAR / NVARCHAR
-- ❌ số → convert cột → Index Scan
SELECT * FROM orders WHERE payment_id = 57013925718;

-- ✅ cùng kiểu với cột
SELECT * FROM orders WHERE payment_id = '57013925718';
```

Các cặp kiểu cần kiểm tra trên SQL Server:


| Cột                          | Tham số / literal                    | Kết quả               |
| ---------------------------- | ------------------------------------ | --------------------- |
| `VARCHAR` | `NVARCHAR` | Có thể convert cột; ảnh hưởng seek tùy collation/plan |
| `DATE` | `DATETIME` | DATE có precedence thấp hơn; kiểm tra cast/dynamic seek |
| `DATETIME2` | `DATETIME` | Thường convert tham số sang DATETIME2, không phải cột |
| `INT` | `VARCHAR` | Thường convert tham số sang INT; chuỗi không hợp lệ có thể lỗi |
| `VARCHAR` | `INT` | Thường convert cột sang INT, có thể mất seek hoặc lỗi dữ liệu |
| Collation khác nhau khi JOIN | Kiểu chuỗi | Có thể lỗi collation conflict hoặc cần chuyển đổi |


SQL Server chuyển kiểu có precedence thấp sang kiểu cao. Xem CONVERT_IMPLICIT nằm ở đâu và actual reads; conversion trên tham số vẫn có thể seek. Với ADO.NET/Dapper, khai báo SqlDbType, Size hoặc DbString.IsAnsi theo schema; EF cần mapping Unicode/độ dài phù hợp. [Data type precedence](https://learn.microsoft.com/en-us/sql/t-sql/data-types/data-type-precedence-transact-sql?view=sql-server-ver17).

### Truy vấn chỉ từ index: Không cần chạm vào bảng dữ liệu

Đủ dữ liệu trong index là điều kiện để tránh lookup lấy payload. PostgreSQL còn phải kiểm tra MVCC visibility; InnoDB cũng có thể cần truy cập clustered record để xác định visibility. Vì vậy covering không phải lời hứa “không bao giờ đọc bảng”. [InnoDB MVCC và secondary index](https://dev.mysql.com/doc/refman/8.4/en/innodb-multi-versioning.html).

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

INCLUDE chứa payload ở leaf, không cung cấp seek/order và không tham gia UNIQUE. Cột INCLUDE vẫn có thể dùng cho residual filter nếu engine chọn; cột cần định vị/giữ thứ tự mới phải là key. MySQL không có INCLUDE, thêm payload vào key làm cây rộng hơn. Tránh index covering mọi cột theo SELECT *; đo dung lượng và chi phí ghi.

PostgreSQL Index Only Scan vẫn có Heap Fetches khi page chưa all-visible. Kiểm tra autovacuum, churn và transaction dài giữ snapshot; VACUUM không luôn đạt heap fetch=0 trên bảng đang ghi liên tục. SQL Server đủ cột thường tránh Key Lookup để lấy dữ liệu. [Index-only scans và visibility map](https://www.postgresql.org/docs/18/indexes-index-only-scans.html).

### Lọc và sắp xếp khi JOIN: Tối ưu trước khi denormalize

Một index thông thường không trải qua hai bảng, nhưng filter/join trên mỗi bảng vẫn có thể giảm công việc đáng kể. Trước khi denormalize, kiểm tra join key, driving set, statistics, projection, index order và LIMIT. Chỉ sao chép thuộc tính khi đo chứng minh cần, đồng thời thiết kế cách đồng bộ khi project đổi trạng thái.

```sql
-- PostgreSQL: ví dụ denormalization, chỉ dùng nếu có cơ chế đồng bộ project_status
ALTER TABLE tasks ADD COLUMN project_status VARCHAR(20);
SELECT * FROM tasks
WHERE team_id = 4 AND status = 'open' AND project_status = 'open';
```


### Vượt giới hạn kích thước index

Giới hạn được tính theo **byte**, không chỉ số ký tự. PostgreSQL B-tree có giới hạn kích thước entry theo page (xấp xỉ một phần ba page); SQL Server rowstore từ 2016 có key tối đa 900 byte clustered / 1700 byte nonclustered, INCLUDE không tính vào key limit nhưng vẫn tăng leaf. InnoDB giới hạn tùy row format/page size, thường 3072 byte với cấu hình hiện đại.

Cách xử lý:

1. Giảm độ rộng/đổi kiểu đúng semantics: UUID native/binary, cột số cho số, tránh đưa nội dung dài vào key.
2. MySQL prefix index `title(20)` hoặc expression LEFT/SUBSTRING có thể giúp tìm candidate. Phải so sánh **toàn bộ giá trị gốc** để xác nhận equality; unique prefix sẽ cấm hai chuỗi khác nhau có cùng prefix.
3. Hash digest cố định giúp tìm equality, nhưng cần kiểm tra giá trị gốc để loại **hash collision**. Không dùng UNIQUE(hash) như bằng chứng duy nhất cho uniqueness của chuỗi gốc.
4. PostgreSQL hash index hỗ trợ equality và không hỗ trợ UNIQUE/range/order. SQL Server hash index chỉ dành cho memory-optimized table; không phải thay thế chung cho rowstore B-tree.

[SQL Server key limits](https://learn.microsoft.com/en-us/sql/t-sql/statements/create-index-transact-sql?view=sql-server-ver17), [MySQL prefix/functional indexes](https://dev.mysql.com/doc/refman/8.4/en/create-index.html).

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
-- Giả định property email đã được validate là chuỗi tối đa 255 ký tự.
ALTER TABLE contacts ADD email AS
  (CAST(JSON_VALUE(attributes, '$.email') AS NVARCHAR(255))) PERSISTED;
CREATE INDEX contacts_email ON contacts (email);
SELECT * FROM contacts WHERE email = 'admin@example.com';
```

Giả định attributes kiểu JSONB trên PostgreSQL và JSON hợp lệ trên các hệ còn lại. Expression B-tree trên attributes->>'email' phục vụ equality/range của property đó; GIN trên toàn JSONB phục vụ các operator như @>, không tự tăng tốc mọi path/so sánh. jsonb_path_ops là lựa chọn có tập operator khác mặc định, nên chọn theo query.

SQL Server JSON_VALUE trả NVARCHAR(4000) theo dạng scalar thông thường; index trực tiếp biểu thức này có thể vượt key limit khi dữ liệu dài. Cast về kiểu/độ dài nghiệp vụ và validate trước để tránh truncation làm sai so sánh. [PostgreSQL JSON indexing](https://www.postgresql.org/docs/18/datatype-json.html#JSON-INDEXING), [SQL Server JSON indexes](https://learn.microsoft.com/en-us/sql/relational-databases/json/index-json-data?view=sql-server-ver17).


### Ràng buộc duy nhất và giá trị NULL: Khác nhau giữa các hệ

UNKNOWN trong phép so sánh SQL không quyết định quy tắc của UNIQUE; quy tắc này khác giữa engine.

| Hệ | UNIQUE(customer_id, shipment_id), customer_id NOT NULL |
| --- | --- |
| PostgreSQL mặc định / MySQL | Có thể có nhiều row (17, NULL) |
| PostgreSQL 15+ NULLS NOT DISTINCT | Không cho trùng (17, NULL) |
| SQL Server | UNIQUE thông thường đã không cho trùng (17, NULL); một cột UNIQUE nullable chỉ có tối đa một NULL |

```sql
-- PostgreSQL 15+: coi NULL bằng nhau cho việc kiểm tra unique
CREATE UNIQUE INDEX orders_shipment_unique
ON orders (customer_id, shipment_id) NULLS NOT DISTINCT;

-- SQL Server: cùng customer chỉ có một shipment_id, kể cả một NULL
CREATE UNIQUE INDEX orders_shipment_unique
ON orders (customer_id, shipment_id);

-- SQL Server: phương án KHÁC, cho nhiều NULL nhưng shipment_id đã gán phải unique
CREATE UNIQUE INDEX orders_shipped_unique
ON orders (customer_id, shipment_id)
WHERE shipment_id IS NOT NULL;

-- Nếu chỉ cần “mỗi khách tối đa một đơn chưa giao”, dùng index riêng:
-- PostgreSQL / SQL Server; SQL Server INCLUDE tránh hạn chế IS NULL đã biết.
CREATE UNIQUE INDEX orders_one_pending
ON orders (customer_id) INCLUDE (shipment_id)
WHERE shipment_id IS NULL;
```

Các index trên thể hiện **các quy tắc khác nhau**; chọn theo nghiệp vụ, không chạy tất cả. Tránh sentinel như -1 nếu nó là giá trị hợp lệ. [PostgreSQL unique constraints](https://www.postgresql.org/docs/18/ddl-constraints.html), [SQL Server UNIQUE và NULL](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/create-unique-indexes?view=sql-server-ver17).

### Tìm và dọn dẹp index không sử dụng

Theo dõi ít nhất chu kỳ nghiệp vụ có báo cáo/batch quan trọng. Không drop index chỉ vì cùng prefix hay counter đọc bằng 0: nó có thể bảo vệ PK/UNIQUE/FK hoặc phục vụ query hiếm. Statistics có thể reset, và replica có workload khác primary.

```sql
-- PostgreSQL: số lần scan và dung lượng, không phải danh sách chắc chắn thừa
SELECT schemaname, relname AS table_name, indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan;

-- SQL Server: LEFT JOIN để thấy cả index chưa có dòng trong usage DMV
SELECT OBJECT_SCHEMA_NAME(i.object_id) AS schema_name,
       OBJECT_NAME(i.object_id) AS table_name, i.name AS index_name,
       i.is_primary_key, i.is_unique, i.is_unique_constraint,
       COALESCE(s.user_seeks, 0) AS user_seeks,
       COALESCE(s.user_scans, 0) AS user_scans,
       COALESCE(s.user_lookups, 0) AS user_lookups,
       COALESCE(s.user_updates, 0) AS user_updates
FROM sys.indexes AS i
JOIN sys.tables AS t ON t.object_id = i.object_id
LEFT JOIN sys.dm_db_index_usage_stats AS s
  ON s.database_id = DB_ID()
 AND s.object_id = i.object_id AND s.index_id = i.index_id
WHERE i.index_id > 0 AND i.is_hypothetical = 0;

-- MySQL: tách counter fetch khỏi insert/update/delete
SELECT object_schema, object_name, index_name,
       count_fetch, count_insert, count_update, count_delete
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE object_schema = 'app'
ORDER BY count_fetch;
```

user_updates SQL Server đếm operation, không phải số row hay thời gian duy trì index. Trước khi drop, lưu DDL, dependency và baseline của các query liên quan.

- **MySQL:** thử INVISIBLE nếu loại index cho phép. Nó vẫn được duy trì và vẫn enforce UNIQUE; không mô phỏng toàn bộ tác động ghi của DROP.
- **SQL Server:** DISABLE không tương đương invisible. Disable clustered index làm dữ liệu bảng không truy cập được; unique/constraint index còn ảnh hưởng constraint/FK. Với nonclustered thường cũng phải REBUILD để phục hồi. Ưu tiên thử trên bản sao/staging.
- **PostgreSQL:** không có invisible index tích hợp. **Đổi tên không ngăn optimizer dùng index.** Thử trên bản sao với workload đại diện.

[SQL Server: ảnh hưởng khi disable index](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/disable-indexes-and-constraints?view=sql-server-ver17).

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

Chỉ thêm predicate thừa khi invariant được bảo đảm bằng schema/transaction hoặc hợp đồng nghiệp vụ đã kiểm chứng, kể cả dữ liệu lịch sử. User đổi org có thể làm ví dụ không còn tương đương. deleted_at/status thường là filter nghiệp vụ thật, không tự coi là “ma”. Tenant predicate phải lấy từ context tin cậy; index không bảo vệ phân quyền.

### Tìm kiếm theo vị trí: Khi hai điều kiện phạm vi đụng nhau

Bounding box theo longitude/latitude có hai range; B-tree ghép thường không thu hẹp cả hai hiệu quả như spatial index. Nếu workload thực sự spatial, thử GiST/R-tree/spatial index và toán tử phù hợp; bounding box chỉ là candidate, có thể cần kiểm tra hình học chính xác.

```sql
-- PostgreSQL: location là geometry với SRID 4326, type là scalar hỗ trợ btree_gist
CREATE EXTENSION IF NOT EXISTS postgis;
CREATE EXTENSION IF NOT EXISTS btree_gist;
CREATE INDEX search_idx ON businesses USING GIST (type, location);
SELECT * FROM businesses
WHERE type = 'restaurant'
  AND location && ST_MakeEnvelope(-74.0083, 40.7216, -73.9752, 40.7422, 4326);

-- SQL Server: giả định location kiểu geography; bảng có clustered primary key
CREATE SPATIAL INDEX search_idx ON businesses (location);
SELECT * FROM businesses
WHERE type = 'restaurant'
  AND location.STIntersects(geography::STGeomFromText(
    'POLYGON((-74.0083 40.7216,-73.9752 40.7216,-73.9752 40.7422,-74.0083 40.7422,-74.0083 40.7216))',
    4326)) = 1;

-- MySQL 8+: giả định location NOT NULL và có SRID xác định (ví dụ 4326)
CREATE SPATIAL INDEX search_idx ON businesses (location);
```

PostgreSQL GiST nhiều cột: scalar + geometry cần `btree_gist`. MySQL spatial index chỉ 1 cột. SQL Server: `geography` / `geometry` + spatial index riêng; filter `type` thường cần index B-tree riêng (spatial index không “gói” equality như GiST đa cột).

### Tìm kiếm ký tự đại diện ở đầu: Trường hợp đặc biệt

Prefix LIKE có thể tạo range theo điều kiện collation/operator class ở Phần 4. Suffix/substring thường không tạo seek trên B-tree gốc, nhưng covering scan hay filter bằng key khác vẫn có thể hữu ích. Suffix cố định còn có thể index REVERSE(name) rồi query prefix trên chuỗi đảo, nếu semantics phù hợp.

```sql
-- PostgreSQL: trigram (substring, ILIKE)
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX users_name_trgm ON users USING GIN (name gin_trgm_ops);
SELECT * FROM users WHERE name ILIKE '%nguyen%';

-- PostgreSQL: full-text (từ, ngôn ngữ, ranking — không phải substring thuần)
CREATE INDEX articles_fts ON articles USING GIN (
  to_tsvector('simple', COALESCE(title, '') || ' ' || COALESCE(body, ''))
);
SELECT * FROM articles
WHERE to_tsvector('simple', COALESCE(title, '') || ' ' || COALESCE(body, ''))
  @@ plainto_tsquery('simple', 'indexing btree');

-- SQL Server: Full-Text (không phải B-tree)
-- CREATE FULLTEXT CATALOG ft AS DEFAULT;
-- CREATE FULLTEXT INDEX ON users(name) KEY INDEX pk_users;
SELECT * FROM users WHERE CONTAINS(name, '"nguyen"');

-- MySQL: FULLTEXT theo token; muốn ngram phải chọn parser riêng, không có trong ví dụ này
ALTER TABLE users ADD FULLTEXT INDEX users_name_ft (name);
SELECT * FROM users WHERE MATCH(name) AGAINST ('nguyen' IN BOOLEAN MODE);
```

GIN trigram không cung cấp ORDER BY B-tree; FTS có semantics riêng. pg_trgm có hỗ trợ equality nhưng không thay UNIQUE/FK hay mọi workload B-tree. Search + sort thường cần sort ngoài; hai index riêng không bảo đảm dùng đồng thời và giữ order. [pg_trgm](https://www.postgresql.org/docs/18/pgtrgm.html).

### OR trên hai cột: Một index không đủ phễu

`WHERE email = @e OR phone = @p` thường cần access path cho từng cột, có thể dùng bitmap/index union/merge hoặc scan. OR là cách viết hợp lệ; thử UNION ALL nếu đo cho thấy có lợi, kèm guard để giữ đúng multiplicity và NULL semantics:

```sql
-- Câu gốc: optimizer có thể kết hợp index email và phone
SELECT * FROM users WHERE email = @e OR phone = @p;

-- Phương án để so plan, mỗi nhánh có thể dùng index riêng
SELECT * FROM users WHERE email = @e
UNION ALL
SELECT * FROM users WHERE phone = @p
  AND (@e IS NULL OR email IS DISTINCT FROM @e);
-- PostgreSQL / SQL Server 2022+; @e là placeholder ứng dụng.
-- SQL Server bản cũ: AND (@e IS NULL OR email <> @e OR email IS NULL)
-- MySQL: AND (? IS NULL OR NOT (email <=> ?))
-- Guard @e IS NULL cần thiết: phép email=@e ở nhánh đầu không match khi @e NULL.
```

`OR` trên **cùng** cột (`status IN (...)` / `status = 'a' OR status = 'b'`) vẫn là equality — nguyên tắc 1, không phải pattern này.

### Index phình to: Bloat, fragmentation, REINDEX

DELETE/UPDATE có thể để lại dead/ghost entries hoặc chỗ trống; engine có cơ chế cleanup và tái sử dụng, nhưng không bảo đảm trả ngay dung lượng về hệ điều hành. Phân biệt logical fragmentation, page density và bloat trước khi chọn bảo trì.

```sql
-- PostgreSQL: đo SIZE, không phải công thức đo bloat; pgstattuple/pgstatindex giúp khảo sát thêm
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid)) AS index_size, idx_scan
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

Chỉ rebuild/reorganize khi số liệu cho thấy lợi ích đủ bù chi phí I/O, log và blocking. Page density thấp còn ảnh hưởng cache/độ sâu, kể cả lookup. Không dùng ngưỡng fragmentation cố định cho mọi index; nhiều trường hợp cải thiện sau rebuild thực chất đến từ statistics mới. Fillfactor thấp hơn có thể giảm split nhưng tăng số page. [SQL Server: bảo trì theo workload và page density](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/reorganize-and-rebuild-indexes?view=sql-server-ver17).

## Phần 7. Kỹ thuật thao tác dữ liệu hiệu quả


### Tranh chấp khóa: Khi bộ đếm bị "nghẽn cổ chai"

Khi đo thấy một counter row là điểm tranh chấp, có thể phân tán thành nhiều bucket để update song song. Giả định PK/UNIQUE(post_id, fanout), likes_count kiểu BIGINT NOT NULL; số bucket phải được đo và tổng đọc sẽ tốn hơn. Retry cần idempotency để tránh cộng hai lần.

```sql
-- MySQL 8.0.19+ có row alias; VALUES() trong update deprecated từ 8.0.20
INSERT INTO post_statistics (post_id, fanout, likes_count)
VALUES (1475870220422107137, FLOOR(RAND() * 100), 1) AS new_row
ON DUPLICATE KEY UPDATE likes_count = likes_count + new_row.likes_count;

-- PostgreSQL
INSERT INTO post_statistics (post_id, fanout, likes_count)
VALUES (1475870220422107137, FLOOR(random() * 100)::int, 1)
ON CONFLICT (post_id, fanout)
DO UPDATE SET likes_count = post_statistics.likes_count + EXCLUDED.likes_count;

-- SQL Server
MERGE post_statistics WITH (HOLDLOCK) AS t
USING (SELECT CAST(1475870220422107137 AS bigint) AS post_id,
              ABS(CAST(CHECKSUM(NEWID()) AS bigint)) % 100 AS fanout,
              1 AS likes_count) AS s
ON t.post_id = s.post_id AND t.fanout = s.fanout
WHEN MATCHED THEN UPDATE SET likes_count = t.likes_count + 1
WHEN NOT MATCHED THEN INSERT (post_id, fanout, likes_count) VALUES (s.post_id, s.fanout, s.likes_count);

SELECT SUM(likes_count) FROM post_statistics WHERE post_id = 1475870220422107137;
```

SQL Server: cast trước ABS tránh overflow của INT_MIN. MERGE vẫn cần kiểm chứng concurrency/deadlock; xem mẫu transaction UPDATE/INSERT ở mục UPSERT. Không coi fanout là cách chữa mọi lock wait.


### Cập nhật dữ liệu từ bảng khác: JOIN trong UPDATE

Giả định categories.category_id là UNIQUE/PK, mỗi product match tối đa một source row. Nếu JOIN trả nhiều source cho một target, giá trị cập nhật có thể không xác định; aggregate hoặc loại trùng source theo quy tắc rõ ràng trước.

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

Dùng dữ liệu từ chính câu DML giúp tránh query lại và race. SQL Server OUTPUT có thể phát row dù statement bị lỗi/rollback; ứng dụng chỉ công bố kết quả sau khi xác nhận thành công/commit. Với enabled trigger, OUTPUT trực tiếp có hạn chế, có thể cần OUTPUT INTO. [OUTPUT clause](https://learn.microsoft.com/en-us/sql/t-sql/queries/output-clause-transact-sql?view=sql-server-ver17).

```sql
-- PostgreSQL
DELETE FROM sessions WHERE ip = '127.0.0.1'
RETURNING id, user_agent, last_access;

-- SQL Server
DELETE FROM sessions
OUTPUT deleted.id, deleted.user_agent, deleted.last_access
WHERE ip = '127.0.0.1';

-- MariaDB: RETURNING (INSERT/DELETE/REPLACE). MySQL 8.x và 9.x không có RETURNING.
```


### Xóa dòng trùng lặp: Dùng CTE thay vì xử lý ở tầng ứng dụng

```sql
-- PostgreSQL
WITH duplicates AS (
  SELECT id, ROW_NUMBER() OVER (
    PARTITION BY firstname, lastname, email
    ORDER BY age DESC, id DESC
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
    ORDER BY age DESC, id DESC
  ) AS rownum
  FROM contacts
)
DELETE c
FROM contacts c
INNER JOIN duplicates d ON c.id = d.id
WHERE d.rownum > 1;
```

Chọn row giữ lại theo ORDER BY có tie-breaker id; xác nhận NULL semantics của khóa trùng. Sau cleanup, thêm UNIQUE phù hợp và kiểm soát write đồng thời để dữ liệu không trùng trở lại.

### UPSERT: Giữ đúng dữ liệu khi có concurrency

SELECT rồi INSERT/UPDATE ở app có race khi nhiều request cùng chọn một key chưa tồn tại. Cần PK/UNIQUE trên khóa nghiệp vụ và cơ chế concurrency phù hợp; không suy ra “một statement luôn an toàn” cho mọi engine.

```sql
-- PostgreSQL
INSERT INTO settings (user_id, theme)
VALUES (42, 'dark')
ON CONFLICT (user_id)
DO UPDATE SET theme = EXCLUDED.theme, updated_at = now()
RETURNING *;

-- SQL Server: mẫu batch sở hữu transaction, giả định UNIQUE(user_id).
-- UPDLOCK + SERIALIZABLE giữ cả key/range chưa tồn tại đến hết transaction.
SET XACT_ABORT ON;
BEGIN TRY
  BEGIN TRAN;
  UPDATE settings WITH (UPDLOCK, SERIALIZABLE)
  SET theme = 'dark', updated_at = SYSUTCDATETIME()
  WHERE user_id = 42;

  IF @@ROWCOUNT = 0
    INSERT INTO settings (user_id, theme, updated_at)
    VALUES (42, 'dark', SYSUTCDATETIME());

  COMMIT;
END TRY
BEGIN CATCH
  IF XACT_STATE() <> 0 ROLLBACK;
  THROW;
END CATCH;

-- MySQL 8.0.19+ có row alias; VALUES() trong update deprecated từ 8.0.20
INSERT INTO settings (user_id, theme)
VALUES (42, 'dark') AS new_row
ON DUPLICATE KEY UPDATE theme = new_row.theme;
```

SQL Server MERGE là phương án khác, cần unique key, locking phù hợp và kiểm chứng trên bản cập nhật đang dùng; nó bắt buộc dấu chấm phẩy kết thúc. Transaction thường không có SERIALIZABLE/key-range protection vẫn có thể race với UPDATE-then-INSERT. Deadlock/serialization failure cần retry cả đơn vị transaction có kiểm soát. [MERGE concurrency](https://learn.microsoft.com/en-us/sql/t-sql/statements/merge-transact-sql?view=sql-server-ver17).

PostgreSQL ON CONFLICT cần arbiter unique phù hợp (partial index có điều kiện inference riêng). MySQL ON DUPLICATE KEY UPDATE có thể match bất kỳ UNIQUE key liên quan, không chỉ user_id; thiết kế nhiều unique key phải xét tác động này.

### Xóa / sửa theo lô: Đừng nuốt cả bảng trong một transaction

`DELETE FROM logs WHERE created_at < …` mười triệu row = WAL/log khổng lồ, lock lâu, replica tụt, `VACUUM` / index phình (Phần 6). Cắt lô nhỏ, commit giữa chừng:

```sql
-- PostgreSQL: commit từng lô, lặp đến khi không còn row. Dùng PK làm ID ổn định.
WITH batch AS (
  SELECT id FROM logs
  WHERE created_at < DATE '2024-01-01'
  ORDER BY created_at, id
  LIMIT 5000
)
DELETE FROM logs
USING batch
WHERE logs.id = batch.id
  AND logs.created_at < DATE '2024-01-01';  -- recheck nếu row đổi giữa hai bước

-- SQL Server
DELETE TOP (5000)
FROM logs
WHERE created_at < '20240101';

-- MySQL
DELETE FROM logs
WHERE created_at < '2024-01-01'
LIMIT 5000;
```

Ví dụ PostgreSQL phù hợp với index (created_at, id). TOP/LIMIT không ORDER BY chọn lô không xác định; khi cần thứ tự, dùng CTE chọn PK có ORDER BY rồi DELETE JOIN. Cân nhắc predicate retention ở DELETE cuối nếu row có thể đổi trong lúc chạy. Điều chỉnh batch size theo thời gian khóa/log/replication lag; xóa partition còn có điều kiện DDL và khóa (Phần 9).

`UPDATE` hàng loạt cùng bài: đừng set 5 triệu row một phát (lock + WAL + index maintain). Cột nằm trong index: mỗi lô còn đắt hơn (Phần 10, HOT).

### Nạp dữ liệu hàng loạt: COPY, BULK INSERT, SqlBulkCopy

`INSERT` từng row từ app = parse + plan + WAL + index cho mỗi câu. Nạp file / stream:

```sql
-- PostgreSQL (file trên server; từ client: \copy trong psql, hoặc Npgsql COPY)
COPY staging_users (email, name)
FROM '/tmp/users.csv' WITH (FORMAT csv, HEADER true);
-- UNLOGGED TABLE PostgreSQL không crash-safe và không được replicate sang standby.
-- PostgreSQL không có DISABLE INDEX; staging mới có thể nạp trước rồi CREATE INDEX.

-- SQL Server
BULK INSERT staging_users
FROM 'C:\tmp\users.csv'
WITH (FORMAT = 'CSV', FIRSTROW = 2, ROWTERMINATOR = '0x0a', TABLOCK);
-- FORMAT=CSV từ SQL Server 2017; đối chiếu encoding và line ending của file.

-- MySQL
LOAD DATA LOCAL INFILE '/tmp/users.csv'
INTO TABLE staging_users
FIELDS TERMINATED BY ',' IGNORE 1 LINES (email, name);
```

App: PostgreSQL COPY FROM STDIN; SQL Server SqlBulkCopy với batch/transaction phù hợp. TABLOCK ảnh hưởng concurrency và không tự bảo đảm minimal logging; còn phụ thuộc recovery model, bảng và index. SqlBulkCopy có tùy chọn kiểm tra constraint/trigger, cần chọn theo nghiệp vụ. LOAD DATA LOCAL phụ thuộc cấu hình client/server. Load xong kiểm tra chất lượng dữ liệu, constraint và cập nhật statistics khi cần.

## Phần 8. Viết query như chuyên gia


### Phân trang đúng cách: Phân trang theo khóa

- Thêm PK vào `ORDER BY` để thứ tự ổn định
- Keyset pagination thay vì `OFFSET` lớn:

```sql
-- PostgreSQL / MySQL: row constructor (cú pháp này không bắt đầu từ 8.0.13)
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

Index đề xuất: (firstname, lastname, id), với cùng collation và chiều ORDER BY. Tuple comparison gọn trên PostgreSQL; MySQL có thể không tận dụng hết key trong một số query, cần so với predicate bậc thang. [MySQL row-constructor optimization](https://dev.mysql.com/doc/refman/8.4/en/row-constructor-optimization.html).

Cursor phải mang **toàn bộ sort key**, tenant/filter tương ứng. Với DESC dùng dấu <; mixed ASC/DESC cần bậc thang theo từng chiều. Nếu sort key nullable, định nghĩa NULL ordering và normalize hoặc xử lý riêng thay vì dùng tuple comparison rồi bỏ sót NULL.

Keyset giảm công việc OFFSET sâu, không tự cung cấp snapshot nhất quán qua nhiều request. UPDATE sort key hoặc dữ liệu mới có thể gây miss/repeat; nếu cần ảnh chụp cố định, dùng snapshot/version/as-of phù hợp. Cursor nên được validate và giữ filter cố định.

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

FOR UPDATE/UPDLOCK giữ khóa đến hết transaction. Index account_id giảm công việc tìm và phạm vi khóa; engine vẫn có thể dùng key/page/table/gap locks tùy isolation và lock escalation. Khóa theo cùng thứ tự, giữ transaction ngắn, retry deadlock có kiểm soát.

Nếu chỉ trừ tiền, thường dùng một statement atomic: UPDATE account SET balance=balance-@amount WHERE account_id=@id AND balance>=@amount, rồi kiểm tra row count. Trường hợp đọc-tính-ghi từ app còn có thể dùng optimistic concurrency/version token. Lock không thay kiểm tra nghiệp vụ.

### SKIP LOCKED: Claim công việc trong transaction ngắn

SKIP LOCKED/READPAST giúp worker bỏ qua row đang khóa. Phải **đổi trạng thái claim trong cùng transaction**, rồi commit trước khi xử lý tác vụ dài; SELECT rồi COMMIT mà vẫn để pending có thể giao cùng job cho worker kế tiếp.

```sql
-- PostgreSQL: một statement atomic; mặc định autocommit khi không có transaction ngoài
WITH claim AS (
  SELECT id
  FROM job_queue
  WHERE status = 'pending'
  ORDER BY id
  LIMIT 1
  FOR UPDATE SKIP LOCKED
)
UPDATE job_queue AS j
SET status = 'processing'
FROM claim
WHERE j.id = claim.id
RETURNING j.id, j.payload;

-- SQL Server: READ COMMITTED, dùng locking cả khi database bật RCSI.
-- READCOMMITTEDLOCK và ROWLOCK thuộc cùng nhóm hint, không ghép cả hai.
;WITH claim AS (
  SELECT TOP (1) id, payload, status
  FROM job_queue WITH (UPDLOCK, READPAST, READCOMMITTEDLOCK)
  WHERE status = 'pending'
  ORDER BY id
)
UPDATE claim SET status = 'processing'
OUTPUT inserted.id, inserted.payload;

-- MySQL/InnoDB 8.0: giữ transaction trong bước chọn + đổi trạng thái
START TRANSACTION;
SET @job_id = NULL;
SELECT id INTO @job_id
FROM job_queue
WHERE status = 'pending'
ORDER BY id
LIMIT 1
FOR UPDATE SKIP LOCKED;
UPDATE job_queue SET status = 'processing' WHERE id = @job_id;
SELECT id, payload FROM job_queue WHERE id = @job_id;
COMMIT;
```

Giả định id là PK; index (status, id) hoặc partial/filtered pending phù hợp. Chỉ xử lý payload sau khi biết claim đã commit; nếu có transaction ngoài thì phải chờ commit ngoài. SQL Server READPAST không bỏ page lock, và vẫn có thể block ở FK/index/metadata; ROWLOCK cũng không bảo đảm không escalation.

Production cần thêm lease/locked_until, claim token và retry khi worker chết; UPDATE hoàn tất phải kiểm tra token để worker cũ không ghi đè claim mới. Side effect cần idempotency. SKIP LOCKED không bảo đảm FIFO tuyệt đối hay exactly-once và không dùng cho báo cáo cần đọc đủ row. [PostgreSQL locking clause](https://www.postgresql.org/docs/18/sql-select.html#SQL-FOR-UPDATE-SHARE), [SQL Server READPAST và RCSI](https://learn.microsoft.com/en-us/sql/t-sql/queries/hints-transact-sql-table?view=sql-server-ver17#readpast).

### EXISTS và NOT EXISTS: Semi-join / anti-join

EXISTS chỉ yêu cầu biết có row match. IN có thể đúng cho semi-join; bẫy NULL chủ yếu cần chú ý với NOT IN: nếu không có match nhưng subquery chứa NULL, kết quả có thể UNKNOWN và row bị loại. NOT EXISTS thể hiện rõ anti-join theo điều kiện tương quan.

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

EXISTS diễn đạt ý định tốt hơn COUNT(*)>0; optimizer có thể biến đổi cả hai trong một số trường hợp. Semi/anti join có thể dùng nested loop, hash hoặc merge. Hash đọc nhiều orders có thể hợp lý khi tập khách lớn, không tự chứng minh thiếu index.

### LATERAL / CROSS APPLY: Top-N mỗi nhóm

ROW_NUMBER rồi lọc rn<=3 có thể scan input đã ordered hoặc cần sort; không luôn sort cả orders. LATERAL/APPLY có thể seek top-N theo từng customer với index (customer_id, created_at DESC, id DESC); thêm id làm tie-breaker.

```sql
-- PostgreSQL
SELECT c.id, o.id AS order_id, o.created_at
FROM customers c
CROSS JOIN LATERAL (
  SELECT id, created_at
  FROM orders
  WHERE customer_id = c.id
  ORDER BY created_at DESC, id DESC
  LIMIT 3
) o;

-- SQL Server
SELECT c.id, o.id AS order_id, o.created_at
FROM customers c
CROSS APPLY (
  SELECT TOP (3) id, created_at
  FROM orders
  WHERE customer_id = c.id
  ORDER BY created_at DESC, id DESC
) o;
```

So tổng công việc giữa top-N seek từng nhóm và ordered scan/window. Số nhóm, mật độ đơn mỗi nhóm, covering, cache và filter quyết định lựa chọn; không có ngưỡng nhóm cố định. CROSS JOIN LATERAL/CROSS APPLY bỏ khách không có đơn; LEFT JOIN LATERAL/OUTER APPLY giữ khách đó với cột đơn NULL.

### Biểu thức bảng tạm (CTE): Xử lý query phức tạp

Chia query thành bước nhỏ, mỗi CTE test độc lập — dễ debug hơn subquery lồng.

CTE là biểu thức query có tên, không phải bảng tạm có thể tự CREATE INDEX. PostgreSQL **đến 11** thường materialize CTE như optimization fence; **12+** có thể inline CTE SELECT không recursive, không volatile, thường khi được tham chiếu một lần. MATERIALIZED/NOT MATERIALIZED điều chỉnh lựa chọn khi áp dụng được; materialization cũng có thể spill ra đĩa. SQL Server không bảo đảm cache kết quả CTE, optimizer có thể dùng spool. [PostgreSQL 11 WITH](https://www.postgresql.org/docs/11/queries-with.html), [PostgreSQL 18 WITH](https://www.postgresql.org/docs/18/queries-with.html).

Cần index/thống kê trên tập trung gian thì dùng [bảng tạm và bảng biến](#bảng-tạm-temp-và-bảng-biến-table-sql-server).

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
  WHERE tree.depth < 20          -- giới hạn độ sâu, không thực sự phát hiện chu trình
)
SELECT * FROM tree;

-- SQL Server: WITH tree AS (anchor UNION ALL recursive) — không ghi RECURSIVE
```

Với graph có chu trình, dùng visited path/CYCLE (PostgreSQL 14+) hoặc logic phát hiện phù hợp; depth limit có thể cắt cả dữ liệu hợp lệ. SQL Server có MAXRECURSION. Materialized path hữu ích cho cây đọc nhiều nhưng có chi phí cập nhật khi chuyển nhánh.

### Bảng tạm `#temp` và bảng biến `@table` (SQL Server)

CTE không nhận `CREATE INDEX`. Khi tập trung gian của SQL Server cần index, dùng bảng tạm hoặc bảng biến. Cả hai nằm trong **tempdb** (bảng biến không "chỉ ở RAM").

| | `#orders` bảng tạm | `@orders` bảng biến |
| --- | --- | --- |
| Index | Clustered và nonclustered, thêm lúc nào cũng được | Chỉ khai báo trong `DECLARE` (SQL Server 2014+). Không `CREATE INDEX` sau |
| Thống kê | Histogram, tự cập nhật, có thể recompile | Không histogram. Compatibility 150+ (SQL Server 2019) biết **số dòng** lúc biên dịch lần đầu, vẫn không biết phân bố cột |
| Transaction | `ROLLBACK` hủy luôn dòng đã ghi | `ROLLBACK` **không** xóa dữ liệu trong `@table` |
| Phạm vi | Session; procedure con nhìn thấy | Batch / procedure hiện tại; procedure con không thấy |
| DDL | `CREATE INDEX`, `ALTER`, `TRUNCATE` | Không `ALTER`, không `TRUNCATE` (dùng `DELETE`) |

```sql
-- Bảng tạm: clustered (PK) + nonclustered
CREATE TABLE #orders (
  id bigint NOT NULL PRIMARY KEY,       -- clustered
  customer_id int NOT NULL,
  created_at datetime2 NOT NULL
);
CREATE INDEX ix_orders_customer
  ON #orders (customer_id, created_at DESC);

-- Có thể nạp trước rồi CREATE INDEX để build một lần, hoặc tạo trước nếu
-- index/constraint giúp load và query; bảng #orders trên đã có clustered PK.

-- Bảng biến: index phải nằm trong DECLARE
DECLARE @orders TABLE (
  id bigint NOT NULL PRIMARY KEY CLUSTERED,
  customer_id int NOT NULL,
  created_at datetime2 NOT NULL,
  INDEX ix_orders_customer NONCLUSTERED (customer_id, created_at DESC)
);
```

#temp thường phù hợp khi cần histogram/index linh hoạt hoặc nhiều phép join. @table phù hợp khi scope/chi phí compile đơn giản và plan vẫn tốt. Không có ngưỡng số row cố định; deferred compilation biết row count lần compile nhưng plan tái dùng vẫn có thể lệch ở lần sau. So plan thực tế trước khi đổi.

`##orders` là bảng tạm global: session khác thấy được, xóa khi session tạo nó kết thúc và không còn ai dùng. Hiếm khi cần.

Sort spill / Index Spool trong plan cũng dùng `tempdb`, nhưng đó là cấu trúc optimizer tự dựng cho một câu query. Không phải `#temp` hay `@table` của bạn.

PostgreSQL không có bảng biến. Tập trung gian có index: `CREATE TEMP TABLE` rồi `CREATE INDEX`, và `ANALYZE` bảng đó (autovacuum không đụng temp table).

### Các tips query hữu ích khác

```sql
-- Tránh division by zero
SELECT visitors_today / NULLIF(visitors_yesterday, 0) FROM stats;

-- Gap-filling (PostgreSQL)
SELECT dates.day, COALESCE(SUM(stats.count), 0)
FROM generate_series(CURRENT_DATE - INTERVAL '14 days', CURRENT_DATE, '1 day') AS dates(day)
LEFT JOIN statistics stats ON stats.day = dates.day
GROUP BY dates.day;

-- SQL Server 2022+, compatibility level 160: GENERATE_SERIES chỉ nhận số.
-- Sinh offset số rồi DATEADD, giả định statistics.day kiểu DATE.
DECLARE @today date = CAST(SYSUTCDATETIME() AS date);
SELECT d.day, COALESCE(SUM(stats.[count]), 0) AS total_count
FROM GENERATE_SERIES(0, 14, 1) AS g
CROSS APPLY (VALUES (DATEADD(day, g.value - 14, @today))) AS d(day)
LEFT JOIN statistics AS stats ON stats.day = d.day
GROUP BY d.day
ORDER BY d.day;

-- Multiple aggregates
-- PostgreSQL:
SELECT
  COUNT(*) FILTER (WHERE release_year = 2024) AS released_2024,
  COUNT(*) FILTER (WHERE director = 'Nolan') AS nolan_movies
FROM movies;
-- MySQL:
SELECT
  COALESCE(SUM(release_year = 2024), 0) AS released_2024,
  COALESCE(SUM(director = 'Nolan'), 0) AS nolan_movies
FROM movies;
-- SQL Server:
SELECT
  COUNT_BIG(CASE WHEN release_year = 2024 THEN 1 END) AS released_2024,
  COUNT_BIG(CASE WHEN director = 'Nolan' THEN 1 END) AS nolan_movies
FROM movies;

-- DISTINCT ON (PostgreSQL): đơn đắt nhất mỗi customer
SELECT DISTINCT ON (customer_id) *
FROM orders
WHERE created_at >= TIMESTAMP '2024-01-01'
  AND created_at <  TIMESTAMP '2025-01-01'
ORDER BY customer_id ASC, price DESC, id DESC;
-- (tránh EXTRACT(YEAR FROM created_at) / YEAR(created_at) — hàm trên cột = không SARGable)

-- SQL Server / MySQL: cùng ý tưởng bằng ROW_NUMBER()
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY price DESC, id DESC) AS rn
  FROM orders
  WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
) t WHERE rn = 1;
```

GENERATE_SERIES của SQL Server tạo cột value kiểu số, không nhận DATE/TIMESTAMP như PostgreSQL. Ví dụ aggregate dùng release_year kiểu số; nếu released_at là thời điểm, dùng range thời gian đúng ngữ nghĩa. [GENERATE_SERIES](https://learn.microsoft.com/en-us/sql/t-sql/functions/generate-series-transact-sql?view=sql-server-ver17).

### N+1: Vòng lặp query vs một câu SQL

ORM/`foreach` khách rồi `SELECT * FROM orders WHERE customer_id = @id` = N+1 round-trip. Index `(customer_id)` **không** cứu latency mạng.

```sql
-- 1 round-trip: IN / JOIN / LATERAL (Phần 8 trên)
SELECT * FROM orders
WHERE customer_id = ANY(@ids);          -- PostgreSQL
-- SQL Server: WHERE customer_id IN (SELECT id FROM @ids)

-- Chi tiết sau khi có id: hai bước (covering hẹp rồi hydrate)
SELECT id FROM orders WHERE customer_id = @c AND created_at >= @from
ORDER BY created_at DESC, id DESC LIMIT 50;
SELECT * FROM orders WHERE id = ANY(@page_ids);
```

Bước 1 có thể dùng index (customer_id, created_at DESC, id DESC); INCLUDE payload theo nhu cầu. Bước 2 lấy theo ID **không giữ thứ tự trang** nếu thiếu ORDER BY/ordinal: ứng dụng phải ghép lại đúng thứ tự. Hai query cũng có thể thấy dữ liệu khác nhau khi update/delete giữa hai lượt; dùng isolation phù hợp nếu cần nhất quán. Với chỉ 50 row, một SELECT trực tiếp + lookup thường đã đủ tốt, nên đo trước khi tách hai bước.

### Optional filter: `OR col IS NULL` / `@p IS NULL OR col = @p`

Form search “mọi field tùy chọn”:

```sql
-- Catch-all filter: plan tái dùng có thể scan; không phải luôn mất seek
WHERE (@status IS NULL OR status = @status)
  AND (@from  IS NULL OR created_at >= @from);
```

SQL Server hay scan; PostgreSQL generic plan cũng dễ bỏ index. Cách làm:

1. **Dynamic SQL** / query builder: chỉ `AND` khi user chọn filter — mỗi combo một plan, SARGable.
2. SQL Server: `OPTION (RECOMPILE)` cho báo cáo thưa (compile mỗi lần).
3. Hai query: “có filter” vs “không filter”, không nhét một câu catch-all.

SQL Server 2025 với compatibility level 170 và OPTIONAL_PARAMETER_OPTIMIZATION bật có **OPPO** cho các optional predicate đủ điều kiện: có thể tách plan khi parameter NULL/non-NULL. PSP và OPPO giải quyết các tình huống khác nhau; xác minh query có được tối ưu không. Dynamic SQL cần parameterize giá trị, chỉ ghép cấu trúc đã kiểm soát. [OPPO](https://learn.microsoft.com/en-us/sql/relational-databases/performance/optional-parameter-optimization?view=sql-server-ver17).

COALESCE(@status,status)=status không luôn tương đương catch-all: khi @status NULL, row status NULL có NULL=NULL (UNKNOWN) nên bị loại. Không đổi dạng chỉ vì nghĩ nhanh hơn.

### LEFT JOIN … IS NULL: Anti-join dễ viết sai

`NOT EXISTS` (trên) là anti-join chuẩn. Người ta hay viết:

```sql
SELECT c.id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.id IS NULL;
```

Phải kiểm tra cột phía phải được bảo đảm NOT NULL khi row match (thường PK o.id), nếu không row có đơn với cột kiểm tra NULL có thể bị nhận nhầm là không có đơn. Filter thời gian đặt ở ON để nghĩa là “không có đơn trong khoảng đó”; đặt ở WHERE sẽ loại null-extended row. Anti join có thể dùng hash/merge/loops theo số liệu, không suy ra thiếu FK index chỉ từ Hash.

### COUNT lớn: Đừng đếm cả bảng mỗi request

COUNT(*) chính xác thường cần đọc các row/entry phù hợp với snapshot trên PostgreSQL, SQL Server và InnoDB, có thể scan index hẹp. MyISAM là ví dụ có metadata count cho một số query không WHERE; không áp kết luận này cho mọi engine. Dashboard cần quyết định độ chính xác/độ mới thay vì đếm lại toàn bộ mỗi request.

- Ước lượng: PostgreSQL `reltuples` (`pg_class`); SQL Server `sys.dm_db_partition_stats` / `sp_spaceused`.
- Đếm theo điều kiện hẹp: index khớp predicate, giảm dữ liệu phải đọc; COUNT(*) không kéo mọi cột bảng. SQL Server dùng COUNT_BIG khi số row có thể vượt giới hạn INT.
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

Keyset theo thời gian cần cursor (created_at,id) và comparator theo chiều ORDER BY. OFFSET không mặc nhiên sai; nó đắt khi sâu và có thể lệch khi dữ liệu đổi giữa các request. Ưu tiên range trực tiếp thay hàm trên cột; một số cast SQL Server có dynamic seek nên vẫn cần xem plan. Timezone: Phần 9.

## Phần 9. Thiết kế Schema: Nền móng vững chắc


### UUID vs Auto-increment: Lựa chọn Primary Key


| Tiêu chí           | Auto-increment               | UUIDv4                 | UUIDv7/ULID         |
| ------------------ | ---------------------------- | ---------------------- | ------------------- |
| Insert locality | Thường tốt; có thể tranh chấp page cuối | Phân tán | Có thể tốt nếu comparator/storage phù hợp |
| Size | 4–8 byte cho INT/BIGINT | 16 byte native/binary | 16 byte binary; dạng text rộng hơn |
| Dễ suy đoán / lộ thời gian | Dễ đoán thứ tự | Có entropy khi dùng generator đúng | Có timestamp; không phải secret |
| Sinh ID phân tán | Cần cấp dải/sequence hoặc phối hợp | Có thể sinh độc lập | Có thể sinh độc lập, cần xử lý clock/monotonicity |


Một phương án phổ biến là BIGINT cho clustered/internal key và unique UUID cho public ID. Nó thêm index nên cần đo tác động ghi. UUIDv4/v7 đều có thể làm PK hợp lệ; ID khó đoán **không thay authorization**. Không dùng ID có timestamp hay NEWSEQUENTIALID như secret/token bảo mật.

```sql
-- PostgreSQL 18+: uuidv7() time-ordered. gen_random_uuid() / uuidv4() = random
ALTER TABLE users ADD COLUMN uuid UUID NOT NULL DEFAULT uuidv7();
CREATE UNIQUE INDEX users_uuid ON users (uuid);

-- SQL Server: public ID random; clustered key hẹp riêng nếu cần locality
ALTER TABLE users ADD uuid UNIQUEIDENTIFIER NOT NULL DEFAULT NEWID();
CREATE UNIQUE INDEX users_uuid ON users (uuid);

-- MySQL: UUID() là v1, không phải UUIDv7 hay token bí mật.
-- Nếu sinh UUIDv7 ở app: UUID_TO_BIN(@uuid_v7, 0); không dùng swap=1 cho v7.
ALTER TABLE users ADD COLUMN uuid BINARY(16) NOT NULL DEFAULT (UUID_TO_BIN(UUID(), 1));
CREATE UNIQUE INDEX users_uuid ON users (uuid);
```


DEFAULT/backfill khi thêm cột lên bảng lớn có thể tốn thời gian và khóa; cân nhắc migration theo giai đoạn. NEWSEQUENTIALID chỉ gọi trong DEFAULT, có đặc điểm suy đoán và restart/failover riêng. [UUID PostgreSQL 18](https://www.postgresql.org/docs/18/functions-uuid.html), [NEWSEQUENTIALID](https://learn.microsoft.com/en-us/sql/t-sql/functions/newsequentialid-transact-sql?view=sql-server-ver17).

UUID_TO_BIN(...,1) dành cho cách bố trí thời gian của UUIDv1; đổi swap flag khi lưu/đọc sẽ làm sai giá trị UUID. [MySQL UUID_TO_BIN](https://dev.mysql.com/doc/refman/8.4/en/miscellaneous-functions.html#function_uuid-to-bin).

### JSON Column: Khi NoSQL gặp SQL

Dùng JSON khi: metadata/settings ít query; thay EAV; giảm JOIN cho data seldom-used.

Vẫn dùng relational cho data chính. Tránh deeply nested JSON. Đừng lưu FK trong JSON.

```sql
-- MySQL 8.0.17+
ALTER TABLE products ADD CONSTRAINT attributes_schema CHECK (
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

CHECK chỉ từ chối FALSE; UNKNOWN do NULL thường vẫn vượt qua. Các cột bắt buộc cần NOT NULL riêng. Ví dụ dưới giả định checkin_at, checkout_at, is_eu đã NOT NULL.

```sql
-- PostgreSQL, is_eu kiểu BOOLEAN
ALTER TABLE reservations
ADD CONSTRAINT start_before_end CHECK (checkin_at < checkout_at);

ALTER TABLE invoices
ADD CONSTRAINT eu_vat CHECK (NOT is_eu OR vatid IS NOT NULL);

-- SQL Server, is_eu kiểu BIT NOT NULL: phương án thay thế cho CHECK phía trên
-- ALTER TABLE invoices ADD CONSTRAINT eu_vat CHECK (is_eu = 0 OR vatid IS NOT NULL);
```

Constraint được bật và enforce bảo vệ dữ liệu dù writer không đi qua ứng dụng. Kiểm tra constraint bị disable/not validated/untrusted, và validate dữ liệu cũ khi bật lại. MySQL chỉ enforce CHECK từ 8.0.16; NOCHECK trên SQL Server có thể làm constraint không trusted. CHECK trên JSON cũng không tự cấm SQL NULL. [PostgreSQL CHECK/NULL](https://www.postgresql.org/docs/18/ddl-constraints.html).

### Ràng buộc loại trừ: Chống chồng chéo (PostgreSQL)

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;  -- operator class GiST cho INT =
CREATE TABLE bookings (
  room_number INT NOT NULL,
  reservation TSTZRANGE NOT NULL,
  CHECK (NOT isempty(reservation)),
  EXCLUDE USING GIST (room_number WITH =, reservation WITH &&)
);
-- Dùng khoảng [checkin, checkout) để hai booking nối tiếp không bị overlap.
```


GiST ở đây kiểm tra đồng thời room_number bằng nhau và reservation overlap. NOT NULL/không rỗng tránh lỗ hổng nghiệp vụ; nếu không cho thời gian vô hạn, thêm kiểm tra bounds. [btree_gist: exclusion constraints](https://www.postgresql.org/docs/18/btree-gist.html).

### Đường dẫn vật lý hóa: Lưu trữ cây đơn giản

```sql
-- PostgreSQL
CREATE EXTENSION IF NOT EXISTS ltree;
CREATE TABLE categories (path LTREE);
INSERT INTO categories VALUES ('Food'), ('Food.Fruit'), ('Food.Fruit.Cherry');
SELECT * FROM categories WHERE path ~ 'Food.Fruit.*{1,}';
CREATE INDEX categories_path_gist ON categories USING GIST (path);

-- SQL Server: hierarchyid
CREATE TABLE categories (path HIERARCHYID PRIMARY KEY);
-- GetDescendant / IsDescendantOf để duyệt cây
```


### Partition: Retention và pruning có điều kiện

Partition giúp retention theo một tập row đã tách sẵn và partition pruning khi predicate khớp partition key. Nó không thay index bên trong partition hay tự tăng tốc mọi query. Drop/detach/switch thường ít log hơn DELETE từng row nhưng vẫn có DDL/metadata lock và dependency.

```sql
-- MySQL
ALTER TABLE logs DROP PARTITION logs_2023_january;

-- PostgreSQL
DROP TABLE logs_2023_january;  -- hoặc DETACH PARTITION rồi DROP

-- SQL Server
TRUNCATE TABLE logs WITH (PARTITIONS (1));  -- cần index alignment và điều kiện hỗ trợ
```

PostgreSQL DROP partition trực tiếp có thể cần khóa mạnh trên parent; DETACH PARTITION CONCURRENTLY có điều kiện và không chạy trong transaction block. PK/UNIQUE trên bảng partitioned PostgreSQL phải bao gồm partition key; MySQL cũng có hạn chế unique key với partition expression. SQL Server SWITCH cần schema/index/constraint tương thích. Xác minh toàn bộ dữ liệu trong partition đã hết retention, không chỉ tên partition. [PostgreSQL partitioning](https://www.postgresql.org/docs/18/ddl-partitioning.html).


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

-- SQL Server: clustered index giữ thứ tự lá theo key (mặc định).
-- Page split làm các page không nằm liền trên đĩa — xem Bonus.
-- CREATE CLUSTERED INDEX ... ON product_comments (product_id, comment_id);
```


### Tính toán trước: Khi index cũng không đủ nhanh

Lưu sẵn aggregate khi đọc lặp lại tốn quá nhiều công việc. Xác định độ mới cho phép, writer chịu trách nhiệm cập nhật, cơ chế idempotency/rebuild và cách sửa sai khi event bị lặp/miss:

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
INCLUDE (deleted_at)
WHERE deleted_at IS NULL;
```

Query phải có predicate để optimizer chứng minh chỉ cần row active. SQL Server có hạn chế đã biết với filtered IS NULL khi cột kiểm tra không nằm trong index, nên ví dụ INCLUDE(deleted_at). [Microsoft: filtered IS NULL index](https://learn.microsoft.com/en-us/troubleshoot/sql/database-engine/performance/filtered-index-with-column-is-null).

Unique(email) WHERE deleted_at IS NULL cho phép tái dùng email đã xóa mềm. Với multi-tenant, unique key thường là (tenant_id,email). MySQL không có filtered index: một phương án là generated active_email = CASE WHEN deleted_at IS NULL THEN email ELSE NULL END rồi UNIQUE(tenant_id,active_email), với email active NOT NULL và collation đúng yêu cầu. Restore có thể thất bại nếu email đã được tái dùng.

Thay deleted_at làm thay đổi membership của partial index, nên vẫn có chi phí ghi và ảnh hưởng HOT PostgreSQL; không né chi phí đó chỉ bằng việc để nó ngoài key.

### Khóa clustered hẹp: Đừng nhét UUID vào mọi secondary

SQL Server và InnoDB: mọi nonclustered index **mang theo clustered key** (thường là PK) ở leaf. PK `UNIQUEIDENTIFIER` / `CHAR(36)` 16–36 byte → nhân với số index → leaf phình, cache miss.

| PK clustered | Secondary `(email)` leaf chứa |
| ------------ | ----------------------------- |
| `INT` / `BIGINT` identity | email + 4–8 byte |
| UUIDv4 | email + 16 byte (+ fragmentation, Phần 1) |

PK/clustered key hẹp, ít thay đổi và có locality tốt là một phương án cho nhiều workload. UUID/ULID có thể là unique key riêng cho API (đầu Phần 9), hoặc chính PK khi yêu cầu phân tán phù hợp. PostgreSQL không tự mang PK vào mọi secondary locator; độ rộng PK vẫn ảnh hưởng heap, PK index và các FK lưu giá trị đó.

### Multi-tenant: `tenant_id` đứng đầu schema

Nếu query chủ yếu theo tenant, tenant_id ở đầu các index tương ứng thường hữu ích. Không bắt buộc mọi PK/index đều có prefix này: admin/report xuyên tenant có access pattern khác. FK nhiều cột giúp bảo đảm child không tham chiếu dữ liệu của tenant khác.

```sql
-- PostgreSQL: hai bảng dùng khóa tenant + ID, ngăn tham chiếu chéo tenant
CREATE TABLE customers (
  tenant_id INT NOT NULL,
  id BIGINT GENERATED ALWAYS AS IDENTITY,
  PRIMARY KEY (tenant_id, id)
);
CREATE TABLE orders (
  tenant_id   INT NOT NULL,
  id          BIGINT GENERATED ALWAYS AS IDENTITY,
  customer_id BIGINT NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (tenant_id, id),
  FOREIGN KEY (tenant_id, customer_id) REFERENCES customers (tenant_id, id)
);

CREATE INDEX orders_customer
ON orders (tenant_id, customer_id, created_at DESC);
```

Thiếu tenant prefix có thể làm đọc nhiều rồi filter, nhưng nếu customer_id đã unique toàn cục thì lookup vẫn có thể rất hẹp. PostgreSQL và SQL Server đều có **Row-Level Security** native; phân quyền và index là hai trách nhiệm riêng. RLS không tự tạo index, và index không ngăn truy cập chéo tenant. PostgreSQL current_setting trả TEXT nên cần cast đúng kiểu tenant_id; context/policy và quyền bypass phải được thiết kế cẩn thận.

[PostgreSQL row security](https://www.postgresql.org/docs/18/ddl-rowsecurity.html), [SQL Server RLS](https://learn.microsoft.com/en-us/sql/relational-databases/security/row-level-security?view=sql-server-ver17).

### Status: ENUM, lookup, hay VARCHAR

- **VARCHAR + CHECK**: dễ đọc, nhưng thay danh sách giá trị vẫn có thể cần DDL lock/validation và migration dữ liệu. Index B-tree bình thường; xem phân bố từng giá trị.
- **Lookup table** (`status_id` FK): thêm trạng thái không `ALTER TYPE`. Index FK (Phần 10). JOIN nhỏ, covering được.
- **ENUM PostgreSQL**: có thể ADD VALUE trong transaction ở bản hiện đại, nhưng không dùng giá trị mới trước khi transaction commit. Có RENAME VALUE; xóa/sắp lại label thường cần đổi type/migration. MySQL ENUM dùng ordinal cho thứ tự, khác thứ tự từ điển của chuỗi. [PostgreSQL ALTER TYPE](https://www.postgresql.org/docs/18/sql-altertype.html).

Nếu query lấy nhiều cột của một trạng thái rất phổ biến, index status đơn có thể không có lợi. Tuy vậy index hẹp có thể giúp count/group hoặc tìm giá trị hiếm. So với (tenant_id,status,created_at) hoặc partial index theo workload, không chỉ tỷ lệ open toàn bảng.

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

Surrogate id vẫn hợp lệ nếu cần identity ổn định cho relationship, công cụ/ORM hoặc các thuộc tính riêng; phải giữ UNIQUE(user_id,role_id) khi nghiệp vụ không cho trùng. Composite PK là phương án gọn cho bảng nối thuần. Index ngược giúp query theo role, nhưng có cần hay không tùy workload và constraint.

### Thời gian: Chọn kiểu theo mốc thời điểm và lịch địa phương

Phân biệt **mốc thời điểm tuyệt đối** và **giờ/ngày theo lịch địa phương**. PostgreSQL timestamp without time zone không mang timezone; DATETIME2 SQL Server có thể lưu UTC theo hợp đồng ứng dụng. SQL Server TIMESTAMP lại là ROWVERSION, **không phải kiểu thời gian**. Lẫn kiểu/session timezone dễ khiến range lọc sai.

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

PostgreSQL TIMESTAMPTZ lưu mốc thời điểm, không giữ tên timezone hay offset gốc; nếu cần lịch theo vùng, lưu thêm zone ID. DATETIMEOFFSET giữ offset, không giữ đầy đủ quy tắc timezone/DST. MySQL TIMESTAMP chuyển theo session time_zone và có giới hạn phạm vi khác DATETIME.

Ngày sinh/ngày hóa đơn dùng DATE; giờ hẹn theo lịch địa phương có thể cần local datetime + zone ID. Tính cận [from,to) trong timezone nghiệp vụ rồi bind tham số đúng kiểu, tránh biến đổi cột đã index. [PostgreSQL date/time types](https://www.postgresql.org/docs/18/datatype-datetime.html).

## Phần 10. Checklist & lỗ hổng hay quên

Những điểm dưới đây hay bị bỏ khi tối ưu index — không thay thế các phần trên, chỉ là lưới an toàn trước khi merge.

### Kiểm tra index phía tham chiếu của khóa ngoại

PK/UNIQUE của bảng parent thường đã có index. Cột FK của child là chuyện khác: PostgreSQL và SQL Server **không tự tạo** index này; MySQL/InnoDB yêu cầu index phù hợp và tự tạo nếu thiếu.

```sql
-- orders.customer_id REFERENCES customers(id)
CREATE INDEX orders_customer_id ON orders (customer_id);
-- Nếu đã có (customer_id, created_at DESC), thường không cần thêm index đơn chỉ vì FK.
```

Index child hỗ trợ join và kiểm tra/cascade khi xóa hoặc sửa parent key, giúp giảm scan/blocking. Với bảng nhỏ hay FK không đi qua workload đó, đo lợi ích thay vì coi đây là luật tuyệt đối. FK ghép cần xét toàn bộ các cột; MySQL có yêu cầu thứ tự prefix cụ thể.

[PostgreSQL FK indexing](https://www.postgresql.org/docs/18/ddl-constraints.html#DDL-CONSTRAINTS-FK), [InnoDB FK indexes](https://dev.mysql.com/doc/refman/8.4/en/create-table-foreign-keys.html).

### Đừng `SELECT *` nếu muốn covering

Muốn lấy payload từ index, các cột cần cho SELECT/join/filter/order phải có trong access path hoặc được suy ra từ predicate/constraint phù hợp. SELECT * dễ làm index không covering và tăng network/materialization; lấy đúng cột cần. COUNT(*) không tương đương SELECT * và không yêu cầu mọi cột bảng. PostgreSQL còn có visibility requirement (Phần 6).

### HOT update (PostgreSQL) và “cột không nằm trong index”

UPDATE tạo tuple version mới. HOT có thể tránh tạo entry B-tree mới khi không sửa cột được non-summarizing index tham chiếu và tuple mới vừa cùng heap page. Nếu không đủ chỗ, sửa cột ngoài index vẫn có thể phải cập nhật các index. Cột key, INCLUDE, expression và partial predicate đều cần xét.

HOT chain giữ entry index trỏ tới gốc/redirect; ctid của phiên bản mới vẫn có thể đổi. BRIN là summarizing index và có quy tắc ngoại lệ ở bản hỗ trợ. Giảm **heap fillfactor** có thể tạo chỗ cho HOT; giảm index fillfactor không thay việc đó. Theo dõi n_tup_hot_upd/n_tup_upd trong pg_stat_user_tables. [PostgreSQL HOT](https://www.postgresql.org/docs/18/storage-hot.html).

### BRIN / columnstore: Chọn theo workload

B-tree không phải câu trả lời duy nhất:

| Tình huống | Công cụ |
| ---------- | ------- |
| Bảng lớn, created_at tương quan mạnh với vị trí heap | PostgreSQL BRIN trên created_at; nhỏ nhưng lossy và cần recheck/summarization |
| Analytics, scan/aggregate nhiều row và ít cột | SQL Server columnstore; xét rowgroup quality, write pattern và maintenance |
| Equality thuần, không range | PostgreSQL `USING HASH` (PG 10+ đã WAL-safe) — vẫn không thay B-tree cho `ORDER BY` / range |

### Thống kê, histogram, và “index có mà không Seek”

Optimizer **đoán** cardinality. Thống kê cũ sau bulk load = plan sai (Phần 5). SQL Server: xem histogram `DBCC SHOW_STATISTICS`. PostgreSQL: `pg_stats`, `ANALYZE`. MySQL: histogram 8.0 (`ANALYZE TABLE ... UPDATE HISTOGRAM`).

### Invisible / disable trước khi DROP

INVISIBLE MySQL vẫn duy trì/enforce index. DISABLE SQL Server có thể mất truy cập clustered table hoặc ảnh hưởng constraint, nên không xem là toggle thử nghiệm tương đương. Đổi tên PostgreSQL không vô hiệu hóa index. Counter có thể reset; kiểm tra thời điểm reset và chu kỳ nghiệp vụ, không suy ra index vô dụng chỉ từ idx_scan=0.

### Trước khi CREATE INDEX

1. Query trả đúng dữ liệu chưa? Giữ NULL, duplicate, collation, timezone và thứ tự ổn định khi rewrite.
2. Chậm vì đọc/CPU hay blocking/network? Có baseline actual plan, reads, latency và tham số đại diện chưa?
3. Key có khớp equality/range/order không? INCLUDE có làm index quá rộng hoặc tăng update cost không?
4. Đã kiểm tra index hiện có, UNIQUE/FK, predicate và prefix overlap chưa?
5. Plan mới có giảm công việc không, hay chỉ đổi tên Scan thành Seek? Có regression cho tham số/query khác không?
6. Rollout có xét DDL lock, edition/version, transaction của migration, dung lượng/log và replica lag không?
7. Sau deploy: index valid/usable, query thực tế dùng đúng, write latency không vượt mức chấp nhận, có cách phục hồi DDL cũ.

Không có bộ index cố định áp dụng cho mọi bảng; xem [quy trình thực hành](#phần-11-quy-trình-tối-ưu-và-thực-hành).

## Phần 11. Quy trình tối ưu và thực hành

### Đo trước và sau theo workload

1. Lấy query **thực sự được ứng dụng gửi**, kèm kiểu/độ dài tham số, tenant, khoảng thời gian, page size và số lần chạy. ORM SQL hoặc parameter mapping có thể khác câu SQL thử tay.
2. Ghi baseline: latency p50/p95/p99, CPU, logical/physical reads, số row trả về, execution plan và waits. Một query nhanh nhưng chạy quá nhiều lần vẫn có thể là nguồn tải chính.
3. Kiểm tra tính đúng đắn trước khi rewrite: NULL, duplicate, LEFT JOIN, timezone, collation và ORDER BY có tie-breaker. So tập kết quả và multiplicity, không chỉ thời gian.
4. Thử index trên dữ liệu có kích thước/skew gần thực tế. So cả tham số hiếm/phổ biến, tenant nhỏ/lớn, range ngắn/dài và LIMIT khác nhau. Đo nhiều lượt, ghi rõ cache warm/cold; không flush cache production để benchmark.
5. Theo dõi read gain cùng write cost: insert/update p95, CPU, WAL/log, index size và khả năng giữ cache. Kiểm tra query khác dùng chung bảng/index.
6. Triển khai có DDL đã review, cơ chế build phù hợp, giám sát blocking và dung lượng. Sau deploy đối chiếu lại plan/latency và chuẩn bị DDL rollback.

SQL Server Query Store giúp theo dõi plan/runtime theo thời gian; PostgreSQL pg_stat_statements cần extension/configuration phù hợp; MySQL Performance Schema statement digests/slow log giúp chọn workload. Tín hiệu missing-index của optimizer là gợi ý cho từng query, có thể trùng nhau và không phản ánh đầy đủ chi phí ghi: không tạo tất cả tự động.

[Query Store](https://learn.microsoft.com/en-us/sql/relational-databases/performance/monitoring-performance-by-using-the-query-store?view=sql-server-ver17), [pg_stat_statements](https://www.postgresql.org/docs/18/pgstatstatements.html), [MySQL statement summaries](https://dev.mysql.com/doc/refman/8.4/en/performance-schema-statement-summary-tables.html).

### Bài thực hành PostgreSQL: Equality, range, order và covering

Chạy trong **database thử nghiệm riêng**, ngoài transaction block khi gọi VACUUM. Ví dụ đầy đủ dưới đây tạo 200.000 row với 100 tenant và khoảng 5% pending; không dùng bảng nghiệp vụ hiện có.

```sql
CREATE TABLE indexing_demo_orders (
  id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tenant_id INTEGER NOT NULL,
  status TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL,
  total NUMERIC(12, 2) NOT NULL
);

INSERT INTO indexing_demo_orders (tenant_id, status, created_at, total)
SELECT 1 + (g % 100),
       CASE WHEN (g / 100) % 20 = 0 THEN 'pending' ELSE 'done' END,
       TIMESTAMPTZ '2024-01-01 00:00:00+00' + g * INTERVAL '1 minute',
       (g % 10000)::numeric / 100
FROM generate_series(1, 200000) AS s(g);

VACUUM (ANALYZE) indexing_demo_orders;

-- Baseline: chỉ có PK id, chưa có index khớp filter/order
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, created_at, total
FROM indexing_demo_orders
WHERE tenant_id = 42
  AND status = 'pending'
  AND created_at >= TIMESTAMPTZ '2024-02-01 00:00:00+00'
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Equality prefix → range cùng sort key → payload INCLUDE
CREATE INDEX indexing_demo_orders_search
ON indexing_demo_orders (tenant_id, status, created_at DESC, id DESC)
INCLUDE (total);

-- Chạy lại đúng query/parameter, đối chiếu kết quả và công việc đọc
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, created_at, total
FROM indexing_demo_orders
WHERE tenant_id = 42
  AND status = 'pending'
  AND created_at >= TIMESTAMPTZ '2024-02-01 00:00:00+00'
ORDER BY created_at DESC, id DESC
LIMIT 20;

SELECT pg_size_pretty(pg_relation_size('indexing_demo_orders_search')) AS index_size;
```

Kỳ vọng để kiểm chứng, không phải plan/time cố định:

- Baseline có thể scan rồi top-N sort. Index mới có thể cung cấp equality/range và đúng thứ tự, dừng sau đủ kết quả.
- Đối chiếu Buffers, Rows Removed by Filter, Sort, actual rows/loops và Heap Fetches. Index Only Scan vẫn phụ thuộc visibility map; không dùng tên operator làm tiêu chí duy nhất.
- Thử status='done', bỏ LIMIT, lấy thêm cột ngoài index hoặc cập nhật total rồi đo lại. total nằm trong INCLUDE nên thay đổi nó có chi phí duy trì index và ảnh hưởng HOT.
- Với query pending cố định, thử riêng partial index (tenant_id,created_at DESC,id DESC) INCLUDE(total) WHERE status='pending'. Nó nhỏ hơn nhưng không phục vụ done hoặc generic parameter plan chưa chứng minh predicate.

### Các thông tin cần giữ khi bàn giao một index

| Thông tin | Nội dung cần ghi |
| --- | --- |
| Query/workload | SQL, tham số đại diện, tần suất và SLA |
| Schema | Engine/version/compatibility, key/order/INCLUDE/predicate, collation |
| Bằng chứng đọc | Plan trước/sau, rows/reads/CPU/latency, kết quả đúng |
| Bằng chứng ghi | Write latency, dung lượng và log/WAL trên tải đại diện |
| Rollout | DDL, điều kiện online/concurrent, thời điểm, giám sát và rollback |

## Bonus. Hiểu sâu B+ Tree trong RDBMS

Phần này giải thích cơ chế page split, row locator, fanout và fillfactor để bổ sung cho mô hình truy cập ở Phần 1–2.

Các B-tree rowstore thông thường dùng mô hình B+ tree hoặc biến thể: leaf chứa index entries, internal page chủ yếu dẫn đường. Với PostgreSQL/nonclustered index, entry trỏ tới dữ liệu ở nơi khác; chỉ clustered leaf chứa dữ liệu hàng. Không áp mô hình này cho GIN/BRIN/columnstore.

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

Seek bắt đầu bằng đi qua nhánh đến leaf; entry trùng hoặc nhiều kết quả có thể trải qua nhiều leaf page. Range scan cơ bản đi tiếp theo sibling link; skip scan/multiple ranges có thể tìm lại nhiều lần. Phân biệt tìm vị trí với toàn bộ công việc trả kết quả.

### Page, fanout, chiều cao cây

Index không lưu từng byte một file phẳng. Đơn vị I/O là **page** (PostgreSQL mặc định 8 KB, SQL Server 8 KB, InnoDB 16 KB thường gặp).

Một page nhánh chứa hàng trăm separator → **fanout** (số con mỗi node) ~ 100–500 tùy độ rộng key. Chiều cao:

| Số row (lá ~300 key/page) | Chiều cao (xấp xỉ) |
| ------------------------- | ------------------ |
| 300                       | 1 (gần như chỉ lá) |
| 90.000                    | 2                  |
| 27 triệu                  | 3                  |
| 8 tỷ                      | 4                  |

Bảng này giả định khoảng 300 entry mỗi leaf và fanout 300, bỏ qua metadata/fillfactor. Cây thực tế có thể cao hơn; root là leaf khi cây nhỏ. Key rộng làm giảm fanout/mật độ, ảnh hưởng cache và có thể tăng chiều cao. Heap/clustered lookup thường là phần đáng chú ý, nhưng phải đo I/O/CPU thực tế.

Root/tầng trên thường có tỷ lệ cache hit cao; không bảo đảm luôn ở RAM khi cache bị áp lực. Random insert thường gây thêm chi phí ở leaf/data page và split.

### Lá nối nhau: Vì sao range chỉ đi một hướng

Lá là danh sách móc **hai chiều** (`prev` / `next`). `BETWEEN` / `LIKE 'abc%'` / `ORDER BY key`:

1. Xuống cây tới lá chứa cận trái  
2. Đọc tuần tự lá → lá kế, **một hướng**

Backward scan đảo toàn bộ hướng key; ORDER BY score DESC, created_at ASC thường cần mixed-direction index hoặc Sort, trừ khi predicate cố định một key. <> có thể được thực hiện bằng nhiều range, không bắt buộc scan mọi entry (Phần 4).

InnoDB / SQL Server clustered: **bảng** cũng là B+ Tree — range trên PK là scan lá clustered, sequential theo key, không theo thời điểm insert (trừ khi PK tăng dần).

### Page split: Random insert đắt hơn sequential

Lá đầy mà phải chèn key nằm giữa → **split**: một page thành hai, ~một nửa key sang page mới, sửa pointer tầng trên (đôi khi split lan lên root).

```
Lá đầy:  [10|20|30|40|50|60|70|80]
Insert 45 → split
         [10|20|30|40|45]  [50|60|70|80]
                ↑ page mới, I/O + fragment
```

- **Sequential** (IDENTITY hoặc key có thứ tự theo comparator): thường tập trung ở edge page; engine vẫn cấp page/split khi đầy, và concurrency có thể gây last-page contention.
- **Random** (UUIDv4): truy cập nhiều leaf, split giữa có thể làm giảm mật độ và locality. Mức ảnh hưởng tùy cache, insert pattern và engine.

SQL Server dm_db_index_operational_stats có leaf_allocation_count/nonleaf_allocation_count, không có cột tên page_split. Đây là chỉ báo allocation/split, không tách rõ mọi loại split; cần Extended Events khi muốn điều tra cụ thể. PostgreSQL pg_stat_all_indexes không trực tiếp đo page split/bloat; cần khảo sát page/size và công cụ như pgstatindex. [SQL Server operational counters](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-objects/sys-dm-db-index-operational-stats-transact-sql?view=sql-server-ver17).

### Leaf chứa gì: TID (heap) vs clustered key

Sau khi Seek tới lá, engine phải ra **row**:

| | PostgreSQL (heap mặc định) | SQL Server clustered / InnoDB |
| --- | --- | --- |
| B-tree secondary leaf | key + TID (block, tuple offset trong relation) | key + clustered key (InnoDB là PK); SQL Server heap dùng RID |
| Bước tiếp | Heap fetch theo TID (có thể random I/O) | Seek thêm trên cây clustered bằng PK |

Đủ payload trên leaf cho phép tránh lookup lấy dữ liệu; PostgreSQL còn cần visibility map để bỏ heap fetch. Nếu bảng nhỏ và index đã có tất cả cột, SELECT * vẫn có thể covering, nhưng thường làm access path rộng hơn cần thiết (Phần 6, 10).

PostgreSQL HOT không có nghĩa ctid của tuple mới giữ nguyên: index entry gốc dẫn qua HOT chain. InnoDB đổi PK cần di chuyển clustered record và duy trì secondary locator; SQL Server đổi clustered key có tác động tương tự. Nếu SQL Server PK nonclustered, đổi PK không đồng nghĩa đổi clustered key. Tránh dùng khóa hay thay đổi làm locator.

Nonclustered/secondary leaf chỉ có key, locator và payload đã khai báo. SQL Server hỗ trợ INCLUDE; InnoDB phải thêm payload vào key. Độ rộng locator ảnh hưởng dung lượng ngay cả khi chỉ lấy vài cột (Phần 9).

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

- SQL Server FILLFACTOR=0 tương đương 100 khi build/rebuild; không tự duy trì tỷ lệ trống sau đó. Giảm chỉ khi đo thấy split đáng kể và lợi ích bù mật độ/cache giảm.
- PostgreSQL B-tree mặc định fillfactor=90, khác heap mặc định=100. Heap fillfactor dành chỗ cho tuple version/HOT, còn index fillfactor dành chỗ trong page index. ALTER TABLE SET fillfactor không tự viết lại mọi page cũ.
- Không áp mức 70–90 cho mọi bảng update nhiều: update payload/key có cơ chế khác nhau. InnoDB có tham số và quy tắc page riêng, không dùng cú pháp FILLFACTOR của SQL Server/PostgreSQL.

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
| Substring LIKE / tìm theo token | Trigram cho substring; FULLTEXT cho token, semantics khác nhau | Phần 4, 6 |

CREATE INDEX mặc định của PostgreSQL và rowstore thông thường là B-tree; SQL Server memory-optimized/columnstore và engine khác có ngoại lệ. Chọn cấu trúc theo operator, thứ tự cần dùng và workload.

---

Bốn nguyên tắc mô tả cách truy cập B-tree cơ bản: tìm vị trí, scan theo thứ tự, tận dụng prefix và đọc range. Skip scan, multiple seek và các access method khác bổ sung khả năng ngoài mô hình đó. Dùng mô hình để đề xuất index, rồi quyết định bằng số liệu.
