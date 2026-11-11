# SAGA Pattern Concepts

**Unit 4:** Distributed Transactions (SAGA Pattern) **Topic Code:** DT-401 **Reading Time:** ~45 minutes

------------------------------------------------------------------------

## Learning Objectives

- Explain the challenges of transactions in a distributed system (BASE vs. ACID).

- Define the SAGA pattern as a solution for managing distributed transactions.

- Describe the Choreography-based SAGA pattern using events and message queues (like RabbitMQ).

- Illustrate the concept of local transactions, compensating transactions (failure path), and the happy path.

- Compare and contrast the Choreography and Orchestration SAGA patterns.

------------------------------------------------------------------------

## Section 1: Concept/overview

### 1.1 Introduction

In traditional Monolith architecture, managing data consistency is relatively simple thanks to **ACID** (Atomicity, Consistency, Isolation, Durability) transactions. You can start a transaction, perform multiple write operations to different tables in the same database, and then `COMMIT` or `ROLLBACK` everything. If an error occurs, the entire transaction is restored to its original state, ensuring data is always consistent.

However, when we move to Microservices architecture, everything becomes more complicated. Each microservice typically owns and manages its own database. A business process, such as “Place Order”, may involve multiple services: `Order`` ``Service`, `Payment`` ``Service`, `Inventory`` ``Service`, `Notification`` ``Service`. How do we ensure that if the `Payment`` ``Service` fails, the order created in the `Order`` ``Service` and the goods deducted in the `Inventory`` ``Service` are undone correctly? This is the problem of **distributed transactions**. Using traditional mechanisms like Two-Phase Commit (2PC) is often not feasible in a microservices environment because it requires resource locking across multiple services, reducing availability and increasing coupling between services. The SAGA pattern was born to solve exactly this problem, providing a method to manage data consistency across multiple services without strict locking transactions.

### 1.2 Formal Definition

**SAGA** is a design pattern for managing distributed transactions, described as a sequence of **local transactions**. Each local transaction updates data within a single service and then publishes an event to trigger the next local transaction in another service.

If a local transaction fails, SAGA executes a series of **compensating transactions** to undo the changes made by the previous local transactions. Compensating transactions must be designed to execute successfully and not fail (idempotent and re-triable).

Unlike ACID, SAGA adheres to the **BASE** model (Basically Available, Soft state, Eventually consistent). Data will not be consistent immediately, but will achieve a consistent state after a period of time (eventual consistency).

### 1.3 Analogy

Imagine you are planning a trip and need to book 3 things: **Flight Ticket**, **Hotel**, and **Rental Car**. Each of these actions is like a local transaction in a separate microservice.

1.  **Step 1: Book Flight (Local Transaction 1)**. You book the ticket successfully.

2.  **Step 2: Book Hotel (Local Transaction 2)**. After having the flight ticket, you book the hotel and succeed as well.

3.  **Step 3: Rent Car (Local Transaction 3)**. You try to rent a car, but there are no cars available. This step **fails**.

Now, your trip cannot be realized. You cannot simply “rollback” the entire process. Instead, you must perform compensating actions:

- **Step 4: Cancel Hotel (Compensating Transaction 2)**. You call the hotel to cancel the booked room.

- **Step 5: Cancel Flight (Compensating Transaction 1)**. You call the airline to cancel the ticket.

The sequence “Book Flight -\> Book Hotel -\> Rent Car” is the **Happy Path**. The sequence “Cancel Hotel -\> Cancel Flight” is the **Failure Path** using compensating transactions. This entire process is a SAGA.

### 1.4 History of Development

The SAGA concept was first introduced in a 1987 scientific paper by Hector Garcia-Molina and Kenneth Salem at Princeton University. Initially, it was proposed as a model for “Long Lived Transactions” (LLTs) in database systems. It wasn’t until microservices architecture became popular that the SAGA pattern was widely applied and became a standard solution for solving distributed transaction problems.

------------------------------------------------------------------------

