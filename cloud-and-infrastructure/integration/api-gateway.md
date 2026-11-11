# Implementing API Gateway

**Unit 2:** API Gateway (Kong) **Topic Code:** MS-S2-U2-T1 **Reading Time:** ~40 minutes

------------------------------------------------------------------------

## Learning Objectives

- Explain the role and benefits of an API Gateway in a microservices architecture.

- Describe core API Gateway features: Routing, Authentication, and Rate Limiting.

- Introduce Kong as a popular open-source API Gateway.

- Understand Kong’s core concepts: Services, Routes, Consumers, and Plugins.

- Outline the steps to set up and configure Kong for basic routing.

------------------------------------------------------------------------

## Section 1: Concept/overview

### 1.1 Introduction

In traditional Monolithic architecture, the client (web/mobile app) only needs to communicate with a single backend. Everything is quite simple. However, as systems grow and transition to Microservices architecture, we face a major problem: instead of one backend, the client now has to communicate with dozens, or even hundreds, of different microservices.

This creates a series of challenges:

- **Client Complexity:** The client must know the address of each service, handle different authentication mechanisms for each service, and the logic for calling APIs becomes complex.

- **Security:** Each service must implement its own security mechanisms (authentication, authorization, rate limiting). This leads to code duplication and difficulty in consistent management.

- **Management & Monitoring:** Monitoring, logging, and metrics collection become dispersed and difficult. When a request fails, tracing through multiple services is a nightmare.

- **Refactoring:** If you want to change the internal structure of microservices (e.g., merging 2 services into one, or splitting one service into three), all clients must update their API calling logic.

API Gateway was born to solve exactly these problems. It acts as a single “gatekeeper” for all requests from the client to the microservices system, helping to simplify, secure, and manage the system effectively.

### 1.2 Formal Definition

An **API Gateway** is a server that acts as a single entry point into a system. It is a reverse proxy that accepts all application programming interface (API) calls, aggregates the various services required to fulfill them, and returns the appropriate result.

Simply put, an API Gateway is an intermediary layer located between the client and your microservices. All requests from the outside must go through the Gateway before reaching the internal services. The Gateway is responsible for handling “cross-cutting concerns” (common issues that many services care about) such as:

- **Routing:** Routing requests to the correct microservice.

- **Authentication & Authorization:** Authenticating users and checking permissions.

- **Rate Limiting:** Limiting the number of requests from a client within a period.

- **Load Balancing:** Distributing load among multiple instances of a service.

- **Caching:** Storing frequently requested responses to reduce load on the backend.

- **Logging & Monitoring:** Logging and collecting data on all requests.

- **Protocol Translation:** Converting between protocols (e.g., from REST to gRPC).

### 1.3 Analogy

Imagine a large office building with hundreds of different companies (microservices).

- **Without API Gateway:** A client wants to meet an employee at company A, then pick up documents at company B, and finally verify papers at company C. This guest has to find out for themselves which floor and room company A is on; where company B is; where company C is. Each time they visit a company, they have to present identification (authentication) and comply with that company’s specific regulations. Very complicated and time-consuming.

- **With API Gateway:** This building has a professional reception desk (API Gateway) in the main lobby. The client just needs to go to the reception desk and state their request (“I want to meet A, pick up document B, verify paper C”). The receptionist will:

1.  **Authentication:** Check the client’s identity once.

2.  **Routing:** Direct or connect the client to the correct companies A, B, C in the correct order.

3.  **Monitoring/Logging:** Record guest information entering and leaving the building.

4.  **Control (Rate Limiting):** Ensure not too many people enter a small company at the same time to avoid overcrowding.

The reception desk has hidden the entire complexity of the building’s internal structure, providing a single, safe, and effective interaction point for the client. That is exactly the role of the API Gateway.

### 1.4 History

**Kong** was created by the company **Mashape** in 2015 and later released as open source. Mashape later renamed itself to **Kong Inc.** to focus entirely on developing this product.

