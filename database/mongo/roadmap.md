# MongoDB Roadmap

## 1. Giới thiệu MongoDB

- MongoDB là gì?
- NoSQL vs SQL
- Ưu và nhược điểm của MongoDB
- Khi nào nên dùng MongoDB?

## 2. Cài đặt và công cụ

- Cài MongoDB (local hoặc Docker)
- MongoDB Compass (UI client)
- Mongo Shell (mongosh)
- MongoDB Atlas (cloud)

## 3. Kiến trúc MongoDB

- Document, Collection, Database
- BSON vs JSON
- ObjectId
- Sharding (chia nhỏ dữ liệu)
- Replica Set (replication dữ liệu)

## 4. CRUD cơ bản

- `insertOne`, `insertMany`
- `find`, `findOne` với filter, projection
- `updateOne`, `updateMany`, `$set`, `$unset`, `$inc`, `$push`, `$addToSet`
- `deleteOne`, `deleteMany`
- `replaceOne`

## 5. Truy vấn nâng cao

- Filter nâng cao: `$or`, `$and`, `$in`, `$regex`, `$gte`, `$lt`, ...
- Sorting, limit, skip
- Indexing: tạo index, compound index, text index
- Query performance và explain()

## 6. Aggregation Framework

- `$match`, `$group`, `$project`, `$sort`, `$lookup`, `$unwind`, `$count`
- Pipeline cơ bản và nâng cao
- Tính toán số liệu, join collection
- Aggregation với array và embedded document

## 7. Schema Design

- Schema-less vs Schema design
- Embedded vs Reference
- Chọn ObjectId hay custom _id
- Best practices

## 8. Index và Tối ưu hiệu năng

- Index types: single, compound, multikey, text, hashed
- Phân tích truy vấn với `explain()`
- Sử dụng index hợp lý trong aggregation
- Caching và performance tuning

## 9. Transaction và Lock

- Multi-document transaction (MongoDB 4.0+)
- Session và atomic operation
- Locking và isolation trong MongoDB

## 10. Bảo mật và Backup

- Authentication và Role-based Access Control (RBAC)
- IP whitelist, TLS/SSL
- Backup/restore: `mongodump`, `mongorestore`
- MongoDB Atlas backup

## 11. Kết nối và tích hợp

- Dùng MongoDB với:
  - Node.js (mongoose)
  - Java (Spring Data MongoDB)
  - Python (pymongo)
  - PHP (MongoDB extension)
- Kết nối qua URI connection string

## 12. Công cụ mở rộng

- MongoDB Atlas (cloud)
- MongoDB Charts (visualize dữ liệu)
- MongoDB Trigger và Realm
- Monitoring với Atlas Metrics hoặc Ops Manager

## 13. Dự án thực hành

- Todo app đơn giản (Node.js + MongoDB)
- Blog CMS với Mongo
- RESTful API CRUD (Express hoặc Spring Boot)
- Aggregation cho thống kê đơn hàng
- Hệ thống ghi log hoặc analytics sử dụng MongoDB

