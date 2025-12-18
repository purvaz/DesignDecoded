# E-commerce Platform

An ecommerce platform includes the most common websites like Amazon, eBay, etc, which allows users to view and buy products. 

## 1. Requirements:
For any ecommerce platform like Amazon, there are some functional and non-functional requirements we need to consider. 

#### Functional Requirements
1. User Management: Account creation, authentication, profiles (buyer & seller), role management.
2. Product Management: Create, edit, categorize listings and manage inventory.
3. Search and Browse: Full text search, category filters (price, brands, ratings) and sorting.
4. Shopping cart and checkout: Add/remove items, update quantities, apply coupons. 
5. Ordering and Payment: Order creation, payments (credit card, wallet, 3rd party apps), refunds/returns.
6. Reviews and Ratings: Buyers can rate products and provide feedback. 
7. Seller Analytics: Dashboard with sales reports, order status and inventory tracking. 
8. Notifications: Order updates, promotions and shipping tracking.

#### Nonfunctional Requirements
1. Scalability: Support high traffic, spikes and large product catalogs. 
2. Reliability: 99.99% uptime and handle failures.
3. Performance: Fast page loads, quick searches and smooth experience.
4. Security and Compliance: Protect sensitive data, comply with norms.
5. Maintainability: Choose right architecture (monolith vs microservices), enable CI/CD.
6. Observability: Logging, monitoring and alerting for system health.
7. Cost efficiency: Optimize resource usage while balancing cost and performance. 

#### Assumptions:
- 1 billion users around the globe
- 100 million products, 1kb per product = 100GB
- 100 million orders per day
- High concurrency, multiple writes

## 2. Core Entities:
- User
- Product
- Cart
- Orders
- Payment & Wallet
- Reviews
- Notifications
- Analytics

## 3. APIs and Interfaces:
For this, it is pretty straightforward when you have the functional requirements in place. Just go to the functional requirements and one by one create the APIs to satisfy them. 

#### a. User Service:

**Database:** Users DB (Relational: MySQL/PostgreSQL)
**Attributes:** user_id, username, email, phone_number, hashed_password, created_at, updated_at
**Core Responsibilities:**\
Manage user registration, authentication, profile updates\
Manage JWT tokens or server-side sessions\
Provide user information to other services via REST APIs\
**APIs:**
POST /auth/signup\
POST /auth/login\
GET /users/{user_id}\
PATCH /users/{user_id}\

#### b. Product Catalog Service:

**Database:** 
Products DB: Document-Based (NoSQL) 
Categories DB: Document-Based (NoSQL) 
**Attributes:** 
Products DB: product_id (PK), seller_id (FK), category_id (FK), name, description, price, stock_available, created_at, updated_at
Categories DB: category_id (PK), name, parent_id (NULL if top-level)
**Core Responsibilities:** 
Store product information, maintain category hierarchy
Provide product listings, creation, updates
Update inventory on successful order purchase
**APIs:**
POST /products
GET /products/{product_id}
PATCH /products/{product_id}
GET /categories