Kong is built on the high-performance web server **Nginx** and uses **LuaJIT** (an extremely fast Lua interpreter) allowing customization and extension through a Plugin system. This combination gives Kong superior performance and extremely low latency, making it one of the most popular and trusted open-source API Gateways today in large-scale microservices systems.

------------------------------------------------------------------------

## Section 2: Core Components

### 2.1 Architecture overview

Kong’s architecture is quite simple but powerful. It consists of two main components: **Kong Proxy** and **Admin API**.

``` mermaid
        graph TD
    Clients["Clients<br>(Web, Mobile, IoT, Partners)"]
    Clients -->|"Public Traffic<br>Port 80, 443"| Proxy

    subgraph Kong["KONG API GATEWAY"]
        Proxy["Kong Proxy<br>(Nginx — Port 8000)"]
        Plugins["Plugins<br>(Auth, Rate Limit, Logging)"]
        Admin["Admin API<br>(RESTful — Port 8001)"]
        DB["Data Store<br>(PostgreSQL, Cassandra)"]
        Plugins --> Proxy
        Admin --> DB
        DB --> Proxy
        DB --> Plugins
    end

    Proxy -->|Upstream Traffic| A["Microservice A"]
    Proxy -->|Upstream Traffic| B["Microservice B"]
    Proxy -->|Upstream Traffic| C["Microservice C"]
```

- **Kong Proxy (Port 8000/8443):** This is the main gate handling all traffic from the client. It receives requests, applies configured Plugins (like authentication, rate limit), then routes requests to the corresponding microservices (upstream services).

- **Admin API (Port 8001):** This is a RESTful API that allows you to configure everything in Kong: create Services, Routes, add Plugins, manage Consumers… All changes you make via the Admin API will be saved to the Data Store and applied immediately to the Kong Proxy.

- **Data Store:** Kong needs a database to store its configuration. It supports PostgreSQL and Cassandra. Kong also has a “DB-less” mode, where configuration is stored in a YAML or JSON file, very suitable for CI/CD and GitOps.

### 2.2 Key Components

To use Kong, you need to understand its 4 core concepts.

**Component 1: Service**

- **Definition:** A `Service` is an entity in Kong representing an upstream API or a microservice of yours. Example: `user-service`, `product-service`, `order-service`.

- **Role:** `Service` contains information on how to connect to that microservice, including protocol (http/https), host, port, and path. It abstracts your physical microservice.

- **Syntax (using Admin API):**

``` bash
# Create a Service that points to a user management microservice
# running at http://user-api:5000
curl -i -X POST http://localhost:8001/services/ \
  --data name=user-service \
  --data url='http://user-api:5000'
```

**Component 2: Route**

- **Definition:** A `Route` defines rules to match requests coming from the client and forward them to a specific `Service`. A `Service` can have multiple `Routes`.

- **Role:** `Route` is the “door” for clients to access the `Service`. You can route based on host, path, HTTP method, header….

- **Syntax:**

``` bash
# Create a Route that maps requests with path "/api/users"
# to the 'user-service' we created earlier.
curl -i -X POST http://localhost:8001/services/user-service/routes \
  --data 'paths[]=/api/users' \
  --data name=user-service-route
```

Now, any request to Kong Proxy at `http://localhost:8000/api/users` will be forwarded to `http://user-api:5000`.

**Component 3: Consumer**

- **Definition:** A `Consumer` represents a user or a client application using your API.

- **Role:** `Consumer` is very important for authentication and tracking. You can attach Plugins (like API key, JWT) to a `Consumer` to manage access rights and apply rate limiting policies specifically for each user/application.

- **Syntax:**

``` bash
# Create a Consumer representing a mobile application
curl -i -X POST http://localhost:8001/consumers/ \
  --data username=mobile-app-v1 \
  --data custom_id=app-uuid-1234
```

**Component 4: Plugin**

- **Definition:** `Plugin` is the heart of Kong. They are modules that provide additional features to your API before the request is forwarded to the upstream service.

