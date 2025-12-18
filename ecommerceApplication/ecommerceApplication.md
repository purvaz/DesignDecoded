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

### a. User Service:

**Database:** Users DB (Relational: MySQL/PostgreSQL)  
**Attributes:** user_id, username, email, phone_number, hashed_password, created_at, updated_at  
**Core Responsibilities:**  
&emsp; Manage user registration, authentication, profile updates  
&emsp; Manage JWT tokens or server-side sessions  
**APIs:**  
&emsp; Provide user information to other services via REST APIs  
&emsp; POST /auth/signup  
&emsp; POST /auth/login  
&emsp; GET /users/{user_id}  
&emsp; PATCH /users/{user_id}  

### b. Product Catalog Service:

**Database:**   
&emsp; Products DB: Document-Based (NoSQL)  
&emsp; Categories DB: Document-Based (NoSQL)   
**Attributes:**   
&emsp; Products DB: product_id (PK), seller_id (FK), category_id (FK), name, description, price, stock_available, created_at, updated_at  
&emsp; Categories DB: category_id (PK), name, parent_id (NULL if top-level)  
**Core Responsibilities:**   
&emsp; Store product information, maintain category hierarchy  
&emsp; Provide product listings, creation, updates  
&emsp; Update inventory on successful order purchase  
**APIs:**  
&emsp; POST /products  
&emsp; GET /products/{product_id}  
&emsp; PATCH /products/{product_id}  
&emsp; GET /categories  

### c. Search Service:

**Database:** search-index DB (Elasticsearch)  
**Attributes:**   
&emsp; Search documents (indexed from products catalog)  
&emsp; Typical fields include product_id, name, description, price, category, keywords  
**Core Responsibilities:**  
&emsp; Full text search, auto-suggest, filtering  
&emsp; Update indexes in near real-time or batches from Product Catalog Service  
**APIs:**  
&emsp; GET /search?q=iphone  
&emsp; GET /suggest?prefix=iph  

### d. Shopping Cart Service:

**Database:** Carts-DB (Relational for persistence), Cart_Items DB  
**Attributes:**  
&emsp; Cart (cart_id (PK), user_id, status (active/abandoned), created_at, updated_at)  
&emsp; Cart_items (cart_item_id (PK), cart_id (FK), product _id, quantity, price_when_added)  
**Core Responsibilities:**  
&emsp; Maintains user cart states (items, quantities, sub-total)  
&emsp; Provides APls to add/remove items, fetch/update cart  
**APls:**  
&emsp; POST / cart/add  
&emsp; POST / cart/remove  
&emsp; GET / cart  

### e. Order Service:

**Database:** Orders DB (Relational, with ACID compliance)  
**Attributes:**  
&emsp; Orders: order_id (PK), user_id (FK), total_amount, status, created_at, updated_at  
&emsp; Order_items: order_item_id(PK), order_id(FK), product_id(FK), quantity, price  
**Core Responsibilities:**  
&emsp; Create and manage orders, track lifestyle (pending, paid, shipped, delivered)  
&emsp; Coordinate with Payment service, update inventory in product catalog  
&emsp; Issue notifications  
**APIs:**  
&emsp; POST /orders(checkout)  
&emsp; GET /orders/{order_id}  
&emsp; POST /notifications/{order_id, status}  
&emsp; POST /inventory/{product_id, quantity}  

### f. Payments Service:

**Database:** Payments DB (Relational with strong consistency)  
**Attributes:** payment_id (PK), order_id(FK), wallet_id(FK), payment_method, transaction_ref, created_at, updated_at  
**Core Responsibilities:**  
&emsp; Integrate with external gateways (PayPal, Stripe)  
&emsp; Manage payment auth, captures, refunds  
&emsp; Store transaction details for audits  
**APIs:**  
&emsp; POST /payments/pay (initiate)  
&emsp; GET /payments/refunds  

### g. Reviews and Ratings Service:

**Database:** Reviews DB (Relational or Documents-Based)  
**Attributes:** review_id (PK), user_id(FK), product_id(FK), rating, comment, created_at, updated_at  
**Core Responsibilities:**  
&emsp; Store user reviews and ratings  
&emsp; Aggregate ratings  
**APIs:**  
&emsp; POST /reviews  
&emsp; GET /reviews?product_id=xxx  

### h. Notifications Service:

**Database:** Notifications DB (Optional) - can be Relational or Key-Value based  
**Attributes:** notifications_id (PK), user_id(FK), type (email, SMS, in-app), status (sent/failed), created_at, updated_at  
**Core Responsibilities:**  
&emsp; Send emails, SMS, push notifications based on events  
&emsp; Subscribe to event bus, can be called directly by other services  
**APIs:**  
&emsp; POST /notifications  

### i. Analytics Service:

**Database:** Analytics DB - Data Warehouse (OLAP)  
**Attributes:**  
&emsp; Various fact tables (orders, product views, user registrations, reviews)  
&emsp; Dimension tables for products, users, time  
**Core Responsibilities:**  
&emsp; Collect data from orders, products, user activity for reporting  
&emsp; Provides dashboards or reports (seller performance, user engagement)  
**APIs:**  
&emsp; Batch ETL or real-time streaming from other microservices  
&emsp; Internal dashboard for analytics  

## 4. High-Level Design:

