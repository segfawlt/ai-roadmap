# Monolith vs. Microservices: Principles, Pros & Cons

**Unit 1:** Fundamentals & Refactoring **Topic Code:** FR-ARC-001 **Reading Time:** ~40 minutes

------------------------------------------------------------------------

## Learning Objectives

- Explain the architectural differences between a Monolith and Microservices.

- Analyze the pros and cons of adopting a microservices architecture.

- Define and identify Bounded Contexts within a business domain using Domain-Driven Design (DDD) principles.

- Understand the importance of the ‘Database per Service’ pattern.

- Describe the ‘Strangler Fig’ pattern as a strategy for migrating from a monolith to microservices.

------------------------------------------------------------------------

## Section 1: Concept/overview

### 1.1 Introduction

In the world of software development, choosing the right architecture is one of the most foundational decisions with the most far-reaching influence on a project’s success. Imagine you are building a small e-commerce application. Initially, building all functions—product management, shopping cart, payments, shipping—in a unified block (Monolith) seems like the fastest and simplest way. Everything is in one place, easy to develop and deploy.

However, as your business grows, the application starts to bloat. A small change in the payment module requires you to test and redeploy the entire system, a huge risk. Scaling becomes difficult; you have to clone the entire bulky application just to handle a traffic spike on the product page. New development teams struggle to learn a massive codebase. These are the “pains” that Microservices architecture was born to solve. This topic will help you clearly understand the nature, differences, pros, and cons of these two architectural approaches, while equipping you with key principles and patterns to make correct architectural decisions for future projects.

### 1.2 Formal Definition

- **Monolith (Monolithic Architecture):** An architectural model where the entire application is built and deployed as a single, inseparable unit. Logical components such as the User Interface (UI), business logic, and Data Access Layer are packaged together in one codebase, run in a single process, and typically connect to a single large database.

- **Microservices (Microservices Architecture):** A software architectural style where a large application is composed of a collection of small, independent services. Each microservice is built around a specific business capability, has its own codebase, is deployed independently, and communicates with other services through well-defined APIs (usually HTTP/REST or message queues). Each service can manage its own database.

### 1.3 Analogy

To easily visualize the difference, let’s compare building software to operating a restaurant.

- **Monolith is like a Traditional Restaurant:**

- There is a **single huge kitchen** where all chefs (developers) work together.

- Every dish (features) from appetizers to main courses to desserts is prepared in the same space.

- **Pros:** Communication between chefs is very fast. Easy to manage when the restaurant is small.

- **Cons:** When the restaurant becomes famous and crowded, the kitchen becomes chaotic. A small incident, like a broken oven, can halt the preparation of all dishes. To expand, you have to build another identical huge kitchen.

- **Microservices is like a Food Court:**

- There are many **small, specialized stalls** (microservices), each selling only one type of food (Pizza, Sushi, Bubble Tea).

- Each stall has its own chef, recipe, and equipment. They operate independently.

- **Pros:** If the Pizza stall runs out of dough, you can still buy Sushi. The Bubble Tea stall can upgrade its machines without affecting other stalls. If the Pizza stall is too crowded, the owner can easily open a second Pizza stall right next to it to serve.

- **Cons:** Managing the overall food court is more complex. A common ordering and payment system (API Gateway) is needed so customers don’t get confused.

### 1.4 History of Development

Monolithic architecture was the default approach for decades, especially during the era of client-server and early web applications. It was simple and effective for systems of moderate complexity.

However, around the 2000s, huge tech companies like Amazon, Netflix, and eBay began facing scale issues that monolithic architecture could not solve effectively. They needed to develop faster, with large and distributed teams, while ensuring high system fault tolerance. From these practical needs, they pioneered breaking down their massive monoliths into smaller, independent services. The term “Microservices” became widely popularized around 2011-2012, with influencers like Martin Fowler and James Lewis helping to define and standardize its principles.

------------------------------------------------------------------------

## Section 2: Core Components

### 2.1 Architecture overview

**Monolithic Architecture:**

