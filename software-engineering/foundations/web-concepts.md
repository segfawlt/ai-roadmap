# Web Concepts

This page introduces the foundational concepts behind the Web, including how clients and servers communicate over HTTP, the structure of URLs, and the role of requests and responses in building web applications.

## What is the Web?

Topic cover:

- Definition of the World Wide Web.

- Difference between the Internet and the Web.

- How clients and servers communicate.

### **1. Definition of the World Wide Web**

- The **World Wide Web (WWW)** is a system of interlinked hypertext documents accessed via the Internet.

- It allows users to view and interact with web pages containing text, images, videos, and other multimedia.

- Invented by **Tim Berners-Lee** in 1989, the Web uses **HTTP** as its communication protocol.

### **2. Difference Between the Internet and the Web**

- **Internet**: The global network of interconnected computers and devices. It provides the infrastructure for communication.

- **Web**: A service that runs on the Internet, enabling access to websites and web applications.

- Analogy: The Internet is like a **road system**, and the Web is like **cars and destinations** that use those roads.

### **3. How Clients and Servers Communicate**

- **Client**: Usually a web browser or mobile app that requests resources.

- **Server**: A machine that hosts websites or APIs and responds to client requests.

- Communication happens via the **HTTP protocol**:

  - **Client sends a request** (e.g., `GET`` ``/index.html`).

  - **Server processes the request** and sends back a response (e.g., HTML page or JSON data).

- This is called the **Request-Response Cycle**.

**Diagram Idea for Slides:**

```
[Client (Browser)] ---> HTTP Request ---> [Server]
[Server] ---> HTTP Response ---> [Client]
```

------------------------------------------------------------------------

## HTTP Basics

Topic cover:

- What is HTTP and HTTPS?

- Request-Response cycle.

- Common HTTP methods: GET, POST, PUT, DELETE.

- Status codes (2xx, 3xx, 4xx, 5xx).

### **1. What is HTTP and HTTPS?**

- **HTTP (HyperText Transfer Protocol)**:

  - A protocol used for communication between clients (browsers) and servers.

  - Operates over **port 80** by default.

  - Data is sent in plain text, which makes it less secure.

- **HTTPS (HTTP Secure)**:

  - Secure version of HTTP using **SSL/TLS encryption**.

  - Operates over **port 443**.

  - Ensures confidentiality, integrity, and authentication of data.

### **2. Request-Response Cycle**

- The fundamental communication pattern of the Web:

  1.  **Client sends a request** to the server (e.g., `GET`` ``/index.html`).

  2.  **Server processes the request** and prepares a response.

  3.  **Server sends back a response** (HTML, JSON, XML, etc.).

- Example:

  ```
  Client: GET /products HTTP/1.1
  Server: 200 OK
  Content-Type: application/json
  {"id": 1, "name": "Laptop"}
  ```

### **3. Common HTTP Methods**

- **GET**: Retrieve data from the server.

- **POST**: Send data to the server (e.g., create a resource).

- **PUT**: Update an existing resource.

- **DELETE**: Remove a resource.

- Other methods: **PATCH**, **HEAD**, **OPTIONS**.

### **4. Status Codes**

- **2xx – Success**

  - `200`` ``OK`: Request succeeded.

  - `201`` ``Created`: Resource created successfully.

- **3xx – Redirection**

  - `301`` ``Moved`` ``Permanently`: Resource moved to a new URL.

  - `302`` ``Found`: Temporary redirect.

- **4xx – Client Errors**

  - `400`` ``Bad`` ``Request`: Invalid request syntax.

  - `401`` ``Unauthorized`: Authentication required.

  - `404`` ``Not`` ``Found`: Resource not found.

- **5xx – Server Errors**

  - `500`` ``Internal`` ``Server`` ``Error`: Generic server error.

  - `503`` ``Service`` ``Unavailable`: Server overloaded or down.

## REST Architecture

Topic cover:

- Principles of REST.

- Resources and endpoints.

- Statelessness and scalability.

Here’s a clear and structured content section for **“REST Architecture”**:

### **1. Principles of REST**

- **REST (Representational State Transfer)** is an architectural style for designing networked applications.