- **Role:** Execute cross-cutting concerns. Kong has a rich Plugin ecosystem: Key Authentication, Rate Limiting, JWT, OAuth 2.0, Logging, CORS, etc. You can apply Plugins globally, for a `Service`, a `Route`, or a `Consumer`.

- **Syntax:**

``` bash
# Enable the rate-limiting plugin on the 'user-service-route'
# This limits clients to 5 requests per minute
curl -i -X POST http://localhost:8001/routes/user-service-route/plugins \
    --data "name=rate-limiting" \
    --data "config.minute=5" \
    --data "config.policy=local"
```

### 2.3 Comparison of Approaches

There are many API Gateway solutions on the market. Below is a comparison between Kong and a few popular competitors.

| Approach | Pros | Cons | When to use |
|----|----|----|----|
| **Kong (Open-Source)** | Extremely high performance (Nginx-based), low latency. Rich, flexible plugin ecosystem. Large community. Easy to install and scale. | Admin interface (Kong Manager) only available in Enterprise version. Need to manage infrastructure yourself. | When high performance, high customization is needed, and full infrastructure control is desired. Suitable for both startups and large enterprises. |
| **AWS API Gateway** | Fully managed service. Deep integration with AWS ecosystem (Lambda, IAM). Auto-scaling. Pay-as-you-go. | Cost can rise high with large request volume. Locked into AWS ecosystem (vendor lock-in). Less flexible than Kong. | When already using AWS ecosystem and wanting a solution without infrastructure management. Suitable for serverless applications. |
| **Apigee (Google Cloud)** | Provides comprehensive API management toolkit (API lifecycle, analytics, monetization). Strong security. | Complex and expensive, usually aimed at large enterprises. Takes time to learn and implement. | When a comprehensive enterprise-grade API management solution is needed, especially features for analytics and API monetization. |
| **Tyk (Open-Source)** | Easy to use, has free admin interface. Good GraphQL support. Written in Go, no complex dependencies. | Performance might not equal Kong in high-load benchmarks. Smaller community than Kong. | When needing an easy-to-install solution, available free UI, and good GraphQL support. Suitable for small and medium teams. |

------------------------------------------------------------------------

## Section 3: Implementation

### Level 1 - Basic (Beginner)

**Goal:** Install Kong and a mock service using Docker, then configure a basic `Service` and `Route` to forward requests.

**Step 1: Create `docker-compose.yml` file**

Create a file named `docker-compose.yml` with the following content:

``` yaml
# docker-compose.yml
version: "3.7"

services:
  # Mock service to act as our backend API
  mock-api:
    image: mockserver/mockserver:mockserver-5.13.2
    environment:
      MOCKSERVER_INITIALIZATION_JSON_PATH: /config/initializer.json
    volumes:
      - ./mock-config:/config
    networks:
      - kong-net

  # Kong database (PostgreSQL)
  kong-db:
    image: postgres:9.6
    environment:
      - POSTGRES_USER=kong
      - POSTGRES_DB=kong
      - POSTGRES_PASSWORD=kong
    networks:
      - kong-net
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "kong"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Kong database migration
  kong-migration:
    image: kong:2.8
    depends_on:
      kong-db:
        condition: service_healthy
    environment:
      - KONG_DATABASE=postgres
      - KONG_PG_HOST=kong-db
      - KONG_PG_USER=kong
      - KONG_PG_PASSWORD=kong
    command: "kong migrations bootstrap"
    networks:
      - kong-net

  # Kong Gateway
  kong:
    image: kong:2.8
    depends_on:
      kong-migration:
        condition: service_completed_successfully
    environment:
      - KONG_DATABASE=postgres
      - KONG_PG_HOST=kong-db
      - KONG_PG_USER=kong
      - KONG_PG_PASSWORD=kong
      - KONG_PROXY_ACCESS_LOG=/dev/stdout
      - KONG_ADMIN_ACCESS_LOG=/dev/stdout
      - KONG_PROXY_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_LISTEN=0.0.0.0:8001
    ports:
      # Port for client requests to the gateway
      - "8000:8000"
      # Port for the Admin API
      - "8001:8001"
    networks:
      - kong-net

networks:
  kong-net:
    driver: bridge
```

