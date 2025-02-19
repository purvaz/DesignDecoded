# Tiny URL

For any system that takes a URL and converts it into a tiny URL, we need to understand a few aspects. 

- What is the maximum length of the tiny URL generated?
- What is the traffic that has to be handled?
- How long do we have to store the tiny URL in the database?
- What characters are allowed? Can it have lowercase and uppercase characters? Or is just one of the two allowed? Are numbers allowed?

## Assumptions:
1. There are X number of requests per seconds, and the data is stored for 10 years. So, the total number of requests in 10 years would be calculated as 
    <p align="center"> Y = X × 60 × 60 × 24 × 365 × 10 </p>
2. The URL consists of uppercase, lowercase characters and numbers [a-z, A-Z, 0-9]. Hence, total number of characters allowed are 62.
3. If 62 characters are allowed, then the total number of combinations given a particular length of the URL would be as follows:
 <p align="center">
   1 → 62 <br/>
   2 → 62<sup>2</sup> = 3,844<br/>
   6 → 62<sup>6</sup> ≈ 58 billion<br/>
   7 → 62<sup>7</sup> ≈ 3.5 trillion<br/>
 </p>
	&nbsp;&nbsp;&nbsp;&nbsp; We are assuming that a 7 character long URL is enough to store all the requests.

## Functional Requirements:
1. The original (long) URL is sent to the tinyURL system and the short URL is returned. This might be used by individual users or via another system. 
2. The short URL is sent to the system and the original long URL is returned. 
3. Analytics needs to be stored (optional, is not a hard functional requirement).

## Design Thinking Pattern:
- The Shorten URL Service is the service that performs the actual shortening and saves the long and the short URLs in the database. 
- The Shorten URL Service has multiple instances to support scalability and low latency. The redirection is handled by a Load Balancer, which redirects the requests to different instances based on the load. 
- To support scalability, the multiple instances approach works well, but this might result in duplicate short URLs being generated for two or more long URLs due to race conditions. This would result in a loss of reliability and integrity. 
- To avoid this, we can have a Redis DB that every instance of the service could hit and get a unique number that self-increments after every hit. However, since all the service instances are connected to the Redis DB, it could turn out to be a single point of failure. 
- To handle this, we can have multiple instances of Redis to handle different instances of the service. This would be useful to avoid the Redis DB being the single point of failure. However, this only works till the Redis instances start generating duplicate numbers. 
- To solve this issue, we can have the Redis DBs give out the unique numbers in a specific range. This works even when more than 2 Redis instances are added. However, then we need to add some sort of management component for the Redis instances so that they do not work on the same range of IDs. 
- This becomes too complicated, hence we form a separate service for token generation called the Token Service. This will provide the Shorten URL Service instances a range (eg. 1001-2000, 5001-6000) instead of a certain number. This will also reduce the number of calls to the token service. 
- This will make sure that the token is always unique and the same range is not assigned to two different services. This is a simple MySQL database and is maintained on a transaction basis. It would typically have a range and a flag, whether the particular range is assigned to a service or not. 
- This token service is also scalable and can have multiple instances. However, since it is MySQL-based and has ACID properties, there is no chance or duplicate ranges being assigned. Additionally, we can keep the range in millions instead of thousands so that the Token service does not get frequent requests. 
- There can occur a scenario where a range is assigned to one of the Shorten URL Services and after assigning a few unique tokens, the service shuts down. Now, the remaining unassigned tokens do not have any record. But keeping track of all these unassigned tokens is a very complicated system and need not be created, as these ranges are fairly small in front of the actual capacity (eg. letting go of approximately a few thousand tokens is nothing in front of 3.5 trillion capacity. 
- As for the conversion of a short URL to retrieve the long URL, the request will be sent to the service which will retrieve the long URL from the database and send it back to the caller. 
- The reason why we are using Cassandra here is for the sheer ability to store and handle a large capacity of billion or trillion records. MySQL would also work if it is sharded appropriately. But with Cassandra, there is no need to do additional work of sharding and handling. 
- We can even implement a cache for storing the frequently accessed URLs instead of making calls to the database every time. 