## Section 2: Core Components

### 2.1 Architecture overview

In SAGA, there are two main approaches to coordinating the sequence of local transactions: **Choreography** and **Orchestration**. We will focus on Choreography, which is the popular and decentralized approach.

**Choreography-based SAGA Architecture:**

```
      +----------------+      OrderCreated      +----------------+      PaymentProcessed      +-----------------+
      |  Order Service | ---------------------> | Message Broker | -------------------------> | Payment Service |
      | (Local Tx #1)  |      (Event)           |  (e.g. RabbitMQ) |      (Event)             | (Local Tx #2)   |
      +----------------+                         +----------------+                          +-----------------+
             ^                                                                                         |
             | OrderFailed (Compensating Event)                                                        | PaymentFailed (Event)
             +-----------------------------------------------------------------------------------------+
```

In this model:

- Each service performs its own local transaction.

- After completion, it publishes an event to the Message Broker.

- Other services listen for events they are interested in and perform corresponding actions.

- There is no central “conductor” coordinating the flow. Services “talk” to each other via events.

### 2.2 Key Components

**Component 1: Local Transaction**

- **Definition:** A standard ACID transaction executed entirely within the scope of a single microservice and a single database. This is the most basic unit of work in a SAGA.

- **Role:** Ensures atomicity for data changes within a service. For example, `Order`` ``Service` creates a new order and saves it to its `Orders` table in a local transaction.

- **Syntax (Example with Python/SQLAlchemy):**

``` python
# Example using a session from a database framework like SQLAlchemy
from database import session

def create_order_local_tx(order_details):
    try:
        # Start of the local transaction
        new_order = Order(
            customer_id=order_details['customer_id'],
            status='PENDING'
        )
        session.add(new_order)
        session.commit() # Atomic commit to the Order Service's database
        return new_order.id
    except Exception as e:
        session.rollback() # Rollback if anything fails within this service
        raise e
```

**Component 2: Compensating Transaction**

- **Definition:** A transaction designed to undo the operations of a previously successful local transaction. It is not a real “rollback”, but a reverse business action.

- **Role:** Returns the system to a consistent state when a step in the SAGA fails. Example: The compensating transaction for “Create Order” is “Cancel Order”.

- **Syntax (Example with Python/SQLAlchemy):**

``` python
# Example of a compensating transaction for create_order
from database import session

def cancel_order_compensating_tx(order_id):
    try:
        order_to_cancel = session.query(Order).filter_by(id=order_id).first()
        if order_to_cancel and order_to_cancel.status != 'CANCELLED':
            # Business logic to undo the 'create_order' action
            order_to_cancel.status = 'CANCELLED'
            # Maybe restore product stock if it was part of the original transaction
            # restore_stock(order_to_cancel.product_id, order_to_cancel.quantity)
            session.commit()
        return True
    except Exception as e:
        # A compensating transaction SHOULD NOT fail.
        # Implement retries or log for manual intervention.
        session.rollback()
        raise e # Or handle it more gracefully
```

**Component 3: Message Broker & Events**

- **Definition:** Message Broker (e.g., RabbitMQ, Kafka) is an intermediary system that receives and distributes messages (events) between services. An event is a notification about an occurrence (e.g., `OrderCreated`, `PaymentFailed`).

- **Role:** Ensures asynchronous communication and reduces dependency (decoupling) between microservices. It allows `Order`` ``Service` to not need to know about the existence of `Payment`` ``Service`, but only needs to emit an `OrderCreated` event.

- **Syntax (Example publishing event with Pika/RabbitMQ):**

``` python
# Example of publishing an event after a successful local transaction
import pika
import json

def publish_event(event_name, payload):
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()
    channel.exchange_declare(exchange='saga_events', exchange_type='topic')

    routing_key = f"order.{event_name}" # e.g., order.created
    message = json.dumps(payload)

    channel.basic_publish(
        exchange='saga_events',
        routing_key=routing_key,
        body=message
    )
    print(f" [x] Sent event {routing_key} with payload {message}")
    connection.close()

# Usage
# order_id = create_order_local_tx(...)
# publish_event('created', {'order_id': order_id, ...})
```

