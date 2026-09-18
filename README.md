# Ecommerce-Microservices

A Java Spring Boot based e-commerce application built using a microservices architecture.

The application is divided into independent services for user management, products, inventory, cart, orders, payments and notifications. Services communicate using REST/Feign for synchronous operations and Apache Kafka for asynchronous event-based communication.

## Architecture


                         Client
                           |
                           v
                    API Gateway :8080
                           |
        +------------------+------------------+
        |                  |                  |
        v                  v                  v
   User Service      Product Service      Cart Service
      :8082               :8083               :8085
        |                  |                  |
        v                  v                  v
     User DB           Product DB           Cart DB

                           |
                           v
                    Order Service :8086
                           |
              +------------+------------+
              |                         |
              v                         v
       Payment Service           Inventory Service
           :8087                     :8084
              |                         |
              v                         v
          Razorpay                 Inventory DB
              |
              v
          Payment DB

                           |
                           v
                         Kafka
                           |
                           v
                 Notification Service
                        :8088


## Services

| Service              | Port | Responsibility                          |
| -------------------- | ---: | --------------------------------------- |
| API Gateway          | 8080 | Entry point for client requests         |
| User Service         | 8082 | Registration, login and user management |
| Product Service      | 8083 | Product creation, update and search     |
| Inventory Service    | 8084 | Stock and inventory management          |
| Cart Service         | 8085 | Cart and cart item management           |
| Order Service        | 8086 | Order creation and order processing     |
| Payment Service      | 8087 | COD and Razorpay payments               |
| Notification Service | 8088 | Order-related email notifications       |

Each service is maintained as an independent GitHub repository.

## Technology Stack

* Java 17
* Spring Boot
* Spring Security
* JWT
* BCrypt
* Spring Data JPA
* Hibernate
* MySQL
* OpenFeign
* Apache Kafka
* Razorpay
* Resilience4j
* JavaMailSender
* Maven
* Git & GitHub

## Authentication

User Service uses Spring Security, JWT and BCrypt for authentication.

### Login Flow


Client -> Email + Password -> User Service -> Authentication -> JWT Token -> Client ->  Authorization: Bearer <JWT> -> Protected APIs



### User Service APIs

| Method | Endpoint                  | Description                        | Access                          |
| ------ | ------------------------- | ---------------------------------- | ------------------------------- |
| POST   | `/auth/register`          | Register a new user                | Public                          |
| POST   | `/auth/login`             | Authenticate user and generate JWT | Public                          |
| GET    | `/auth/getuser/{uid}`     | Get basic user information         | Based on security configuration |
| GET    | `/auth/UserDetails/{uid}` | Get user details with role         | ADMIN                           |

Admin access is enforced using:


@PreAuthorize("hasRole('ADMIN')")


## Main E-Commerce Flow

User → Register / Login → User Service → JWT → Product Service → Select Product → Cart Service → Add to Cart → Order Service


Order Service
      ↓
      ├──→ Payment Service → Razorpay
      │
      └──→ Inventory Service → Inventory DB

Payment / Inventory Events → Kafka → Notification Service



## Order Processing

The Order Service coordinates the order creation process.

The Order Service does not directly manage inventory data. Inventory operations are handled by the Inventory Service.

Similarly, cart data is managed by the Cart Service.

### COD Flow


Create Order
     |
     v
Order Service
     |
     v
Payment Service
     |
     | Payment = PENDING
     v
Order Confirmed
     |
     v
Inventory Processing
     |
     v
Order Delivered
     |
     v
Payment = PAID


### Online Payment Flow


Create Order
     |
     v
Order Service
     |
     v
Payment Service
     |
     v
Create Razorpay Order
     |
     v
Razorpay Checkout
     |
     v
Payment
     |
     v
Razorpay Webhook
     |
     v
Signature Verification
     |
     v
Payment = PAID


## Payment Integration

The Payment Service supports:

* Cash on Delivery
* Online payment using Razorpay

### COD

For COD orders, the Payment Service creates a payment record with:


Payment Method : COD
Payment Status : PENDING

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

Current application topics include:


order.created
inventory.reserve
inventory.reserved
inventory.updated
payment.success
order.delivered


Example:

Order Service
     |
     | order.created
     v
   Kafka
     |
     v
Inventory / Other Consumers

Another example:


Order Service
     |
     | order.delivered
     v
   Kafka
     |
     v
Notification Service


Kafka helps keep service responsibilities separate and avoids unnecessary direct dependencies between services.

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


Ecommerce-Microservices
│
├── API Gateway
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


MySQL
Kafka
ZooKeeper


Kafka is currently configured with ZooKeeper for local development.

### Application Ports

| Service              | Port |
| -------------------- | ---: |
| API Gateway          | 8080 |
| User Service         | 8082 |
| Product Service      | 8083 |
| Inventory Service    | 8084 |
| Cart Service         | 8085 |
| Order Service        | 8086 |
| Payment Service      | 8087 |
| Notification Service | 8088 |

Start the required infrastructure first and then start the Spring Boot services.

## Configuration

Database and service-specific configuration is maintained inside each microservice.

Sensitive configuration values should not be committed to GitHub.

Typical configuration includes:


DB_USERNAME
DB_PASSWORD
JWT_SECRET
RAZORPAY_KEY_ID
RAZORPAY_KEY_SECRET
MAIL_USERNAME
MAIL_PASSWORD


For local development, configure these values in the appropriate application configuration or environment variables.

## Git Workflow

The project follows a feature-based Git workflow.


feature/*
     |
     v
  develop
     |
     v
    main


Development work is first performed on a feature branch.

After testing, the feature branch is merged into `develop`.

Stable changes are then merged into `main`.

## Repository Links

Individual microservices are maintained in separate repositories:

* API Gateway
* User Service
* Product Service
* Inventory Service
* Cart Service
* Order Service
* Payment Service
* Notification Service

Repository links will be added here as the individual repositories are published.

## Author

**Deepak Kumar**

Java Backend Developer
