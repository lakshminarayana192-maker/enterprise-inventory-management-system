
# High-Level Architecture

## 1. System Overview

InvenTrack Enterprise is an enterprise inventory, procurement, order, payment, and reporting platform.

The system will provide a React web application for users such as administrators, warehouse managers, inventory clerks, procurement officers, finance users, sales users, and auditors.

The backend will use Java Spring Boot microservices. Each service will own a specific business capability and its related data.

The application will initially run locally using Docker and Docker Compose. Later, it will be deployed to Kubernetes using Helm.

---

## 2. High-Level Architecture

```text
                         React Frontend
                               |
                         API Gateway
                               |
    ---------------------------------------------------------
    |            |            |            |                |
Identity     Product      Warehouse     Inventory       Supplier
Service      Service       Service        Service        Service
    |            |            |            |                |
PostgreSQL   PostgreSQL   PostgreSQL    PostgreSQL      PostgreSQL

    ---------------------------------------------------------
    |             |             |             |             |
Procurement    Order         Payment      Notification    Audit
Service        Service       Service        Service       Service
    |             |             |                              |
PostgreSQL    PostgreSQL    PostgreSQL                      PostgreSQL

                         Reporting Service
                                |
                           PostgreSQL
```

Shared infrastructure:

```text
Kafka
Redis
Object Storage
Email Provider
Payment Gateway
Prometheus
Grafana
Centralized Logging
Distributed Tracing
```

---

## 3. Bounded Contexts

The application is divided into the following bounded contexts:

* Identity and Access Management
* Product Catalogue
* Warehouse Management
* Inventory Management
* Supplier Management
* Procurement Management
* Customer Order Management
* Payment Management
* Notification Management
* Reporting
* Audit and Compliance

Each bounded context owns its business rules, terminology, APIs, and data.

---

## 4. Microservices

### 4.1 API Gateway

Responsibilities:

* Provide a single entry point for the frontend
* Route requests to backend services
* Validate authentication tokens
* Apply CORS rules
* Apply rate limiting
* Generate or propagate correlation IDs
* Log incoming requests
* Forward user identity information where appropriate

The API Gateway must not contain core business logic.

---

### 4.2 Identity Service

Responsibilities:

* User management
* Role management
* Permission management
* Authentication
* JWT access-token generation
* Refresh-token handling
* Password reset
* Account locking
* Login audit history

Owned data:

* Users
* Roles
* Permissions
* User-role mappings
* Role-permission mappings
* Refresh tokens

---

### 4.3 Product Service

Responsibilities:

* Product management
* Category management
* Brand management
* Unit-of-measure management
* Product status
* Product images
* Product search
* Filtering
* Sorting
* Pagination
* Product PDF export

Owned data:

* Products
* Categories
* Brands
* Units of measure
* Product metadata
* Product image metadata

---

### 4.4 Warehouse Service

Responsibilities:

* Warehouse management
* Warehouse address management
* Aisle management
* Rack management
* Shelf management
* Bin management
* Warehouse-user assignment
* Warehouse capacity configuration

Owned data:

* Warehouses
* Warehouse locations
* Aisles
* Racks
* Shelves
* Bins

---

### 4.5 Inventory Service

Responsibilities:

* Maintain on-hand stock
* Maintain reserved stock
* Calculate available stock
* Prevent negative stock
* Manage inventory reservations
* Manage inventory adjustments
* Record stock movements
* Release expired reservations
* Handle concurrent stock updates
* Manage stock transfers

Owned data:

* Inventory balances
* Inventory reservations
* Inventory adjustments
* Stock movement ledger
* Transfer records

Inventory formula:

```text
Available Stock = On-hand Stock - Reserved Stock
```

---

### 4.6 Supplier Service

Responsibilities:

* Supplier registration
* Supplier contact management
* Supplier status
* Supplier categories
* Supplier-product relationships
* Supplier performance information

Owned data:

* Suppliers
* Supplier contacts
* Supplier-product mappings
* Supplier documents metadata

---

### 4.7 Procurement Service

Responsibilities:

* Purchase requisitions
* Purchase requisition approvals
* Purchase orders
* Purchase-order approvals
* Goods receipt notes
* Partial deliveries
* Supplier invoices
* Invoice verification
* Procurement history

Owned data:

* Purchase requisitions
* Purchase orders
* Purchase-order items
* Approval records
* Goods receipts
* Supplier invoices

---

### 4.8 Order Service

Responsibilities:

* Customer management
* Customer-order creation
* Order-item management
* Order status management
* Discount and tax calculation
* Inventory reservation coordination
* Order cancellation
* Order history
* Saga coordination for order, inventory, and payment flow

Owned data:

* Customers
* Customer orders
* Order items
* Order status history

---

### 4.9 Payment Service

Responsibilities:

* Create payment requests
* Integrate with payment gateways
* Process payment webhooks
* Verify webhook signatures
* Prevent duplicate payment processing
* Process full and partial refunds
* Maintain payment history
* Perform payment reconciliation
* Publish payment events

Owned data:

* Payments
* Refunds
* Webhook events
* Processed event records
* Reconciliation records

---

### 4.10 Notification Service

Responsibilities:

* Send email notifications
* Create in-app notifications
* Support optional SMS notifications
* Manage notification templates
* Retry failed notifications
* Record delivery status
* Consume events from Kafka

