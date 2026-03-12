## Notes

*   **Owning vs. inverse side**  
    In a bidirectional **One‑to‑Many / Many‑to‑One**, the **`@ManyToOne` side is the owning side** that writes the foreign key; the `@OneToMany` side is **inverse** and must declare `mappedBy` to point to the owning attribute. [\[baeldung.com\]](https://www.baeldung.com/hibernate-one-to-many)

*   **`mappedBy` meaning**  
    `mappedBy = "customer"` on `Customer.orders` declares that **`Order.customer` is the owner**; the collection is a mirror that JPA/Hibernate does **not** use to update the FK. [\[baeldung.com\]](https://www.baeldung.com/hibernate-one-to-many)

*   **Cascade is propagation, not relationship setting**  
    `cascade = PERSIST` (or `ALL`) **propagates** the `persist` (or other operations) from parent to children, but it **does not set the foreign key**. The FK update still comes from the **owning side**. [\[stackoverflow.com\]](https://stackoverflow.com/questions/20934130/owning-side-vs-non-owning-side-in-hibernate-and-its-usage-in-reference-of-mapped), [\[baeldung.com\]](https://www.baeldung.com/hibernate-one-to-many)

*   **Who must you save?**  
    Save the entity **where you placed cascade**, provided the **owning side field is set**. Without cascade, you must persist each entity explicitly. [\[stackoverflow.com\]](https://stackoverflow.com/questions/20934130/owning-side-vs-non-owning-side-in-hibernate-and-its-usage-in-reference-of-mapped)

*   **Spring Data JPA `save()`**  
    Repository `save()` delegates to the JPA `EntityManager` (persist/merge) and does **not** add extra cascade behavior beyond JPA mappings. Repository methods are transactional by default (writes), and reads are `readOnly=true` by default. [\[baeldung.com\]](https://www.baeldung.com/hibernate-mapsid-annotation)

*   **Flush/commit execution**  
    Hibernate performs **dirty checking** and synchronizes managed entities at **flush/commit**, issuing SQL for inserts/updates/deletes accordingly. [\[jakarta.ee\]](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2)

***

## Key Points

1.  **Define ownership correctly**
    *   `Order.customer` → `@ManyToOne` (owning side)
    *   `Customer.orders` → `@OneToMany(mappedBy = "customer")` (inverse side)  
        The owning side controls FK updates; inverse side is informational for JPQL navigation and fetch planning. [\[baeldung.com\]](https://www.baeldung.com/hibernate-one-to-many)

2.  **Keep both sides in sync in memory**  
    JPA requires **you** to maintain consistency of bidirectional references. Set **both**: add to `Customer.orders` **and** call `order.setCustomer(customer)` so the owning side can write the FK. [\[stackoverflow.com\]](https://stackoverflow.com/questions/20934130/owning-side-vs-non-owning-side-in-hibernate-and-its-usage-in-reference-of-mapped)

3.  **Cascade decides propagation, not linkage**  
    `cascade = PERSIST` lets `persist(customer)` also `persist(order)`, but you **still** must set `order.setCustomer(customer)`; otherwise the FK can remain `NULL`. [\[stackoverflow.com\]](https://stackoverflow.com/questions/20934130/owning-side-vs-non-owning-side-in-hibernate-and-its-usage-in-reference-of-mapped)

4.  **Spring transaction boundary**  
    Place `@Transactional` at the **service** method. Spring opens a transaction, binds an `EntityManager` (persistence context), and triggers flush before commit—no extra magic beyond JPA. [\[baeldung.com\]](https://www.baeldung.com/jpa-hibernate-associations)

***

## Example (Spring Boot / JPA)

```java
@Entity
class Customer {
  @Id @GeneratedValue
  private Long id;

  @OneToMany(mappedBy = "customer", cascade = CascadeType.PERSIST) // inverse side
  private List<Order> orders = new ArrayList<>();

  // helper keeps both sides consistent (required by JPA contract)
  public void addOrder(Order o) {
    orders.add(o);          // inverse
    o.setCustomer(this);    // owning side (writes FK)
  }
}

@Entity
class Order {
  @Id @GeneratedValue
  private Long id;

  @ManyToOne(optional = false) // owning side
  private Customer customer;

  public void setCustomer(Customer c) { this.customer = c; }
}

// Service layer
@Service
class CheckoutService {
  private final CustomerRepository customers;

  CheckoutService(CustomerRepository customers) { this.customers = customers; }

  @Transactional
  public Long placeOrder() {
    Customer c = new Customer();
    Order o1 = new Order();
    Order o2 = new Order();

    // Maintain both sides; cascade=PERSIST will propagate persist to orders
    c.addOrder(o1);
    c.addOrder(o2);

    customers.save(c); // delegates to JPA; cascade handles orders
    return c.getId();
  }
}
```

*   `mappedBy` on `Customer.orders` states the owner is `Order.customer`. [\[baeldung.com\]](https://www.baeldung.com/hibernate-one-to-many)
*   Cascade propagates `persist` from `Customer` to its `Order`s, but the FK is written because **`Order.customer`** is set. [\[stackoverflow.com\]](https://stackoverflow.com/questions/20934130/owning-side-vs-non-owning-side-in-hibernate-and-its-usage-in-reference-of-mapped)
*   Spring repository defaults and transaction handling align with JPA: no extra cascade is introduced. [\[baeldung.com\]](https://www.baeldung.com/hibernate-mapsid-annotation), [\[baeldung.com\]](https://www.baeldung.com/jpa-hibernate-associations)

***

## Diagram (Ownership, Cascade, and Persist Flow)

             (inverse)                            (owning)
    Customer ----------------------< Order
       ^  OneToMany (mappedBy)           ManyToOne (writes FK)
       |     cascade=PERSIST             field: customer
       |                                 FK column: orders.customer_id
       |
       |  addOrder(o):
       |    orders.add(o)  // inverse only
       |    o.setCustomer(this)  // OWNING FIELD set -> FK written on flush

    Persist flow (Spring @Transactional):
    -------------------------------------
    service.saveCustomer():
      1) persist(customer)        -- due to cascade=PERSIST -->
      2) persist(order1, order2)  -- owning side has customer set
      3) flush/commit -> SQL:
           INSERT customer(...)
           INSERT orders(..., customer_id=<customer.id>)

*   Owning side = `@ManyToOne`; inverse side uses `mappedBy`; FK updates occur at flush/commit based on the owning side. [\[baeldung.com\]](https://www.baeldung.com/hibernate-one-to-many), [\[jakarta.ee\]](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2)

***

### Mini “what to remember”

*   **Owning side writes FK** → `Order.customer`. Use `mappedBy` on the other side. [\[baeldung.com\]](https://www.baeldung.com/hibernate-one-to-many)
*   **Cascade propagates operations, not links** → always set the owning field. [\[stackoverflow.com\]](https://stackoverflow.com/questions/20934130/owning-side-vs-non-owning-side-in-hibernate-and-its-usage-in-reference-of-mapped)
*   **Spring Data `save()`** honors JPA rules; repos have transactional defaults (reads `readOnly=true`, writes transactional). [\[baeldung.com\]](https://www.baeldung.com/hibernate-mapsid-annotation)

If you want, I can generate a **small test** that prints the exact SQL for:  
(a) both sides set, (b) only inverse side set, (c) only owning side set—so you can see the FK behavior live.
