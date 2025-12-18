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

**Database:** Users DB (Relational: MySQL/PostgreSQL)\
**Attributes:** user_id, username, email, phone_number, hashed_password, created_at, updated_at \
**Core Responsibilities:** \
&emsp; Manage user registration, authentication, profile updates \
&emsp; Manage JWT tokens or server-side sessions\
&emsp; Provide user information to other services via REST APIs \
**APIs:** \
&emsp; POST /auth/signup\
&emsp; POST /auth/login\
&emsp; GET /users/{user_id}\
&emsp; PATCH /users/{user_id}

#### b. Product Catalog Service:

**Database:** \ 
&emsp; Products DB: Document-Based (NoSQL) \ 
&emsp; Categories DB: Document-Based (NoSQL) \ 
**Attributes:** \ 
&emsp; Products DB: product_id (PK), seller_id (FK), category_id (FK), name, description, price, stock_available, created_at, updated_at \
&emsp; Categories DB: category_id (PK), name, parent_id (NULL if top-level) \
**Core Responsibilities:** \ 
&emsp; Store product information, maintain category hierarchy \
&emsp; Provide product listings, creation, updates \
&emsp; Update inventory on successful order purchase \
**APIs:** \
&emsp; POST /products \
&emsp; GET /products/{product_id} \
&emsp; PATCH /products/{product_id} \
&emsp; GET /categories

#### c. Search Service:

**Database:** search-index DB (Elasticsearch) \
**Attributes:** \ 
&emsp; Search documents (indexed from products catalog) \
&emsp; Typical fields include product_id, name, description, price, category, keywords \
**Core Responsibilities:** \
&emsp; Full text search, auto-suggest, filtering \
&emsp; Update indexes in near real-time or batches from Product Catalog Service \
**APIs:** \
&emsp; GET /search?q=iphone \
&emsp; GET /suggest?prefix=iph

#### d. Shopping Cart Service:

**Database:** Carts-DB (Relational for persistence), Cart_Items DB \
**Attributes:** \
&emsp; Cart (cart_id (PK), user_id, status (active/abandoned), created_at, updated_at) \
&emsp; Cart_items (cart_item_id (PK), cart_id (FK), product _id, quantity, price_when_added) \
**Core Responsibilities:** \
&emsp; Maintains user cart states (items, quantities, sub-total) \
&emsp; Provides APls to add/remove items, fetch/update cart \
**APls:** \ 
&emsp; POST / cart/add \
&emsp; POST / cart/remove \
&emsp; GET / cart

#### e. Order Service:

**Database:** Orders DB (Relational, with ACID compliance) \
**Attributes:** \ 
&emsp; Orders: order_id (PK), user_id (FK), total_amount, status, created_at, updated_at \
&emsp; Order_items: order_item_id(PK), order_id(FK), product_id(FK), quantity, price \
**Core Responsibilities:** \ 
&emsp; Create and manage orders, track lifestyle (pending, paid, shipped, delivered) \
&emsp; Coordinate with Payment service, update inventory in product catalog \
&emsp; Issue notifications \ 
**APIs:** \
&emsp; POST /orders(checkout) \
&emsp; GET /orders/{order_id} \
&emsp; POST /notifications/{order_id, status} \
&emsp; POST /inventory/{product_id, quantity}

#### f. Payments Service:

**Database:** Payments DB (Relational with strong consistency) \
**Attributes:** payment_id (PK), order_id(FK), wallet_id(FK), payment_method, transaction_ref, created_at, updated_at \
**Core Responsibilities:** \
&emsp; Integrate with external gateways (PayPal, Stripe) \
&emsp; Manage payment auth, captures, refunds \
&emsp; Store transaction details for audits \
**APIs:** \
&emsp; POST /payments/pay (initiate) \
&emsp; GET /payments/refunds

#### g. Reviews and Ratings Service:

**Database:** Reviews DB (Relational or Documents-Based) \
**Attributes:** review_id (PK), user_id(FK), product_id(FK), rating, comment, created_at, updated_at \
**Core Responsibilities:** \
&emsp; Store user reviews and ratings \
&emsp; Aggregate ratings \ 
**APIs:** \
&emsp; POST /reviews \
&emsp; GET /reviews?product_id=xxx

#### h. Notifications Service:

**Database:** Notifications DB (Optional) - can be Relational or Key-Value based \
**Attributes:** notifications_id (PK), user_id(FK), type (email, SMS, in-app), status (sent/failed), created_at, updated_at \
**Core Responsibilities:** \
&emsp; Send emails, SMS, push notifications based on events\
&emsp; Subscribe to event bus, can be called directly by other services\
**APIs:** \
&emsp; POST /notifications

#### i. Analytics Service:

**Database:** Analytics DB - Data Warehouse (OLAP) \
**Attributes:** \ 
&emsp; Various fact tables (orders, product views, user registrations, reviews) \
&emsp; Dimension tables for products, users, time \
**Core Responsibilities:** \
&emsp; Collect data from orders, products, user activity for reporting\
&emsp; Provides dashboards or reports (seller performance, user engagement)\
**APIs:** \
&emsp; Batch ETL or real-time streaming from other microservices\
&emsp; Internal dashboard for analytics