```
+--------------------------------------------------+
|               Monolithic Application             |
|                                                  |
|  +--------------------------------------------+  |
|  |           User Interface (Web/App)         |  |
|  +--------------------------------------------+  |
|                                                  |
|  +--------------------------------------------+  |
|  |                Business Logic              |  |
|  |  [Products] [Orders] [Users] [Payments]    |  |
|  +--------------------------------------------+  |
|                                                  |
|  +--------------------------------------------+  |
|  |              Data Access Layer             |  |
|  +--------------------------------------------+  |
|                                                  |
+------------------------|-------------------------+
                         |
                         |
           +-------------v-------------+
           |    Single Large Database    |
           +-----------------------------+
```

**Microservices Architecture:**

```
+-----------+      +------------------+      +----------------------+
|  Client   |----->|   API Gateway    |----->|   User Service       |----->(User DB)
+-----------+      | (Routing, Auth)  | |    +----------------------+
                   +------------------+ |
                                        |    +----------------------+
                                        |--->|  Product Service     |----->(Product DB)
                                        |    +----------------------+
                                        |
                                        |    +----------------------+
                                        |--->|   Order Service      |----->(Order DB)
                                             +----------------------+
```

### 2.2 Key Components

These are the core concepts and patterns you need to master when working with microservices.

**Component 1: Bounded Context (Domain-Driven Design)**

- **Definition:** Bounded Context is a central concept of Domain-Driven Design (DDD). It defines a **context boundary** within which a specific domain model has a clear, unambiguous value and meaning. Inside a Bounded Context, each business term (e.g., “Product”) has a unique definition.

- **Role:** Bounded Context helps us “slice” a complex business domain into smaller, more manageable parts. This is the most natural and logical guide for defining the boundaries of a microservice. Instead of splitting services by technical layers (e.g., service for UI, service for DB), we split by business (service for Orders, service for Shipping).

- **Real-world Example:** In an e-commerce system:

- In the **Sales Bounded Context**, a `Product` has attributes like `name`, `description`, `price`, `discount`.

- In the **Shipping Bounded Context**, that same `Product` is viewed from a different angle, with attributes like `weight`, `dimensions`, `isFragile`.

- These two “versions” of the `Product` exist in two different Bounded Contexts and should be managed by two different microservices (`ProductCatalogService` and `ShippingService`).

**Component 2: Database per Service Pattern**

- **Definition:** This is a foundational principle in microservices architecture, stating that each microservice must exclusively own and manage its own database. No other service is allowed to access this database directly. All interactions with data must go through the API of the service that owns it.

- **Role:**

1.  **Ensures Loose Coupling:** Changing the DB schema of one service (e.g., adding a column to the `orders` table) will not affect or break other services.

2.  **Technological Flexibility (Polyglot Persistence):** Each service can choose the database type best suited for its needs. For example, `UserService` might use MySQL (SQL) because it needs strong transactions, while `ProductCatalogService` might use MongoDB (NoSQL) to store flexible product attributes.

- **Example of DB Selection:**

``` sql
-- Order Service (using PostgreSQL for transactional integrity)
CREATE TABLE orders (
    order_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    order_date TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    status VARCHAR(50) NOT NULL
);
```

``` javascript
// Product Service (using MongoDB for flexible product attributes)
db.products.insertOne({
  productId: "SKU-12345",
  name: "Laptop Pro",
  specs: {
    cpu: "M3 Pro",
    ram: "18GB",
    storage: "512GB SSD",
  },
  tags: ["electronics", "apple", "laptop"],
});
```

**Component 3: Strangler Fig Pattern**

- **Definition:** Named after a type of fig tree that grows around and gradually “strangles” the host tree, this is an architectural pattern for gradually migrating from an old Monolith system to a Microservices architecture. The idea is to build a new system (microservices) around the old system, and gradually redirect requests from the old to the new until the old system has no remaining functionality and can be safely removed.

- **Role:** Minimizes the risk of a “big bang rewrite.” Instead of stopping development and rebuilding the entire system over years, this pattern allows businesses to continue operating, add new features to the new system, and move piece by piece safely.

- **How it works:** An intermediary layer called a **Strangler Facade** (usually a Reverse Proxy, API Gateway) is placed in front of the monolith. Initially, it just forwards all requests to the monolith. Later, when a function (e.g., User Profile Management) is rebuilt into a new microservice, the Facade is configured to route requests related to the profile (`/api/profile/*`) to the new service, while other requests still go to the monolith.

- **Example Nginx Configuration as Strangler Facade:**