Owned data:

* Notification records
* Notification templates
* User notification preferences
* Delivery attempts

---

### 4.11 Reporting Service

Responsibilities:

* Generate dashboards
* Generate product reports
* Generate inventory reports
* Generate stock valuation reports
* Generate payment reports
* Generate procurement reports
* Generate scheduled reports
* Support PDF, CSV, and Excel exports

Owned data:

* Reporting projections
* Generated-report metadata
* Scheduled-report configuration

The reporting service should not directly modify operational service data.

---

### 4.12 Audit Service

Responsibilities:

* Record business and security events
* Maintain audit history
* Record old and new values
* Record user, timestamp, service, and correlation ID
* Support audit reporting
* Consume audit events through Kafka

Owned data:

* Audit logs
* Event metadata
* Compliance records

---

## 5. Database Strategy

Each microservice will own its database or isolated database schema.

Examples:

```text
identity_db
product_db
warehouse_db
inventory_db
supplier_db
procurement_db
order_db
payment_db
reporting_db
audit_db
notification_db
```

Rules:

* A service must not directly read or update another service's tables.
* Services communicate through APIs or events.
* Database changes will be managed using Flyway.
* Each service will maintain its own migration scripts.
* Foreign-key relationships will exist only inside the same service database.
* Cross-service relationships will use business identifiers such as product ID, warehouse ID, order ID, and payment ID.

---

## 6. Communication Strategy

### Synchronous communication

REST APIs will be used when an immediate response is required.

Examples:

* Frontend requests a product list
* Order Service requests inventory availability
* Payment Service creates a payment-provider order
* API Gateway requests user validation

### Asynchronous communication

Kafka will be used for events that do not require an immediate response or that must notify several services.

Example events:

```text
ProductCreated
InventoryReserved
InventoryReservationFailed
GoodsReceived
OrderCreated
OrderConfirmed
PaymentCompleted
PaymentFailed
RefundCompleted
StockBelowThreshold
```

### Communication rules

* REST calls must have timeouts.
* Retries must only be used for safe or idempotent operations.
* Events must contain an event ID.
* Kafka consumers must handle duplicate messages.
* Failed events will move to retry or dead-letter topics.
* Correlation IDs must be propagated across services.

---

## 7. Distributed Transaction Strategy

A single database transaction cannot span Order, Inventory, and Payment services.

The system will use the Saga pattern.

Example order flow:

```text
Create Order
     |
Reserve Inventory
     |
Initiate Payment
     |
Payment Successful
     |
Confirm Order
```

Compensation flow:

```text
Payment Failed
     |
Release Inventory Reservation
     |
Mark Order as Payment Failed
```

The Transactional Outbox Pattern will be used where a database change and Kafka event must be committed reliably.

---

## 8. Caching Strategy

Redis will be used for:

* Frequently accessed reference data
* Product categories
* User permissions
* Rate-limiting counters
* Short-lived payment-session data
* Application configuration

Rapidly changing inventory quantities will not be cached without a defined consistency and invalidation strategy.

---

## 9. Security Architecture

The system will use:

* Spring Security
* JWT access tokens
* Refresh tokens
* Role-based access control
* Permission-based authorization
* Password hashing
* Secure headers
* CORS configuration
* Input validation
* Webhook signature verification
* Secret management
* Audit logging
* Rate limiting

The frontend can hide unauthorized actions, but the backend must always enforce authorization.

---

## 10. Observability

The platform will include:

* Structured application logs
* Correlation IDs
* Metrics using Micrometer and Prometheus
* Dashboards using Grafana
* Distributed tracing using OpenTelemetry
* Centralized logging using Loki or the Elastic Stack
* Health endpoints using Spring Boot Actuator

Important metrics include:

* Request count
* Error rate
* Response time
* JVM memory
* Database connection usage
* Kafka consumer lag
* Payment success rate
* Inventory reservation failures
* PDF generation time

---

## 11. Local Docker Compose Architecture

The first working Docker Compose environment will include:

```text
React Frontend
API Gateway
Identity Service
Product Service
Inventory Service
PostgreSQL
Redis
Kafka
Kafka UI
```

Additional services will be introduced incrementally.

Docker Compose will provide:

* Service networking
* Environment configuration
* Health checks
* Persistent volumes
* Local dependency startup
* Repeatable development environments

---

## 12. Future Kubernetes Deployment

Each application service will be deployed as a Kubernetes Deployment.

The Kubernetes environment will include:

* Namespaces
* Deployments
* Services
* ConfigMaps
* Secrets
* Ingress
* PersistentVolumeClaims
* Jobs
* CronJobs
* HorizontalPodAutoscalers
* NetworkPolicies
* Resource requests and limits
* Readiness probes
* Liveness probes
* Startup probes
* Rolling updates
* Rollbacks

Helm will be used to package and configure Kubernetes resources.

A Kubernetes Service will provide internal DNS and load balancing between healthy service pods.

---

## 13. Initial Implementation Decision

The team will not create every microservice at the beginning.

The first working version will include:

```text
React Frontend
API Gateway
Identity Service
Product Service
Inventory Service
PostgreSQL
Docker Compose
```

Warehouse, procurement, orders, payments, Kafka, reporting, notifications, and audit capabilities will be introduced in later sprints.

This reduces initial complexity while preserving the target enterprise architecture.
