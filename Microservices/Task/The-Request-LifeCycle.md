The Complete Request Lifecycle — Every Single Step

```
════════════════════════════════════════════════════════════════════
 REQUEST: POST http://localhost:8081/api/checkout
 BODY:    { "productName": "Laptop", "quantity": 1,
            "userEmail": "user@example.com" }
════════════════════════════════════════════════════════════════════

LAYER 1 — HTTP Request arrives at checkout-service (port 8081)
──────────────────────────────────────────────────────────────
  Spring's DispatcherServlet receives the HTTP request
  Matches URL "/api/checkout" + method POST
        │
        ▼
  CheckoutController.checkout(@RequestBody CheckoutRequest)
  Jackson deserializes JSON body → CheckoutRequest object
  {productName="Laptop", quantity=1, userEmail="user@example.com"}
        │
        ▼

LAYER 2 — Business Logic (CheckoutService)
──────────────────────────────────────────
  CheckoutService.processCheckout(request)

  STEP 2a: Build Order entity
  Order {
    productName = "Laptop"
    quantity    = 1
    userEmail   = "user@example.com"
    status      = "PENDING"          ← not confirmed yet
    trackingNumber = null            ← not assigned yet
    createdAt   = 2024-01-15T10:30:00
  }

  STEP 2b: orderRepository.save(order)
  Hibernate generates SQL:
  INSERT INTO orders (product_name, quantity, user_email, status, created_at)
  VALUES ('Laptop', 1, 'user@example.com', 'PENDING', '2024-01-15 10:30:00')
  MySQL returns: id = 1
        │
        ▼

LAYER 3 — REST call to shipping-service (via Eureka)
────────────────────────────────────────────────────
  STEP 3a: Build ShippingRequest
  ShippingRequest {
    orderId        = 1
    recipientEmail = "user@example.com"
    productName    = "Laptop"
    quantity       = 1
  }

  STEP 3b: restTemplate.postForObject("http://SHIPPING-SERVICE/api/shipping", ...)

  What happens inside @LoadBalanced RestTemplate:
  ┌─────────────────────────────────────────────────────────┐
  │ 1. Intercepts "SHIPPING-SERVICE" hostname               │
  │ 2. Checks local Eureka registry cache                   │
  │ 3. Finds: SHIPPING-SERVICE → localhost:8082             │
  │ 4. Replaces URL: http://localhost:8082/api/shipping     │
  │ 5. Jackson serializes ShippingRequest → JSON            │
  │ 6. Sends HTTP POST to localhost:8082                    │
  └─────────────────────────────────────────────────────────┘
        │
        │ HTTP POST → localhost:8082/api/shipping
        ▼

LAYER 4 — shipping-service receives request (port 8082)
───────────────────────────────────────────────────────
  ShippingController.createShipment(@RequestBody ShippingRequest)
        │
        ▼
  ShippingService.createShipment(request)

  STEP 4a: Generate tracking number
  UUID.randomUUID() = "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  substring(0,8).toUpperCase() = "A1B2C3D4"
  trackingNumber = "TRK-A1B2C3D4"

  STEP 4b: Build Shipment entity
  Shipment {
    orderId        = 1
    trackingNumber = "TRK-A1B2C3D4"
    recipientEmail = "user@example.com"
    status         = "CREATED"
    createdAt      = 2024-01-15T10:30:01
  }

  STEP 4c: shipmentRepository.save(shipment)
  Hibernate generates SQL:
  INSERT INTO shipments (order_id, tracking_number, recipient_email, status, created_at)
  VALUES (1, 'TRK-A1B2C3D4', 'user@example.com', 'CREATED', '2024-01-15 10:30:01')
  PostgreSQL returns: id = 1

  STEP 4d: Build ShippingResponse DTO
  ShippingResponse {
    shipmentId     = 1
    trackingNumber = "TRK-A1B2C3D4"
    status         = "CREATED"
    message        = "Shipment created successfully"
  }
        │
        │ HTTP 201 response with JSON body
        ▼

LAYER 5 — Back in checkout-service
───────────────────────────────────
  restTemplate.postForObject() returns ShippingResponse
  Jackson deserializes response JSON → ShippingResponse object

  STEP 5a: Update order with tracking number
  savedOrder.setTrackingNumber("TRK-A1B2C3D4")
  savedOrder.setStatus("CONFIRMED")

  STEP 5b: orderRepository.save(savedOrder)
  Hibernate generates SQL:
  UPDATE orders
  SET tracking_number = 'TRK-A1B2C3D4', status = 'CONFIRMED'
  WHERE id = 1
        │
        ▼

LAYER 6 — Publish Event to RabbitMQ
─────────────────────────────────────
  STEP 6a: Build OrderPlacedEvent
  OrderPlacedEvent {
    orderId        = 1
    productName    = "Laptop"
    quantity       = 1
    userEmail      = "user@example.com"
    trackingNumber = "TRK-A1B2C3D4"
  }

  STEP 6b: rabbitTemplate.convertAndSend(
    exchange   = "order.exchange",
    routingKey = "order.placed",
    message    = OrderPlacedEvent
  )

  What happens inside RabbitMQ:
  ┌─────────────────────────────────────────────────────────┐
  │ 1. Jackson converts OrderPlacedEvent → JSON string      │
  │ 2. JSON wrapped in RabbitMQ Message with headers        │
  │ 3. Sent to "order.exchange"                             │
  │ 4. Exchange checks bindings:                            │
  │    "order.placed" routing key → routes to "order.queue" │
  │ 5. Message stored in "order.queue"                      │
  │ 6. convertAndSend() RETURNS IMMEDIATELY                 │
  │    checkout does NOT wait for notification              │
  └─────────────────────────────────────────────────────────┘
        │
        ▼

LAYER 7 — Return response to user
───────────────────────────────────
  CheckoutResponse {
    orderId        = 1
    productName    = "Laptop"
    userEmail      = "user@example.com"
    trackingNumber = "TRK-A1B2C3D4"
    status         = "CONFIRMED"
    message        = "Order placed successfully!"
  }

  HTTP 201 CREATED sent back to user ✅
  USER GETS THEIR RESPONSE HERE
  ↑
  Everything above happened in ~200-500ms

════════════════════════════════════════════════════════════════════
  ASYNC — Happens independently, after the user already got response
════════════════════════════════════════════════════════════════════

LAYER 8 — notification-service consumes from queue
───────────────────────────────────────────────────
  Spring's @RabbitListener detects message in "order.queue"
  Jackson deserializes JSON → OrderPlacedEvent object

  OrderEventListener.handleOrderPlacedEvent(event)
        │
        ▼
  NotificationService.processNotification(event)

  STEP 8a: Build message string
  "Hello user@example.com! Your order for 1 x Laptop
   has been confirmed. Your tracking number is: TRK-A1B2C3D4"

  STEP 8b: Build NotificationLog document
  NotificationLog {
    orderId        = 1
    userEmail      = "user@example.com"
    productName    = "Laptop"
    quantity       = 1
    trackingNumber = "TRK-A1B2C3D4"
    message        = "Hello user@example.com!..."
    sentAt         = 2024-01-15T10:30:02
    status         = "SENT"
  }

  STEP 8c: notificationLogRepository.save(notificationLog)
  MongoDB stores BSON document in notification_logs collection
  MongoDB generates: _id = "65f1a2b3c4d5e6f7a8b9c0d1"

  ✅ Notification logged to MongoDB Atlas
```

