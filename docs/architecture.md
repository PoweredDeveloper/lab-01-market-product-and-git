## Product Choice

- Yandex Go
- [Website](https://taxi.yandex.ru/)
- Yandex Go is a mobile app from Yandex focused on transportation and delivery. It was built on the foundation of Yandex Taxi.

## Main Components

![Yandex Go Component Diagram](/docs/diagrams/out/yandex-go/architecture-component/Component%20Diagram.svg)
[Diagram Source](https://github.com/PoweredDeveloper/lab-01-market-product-and-git/blob/main/docs/diagrams/src/yandex-go/architecture-component.puml)

### 1. Client Applications
Mobile App and Web App act as user-facing clients, communicating with the backend via REST APIs to request rides, pricing, payments, and status updates.

### 2. API Layer
The API Gateway exposes REST endpoints and routes requests to internal services using gRPC, handling concerns like authentication, throttling, and request aggregation.

### 3. Core Services
Domain-specific microservices (User, Dispatch, Pricing, Payment, Maps, Notifications) implement business logic and communicate with each other asynchronously and via RPC.

### 4. Data Layer
This layer provides persistence and data processing using operational databases (YDB), caching (Redis), analytics storage (ClickHouse), and event streaming via Kafka/Logbroker.

### 5. External Services
Third-party Yandex services such as Yandex Pay and Yandex Maps are integrated for payment processing and geolocation, routing, and map data.


## Data flow

![Yandex Go Sequence Diagram](/docs/diagrams/out/yandex-go/architecture-sequence/Sequence%20Diagram.svg)
[Diagram Source](https://github.com/PoweredDeveloper/lab-01-market-product-and-git/blob/main/docs/diagrams/src/yandex-go/architecture-sequence.puml)

### Group: 3. Booking & Async Dispatch

In this flow, the Mobile App sends a `createOrder` request to the API Gateway, which forwards it to the Dispatch Service to initialize a ride with the selected class and payment method. The Dispatch Service validates payment via the Payment Service, persists the order state (`SEARCHING`) in the Operational DB, and performs a geospatial search to find nearby drivers.

The Dispatch Service runs an internal matching algorithm, selects a driver, updates the order state to `ASSIGNED`, and publishes a `RideAssigned` event to Kafka. The Push Service consumes this event to notify the user (“Driver Found”), while the Mobile App updates the map view with driver details.

## Deployment

![Yandex Go Deployment Diagram](/docs/diagrams/out/yandex-go/architecture-deployment/Deployment%20Diagram.svg)
[Diagram Source](https://github.com/PoweredDeveloper/lab-01-market-product-and-git/blob/main/docs/diagrams/src/yandex-go/architecture-deployment.puml)

- **Client side**: The Yandex Go Mobile App runs on user smartphones (iOS/Android), while the Web App runs in the user’s web browser on a personal computer.
- **Edge / entry point**: All client requests enter Yandex Cloud through a public load balancer and are forwarded to the API Gateway over HTTPS.
- **Application tier**: Core backend services (User, Dispatch, Pricing, Payment, Maps & Routing, Notification) are deployed as pods within a Kubernetes cluster.
- **Data & messaging layer**: Redis (cache), Kafka (event bus), ClickHouse (analytics), and YDB (operational database) run in dedicated managed clusters inside Yandex Cloud.
- **External services**: Yandex Pay and Yandex Maps APIs are hosted in external Kubernetes clusters and accessed securely via HTTPS or gRPC.

## Assumptions

- I assume the Dispatch Service runs a real-time matching algorithm that considers driver proximity, availability, and service class when assigning a ride.
- I assume the system is heavily event-driven, with Kafka events used to propagate state changes (ride assigned, payment success) to decouple services and enable scalability.
- I assume Redis is used for low-latency, short-lived state (sessions, ride state, driver locations), while YDB is the source of truth for transactional data.

## Open questions

- How is the dispatch matching algorithm implemented internally, and what optimization criteria (ETA, driver score, fairness) are prioritized?
- What strategies are used to ensure data consistency and idempotency across services during retries and partial failures?
- How are surge pricing rules configured, updated, and rolled out safely without impacting active rides?