**Step 2: Create configuration file for mock service** Create a directory `mock-config` and inside create file `initializer.json`:

``` json
// mock-config/initializer.json
[
  {
    "httpRequest": {
      "method": "GET",
      "path": "/products/123"
    },
    "httpResponse": {
      "statusCode": 200,
      "headers": {
        "Content-Type": ["application/json"]
      },
      "body": {
        "id": 123,
        "name": "Awesome Gadget",
        "price": 99.99
      }
    }
  }
]
```

**Step 3: Launch environment** Open terminal and run command:

``` bash
docker-compose up -d
```

**Step 4: Configure Kong**

1.  **Create a Service** pointing to our `mock-api`.

``` bash
# Create a Service named 'product-service' pointing to our mock API
curl -i -X POST http://localhost:8001/services/ \
  --data name=product-service \
  --data host=mock-api \
  --data port=1080
```

1.  **Create a Route** to expose this service externally via path `/api/products`.

``` bash
# Create a Route that maps requests with the path '/api/products'
# to the 'product-service'
curl -i -X POST http://localhost:8001/services/product-service/routes \
  --data 'paths[]=/api/products' \
  --data strip_path=true \
  --data name=products-route
# 'strip_path=true' means Kong will remove '/api/products' from the path
# before forwarding the request to the upstream service.
# So, '/api/products/123' becomes '/123'. This is a common pattern.
```

**Step 5: Verify result** Send a request to **Kong Proxy (port 8000)**, not `mock-api` directly.

``` bash
curl -i http://localhost:8000/api/products/123
```

**Expected Output:**

```
HTTP/1.1 200 OK
Content-Type: application/json
...
Connection: keep-alive

{"id":123,"name":"Awesome Gadget","price":99.99}
```

Success! Your request went through Kong, was routed correctly to `mock-api`, and you received the response.

**Common Errors:**

- \*\*Error 1: `{"message":"no`` ``route`` ``and`` ``no`` ``API`` ``found`` ``with`` ``those`` ``values"}**`

- **Description:** This error occurs when you call Kong Proxy but no `Route` matches your request (e.g., wrong path).

- **Fix:** Check the `paths` you configured in `Route` again. Ensure the path in your request matches one of the defined `paths`.

- \*\*Error 2: `curl:`` ``(7)`` ``Failed`` ``to`` ``connect`` ``to`` ``localhost`` ``port`` ``8001:`` ``Connection`` ``refused**`

- **Description:** This error means Kong’s Admin API is not ready or the Kong container failed.

- **Fix:** Run `docker-compose`` ``logs`` ``kong` to view error logs. Possibly the database container hadn’t finished starting when Kong connected.

### Level 2 - Intermediate

**Goal:** Secure the API in Level 1 by requiring an API Key and applying Rate Limiting.

**Step 1: Enable `key-auth` plugin** We will require clients to provide an `apikey` in the header to access `product-service`.

``` bash
# Enable the 'key-auth' plugin on the 'product-service'
curl -i -X POST http://localhost:8001/services/product-service/plugins \
  --data name=key-auth
```

Now, if you try the old request again, you will be denied.

``` bash
curl -i http://localhost:8000/api/products/123
```

**Expected Output:**

```
HTTP/1.1 401 Unauthorized
...

{"message":"No API key found in request"}
```

**Step 2: Create Consumer and API Key**

1.  Create a `Consumer` representing a “mobile app”.

``` bash
# Create a consumer for our mobile app
curl -i -X POST http://localhost:8001/consumers/ \
  --data username=mobile_app
```

1.  Create an API key for this `Consumer`.

``` bash
# Create an API key for the 'mobile_app' consumer
curl -i -X POST http://localhost:8001/consumers/mobile_app/key-auth \
  --data key=super-secret-key-123
```

**Step 3: Send request with API Key** Now, add `apikey` to the request header.