``` nginx
# nginx.conf
server {
    listen 80;

    # New User Profile microservice is ready
    location /api/v2/profile/ {
        # Route requests to the new microservice
        proxy_pass http://user_profile_service:5001;
    }

    # All other requests are still handled by the old monolith
    location / {
        proxy_pass http://monolith_app:8000;
    }
}
```

### 2.3 Comparing Approaches

| Criteria | Monolith | Microservices |
|----|----|----|
| **Initial Complexity** | **Low.** Start project fast, no need to worry about network communication, distributed systems. | **High.** Need to set up API Gateway, service discovery, CI/CD for each service, handle network errors. |
| **Deployment** | **Simple.** Just deploy a single artifact (WAR, JAR, binary). | **Complex.** Must deploy and manage multiple services. Needs strong automation tools (CI/CD, containerization). |
| **Scalability** | **Difficult.** Must scale the entire application, even parts rarely used. Resource expensive. | **Flexible.** Can scale each service independently based on actual demand. |
| **Reliability** | **Low.** An error in a small module can crash the entire application. | **High.** Error in one service usually doesn’t affect other services (fault isolation). |
| **Tech Flexibility** | **Limited.** Locked into a single tech stack chosen from the start. | **High.** Each service can choose the language, framework, DB best suited for its function. |
| **Development Speed** | **Fast at first, slowing down** as codebase grows and teams must coordinate tightly. | **Slow at first, faster** later when teams can work independently on their services. |
| **When to use** | Small projects, MVP (Minimum Viable Product), small team, simple business domain. | Large, complex systems, requiring high scalability and reliability, large and distributed development teams. |

------------------------------------------------------------------------

## Section 3: Implementation

We will use Python and the Flask framework to illustrate the concepts.

### Level 1 - Basic (Beginner): Building a Simple E-commerce Monolith

This is a single application handling both products and orders.

``` python
# monolith_app.py
from flask import Flask, jsonify

app = Flask(__name__)

# Simulating a single, shared database
DB = {
    "products": {
        "p1": {"name": "Laptop Pro", "price": 1200},
        "p2": {"name": "Wireless Mouse", "price": 50}
    },
    "orders": [
        {"order_id": "o101", "product_id": "p1", "quantity": 1}
    ]
}

@app.route("/products/<product_id>")
def get_product(product_id):
    """Endpoint to get product details."""
    product = DB["products"].get(product_id)
    if not product:
        return jsonify({"error": "Product not found"}), 404
    return jsonify(product)

@app.route("/orders")
def get_orders():
    """Endpoint to list all orders."""
    # Business logic can directly access product data
    orders_with_details = []
    for order in DB["orders"]:
        product = DB["products"].get(order["product_id"])
        orders_with_details.append({
            "order_id": order["order_id"],
            "product_name": product["name"] if product else "Unknown",
            "quantity": order["quantity"]
        })
    return jsonify(orders_with_details)

if __name__ == "__main__":
    # The entire application runs as a single process
    app.run(port=5000, debug=True)
```

**How to run:**

``` bash
pip install Flask
python monolith_app.py
```

**Expected Output:**

- Accessing `http://127.0.0.1:5000/products/p1` will return: `{"name":"Laptop`` ``Pro","price":1200}`

- Accessing `http://127.0.0.1:5000/orders` will return: `[{"order_id":"o101","product_name":"Laptop`` ``Pro","quantity":1}]`

**Common Errors:**

- **Circular Dependencies:** As the app grows, the `orders` module might import the `products` module and vice versa, causing import errors and making the codebase confusing.

- **Tight Coupling:** The logic of `get_orders` directly accesses `products` data. Any change in `products` data structure risks breaking `orders` functionality.

### Level 2 - Intermediate: Splitting Monolith into 2 Microservices

Now, we will split the application above into `ProductService` and `OrderService`. They will run independently and communicate via API.

**Service 1: Product Service**

``` python
# product_service.py
from flask import Flask, jsonify

app = Flask(__name__)

# This service has its OWN database
PRODUCT_DB = {
    "p1": {"name": "Laptop Pro", "price": 1200},
    "p2": {"name": "Wireless Mouse", "price": 50}
}

@app.route("/products/<product_id>")
def get_product(product_id):
    """Provides product information via an API."""
    product = PRODUCT_DB.get(product_id)
    if not product:
        return jsonify({"error": "Product not found"}), 404
    return jsonify(product)

if __name__ == "__main__":
    # Runs on a different port from the order service
    app.run(port=5001, debug=True)
```