------------------------------------------------------------------------

## Section 3: Implementation

Using an E-commerce scenario: **Order Service**, **Payment Service**, **Inventory Service**.

- **Happy Path:** Create Order -\> Process Payment -\> Update Inventory.

- **Failure Path:** Process Payment fails -\> Cancel Order.

### Level 1 - Basic (Beginner): Happy Path

This example illustrates only 2 services: `OrderService` publishes an event and `PaymentService` consumes that event.

**`order_service_publisher.py`**

``` python
# order_service_publisher.py
import pika
import json
import uuid

def create_order_and_publish_event():
    """
    Simulates creating an order and publishing an 'OrderCreated' event.
    In a real app, this would involve a local DB transaction first.
    """
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()

    # Declare a topic exchange to route events
    channel.exchange_declare(exchange='saga_exchange', exchange_type='topic')

    order_id = str(uuid.uuid4())
    order_details = {
        'order_id': order_id,
        'user_id': 'user-123',
        'amount': 99.99,
        'items': [{'product_id': 'prod-abc', 'quantity': 1}]
    }

    # The routing key helps consumers filter for events they care about.
    routing_key = 'order.created'

    # Here, we assume the local transaction for creating the order was successful.
    print(f"Order {order_id} created locally.")

    # Now, publish the event to notify other services.
    channel.basic_publish(
        exchange='saga_exchange',
        routing_key=routing_key,
        body=json.dumps(order_details),
        properties=pika.BasicProperties(
            content_type='application/json',
            delivery_mode=2, # make message persistent
        )
    )

    print(f" [x] Sent '{routing_key}': '{order_details}'")
    connection.close()

if __name__ == '__main__':
    create_order_and_publish_event()
```

**`payment_service_consumer.py`**

``` python
# payment_service_consumer.py
import pika
import json
import time

def main():
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()

    channel.exchange_declare(exchange='saga_exchange', exchange_type='topic')

    # Create an exclusive queue for this consumer
    result = channel.queue_declare(queue='', exclusive=True)
    queue_name = result.method.queue

    # Bind the queue to the exchange, listening for 'order.created' events
    binding_key = 'order.created'
    channel.queue_bind(exchange='saga_exchange', queue=queue_name, routing_key=binding_key)

    print(' [*] Payment Service waiting for order.created events. To exit press CTRL+C')

    def callback(ch, method, properties, body):
        order_details = json.loads(body)
        print(f" [x] Received event '{method.routing_key}': {order_details}")

        # --- Start of Local Transaction for Payment Service ---
        print(f"Processing payment for order {order_details['order_id']}...")
        time.sleep(2) # Simulate payment processing work
        print(f"Payment successful for order {order_details['order_id']}.")
        # --- End of Local Transaction ---

        # Acknowledge the message was processed successfully
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(
        queue=queue_name,
        on_message_callback=callback
    )

    channel.start_consuming()

if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        print('Interrupted')
```

**Expected Output:**

1.  Run `payment_service_consumer.py` first.

```
 [*] Payment Service waiting for order.created events. To exit press CTRL+C
```

1.  Run `order_service_publisher.py`.

```
Order 123e4567-e89b-12d3-a456-426614174000 created locally.
 [x] Sent 'order.created': '{...}'
```

1.  The payment consumer’s terminal will show:

```
 [x] Received event 'order.created': {'order_id': '...', ...}
Processing payment for order 123e4567-e89b-12d3-a456-426614174000...
Payment successful for order 123e4567-e89b-12d3-a456-426614174000.
```

**Common Errors:**

- **`pika.exceptions.AMQPConnectionError`:** RabbitMQ server is not running or is not accessible at `localhost`.

- **Fix:** Ensure RabbitMQ server is installed and running. Check host, port, username, and password in `pika.ConnectionParameters`.

