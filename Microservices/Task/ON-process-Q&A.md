Why Each Service Has Its Own Database

The Shared Database Anti-Pattern (What NOT to do)
```
                    ❌ WRONG APPROACH
                    
checkout-service ──┐
                   ├──► [ Single Shared Database ]
shipping-service ──┤         (all tables)
                   │
notification-service─┘

Problems:
─────────
1. COUPLING: If DBA renames a column → ALL services break simultaneously
2. SCALING: Can't scale checkout DB independently from shipping DB
3. OWNERSHIP: Who owns the 'orders' table? Checkout? Shipping? Both?
4. DEPLOYMENT: Schema change requires coordinating ALL team deployments
5. FAILURE: One service's bad query can lock tables → ALL services slow down
```

The Golden Rule:

A service's database is its private implementation detail. No other service may access it directly. The only way to get data from another service is through its API.

---
2. Our Database Design — Full Picture
```
checkout_db (MySQL)                    shipping_db (PostgreSQL)
───────────────────                    ────────────────────────
orders                                 shipments
──────                                 ─────────
id            BIGINT PK AUTO           id              BIGINT PK
product_name  VARCHAR(255)             order_id        BIGINT
quantity      INT                      tracking_number VARCHAR(255) UNIQUE
user_email    VARCHAR(255)             recipient_email VARCHAR(255)
tracking_number VARCHAR(255)           status          VARCHAR(50)
status        VARCHAR(50)              created_at      TIMESTAMP
created_at    DATETIME


notification_db (MongoDB Atlas)
────────────────────────────────
notification_logs collection
────────────────────────────
{
  _id: ObjectId,
  orderId: Long,
  userEmail: String,
  productName: String,
  trackingNumber: String,
  message: String,
  sentAt: DateTime
}
```

Why MongoDB for Notification?

```
Notification logs are a perfect fit for MongoDB because:

✅ Schema-less: Different notification types have different fields
   Email notification:  { subject, body, recipient }
   SMS notification:    { phoneNumber, message }
   Push notification:   { deviceToken, title, payload }

✅ Write-heavy: We only INSERT logs, rarely query them
   MongoDB is optimized for high-speed writes

✅ No relationships: Notification logs don't join with anything
   Documents are self-contained - perfect for MongoDB's document model

✅ Flexible: Fields can be added without schema migrations
   No ALTER TABLE needed - just add the field
```

```
JPA (relational)              MongoDB (document)
────────────────              ──────────────────
@Entity                   →   @Document
@Table(name="...")        →   @Document(collection="...")
@Id (jakarta.persistence) →   @Id (springframework.data.annotation)
@Column                   →   (no annotation needed - all fields stored)
@GeneratedValue           →   (MongoDB auto-generates ObjectId)
JpaRepository             →   MongoRepository
```


The Data Relationship Across Services

```
MySQL orders table          PostgreSQL shipments         MongoDB notification_logs
──────────────────          ────────────────────         ───────────────────────
id = 1                      id = 1                       _id = "65f1a2b3..."
status = CONFIRMED          order_id = 1  ←──── same     orderId = 1  ←──── same
tracking = TRK-ABC          tracking = TRK-ABC ←── same   trackingNumber = TRK-ABC
                                                          status = SENT

They're connected by orderId and trackingNumber
NOT by foreign keys or JPA joins
This is called: eventual consistency across service boundaries
```