**Service 2: Order Service**

``` python
# order_service.py
from flask import Flask, jsonify
import requests # Library to make HTTP requests

app = Flask(__name__)

# This service has its OWN database
ORDER_DB = [
    {"order_id": "o101", "product_id": "p1", "quantity": 1},
    {"order_id": "o102", "product_id": "p99", "quantity": 2} # p99 does not exist
]

# The address of the product service
PRODUCT_SERVICE_URL = "http://127.0.0.1:5001"

@app.route("/orders")
def get_orders():
    """
    Lists orders and enriches them with product data
    by calling the Product Service API.
    """
    enriched_orders = []
    for order in ORDER_DB:
        product_id = order["product_id"]
        product_info = {}
        try:
            # Inter-service communication via API call
            response = requests.get(f"{PRODUCT_SERVICE_URL}/products/{product_id}", timeout=0.5)
            response.raise_for_status() # Raise an exception for bad status codes (4xx or 5xx)
            product_info = response.json()
        except requests.exceptions.RequestException as e:
            # Handle cases where Product Service is down or product not found
            print(f"Could not fetch product {product_id}. Error: {e}")
            product_info = {"name": "Product information unavailable"}

        enriched_orders.append({
            "order_id": order["order_id"],
            "product_name": product_info.get("name", "Unknown"),
            "quantity": order["quantity"]
        })

    return jsonify(enriched_orders)

if __name__ == "__main__":
    # Runs on its own port
    app.run(port=5002, debug=True)
```

**How to run:**

1.  Open terminal 1: `python`` ``product_service.py`

2.  Open terminal 2: `python`` ``order_service.py`

3.  Access `http://127.0.0.1:5002/orders`.

**Expected Output:**

``` json
[
  {
    "order_id": "o101",
    "product_name": "Laptop Pro",
    "quantity": 1
  },
  {
    "order_id": "o102",
    "product_name": "Product information unavailable",
    "quantity": 2
  }
]
```

This example shows that `OrderService` still works (resilience) even when product `p99` information is not found from `ProductService`.

### Level 3 - Advanced: Applying Strangler Fig Pattern for Migration

Assume we have an old monolith, and we want to separate `inventory` functionality into a new service without disrupting the system.

**Step 1: Current Monolith**

``` python
# monolith_v1.py
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/api/users/<user_id>")
def get_user(user_id):
    return jsonify({"user_id": user_id, "name": "Monolith User"})

@app.route("/api/inventory/<item_id>")
def get_inventory(item_id):
    # Old, legacy inventory logic
    return jsonify({"item_id": item_id, "stock": 100, "source": "monolith"})

if __name__ == "__main__":
    app.run(port=8000)
```

**Step 2: New Inventory Microservice**

``` python
# inventory_service.py
from flask import Flask, jsonify

app = Flask(__name__)

@app.route("/api/inventory/<item_id>")
def get_new_inventory(item_id):
    # New, improved inventory logic
    return jsonify({"item_id": item_id, "stock": 999, "source": "microservice"})

if __name__ == "__main__":
    app.run(port=8001)
```

**Step 3: Strangler Facade using Nginx**

Create an `nginx.conf` file to route traffic.