### Level 2 - Intermediate: Handling Failures with Compensating Transactions

Now, we assume the `PaymentService` can fail. In that case, it must publish a `payment.failed` event so the `OrderService` can catch it and execute a compensating transaction (cancel order).

**`payment_service_with_failure.py`**

``` python
# payment_service_with_failure.py
import pika
import json
import time
import random

def payment_consumer():
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()
    channel.exchange_declare(exchange='saga_exchange', exchange_type='topic')
    result = channel.queue_declare(queue='payment_queue', durable=True)
    queue_name = result.method.queue
    channel.queue_bind(exchange='saga_exchange', queue=queue_name, routing_key='order.created')

    print(' [*] Payment Service waiting for events.')

    def publish_reply_event(event_name, payload):
        # Helper to publish response events
        channel.basic_publish(
            exchange='saga_exchange',
            routing_key=event_name, # e.g., 'payment.processed' or 'payment.failed'
            body=json.dumps(payload)
        )
        print(f" [>] Sent event '{event_name}'")

    def callback(ch, method, properties, body):
        order_details = json.loads(body)
        order_id = order_details.get('order_id')
        print(f" [<] Received 'order.created' for order {order_id}")

        # Simulate a chance of payment failure
        if random.random() > 0.5:
            # --- Happy Path ---
            print(f"Processing payment for order {order_id}... SUCCESS")
            # In a real app, commit local transaction here
            reply_payload = {'order_id': order_id, 'status': 'PAID'}
            publish_reply_event('payment.processed', reply_payload)
        else:
            # --- Failure Path ---
            print(f"Processing payment for order {order_id}... FAILED")
            # No DB changes, just notify about the failure
            reply_payload = {'order_id': order_id, 'reason': 'Insufficient funds'}
            publish_reply_event('payment.failed', reply_payload)

        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(queue=queue_name, on_message_callback=callback)
    channel.start_consuming()

if __name__ == '__main__':
    payment_consumer()
```

**`order_service_with_compensation.py`**

``` python
# order_service_with_compensation.py
import pika
import json

# This service now also needs to listen for compensation events
def order_service_listener():
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()
    channel.exchange_declare(exchange='saga_exchange', exchange_type='topic')

    # A dedicated queue for order service to listen for replies
    result = channel.queue_declare(queue='order_compensation_queue', durable=True)
    queue_name = result.method.queue

    # Listen for payment failure events
    channel.queue_bind(exchange='saga_exchange', queue=queue_name, routing_key='payment.failed')
    # Can also listen for success events to update order status
    channel.queue_bind(exchange='saga_exchange', queue=queue_name, routing_key='payment.processed')

    print(' [*] Order Service listening for compensation/success events.')

    def callback(ch, method, properties, body):
        event_data = json.loads(body)
        order_id = event_data.get('order_id')

        if method.routing_key == 'payment.failed':
            print(f" [!] Received 'payment.failed' for order {order_id}")
            # --- Start of Compensating Transaction ---
            print(f"Executing compensating transaction: Cancelling order {order_id}...")
            # Here you would update the order status to 'CANCELLED' in the database
            # update_order_status(order_id, 'CANCELLED')
            print(f"Order {order_id} has been cancelled.")
            # --- End of Compensating Transaction ---
        elif method.routing_key == 'payment.processed':
             print(f" [+] Received 'payment.processed' for order {order_id}. Updating status to PAID.")
             # update_order_status(order_id, 'PAID')

        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(queue=queue_name, on_message_callback=callback)
    channel.start_consuming()

if __name__ == '__main__':
    # In a real app, this would run in a separate process/thread
    # from the publisher.
    order_service_listener()
```

*To run this example, you need to run 3 files: `order_service_publisher.py` (to create the order), `payment_service_with_failure.py`, and `order_service_with_compensation.py`.*

### Level 3 - Advanced: Production-ready Patterns

#### 1. Idempotent Consumers