- Key principles:

  - **Client-Server**: Separation of concerns between client and server.

  - **Stateless**: Each request contains all necessary information; server does not store client state.

  - **Cacheable**: Responses should indicate if they can be cached to improve performance.

  - **Uniform Interface**: Consistent way to interact with resources (using HTTP methods).

  - **Layered System**: Architecture can have multiple layers (e.g., load balancers, proxies).

  - **Code on Demand (optional)**: Servers can send executable code to clients.

### **2. Resources and Endpoints**

- **Resource**: Any object or data that can be accessed via a URL (e.g., user, product).

- **Endpoint**: A specific URL that represents a resource or collection of resources.

- Example:

  - Resource: `User`

  - Endpoint: `/users` (collection), `/users/{id}` (specific user)

- Resources are typically represented in **JSON** format for easy data exchange.

### **3. Statelessness and Scalability**

- **Statelessness**:

  - The server does not keep track of client sessions.

  - Each request is independent and contains all necessary context.

- **Why it matters**:

  - Easier to scale horizontally (add more servers).

  - Simplifies server design and improves reliability.

- **Scalability**:

  - REST’s stateless nature allows multiple servers to handle requests without shared session data.

  - Combined with caching, this improves performance for large-scale systems.

------------------------------------------------------------------------

## APIs

Topic cover:

- What is an API?

- Types of APIs: REST, GraphQL, SOAP.

- Why APIs are important in modern applications.

Here’s a well-structured content section for **“APIs”** that you can use in your lecture:

### **1. What is an API?**

- **API (Application Programming Interface)**:

  - A set of rules and protocols that allow different software applications to communicate.

  - Acts as a **bridge** between systems, enabling data exchange and functionality sharing.

- Example:

  - When you use a weather app, it calls an API to fetch real-time weather data from a server.

### **2. Types of APIs**

- **REST (Representational State Transfer)**:

  - Uses HTTP methods (`GET`, `POST`, `PUT`, `DELETE`).

  - Data usually exchanged in **JSON** format.

  - Simple, scalable, and widely used.

- **GraphQL**:

  - Query language for APIs developed by Facebook.

  - Allows clients to request **exact data** they need.

  - Reduces over-fetching and under-fetching of data.

- **SOAP (Simple Object Access Protocol)**:

  - XML-based protocol.

  - More rigid and heavyweight compared to REST.

  - Often used in enterprise systems requiring strict standards.

### **3. Why APIs Are Important in Modern Applications**

- **Integration**:

  - Connects different services (e.g., payment gateways, social media).

- **Scalability**:

  - Enables modular architecture for large systems.

- **Efficiency**:

  - Reduces duplication by reusing existing services.

- **Innovation**:

  - Allows developers to build new apps on top of existing platforms (e.g., Google Maps API).

## JSON and Data Exchange

Topic cover:

- Why JSON is widely used.

- Example of JSON structure.

- Comparison with XML.

Here’s a structured and detailed content section for **“JSON and Data Exchange”**:

### **1. Why JSON is Widely Used**

- **JSON (JavaScript Object Notation)** is a lightweight data-interchange format.

- Key reasons for popularity:

  - **Human-readable** and easy to understand.

  - **Language-independent** but uses conventions familiar to programmers.

  - **Lightweight** compared to XML (less verbose).

  - **Native support** in JavaScript and widely supported in other languages.

- Commonly used in **APIs**, **web services**, and **data storage**.

### **2. Example of JSON Structure**

``` json
{
  "id": 101,
  "name": "Laptop",
  "price": 1200.5,
  "in_stock": true,
  "tags": ["electronics", "computers"]
}
```

- **Key-Value pairs** represent data.

- Supports:

  - Strings, numbers, booleans.

  - Arrays and nested objects.

### **3. Comparison with XML**

| Feature | JSON | XML |
|----|----|----|
| **Format** | Lightweight, less verbose | Verbose with opening/closing tags |
| **Readability** | Easier for humans | Harder to read |
| **Data Types** | Supports arrays, numbers, booleans | Everything is text |
| **Usage** | Modern APIs, web apps | Legacy systems, SOAP services |

- **Why JSON \> XML for APIs**:

  - Faster parsing.

  - Smaller payload size.

  - Easier integration with JavaScript-based frontends.