``` bash
curl -i http://localhost:8000/api/products/123 \
  -H "apikey: super-secret-key-123"
```

**Expected Output:**

```
HTTP/1.1 200 OK
...

{"id":123,"name":"Awesome Gadget","price":99.99}
```

Request succeeded again!

**Step 4: Apply Rate Limiting** Now, limit `mobile_app` to send only 3 requests per minute.

``` bash
# Enable rate-limiting plugin for the 'mobile_app' consumer on the 'products-route'
curl -i -X POST http://localhost:8001/routes/products-route/plugins \
    --data "name=rate-limiting" \
    --data "consumer.username=mobile_app" \
    --data "config.minute=3" \
    --data "config.policy=local"
```

Try calling the API 4 times in a row very quickly:

``` bash
for i in {1..4}; do curl -i http://localhost:8000/api/products/123 -H "apikey: super-secret-key-123"; sleep 1; done
```

**Expected Output:** After 3 successful calls (HTTP 200), the 4th call you will receive:

```
HTTP/1.1 429 Too Many Requests
...

{"message":"API rate limit exceeded"}
```

### Level 3 - Advanced

**Goal:** Manage Kong configuration “declaratively” using a YAML file, instead of using manual `curl` commands. This is best practice for production environments and CI/CD.

**Step 1: Update `docker-compose.yml` to use DB-less mode** We will remove the database and migration, instead just let Kong read configuration from a file.

``` yaml
# docker-compose.yml (updated for DB-less)
version: "3.7"

services:
  # Mock service remains the same
  mock-api:
    image: mockserver/mockserver:mockserver-5.13.2
    environment:
      MOCKSERVER_INITIALIZATION_JSON_PATH: /config/initializer.json
    volumes:
      - ./mock-config:/config
    networks:
      - kong-net

  # Kong Gateway in DB-less mode
  kong:
    image: kong:2.8
    volumes:
      # Mount the declarative config file into the container
      - ./kong-config:/usr/local/kong/declarative
    environment:
      # Set mode to 'db_less'
      - KONG_DATABASE=off
      # Specify the path to the config file
      - KONG_DECLARATIVE_CONFIG=/usr/local/kong/declarative/kong.yml
      - KONG_PROXY_ACCESS_LOG=/dev/stdout
      - KONG_ADMIN_ACCESS_LOG=/dev/stdout
      - KONG_PROXY_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_ERROR_LOG=/dev/stderr
      - KONG_ADMIN_LISTEN=0.0.0.0:8001
    ports:
      - "8000:8000"
      - "8001:8001"
    networks:
      - kong-net

networks:
  kong-net:
    driver: bridge
```

\_Note: You need to remove old containers before re-running: `docker-compose`` ``down`` ``-v_`

\*\*Step 2: Create configuration file `kong.yml**`Create a directory`kong-config`and inside create file`kong.yml`. This file will define all `Service`, `Route`, `Consumer`, `Plugin` that we created in previous levels.

``` yaml
# kong-config/kong.yml
# This is the single source of truth for Kong's configuration

_format_version: "2.1"
_comment: "Declarative configuration for our e-commerce API"

services:
  - name: product-service
    url: http://mock-api:1080
    routes:
      - name: products-route
        paths:
          - /api/products
        strip_path: true
    plugins:
      - name: key-auth
        config:
          key_names:
            - apikey

consumers:
  - username: mobile_app
    keyauth_credentials:
      - key: ${MOBILE_APP_API_KEY} # Use environment variable for secrets

plugins:
  - name: rate-limiting
    route: products-route
    consumer: mobile_app
    config:
      minute: 3
      policy: local
```

**Step 3: Launch and check**

1.  Set an environment variable for the API key to avoid hardcoding secrets.

``` bash
export MOBILE_APP_API_KEY="a-very-secure-key-from-env"
```

1.  Relaunch Docker Compose. Kong will read the `kong.yml` file and self-configure on startup.

``` bash
docker-compose down -v && docker-compose up -d
```

1.  Re-check if everything works as before.

