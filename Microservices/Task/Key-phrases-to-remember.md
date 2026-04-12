```
Key phrases to remember:

Independent deployability — deploy one service without touching others
Failure isolation — one service crashes, others keep running
Independent scalability — if checkout gets high traffic, scale only that service
```
```
"I used the Single Responsibility Principle at the service level.
 Each service owns one business capability and nothing else.

checkout-service owns the order lifecycle — creating orders, tracking their status.
 shipping-service owns everything about physical delivery — generating tracking numbers,
 managing shipment records. notification-service owns customer communication
— it doesn't care how orders work, it just knows it needs to send a message when one is placed.

The technical boundary follows the business boundary. That's the principle — one service,
 one business capability, one reason to change."

Key phrase to remember:

"One service, one business capability, one reason to change"
```

```
When asked about disadvantages — always lead with distributed transactions and network reliability.
Those are the answers that show you actually thought about production problems, not just code organization.
```

```
"When checkout-service starts, Spring Boot reads the application.properties and sees
 eureka.client.service-url.defaultZone=http://localhost:8761/eureka/. The Eureka client
 embedded in checkout-service sends a registration HTTP POST to that URL, announcing:

Service name: CHECKOUT-SERVICE
Instance IP: localhost
Port: 8081
Status: UP
Eureka server receives this and adds CHECKOUT-SERVICE to its registry.

After registration, checkout-service starts sending heartbeats — a small HTTP request
to Eureka every 30 seconds by default, or every 10 seconds in our config because we set
 lease-renewal-interval-in-seconds=10. This tells Eureka: I'm still alive.

Simultaneously, checkout-service fetches the full registry from Eureka and caches it locally.
 Now it knows the addresses of all other registered services.

If checkout-service stops sending heartbeats — because it crashed — Eureka waits for
 lease-expiration-duration-in-seconds=30, then removes it from the registry.
 Other services that refresh their cache will stop seeing CHECKOUT-SERVICE as available."

Key concepts you needed to include:



Registration  → one-time HTTP POST on startup announcing the service
Heartbeat     → repeated signal proving the service is still alive
Cache         → local copy of the registry each client maintains
Expiration    → what happens when heartbeats stop — removal from registry
```

```
The question said "exact code path" — that means class names, method names, annotations.
 You described the concept correctly but an interviewer asking for "exact code path" wants
to hear the actual classes involved on both sides.

How a senior developer would answer:

"Starting from checkout-service:

CheckoutController receives the POST request at /api/checkout via @PostMapping.
 It's annotated with @RequestBody so Jackson automatically deserializes the JSON into
 a CheckoutRequest object. The controller calls CheckoutService.processCheckout().

Inside CheckoutService, we first save the order as PENDING using orderRepository.save().
 Then we build a ShippingRequest DTO and call:

java


restTemplate.postForObject(
    "http://SHIPPING-SERVICE/api/shipping",
    shippingRequest,
    ShippingResponse.class
)
The @LoadBalanced RestTemplate intercepts this call. It sees SHIPPING-SERVICE as the hostname,
queries the local Eureka registry cache, finds localhost:8082, and rewrites the URL to
http://localhost:8082/api/shipping. Jackson serializes the ShippingRequest to JSON and sends the HTTP POST.

On the shipping-service side, ShippingController receives it at @PostMapping("/api/shipping").
 Jackson deserializes the body into ShippingRequest. It calls ShippingService.createShipment(),
 which generates the tracking number, saves the Shipment entity to PostgreSQL via shipmentRepository.save(),
builds a ShippingResponse, and returns it.

The HTTP response travels back to checkout-service. postForObject deserializes the JSON into a
ShippingResponse object. We extract the tracking number, update the order entity, save it as CONFIRMED."

The key addition: Name the actual classes — CheckoutController, CheckoutService, ShippingController,
ShippingService. That's what "exact code path" means.


```

---
# Important

```
"RabbitMQ is a message broker — it sits between services and handles message delivery
 so the sender doesn't need to know anything about the receiver.

The Producer is the service that sends a message — in my project that's checkout-service
publishing an OrderPlacedEvent.

The Exchange receives the message from the producer. It doesn't store anything — 
its only job is to look at the message and decide which queue to route it to.
Think of it as a post office sorting center.

The Queue is where messages are actually stored until a consumer picks them up.
 It's durable in our config — meaning it survives a RabbitMQ restart without losing messages.

The Binding is the rule that connects an exchange to a queue. In our system the binding says:
when a message arrives at order.exchange with routing key order.placed — send it to order.queue.
 Without a binding, messages arrive at the exchange and go nowhere.

The Consumer — notification-service — listens to order.queue via @RabbitListener.
 When a message arrives, Spring automatically deserializes it and calls our handler method.

So the full flow: checkout publishes to exchange → exchange checks binding
 → routing key matches → message goes to queue → notification consumes from queue."

The thing you missed: Routing key. It's the critical piece that makes the binding work. Learn this sentence:

"The binding says: messages arriving at THIS exchange with THIS routing key go to THIS queue."
```

```
Exchange Type    Routing Logic              Use Case
─────────────────────────────────────────────────────
Direct           Exact routing key match    One event → one queue
Fanout           Ignores key, sends all     Broadcast to all queues
Topic            Wildcard pattern match     Flexible routing by pattern
```

