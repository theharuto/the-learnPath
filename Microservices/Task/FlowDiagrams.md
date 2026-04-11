The Complete Data Flow — Visualized

```
POST /api/checkout
Body: {
  "productName": "Laptop",
  "quantity": 1,
  "userEmail": "user@example.com"
}
        │
        ▼
CheckoutController.checkout()
        │
        ▼
CheckoutService.processCheckout()
        │
        ├─ 1. Build Order entity, status = PENDING
        ├─ 2. orderRepository.save() → MySQL checkout_db
        │
        ├─ 3. Build ShippingRequest DTO
        ├─ 4. restTemplate.postForObject("http://SHIPPING-SERVICE/api/shipping")
        │              │
        │              │  @LoadBalanced intercepts
        │              │  asks Eureka: "where is SHIPPING-SERVICE?"
        │              │  Eureka says: "localhost:8082"
        │              │
        │              ▼
        │     ShippingController.createShipment()
        │              │
        │     ShippingService.createShipment()
        │              │
        │              ├─ Generate tracking: "TRK-A1B2C3D4"
        │              ├─ Save Shipment → PostgreSQL shipping_db
        │              └─ return ShippingResponse
        │              │
        │     ◄────────┘ HTTP 201 + ShippingResponse JSON
        │
        ├─ 5. Update Order: trackingNumber + status = CONFIRMED
        ├─ 6. orderRepository.save() → MySQL checkout_db
        │
        └─ 7. Return CheckoutResponse to user

Response: {
  "orderId": 1,
  "productName": "Laptop",
  "userEmail": "user@example.com",
  "trackingNumber": "TRK-A1B2C3D4",
  "status": "CONFIRMED",
  "message": "Order placed successfully!"
}
```
---

RabbitMQ Core Concepts — The Mental Model
```
┌─────────────────────────────────────────────────────────────────┐
│                        RabbitMQ Broker                          │
│                                                                 │
│   Producer          Exchange          Queue         Consumer    │
│   ────────          ────────          ─────         ────────    │
│                                                                 │
│  checkout    ──►   order.exchange  ──►  order.queue  ──►  notification │
│  (publishes)       (routes msg)        (stores msg)       (listens)    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```
| Term | What It Is | Real World Analogy |
| :-- | :-- | :-- |
| Producer | The service that sends a message | Person dropping a letter in a mailbox |
| Exchange | Receives message, decides which queue to route it to | Post office sorting center |
| Queue | Stores messages until a consumer picks them up | The actual mailbox |
| Consumer | The service that reads and processes messages | Person receiving and reading the letter |

---
The relationship between these three beans:


```
  [checkout-service publishes]
           │
           │  routing key: "order.placed"
           ▼
    ┌─────────────────┐
    │  order.exchange  │  (DirectExchange)
    └────────┬────────┘
             │  Binding: "order.placed" → order.queue
             ▼
    ┌─────────────────┐
    │   order.queue   │  (Queue, durable=true)
    └────────┬────────┘
             │
             ▼
  [notification-service consumes]
```

---

 The Complete Async Flow — Visualized

```
POST /api/checkout
        │
        ▼
CheckoutService.processCheckout()
        │
        ├─ Save order PENDING → MySQL
        ├─ Call SHIPPING-SERVICE → PostgreSQL
        ├─ Update order CONFIRMED → MySQL
        │
        ├─ Build OrderPlacedEvent
        ├─ rabbitTemplate.convertAndSend()
        │         │
        │         │  Converts to JSON:
        │         │  {
        │         │    "orderId": 1,
        │         │    "productName": "Laptop",
        │         │    "quantity": 1,
        │         │    "userEmail": "user@example.com",
        │         │    "trackingNumber": "TRK-A1B2C3D4"
        │         │  }
        │         │
        │         ▼
        │    [order.exchange]
        │         │  routing key match: "order.placed"
        │         ▼
        │    [order.queue] ← message sits here
        │
        └─ Returns response to user IMMEDIATELY ✅
                          │
              (milliseconds or minutes later)
                          │
                          ▼
           OrderEventListener.handleOrderPlacedEvent()
                          │
                          ├─ Deserializes JSON → OrderPlacedEvent
                          ├─ Logs notification details
                          └─ (would send real email here)
```