HLD satisfies the Functional Requirements.

**Microservices:**

- To begin with, the user will login/sign up. The user’s request goes to the API gateway and is then redirected to the appropriate microservice, which is the User Management microservice. This will contain the user service which is connected to its own user DB. 
- The user service contains logic to authenticate the username and password and grant access if correct. 
- Auth service, which is responsible for generating the JWT tokens and keeping the session alive can be a part of the user service, but we have separated it to keep the user service’s scope in check. This way the Auth service can be reused later for other services. 
- If other systems want to access the JWT tokens or need to interact with the Auth service, we are gonna publish REST APIs that will enable this interaction. 
- The product management microservice contains the product catalog service which interacts with the product DB. The product DB is a no-sql database as it needs to support a vast variety of category listings. This cannot be effectively done in a traditional sql database. 
- Using a nosql database which supports different categories of products also helps with managing the inventory, since all products have different attributes. 
- User should be able to search and filter the products based on different parameters. This searching mechanism needs to be fast and efficient, and hence, we make use of Elastic Search database. This makes the searching seamless and gives a better user experience. It makes use of indexes which makes the searching so quick. 
- For the checkout, a shopping cart service will be present, which will save all the data in the carts db. If the customer has a promo code, it will be retrieved from the promo db. 
- The promo db is not combined with the shopping cart service and used as an additional service for better modularization. 
- All checkouts will be happening through shopping cart service, hence no other services will be able to access promo service directly, only through shopping cart service. 
- The wallet service handles all the cards or other methods stored for a faster checkout. Whenever we use the checkout option, and if we do a one time payment, then the wallet service is not used. Instead, if we have some stored cards or payment info, the wallet service gets the saved data from the wallet DB and returns to the payment service. All of the information in the wallet DB is encrypted and stored using all compliant protocols. 
- The payment service handles the payment and gets information from wallet, payments DB and any credit card/third party apps. 
- After the payment is complete, the order is placed and all of the orders are processed by the orders service and stored in the Orders DB. 
- Similarly, reviews and ratings have a separate service and a regular relational DB for storing the product reviews. 
- As for the analytics, the sellers or the as a system, we might want to look at the seller/buying analytics and these are usually stored in a Data Warehouse. These analytics are not performed on the live DB as it can be an expensive operation and it might overwhelm the live DB, especially when there is high traffic on the website.
- For this, we have a data warehouse that has a replica of the data and it is nightly synced or periodically synced instead of real time. 
- Whenever a significant event happens, we need to notify the user, and that is when this comes into play. We can use the notification service with any of the other microservices.
- This can be either in app push notification or email or message. 
- The notifications service will be using a Kafka PubSub model, as it will be subscribing to a lot of events and as they occur, the service will push out notifications to the user. 
- The Recommendation system has a Kafka Queue where items get added when the user searches for a product or adds to wishlist, or users with similar profiles buy some items. 
- The information is then passed to the Spark Streaming for near real-time processing and directly added to the Recommendations DB. 
- The Recommendations Engine Service show recommendations on the homepage to the user based on past searches/wishlisted items or similar trends. 

**Architecture:**

- The user request passes through the firewall and is sent to the API gateway. The API gateway acts as a router and a load balancer. If the API gateway doesnt inherently support load balancing, we can add a load balancer explicitly. 
- The user might want to login/signup and these requests are sent to the user management microservice. 
- Similarly, the user might just want to search a few products before loging in or signing up, and these requests are directly sent to the search service. 
- Without doing either of the two (login/signup or search), a user must be able to view a product timeline of all the popular products or items on sale on their feed. This comes from the product catalog service. 
- If a user searches for a product, they will be searching based on the existing product catalog. 
- The search service contains a lot of precursor work like adding indexes and server side sorting on the database so that the search is very quick.
- A user needs to be able to search for products after logging in, and hence the user management service needs to be able to communicate with the search db.
- Also, a user might be a seller, and they need to communicate with the product catalog service as a seller, and the product catalog service needs to make sure the seller is authenticated. Hence this will be a 2 way communication between the user and product catalog services. 
- A user who purchases a product should be able to write a review and also, before buying, the user should be able to look at the reviews. 
- A user should be able to add things into their cart even without logging in or signing up. But they shouldnt be able to place an order without signing up. 
- Next, after a user proceeds to checkout, the order service comes in. Notice that the order service only comes in when the user follows the proper flow of  
<p align="center"> login/signup → search (optional) → select product → add product to cart → checkout </p>  
- Without following the above flow, the user cannot directly check out and pay. Also, the wallet service needs to authenticate the user and hence communicates with the user management service. 
- Only when the payment goes through, an order is generated. Once the order is placed, the inventory management system must be updated as the product count will have decreased. 
- The notifications system must be updated once a payment goes through, whenever there are any changes in the order status, when there are abandoned items in the cart sitting for too long, whenever a product is whishlisted and becomes available, when a user creates an account for the first time or when a payment is successful.
- All of the above microservices are grouped together as one and accessed by the Analytics microservice that has an Analytics DB. The analytics DB is a large data warehouse since it stores massive volumes of data. 
- A CRON job runs periodically to gather the data from all the microservices. The reason why this is periodic and not realtime is because the analytics data need not be real time and can be a few hours old. 
- This service can further be linked to PowerBI or Tableau for report generation.






