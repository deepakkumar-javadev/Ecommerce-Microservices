# Ecommerce-Microservices

A Java Spring Boot based e-commerce application built using a microservices architecture.

The application is divided into independent services for user management, products, inventory, cart, orders, payments and notifications. Services communicate using REST/Feign for synchronous operations and Apache Kafka for asynchronous event-based communication.

## Architecture

```text
                         Ecom Frontend
                            :9090
                              |
                              v
                       API Gateway
                           :8080
                              |
          +-------------------+-------------------+
          |                   |                   |
          v                   v                   v
    User Service       Product Service       Cart Service
       :8082                :8083                :8085
          |                   |                   |
       User DB            Product DB            Cart DB
                              |
                              v
                     Inventory Service
                           :8084
                              |
                        Inventory DB


                    Order Service :8086
                           |
                           v
                    Payment Service
                         :8087
                           |
                        Razorpay
                           |
                     Payment DB


                  Kafka Event Communication
                           |
          +----------------+----------------+
          |                                 |
          v                                 v
   Inventory Events                   Order Events
          |                                 |
          +----------------+----------------+
                           |
                           v
               Notification Service
                      :8088
                           |
                           v
                  Email Notification


                    Eureka Server
                       :8761
                           ^
                           |
                  Service Registration
```

## Services

| Service               | Port | Responsibility                          |
| --------------------- | ---: | --------------------------------------- |
| Ecom Frontend Service | 9090 | E-commerce frontend and user interface  |
| API Gateway           | 8080 | Entry point for client requests         |
| Eureka Server         | 8761 | Service discovery and service registry  |
| User Service          | 8082 | Registration, login and user management |
| Product Service       | 8083 | Product creation, update and search     |
| Inventory Service     | 8084 | Stock and inventory management          |
| Cart Service          | 8085 | Cart and cart item management           |
| Order Service         | 8086 | Order creation and order processing     |
| Payment Service       | 8087 | COD and Razorpay payments               |
| Notification Service  | 8088 | Order-related email notifications       |

Each microservice is maintained as an independent GitHub repository.

## Technology Stack

* Java 17
* Spring Boot
* Spring Security
* Spring Cloud Gateway
* Spring Cloud Eureka
* JWT
* BCrypt
* Spring Data JPA
* Hibernate
* MySQL
* OpenFeign
* Apache Kafka
* Razorpay
* Resilience4j
* SpringDoc OpenAPI
* Swagger UI
* JavaMailSender
* Maven
* Git & GitHub

## Service Discovery

Eureka Server is used for service registration and discovery.

```text
Eureka Server
    :8761
       ^
       |
       +---- User Service :8082
       +---- Product Service :8083
       +---- Inventory Service :8084
       +---- Cart Service :8085
       +---- Order Service :8086
       +---- Payment Service :8087
       +---- Notification Service :8088
       +---- API Gateway :8080
```

The services register themselves with Eureka, allowing service-to-service communication without hardcoding service host information.

## Authentication

User Service uses Spring Security, JWT and BCrypt for authentication.

### Login Flow

```text
Client
  ↓
Email + Password
  ↓
User Service
  ↓
Authentication
  ↓
JWT Token
  ↓
Client
  ↓
Authorization: Bearer <JWT>
  ↓
Protected APIs
```

### User Service APIs

| Method | Endpoint                  | Description                        | Access                          |
| ------ | ------------------------- | ---------------------------------- | ------------------------------- |
| POST   | `/auth/register`          | Register a new user                | Public                          |
| POST   | `/auth/login`             | Authenticate user and generate JWT | Public                          |
| GET    | `/auth/getuser/{uid}`     | Get basic user information         | Based on security configuration |
| GET    | `/auth/UserDetails/{uid}` | Get user details with role         | ADMIN                           |

Admin access is enforced using:

```java
@PreAuthorize("hasRole('ADMIN')")
```

## API Documentation

Swagger UI is integrated with the backend microservices using SpringDoc OpenAPI.

Swagger UI provides interactive API documentation and allows REST APIs to be tested directly from the browser.

### Swagger UI URLs

| Service              | Swagger UI                                    |
| -------------------- | --------------------------------------------- |
| User Service         | `http://localhost:8082/swagger-ui/index.html` |
| Product Service      | `http://localhost:8083/swagger-ui/index.html` |
| Inventory Service    | `http://localhost:8084/swagger-ui/index.html` |
| Cart Service         | `http://localhost:8085/swagger-ui/index.html` |
| Order Service        | `http://localhost:8086/swagger-ui/index.html` |
| Payment Service      | `http://localhost:8087/swagger-ui/index.html` |
| Notification Service | `http://localhost:8088/swagger-ui/index.html` |

Eureka Server does not require a Swagger UI because it is used for service discovery and registration.

## Main E-Commerce Flow

```text
User
 ↓
Register / Login
 ↓
User Service
 ↓
JWT
 ↓
Product Service
 ↓
Select Product
 ↓
Cart Service
 ↓
Add to Cart
 ↓
Order Service
 ↓
Payment Service
 ↓
Razorpay / COD
 ↓
Inventory Service
 ↓
Stock Processing
 ↓
Kafka Events
 ↓
Notification Service
 ↓
Email Notification
```

## Order Processing

The Order Service coordinates the order creation process.

The Order Service does not directly manage inventory data. Inventory operations are handled by the Inventory Service.

Similarly, cart data is managed by the Cart Service.

### COD Flow

```text
Create Order
    ↓
Order Service
    ↓
Payment Service
    ↓
Payment = PENDING
    ↓
Order Confirmed
    ↓
Inventory Processing
    ↓
Order Delivered
    ↓
Payment = PAID
    ↓
order.delivered
    ↓
Kafka
    ↓
Notification Service
    ↓
Email Notification
```