An event may be re-sent multiple times (at-least-once delivery). A consumer must be designed to process an event multiple times without causing side effects.

``` python
# Advanced idempotent consumer snippet
# Assume we have a database connection 'db'

processed_payments = set() # In-memory, for demo. Use a DB table in production!

def idempotent_payment_callback(ch, method, properties, body):
    order_details = json.loads(body)
    order_id = order_details['order_id']

    # Use a unique transaction ID from the message if available, or a key from the data
    transaction_key = f"payment-for-order-{order_id}"

    # CHECK if already processed
    # In production, this would be a lookup in a `processed_messages` table
    if transaction_key in processed_payments:
        print(f" [!] Duplicate event for order {order_id}. Ignoring.")
        ch.basic_ack(delivery_tag=method.delivery_tag)
        return

    print(f"Processing payment for order {order_id} for the first time.")
    # ... process payment logic ...

    # MARK as processed inside the same local transaction as the business logic
    # db.execute("INSERT INTO processed_messages (id) VALUES (?)", (transaction_key,))
    # db.commit()
    processed_payments.add(transaction_key) # For demo

    ch.basic_ack(delivery_tag=method.delivery_tag)
```

#### 2. Transactional Outbox Pattern

**Problem:** What happens if the `Order`` ``Service` successfully commits the transaction to the database, but crashes **before** publishing the `OrderCreated` event? The result is that the order exists but the SAGA never starts, causing inconsistency.

**Solution:** Write the event to an `outbox` table in the same database and same local transaction as the main business logic. A separate process (Relay) will read from this `outbox` table and publish the event to the message broker. This ensures atomicity between saving state and preparing to send the event.

``` python
# Pseudocode for Transactional Outbox in Order Service

def create_order_with_outbox(order_details, db_session):
    """
    Saves the order and the event to be published in a single transaction.
    """
    try:
        # 1. Create the order object
        new_order = Order(status='PENDING', ...)
        db_session.add(new_order)

        # This is CRITICAL: flush to get the new_order.id
        db_session.flush()

        # 2. Create the event payload
        event_payload = {
            'order_id': new_order.id,
            'user_id': order_details['user_id'],
            'amount': order_details['amount']
        }

        # 3. Create the outbox entry for the event
        outbox_event = Outbox(
            event_type='OrderCreated',
            payload=json.dumps(event_payload)
        )
        db_session.add(outbox_event)

        # 4. Commit both the new order and the outbox event atomically
        db_session.commit()
        print("Order and Outbox event committed to DB.")

    except Exception as e:
        db_session.rollback()
        print(f"Transaction failed: {e}")
        raise

# A separate, continuously running process (the "Relay")
def outbox_relay_process():
    while True:
        # Find unpublished events
        events_to_publish = db_session.query(Outbox).filter_by(published=False).limit(100).all()

        for event in events_to_publish:
            try:
                # Publish to message broker
                publish_event_to_broker(event.event_type, event.payload)

                # Mark as published
                event.published = True
                db_session.commit()
            except Exception as e:
                # Handle broker publishing errors (e.g., retry logic)
                db_session.rollback()

        time.sleep(5) # Poll every 5 seconds
```

------------------------------------------------------------------------

## Section 4: Best Practices

### ✅ DO’s

| Practice | Why | Example |
|----|----|----|
| **Make Compensating Transactions Idempotent** | Compensating transactions may be called multiple times due to network errors. Retrying should not cause errors or change state after the first successful run. | `cancel_order(order_id)` should check if `order.status` is already `CANCELLED` and do nothing, instead of reporting an “order already cancelled” error. |
| **Use a Correlation ID** | Assign a unique ID to the entire SAGA sequence. This ID is passed through all events. | Every event (`OrderCreated`, `PaymentProcessed`) contains the field `correlation_id:`` ``"saga-xyz-123"`. This makes tracing and logging extremely easy. |
| **Keep Sagas Short and Simple** | The longer and more steps a SAGA has, the higher the likelihood of failure and the complexity of compensation. | Instead of a huge “User Onboarding” SAGA, break it down into smaller SAGAs like “Create Profile” and “Setup Billing”. |
| **Design for Eventual Consistency** | Accept that data will have a certain delay to become consistent. The user interface must be designed to handle this. | After placing an order, show the user a “Processing” status instead of “Success” immediately. Update the UI when receiving the `OrderConfirmed` event. |