``` bash
# This request should succeed
curl -i http://localhost:8000/api/products/123 -H "apikey: a-very-secure-key-from-env"

# This request should fail
curl -i http://localhost:8000/api/products/123 -H "apikey: wrong-key"
```

This approach is much more powerful: configuration is managed in Git, easy to review, and consistently deployable across different environments.

------------------------------------------------------------------------

## Section 4: Best Practices

### ✅ DO’s - Recommended

| Practice | Why | Example |
|----|----|----|
| **Declarative Configuration** | Helps manage configuration as code (Infrastructure as Code), easy to integrate with CI/CD, review, and ensure consistency between environments. | Use `kong.yml` (DB-less mode) and store in Git repository. |
| **Secure the Admin API** | Admin API has full control over Kong. If exposed, attackers can sabotage your entire API system. | Only expose Admin API in internal network (private network), or require authentication (RBAC in Enterprise version). In Docker, do not map port `8001` to public. |
| **Centralize Logging** | Centralize logs from Kong and all microservices in one place (e.g., ELK Stack, Splunk) for easy monitoring, debugging, and analysis. | Use Kong’s logging plugins (like `http-log`, `tcp-log`, `file-log`) to push logs to a log aggregator. |
| **Version Your APIs** | Use path-based versioning (e.g., `/api/v1/products`, `/api/v2/products`) to manage different API versions clearly. | Create different `Routes` for each version, which can point to the same `Service` or different `Services`. |

### ❌ DON’Ts - Avoid

| Anti-pattern | Consequence | How to avoid |
|----|----|----|
| **Putting Business Logic in Gateway** | Gateway becomes complex, hard to maintain, and becomes a “bottleneck”. It violates the principle “Gateway should only handle cross-cutting concerns”. | Keep logic in Gateway simple (routing, auth, rate-limit). Business logic must reside in microservices. |
| **Creating a Single “God” Gateway** | A single Gateway for the entire company with hundreds of services will become bloated, hard to manage, and an issue can affect everyone. | Apply “Gateway per Team” or “Gateway per Domain” pattern. Each team/business domain will manage its own Gateway. |
| **Hardcoding Secrets in Configuration** | Exposing sensitive info (API keys, credentials) in config file or code poses serious security risk. | Use environment variables, Kong Vaults, or secret management systems (like HashiCorp Vault, AWS Secrets Manager) to inject secrets at runtime. |

### 🔒 Security Considerations

- **HTTPS Everywhere:** Always use TLS/SSL to encrypt traffic between client and Kong, as well as between Kong and upstream services.

- **Validate Inputs:** Use plugins like `request-validator` to check input data (request body, params, headers) right at the Gateway, preventing invalid requests from reaching the backend.

- **Least Privilege Principle:** Only enable truly necessary plugins. Each `Consumer` should only have access to `Routes` they are allowed.

- **Regularly Update Kong:** Always update to the latest Kong version to receive security patches and performance improvements.

### ⚡ Performance Tips

- **Choose Plugins Wisely:** Some plugins can affect performance (especially those requiring external calls). Benchmark and use only what you need.

- **Enable Caching:** Use `proxy-cache` plugin to cache responses of GET requests that don’t change often. This significantly reduces load on upstream services.

- **Horizontal Scaling:** Kong is designed to scale horizontally. When load increases, just add new Kong nodes to the cluster and put a Load Balancer in front of them.

- **DB-less Mode:** For systems requiring extremely low latency and high resilience, DB-less mode usually offers better performance as it doesn’t require database round-trips to fetch configuration for each request.

------------------------------------------------------------------------

## Section 5: Case Study

### 5.1 Scenario

**Company/Project:** “FoodieApp” - a food delivery startup. **Requirements:**

1.  System moving from Monolith to Microservices: `user-service`, `restaurant-service`, `order-service`.

2.  Mobile client app needs a single endpoint to communicate.

3.  Need to protect APIs, only allow registered clients (with API key) access.

4.  `order-service` is sensitive API, only logged-in users (with JWT) can access.

5.  Prevent API spamming from any client.