### Online Payment Flow

```text
Create Order
    ↓
Order Service
    ↓
Payment Service
    ↓
Create Razorpay Order
    ↓
Razorpay Checkout
    ↓
Payment
    ↓
Razorpay Webhook
    ↓
Signature Verification
    ↓
Payment = PAID
    ↓
Inventory Processing
    ↓
Kafka
    ↓
Notification Service
    ↓
Email Notification
```

## Payment Integration

The Payment Service supports:

* Cash on Delivery
* Online payment using Razorpay

### COD

For COD orders, the Payment Service creates a payment record with:

```text
Payment Method : COD
Payment Status : PENDING
```

When the order is delivered, the payment status is updated to `PAID`.

### Razorpay

For online orders, the Payment Service:

1. Creates a Razorpay order.
2. Stores the Razorpay order ID.
3. Returns the payment information to the client.
4. Opens Razorpay Checkout.
5. Receives the Razorpay webhook.
6. Verifies the webhook signature.
7. Updates the payment status to `PAID`.
8. Publishes the required event for further order processing.

Razorpay API calls use Resilience4j retry and rate-limiter mechanisms.

## Kafka Event Communication

Apache Kafka is used for asynchronous communication between services.

### Kafka Topics

```text
order.created
inventory.reserve
inventory.reserved
inventory.updated
payment.success
order.delivered
```

### Example

```text
Order Service
     ↓
order.created
     ↓
Kafka
     ↓
Inventory / Other Consumers
```

### Another Example

```text
Order Service
     ↓
order.delivered
     ↓
Kafka
     ↓
Notification Service
```

Kafka helps keep service responsibilities separate and reduces unnecessary direct dependencies between services.

## Inventory Service

Inventory Service is responsible for maintaining product stock.

Important inventory information includes:

* SKU code
* Available quantity
* Reserved quantity
* Inventory status
* Created date

Inventory reservation and stock updates are handled by Inventory Service rather than Order Service.

## Notification Service

Notification Service handles order-related email notifications.

`JavaMailSender` is used for sending emails.

When notification-related events are received, Notification Service obtains the required user information from User Service and sends the appropriate order notification.

User email information is not maintained inside Inventory Service.

## Database Design

Each microservice has its own MySQL database.

| Service              | Database             | Main Tables           |
| -------------------- | -------------------- | --------------------- |
| User Service         | `ecomuserdb`         | `users`               |
| Product Service      | `ecomproductdb`      | `products`            |
| Inventory Service    | `ecominventorydb`    | `inventory`           |
| Cart Service         | `ecomcartdb`         | `carts`, `cart_items` |
| Order Service        | `ecomorderdb`        | `orders`              |
| Payment Service      | `ecompaymentdb`      | `payments`            |
| Notification Service | `ecomnotificationdb` | `notifications`       |

Services access their own database instead of directly accessing another service's database.

## Project Structure

The application is maintained using separate repositories for each microservice.

```text
Ecommerce-Microservices
│
├── Ecom Frontend Service
├── API Gateway
├── Eureka Server
├── User Service
├── Product Service
├── Inventory Service
├── Cart Service
├── Order Service
├── Payment Service
└── Notification Service
```

The main `Ecommerce-Microservices` repository contains project documentation and links to the individual service repositories.

## Running Locally

### Prerequisites

* Java 17
* Maven
* MySQL
* Apache Kafka
* ZooKeeper
* Git

### Infrastructure

The project currently uses:

```text
MySQL
Kafka
ZooKeeper
```

Kafka is currently configured with ZooKeeper for local development.

### Application Ports

| Service               | Port |
| --------------------- | ---: |
| Ecom Frontend Service | 9090 |
| API Gateway           | 8080 |
| Eureka Server         | 8761 |
| User Service          | 8082 |
| Product Service       | 8083 |
| Inventory Service     | 8084 |
| Cart Service          | 8085 |
| Order Service         | 8086 |
| Payment Service       | 8087 |
| Notification Service  | 8088 |

Start the required infrastructure first and then start the Eureka Server and Spring Boot services.

## Configuration

Database and service-specific configuration is maintained inside each microservice.

Sensitive configuration values should not be committed to GitHub.

Typical configuration includes:

```text
DB_USERNAME
DB_PASSWORD
JWT_SECRET
RAZORPAY_KEY_ID
RAZORPAY_KEY_SECRET
MAIL_USERNAME
MAIL_PASSWORD
RAZORPAY_WEBHOOK_SECRET
```

For local development, configure these values in the appropriate application configuration or environment variables.

## CORS

CORS configuration is used where required for frontend communication with backend services.

The frontend runs on:

```text
http://localhost:9090
```

Backend services run on their respective ports.

Example:

```text
Frontend
http://localhost:9090
       ↓
Backend APIs
localhost:8080 / localhost:8082 / localhost:8083 / ...
```

## Git Workflow

The project follows a feature-based Git workflow.

```text
feature/*
    ↓
develop
    ↓
main
```

Development work is first performed on a feature branch.

After testing, the feature branch is merged into `develop`.

Stable changes are then merged into `main`.

## Repository Links

Individual microservices are maintained in separate repositories:

* Ecom Frontend Service
* API Gateway
* Eureka Server
* User Service
* Product Service
* Inventory Service
* Cart Service
* Order Service
* Payment Service
* Notification Service

Repository links can be added here as the individual service repositories are published.

## Developer

**Deepak Kumar**

Java Backend Developer

**Tech:** Java 17 · Spring Boot · Microservices · REST APIs · Kafka · MySQL · Razorpay · Eureka · Swagger