### ❌ DON’Ts

| Anti-pattern | Consequence | How to avoid |
|----|----|----|
| **Relying on Synchronous Calls within a Saga** | A service directly calling another service via API REST/gRPC creates tight coupling, losing system availability and resilience. | Always use asynchronous communication via a Message Broker. Service A publishes an event, Service B listens and processes. |
| **Forgetting to Define Compensating Transactions** | If a step fails and there is no corresponding compensating transaction, the system will be stuck in an inconsistent state permanently, requiring manual intervention. | For every Local Transaction that can change data, define a corresponding Compensating Transaction immediately. |
| **Creating a “Cyclic” Saga** | Service A emits an event triggering Service B, Service B emits an event triggering Service A, creating an infinite loop. | Analyze the event flow carefully. Ensure the SAGA flow is a Directed Acyclic Graph (DAG). |
| **Returning Data in Events** | Publishing an event containing all data of an object (e.g., `OrderCreated` contains all product and customer details). | Events should only contain IDs and necessary information so other services can query data if needed. This avoids data duplication and staleness. |

### 🔒 Security Considerations

- **Message Broker Security:** Always enable authentication and authorization on the Message Broker. Each service should only have permission to publish/subscribe to the topics/exchanges it needs.

- **Data Encryption:** Encrypt event content (payload) if it contains sensitive information.

- **Input Validation:** Each consumer service must validate data received from events, not blindly trusting data from other services.

### ⚡ Performance Tips

- **Message Serialization:** Use efficient message formats like Protocol Buffers or Avro instead of JSON for systems requiring high performance.

- **Consumer Scaling:** Increase the number of consumer service instances to process messages from a queue in parallel, helping increase throughput.

- **Batch Processing:** If possible, design consumers to process a batch of messages at once instead of one by one to reduce overhead.

- **Broker Tuning:** Tune Message Broker parameters (e.g., prefetch count in RabbitMQ) to optimize message distribution to consumers.

------------------------------------------------------------------------

## Section 5: Case Study

### 5.1 Scenario

**Company/Project:** **Tiki-Clone**, an e-commerce platform transitioning from Monolith to Microservices architecture. **Requirements:** Rebuild the “Order Placement” process in the new architecture, ensuring data consistency across `Order`, `Payment`, and `Inventory` services. **Constraints:** The system must have high availability, services must be independently deployed, and single points of failure must be avoided.

### 5.2 Problem Analysis

In the old Monolith system, the order process was executed in a large database transaction. If any step (create order, deduct money, deduct stock) failed, the entire transaction would be rolled back.

When moving to Microservices, each service has its own database. A sequential API call from `Order`` ``Service` -\> `Payment`` ``Service` -\> `Inventory`` ``Service` is very risky. If `Inventory`` ``Service` fails, undoing the transaction in `Payment`` ``Service` (refund) and `Order`` ``Service` (cancel order) becomes complex, unreliable, and creates tight coupling.

### 5.3 Solution Design

The team decided to apply the **Choreography-based SAGA Pattern** using RabbitMQ as the Message Broker.

**SAGA Flow (Happy Path):**

1.  **Client** sends request `POST`` ``/orders` to **Order Service**.

2.  **Order Service**:

- Executes Local Transaction: Creates an order with `PENDING` status.

- Publishes event `OrderCreated` containing `orderId`, `amount`.

1.  **Payment Service**:

- Listens for `OrderCreated` event.

- Executes Local Transaction: Processes payment.

- Publishes event `PaymentProcessed`.

