# Search Engine

For a regular search engine like Google, Bing or Mozilla Firefox, there are some functional and non-functional requirements we need to consider. 

### Functional Requirements:
1. User enters a search query
2. Search engine finds relevant sites, sort them by most relevant sites
3. User can see a list of titles and short descriptions as a result. 

### Non-functional Requirements:
1. Low latency
2. High throughput
3. Efficient Indexing
4. Fault Tolerance
5. Scalable

### Assumptions:
1. There are 1 trillion pages on the internet
2. Around 100 billion pages are unique
3. Each page is updated once every 10 days on an average
4. Average page is 2MB in size
5. So the BLOB storage size needs to be: 
  <p align="center"> (100 billion pages) * (2MB/page) = ~200 Peta Bytes </p>
&nbsp;&nbsp;&nbsp; 6. And assuming the metadata that needs to be stored in it is the URL (50B), hash (16B), BLOB ID (16B), last updated timestamp (8B), title (60B), desc (150B), priority (4B), the metadata size would be: <br/> <br/>
  <p align="center"> (100 billion pages) * (314 bytes/page of metadata) = ~31.4 TB </p>
 
 ---

### API Design:
_GET /query_

**Params:**
- Search query
- Page #

**Returns:** List of relevant sites
- Title
- Description
- URL

### Database Design:
**Query Patterns:**
- Get page by URL
- Get page by hash
- Search by a word

**Schema:**
- URL
- BLOB ID
- Site content
- Title
- Description
- Hash
- Last updated
- Priority
***
### Design Thinking Pattern:

#### Basic Design:
- When a user searches for some content on the search engine, it goes through an API gateway to the database that contains content for every page. 
- Now, to acquire every single page on the internet and store it in the database, we need some sort of a crawler that crawls through the internet and downloads the HTML pages from the internet and stores them in the database. 
- Now, to get every single URL, we can think of the Web as an actual web, meaning, when we get a single URL, we can go through the HTML page (using the crawler) and find other URLs associated with that page, dig deep and find other URLs in this way. Once we find the URLs, we add the new URLs to a new database. 
- Now, a large number of users are constantly accessing the search engine, searching for new sites. Hence, there needs to be a load balancer between the users and the API gateway. Similarly, the API gateways also contain multiple instances to support scalability. 

#### Storage:
- Now, considering the sheer size of data we calculated earlier (200 PB of pages and 31.4 TB of metadata), all of this cannot be stored on a regular sharded database, let alone a single database. Hence we need some sort of a BLOB storage system like Amazon S3 which will store the site’s content in a binary content form, and we can store the BLOB ID in our metadata DB. 
- The metadata database is going to be sharded, and it can be indexed based on the URL, the hash or the words. Sharding based on the URL and Hash is pretty straightforward, with the URL and Hash being the shard key respectively. In case of word, the word can be the shard key, with the frequency of it occurring in the URL or content being the sorting parameter. 

#### Crawler:
- Next, the crawler’s requirements are pretty straightforward. The crawler simply fetches a page, extracts the URLs on the page, and filters out the excluded URLs, and does this all over again recursively for all the pages.
- For filtering the excluded URLs, the crawler needs to consult the robot.txt file every time we need to crawl a site and if the URL is not excluded, the crawler can access the site. This might add a lot of overhead for the crawler, and hence, we add a robot.txt cache. 
- It would be optimal to have the physical location of the crawler closer to the physical location of the pages it accesses. 

#### URL Frontier:
- The URL frontier is a component that needs to hold every URL and then send them to the crawler in the right order. The order is based on priority (considering that different sites update at different frequencies, and that each site needs to be crawled eventually), and politeness (single crawler must be crawling at a host at a time and other crawlers can crawl different hosts).
- Now, the naive version of a URL frontier would just have a simple queue that would hold the URLs and would feed them one by one to the crawler which would crawl the host of the URL and push the URL back into the queue. This solves the problem of “hold every URL” but doesn't solve the issue of priority or politeness. 
- The lesser naive approach would be having a list of queues arranged in the order of priority. A prioritizer will feed the URLs in the queues according to their priority and a queue selector will select the URLs according to their priorities. 
- To make sure we implement politeness, we have a list of queues similar to the priority structure, but instead of being assigned according to priority, the queues are assigned according to the host. So one host will be assigned one queue. There will be a heap that keeps a tab on all the queues and updates the last time a URL from the queue was crawled. A selector runs over the heap and checks which queue is to be crawled, takes a URL from the queue and passes it on to the crawler. The crawler crawls over the host and passes it back to the prioritizer once done. This ensures only one crawler is working on a single host at a time. 