---

 Full Debug Flow — Tracing One Order
Use these commands to trace order #1 across all services:

```
# Step 1: Place the order
curl -X POST http://localhost:8081/api/checkout \
  -H "Content-Type: application/json" \
  -d '{"productName":"Laptop","quantity":1,"userEmail":"user@example.com"}'

# Response gives you orderId - let's say it's 1

# Step 2: Check order state in checkout-service
curl http://localhost:8081/api/checkout/orders/1

# Step 3: Check shipment state in shipping-service
curl http://localhost:8082/api/shipping/order/1

# Step 4: Check MongoDB Atlas manually in the browser
# Browse Collections → notification_db → notification_logs
# Filter: { "orderId": 1 }

# Step 5: Check RabbitMQ queue state
# http://localhost:15672 → Queues → order.queue
# Should show 0 messages ready (all consumed)
```

---

System State Diagram — All Possible States

```
Order Status Flow (MySQL):
─────────────────────────
                    shipping fails
PENDING ──────────────────────────────► FAILED
   │
   │ shipping succeeds
   ▼
CONFIRMED ──────────────────────────────► (done)
   │
   │ event published to RabbitMQ
   ▼
[order.queue]
   │
   │ notification-service processes
   ▼
MongoDB: status = SENT


Shipment Status Flow (PostgreSQL):
───────────────────────────────────
CREATED (set when shipment is first made)
   │
   │ (future enhancement)
   ▼
IN_TRANSIT
   │
   ▼
DELIVERED


Full System Health Check:
──────────────────────────
Service             How To Check
──────────────────────────────────────────────────────
eureka-server       http://localhost:8761
checkout-service    http://localhost:8081/api/checkout/health
shipping-service    http://localhost:8082/api/shipping/health
notification        No HTTP endpoint - check logs + MongoDB
RabbitMQ            http://localhost:15672
MySQL               SELECT * FROM orders ORDER BY id DESC LIMIT 5
PostgreSQL          SELECT * FROM shipments ORDER BY id DESC LIMIT 5
MongoDB             Atlas dashboard → notification_logs collection
```

---

The Walkt hrough 

```
"A user sends a POST request to checkout-service on port 8081.

The service saves the order as PENDING in MySQL, then makes a 
synchronous REST call to shipping-service. It uses a @LoadBalanced 
RestTemplate — instead of a hardcoded URL, it uses the service name 
'SHIPPING-SERVICE'. Spring Cloud LoadBalancer resolves that name 
to the actual address by querying the local Eureka registry cache.

Shipping-service receives the request, generates a tracking number,
saves the shipment to PostgreSQL, and returns the tracking number.

Back in checkout-service, we update the order status to CONFIRMED
and store the tracking number in MySQL.

Then — asynchronously — we publish an OrderPlacedEvent to RabbitMQ.
We convert the event to JSON and send it to a DirectExchange with
routing key 'order.placed'. The exchange routes it to 'order.queue'.
This call returns immediately. The user gets their response right now.

Independently, notification-service is listening to 'order.queue'
via @RabbitListener. When the message arrives, Spring automatically
deserializes the JSON back to an OrderPlacedEvent and calls our
handler method. We build a notification log document and save it
to MongoDB Atlas.

The key design decisions are:
- REST for checkout→shipping because we need the tracking number synchronously
- RabbitMQ for checkout→notification because it's fire-and-forget
- Eureka so no service has hardcoded URLs
- Separate databases so services are truly independent"
```