1.  **Inventory Service**:

- Listens for `PaymentProcessed` event.

- Executes Local Transaction: Deducts quantity in stock.

- Publishes event `InventoryUpdated`.

1.  **Order Service**:

- Listens for `InventoryUpdated` event.

- Updates order status to `CONFIRMED`.

**SAGA Flow (Failure Path - Payment Fails):**

1.  … (Same as steps 1, 2 above)

2.  **Payment Service**:

- Payment processing fails.

- Publishes event `PaymentFailed` containing `orderId` and `reason`.

1.  **Order Service**:

- Listens for `PaymentFailed` event.

- Executes **Compensating Transaction**: Updates order status to `CANCELLED`.

### 5.4 Implementation

``` python
# A simplified, conceptual implementation of the Order Service consumer logic
# order_service_main_consumer.py

import pika
import json

def order_service_saga_listener():
    # ... connection and channel setup ...
    # ... queue setup, binding to 'payment.failed' and 'inventory.updated'

    def callback(ch, method, properties, body):
        event = json.loads(body)
        routing_key = method.routing_key
        order_id = event.get('order_id')

        print(f"Order Service received event '{routing_key}' for order {order_id}")

        if routing_key == 'inventory.updated':
            # This is the final step of the happy path for the Order Service
            # Start local transaction
            try:
                # order = find_order_in_db(order_id)
                # order.status = 'CONFIRMED'
                # save_order_to_db(order)
                print(f"SAGA successful. Order {order_id} status updated to CONFIRMED.")

                # Optionally, publish an OrderConfirmed event for other services
                # (e.g., Notification Service)
            except Exception as e:
                # Handle DB errors, maybe retry
                print(f"Error updating order {order_id} to CONFIRMED: {e}")
            # End local transaction

        elif routing_key == 'payment.failed':
            # This is the failure path. We must compensate.
            # Start compensating transaction
            try:
                # order = find_order_in_db(order_id)
                # if order.status == 'PENDING':
                #    order.status = 'CANCELLED'
                #    save_order_to_db(order)
                print(f"SAGA failed. Compensating: Order {order_id} status updated to CANCELLED.")
            except Exception as e:
                # A compensating transaction should be as robust as possible.
                # Log for manual intervention if it fails.
                print(f"CRITICAL: Failed to execute compensating transaction for order {order_id}: {e}")
            # End compensating transaction

        ch.basic_ack(delivery_tag=method.delivery_tag)

    # ... channel.start_consuming() ...

# Assume other services (Payment, Inventory) have their own publishers/consumers
# similar to the examples in Section 3.
```

### 5.5 Results & Lessons Learned

- **Improved Metrics:**

- **Availability:** Significantly increased. An incident in `Inventory`` ``Service` no longer crashes the entire order process. Orders can still be created and paid for (and will be processed in inventory after service recovery, or compensated if needed).

- **Scalability:** Each service can be scaled independently. During sale seasons, `Payment`` ``Service` can be increased to 10 instances without affecting other services.

- **Deployment Speed:** Teams can update and deploy their services without coordinating with all other teams.

- **Lessons Learned:**

- **Observability is Key:** Debugging SAGA is very hard. Investing in centralized logging with `correlation_id` and distributed tracing is mandatory, not optional.

- **Eventual Consistency is a Business Decision:** Must work with Product Managers so they understand that some data will not be updated immediately. The user interface must reflect the correct “processing” state of the system.

- **Start with Choreography, but be Ready for Orchestration:** Choreography is great for simple flows. But when returns/exchanges processes were added, with many complex logics and conditions, the team realized that SAGA Orchestration might be a better choice for those processes.

------------------------------------------------------------------------

## References

- [Original SAGA Paper (1987) by Garcia-Molina & Salem](https://dl.acm.org/doi/10.1145/38713.38742)

- [Microservices.io - Saga Pattern by Chris Richardson](https://microservices.io/patterns/data/saga.html)

- [RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html)