**Constraints:**

- Small team size, needs solution easy to deploy and manage.

- Limited budget, prefers open-source solutions.

- System needs scalability for future.

### 5.2 Problem Analysis

Initially, mobile dev team had to call 3 different APIs directly: `users.foodieapp.com`, `restaurants.foodieapp.com`, `orders.foodieapp.com`.

- **Complex Logic:** Client has to manage 3 base URLs, 3 different authentication mechanisms (API key for `restaurant`, JWT for `order`).

- **Inconsistent Security:** Each service implements rate limiting itself, leading to difficulty in overall management.

- **Hard to Develop:** When `order-service` is updated, mobile team has to change code and re-release app.

This is a classic case showing the need for an API Gateway. Kong was chosen because it is open-source, high-performance, and meets all requirements.

### 5.3 Solution Design

Use Kong as central API Gateway.

- **Single Entry Point:** `api.foodieapp.com`

- **Routing:**

- `api.foodieapp.com/users/**` -\> `user-service`

- `api.foodieapp.com/restaurants/**` -\> `restaurant-service`

- `api.foodieapp.com/orders/**` -\> `order-service`

- **Security Plugins:**

- **Global:** Plugin `rate-limiting` applied to all requests to prevent spam (e.g., 100 requests/minute/IP).

- **`restaurant-service`:** Plugin `key-auth` to authenticate partner clients (e.g., food review apps).

- **`order-service`:** Plugin `jwt` to authenticate end-users.

``` mermaid
        graph LR
    Mobile["Mobile App"] --> Kong
    Partner["Partner App"] --> Kong

    Kong["Kong API Gateway<br>(api.foodieapp.com)<br>1. Rate Limiting — Global<br>2. Key-Auth — Restaurants<br>3. JWT — Orders"]

    Kong --> US["user-service"]
    Kong --> RS["restaurant-service"]
    Kong --> OS["order-service"]
```

### 5.4 Implementation

Below is the `kong.yml` file describing the entire configuration for this scenario.

``` yaml
# kong.yml for FoodieApp
_format_version: "2.1"

services:
  - name: user-service
    url: http://user-service.internal:8080
    routes:
      - name: users-route
        paths: ["/users"]
        strip_path: true

  - name: restaurant-service
    url: http://restaurant-service.internal:8080
    routes:
      - name: restaurants-route
        paths: ["/restaurants"]
        strip_path: true
    plugins:
      - name: key-auth # Require API key for this service

  - name: order-service
    url: http://order-service.internal:8080
    routes:
      - name: orders-route
        paths: ["/orders"]
        strip_path: true
    plugins:
      - name: jwt # Require JWT for this service

consumers:
  - username: partner-review-app
    keyauth_credentials:
      - key: partner-secret-key-xyz

# Global plugin applied to all requests
plugins:
  - name: rate-limiting
    config:
      minute: 100
      policy: local
```

### 5.5 Results & Lessons Learned

- **Improved Metrics:**

- **Development Time:** Time for mobile team to integrate a new feature reduced by 30% because they only need to work with a single and consistent API endpoint.

- **Security Incidents:** Reduced spam/brute-force attack requests to nearly 0 thanks to `rate-limiting` and `bot-detection` at Gateway.

- **API Latency:** Average request latency did not increase significantly (\< 5ms) after passing through Kong, proving the solution’s high performance.

- **Key Takeaways:**

- **Start with Gateway:** With new microservices projects, deploy API Gateway from the start. Adding it later will require much more refactoring effort.

- **Configuration as Code is Key:** Using `kong.yml` and Git helped the team manage configuration transparently, securely, and automate the deployment process.

- **Don’t Over-complicate:** Only apply truly necessary plugins. Adding too many plugins can increase latency and system complexity.

------------------------------------------------------------------------

## References

- [Kong Documentation](https://docs.konghq.com/)

- [Microservices Pattern: API Gateway](https://microservices.io/patterns/apigateway.html)

- [Kong Gateway on GitHub](https://github.com/Kong/kong)