``` nginx
# nginx.conf
events {}

http {
    # Define upstreams for our services
    upstream monolith {
        server host.docker.internal:8000; # Use this for Docker, or 127.0.0.1 for local
    }

    upstream inventory_service {
        server host.docker.internal:8001;
    }

    server {
        listen 8080;

        # The "Strangling" rule
        # If the request path starts with /api/inventory/, route it to the new service.
        location /api/inventory/ {
            proxy_pass http://inventory_service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # Catch-all rule
        # All other requests go to the old monolith.
        location / {
            proxy_pass http://monolith;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

**How to run (using Docker to simplify network):**

1.  Save the 3 files above.

2.  Run the Python applications.

3.  Run Nginx: `docker`` ``run`` ``-p`` ``8080:8080`` ``-v`` ``$(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro`` ``nginx`

4.  Send request to Facade (port 8080):

- `curl`` ``http://localhost:8080/api/users/123` -\> Returns: `{"user_id":"123","name":"Monolith`` ``User"}` (from Monolith)

- `curl`` ``http://localhost:8080/api/inventory/abc` -\> Returns: `{"item_id":"abc","source":"microservice","stock":999}` (from New Microservice)

The system has successfully redirected `inventory` traffic to the new service without affecting the remaining functions.

------------------------------------------------------------------------

## Section 4: Best Practices

### ✅ DO’s

| Practice | Why | Example |
|----|----|----|
| **Design around Business Capabilities** | Create highly cohesive and loosely coupled services that accurately reflect organizational structure. | Instead of `UserService`, `OrderService`, `PaymentService`, think of `CustomerManagement`, `OrderFulfillment`, `Billing`. |
| **Decentralize Everything** | Promote autonomy and resilience. Each team can make decisions quickly without permission from other teams. | `Order` team uses Java & PostgreSQL, `Search` team uses Python & Elasticsearch. Each team has its own CI/CD pipeline. |
| **Embrace Automation (CI/CD)** | Managing dozens or hundreds of services manually is impossible. Automation reduces errors and increases deployment speed. | A commit to the `main` branch of `ProductService` automatically triggers build, test, and deploy to staging. |
| **Implement Robust Monitoring & Logging** | In a distributed system, tracing errors is hard. Centralized logging and distributed tracing are mandatory. | Use tools like Datadog, ELK Stack, or Jaeger to track a request through multiple services and find bottlenecks. |

### ❌ DON’Ts

| Anti-pattern | Consequence | How to avoid |
|----|----|----|
| **Distributed Monolith** | You incur all the complexity of microservices (network, deployment) without the benefits of independence. | Strictly adhere to “Database per Service”. Prioritize asynchronous communication via message queue over synchronous API call chains. |
| **Services Too Small (Nanoservices)** | Operational overhead skyrockets, inter-service communication becomes extremely complex, hard to debug. | Stick to Bounded Context. A service should be large enough to perform a meaningful business function, but small enough for one team to manage. |
| **Shared Libraries/Code** | A small change in a shared library may require rebuilding and redeploying a series of services, losing independence. | Avoid sharing business logic code. If sharing is needed, it should only be very generic utility code (e.g., client for logging service). |
| **Migrate Too Early** | Starting with microservices for a new, unproven product will significantly slow down initial development speed. | Start with a “well-structured monolith”. Only move to microservices when issues of scale and team development speed become clear. |

### 🔒 Security Considerations

- **Service-to-Service Authentication:** Do not trust any traffic within the internal network. Use mechanisms like mTLS (Mutual TLS) or OAuth 2.0 Client Credentials Flow to authenticate API calls between services.

- **API Gateway Security:** API Gateway is the single entry point. Authentication (end-user verification), authorization (permission check), rate limiting (DoS protection), and input validation must be enforced here.

- **Secret Management:** Absolutely do not hard-code credentials (database passwords, API keys) in code or config files. Use dedicated tools like HashiCorp Vault, AWS Secrets Manager, or Azure Key Vault.

### ⚡ Performance Tips

- **Asynchronous Communication:** For tasks not requiring immediate response (e.g., sending email notifications, video processing), use a message queue (RabbitMQ, Kafka). This reduces latency for users and increases system fault tolerance.

- **Caching:** Cache frequently accessed data at multiple layers: in service (in-memory), using cache DB (Redis), or at API Gateway.

- **Circuit Breaker Pattern:** When a service (e.g., `PaymentService`) fails, services calling it will continuously fail, wasting resources. Circuit Breaker will “break the circuit” for these calls for a period, allowing the failed service time to recover and preventing cascading failures.

------------------------------------------------------------------------

## Section 5: Case Study

### 5.1 Scenario

**Company/Project:** “TicketNow”, a rapidly growing online event ticketing platform. **Requirements:**

1.  System must handle extreme traffic spikes when selling tickets for hot events (flash sales).

2.  Accelerate development of new features related to event recommendations and user management.

3.  Allow Marketing team to test promotional campaigns without affecting the core system. **Constraints:** Cannot stop system operations to rewrite. Limited budget, cannot “rip and replace”.

### 5.2 Problem Analysis

TicketNow’s current system is a Monolith application written in Django.

- **Root Cause:** When a major event goes on sale, massive requests flooding the `Booking` module overload the shared database. This causes the entire website, including unrelated pages like user profile or blog, to crash.

- **Inefficiency:** The `Recommendation` team wants to use Python with Machine Learning libraries but is stuck with the monolith’s old tech stack. Every small change requires coordination and review from the entire team, slowing progress.

### 5.3 Solution Design

Management decided to apply a strategy of migrating to Microservices using the Strangler Fig Pattern.

- **Architecture Decision:**

1.  Identify Bounded Contexts: `Booking`, `EventCatalog`, `UserManagement`, `Recommendation`.

2.  Prioritize separating `Booking` first because this is causing the biggest performance problem.

3.  Create a new `BookingService`, using Go language (famous for concurrency performance) and ScyllaDB (NoSQL, designed for high throughput).

4.  Set up an API Gateway (using Kong) as the Strangler Facade.

- **Trade-offs:**

- **Pro:** Solves the core performance problem, allows other teams to develop in parallel.

- **Con:** Increases operational complexity. Needs investment in DevOps and monitoring. Will have a phase of maintaining both old and new systems, potentially requiring complex data synchronization.

### 5.4 Implementation

Below is a simplified example of refactoring the monolith to call the new service.

**Monolith Code (before change):**

``` python
# ticketnow_monolith/booking/views.py

class BookingView:
    def create_booking(self, event_id, user_id, num_tickets):
        # ... Complex booking logic directly accessing the main PostgreSQL DB ...
        # This part is the bottleneck
        with transaction.atomic():
            event = Event.objects.get(id=event_id)
            if event.stock < num_tickets:
                raise InsufficientStockError("Not enough tickets!")
            event.stock -= num_tickets
            event.save()
            Booking.objects.create(user_id=user_id, event_id=event_id, ...)
        return "Booking successful in Monolith"
```

**New Booking Microservice:**

``` go
// booking_service/main.go
package main

// ... (import statements) ...

func createBookingHandler(w http.ResponseWriter, r *http.Request) {
    // New, highly-optimized booking logic using ScyllaDB
    // This service only cares about one thing: creating bookings fast.
    log.Println("Booking created via new Microservice!")
    w.WriteHeader(http.StatusCreated)
    w.Write([]byte(`{"status": "success", "source": "microservice"}`))
}

func main() {
    http.HandleFunc("/api/v2/bookings", createBookingHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

**Monolith Code (after refactoring to “strangle”):**

``` python
# ticketnow_monolith/booking/views.py
import requests

BOOKING_SERVICE_URL = "http://booking-service.internal/api/v2/bookings"

class BookingView:
    def create_booking(self, event_id, user_id, num_tickets):
        # The monolith now delegates the critical task to the new service
        # It no longer handles the high-load database transaction itself.
        try:
            payload = {"eventId": event_id, "userId": user_id, "tickets": num_tickets}
            response = requests.post(BOOKING_SERVICE_URL, json=payload)
            response.raise_for_status()
            return "Booking request forwarded to new service"
        except requests.RequestException as e:
            # Fallback to old logic or return an error
            log.error(f"Failed to call booking service: {e}")
            # return self.create_booking_legacy(...)
            raise ServiceUnavailableError("Booking service is temporarily down.")
```

### 5.5 Results & Lessons Learned

- **Improved Metrics:**

- System handled **10 times** higher concurrent ticket booking requests during flash sales.

- Error rate during peak hours reduced by 95%.

- Time for `Recommendation` team to deploy a new algorithm from idea to production reduced from 1 month to 3 days.

- **Lessons Learned:**

- **Start small:** Choosing `Booking` as the first Bounded Context to separate was a correct decision as it solved the biggest “pain”.

- **Invest in the Facade:** API Gateway is not just for routing, it is also a critical checkpoint for security, monitoring, and traffic management. Investing in a good Gateway saved them many times.

- **Data Synchronization is Hard:** The biggest challenge was keeping data (e.g., event info) consistent between the monolith and the new service during the transition. They had to build temporary data synchronization jobs.

------------------------------------------------------------------------

## References

- [Microservices by Martin Fowler](https://martinfowler.com/articles/microservices.html)

- [Domain-Driven Design (DDD) - Bounded Context Explained](https://martinfowler.com/bliki/BoundedContext.html)

- [Pattern: Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html)

- [Pattern: Database per Service](https://microservices.io/patterns/data/database-per-service.html)
