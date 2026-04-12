SECTION 1 — Microservices Fundamentals
Q1.
"You built a microservices system. Explain to me what a microservice is and how it's different from a monolith. Use your own project as the example."

Q2.
"Your project has three services — checkout, shipping, notification. How did you decide what belongs in each service? What is the principle behind that boundary?"

Q3.
"What are the disadvantages of microservices? Don't give me the benefits — I want the problems. Based on what you built, what made it harder than just writing one Spring Boot app?"

Answer all three above. I'll give feedback and then we move to Section 2.

SECTION 2 — Eureka & Service Discovery (after Section 1 feedback)
Q4.
"What is Eureka and why did you use it? What specific problem does it solve in your system?"

Q5.
"Walk me through exactly what happens — step by step — when checkout-service starts up and registers with Eureka."

Q6.
"What is the difference between register-with-eureka and fetch-registry? Why do you set both to false in the Eureka server but true in the clients?"

SECTION 3 — REST Communication & Load Balancing
Q7.
"How does checkout-service call shipping-service? Walk me through the exact code path — from the line that makes the call to the line that receives it."

Q8.
"What does @LoadBalanced do? What happens if you remove it?"

Q9.
"You chose synchronous REST for checkout→shipping. Why? Why not use RabbitMQ for that call too?"

SECTION 4 — RabbitMQ & Async Messaging
Q10.
"Explain RabbitMQ to me. What is an Exchange? What is a Queue? What is a Binding? How do they relate to each other?"

Q11.
"Why did you use DirectExchange specifically? What would FanoutExchange do differently?"

Q12.
"I stop notification-service. I send 5 orders. I restart notification-service 10 minutes later. What happens to those 5 messages? Why?"

Q13.
"What is the difference between a synchronous call and an asynchronous message in terms of what the caller does after sending?"

SECTION 5 — Database Design
Q14.
"Each service has its own database. Why? What specifically breaks if they share one database?"

Q15.
"checkout-service has an order with orderId = 1. shipping-service has a shipment with orderId = 1. There is no foreign key between them. How are they related? What is this pattern called?"

Q16.
"Why does notification-service use MongoDB instead of MySQL? What property of notifications makes a document database a better fit?"

Q17.
"What is spring.jpa.hibernate.ddl-auto=update? Why would you never use create in production?"

SECTION 6 — Architecture Decisions
Q18.
"You have ShippingResponse defined in both checkout-service and notification-service. That's duplication. A teammate says we should create a common-dto shared library. What do you say?"

Q19.
"In CheckoutService, you save the order as PENDING before calling shipping. Why? What would go wrong if you only saved after shipping confirmed?"

Q20.
"We wrapped the RabbitMQ publish in its own try-catch block, separate from the shipping call. Why? What real problem does that solve?"

SECTION 7 — Debugging & Production Thinking
Q21.
"You get a support ticket: 'User placed an order, got a confirmation, but never received a notification email.' Walk me through how you debug this. Step by step."

Q22.
"What is the Eureka self-preservation mode? Why did we disable it in development but you should leave it on in production?"

Q23.
"A junior developer on your team asks: what's the difference between @Component, @Service, and @Repository? How do you explain it?"

SECTION 8 — The "Why" Questions
These are the hardest. Interviewers ask these to see if you understand or just copied code.

Q24.
"Why does @RabbitListener not need you to write a loop or polling mechanism? What does Spring do under the hood?"

Q25.
"You used @Builder from Lombok in every entity and DTO. Why is the Builder pattern better than just using a constructor with all arguments?"

Q26.
"Why do we use constructor injection (@RequiredArgsConstructor + final) instead of @Autowired on the field directly?"

Q27. — Final Question
"Imagine you're presenting this project to a panel of senior engineers. In 60 seconds, explain: what you built, what technologies you used, and the three most important design decisions you made and why."

