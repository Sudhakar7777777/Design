# How to Design a URL Shortening service like TinyURL?

Design a TinyURL Service that converts URLs to short URLs.
eg: bit.ly, goo.gl, qlink.me, etc.

URL shortening is used to create shorter aliases for long URLs.

We call these shortened aliases “short links.” Users are redirected to the original URL when they hit these short links.
Short links save a lot of space when displayed, printed, messaged, or tweeted.

URL shortening is used for optimizing links across devices, tracking individual links to analyze audience and campaign performance, and hiding affiliated original URLs.

Example:

- Original URL : `https://spokeo.com/collection/page/5668639101419520/`
- Short Link : `http://tinyurl.com/jlg8zpc`

## Understanding the requirements: (Scope of discussion, ask clarifying questions)

### Functional Requirements:

1.  Given a URL, our service should generate a shorter and unique alias of it.This is called a short link. This link should be short enough to be easily copied and pasted into applications.
1.  When users access a short link, our service should redirect them to the original link.
1.  Links will expire after a standard default timespan. Users should be able to specify the expiration time.

### Non-Functional Requirements:

1.  The system should be highly available. This is required because, if our service is down, all the URL redirections will start failing.
1.  URL redirection should happen in real-time with minimal latency.
1.  Shortened links should not be guessable (not predictable).
1.  Capacity estimates: **Usage/Throughput/Traffic**

    Assuming 10 million users in the system. And create short links 10 per month. And each URL is read 100 times by other users.

    - Write load `10 * 10 million = 100M writes`
    - Read load `100 * 10 * 10 million = 10B reads`

    Read write ratio of `1000:1`. **Read heavy** system.

    We could calculate the system throughput using the scale.

    **Scale:** 1 request per second = `2.5 million` requests per month

    - Write throughput - `40 request per second`
    - Read throughput - `4000 request per second`

1.  Capacity estimates: **Storage**

    Size of 1 Record is: 1.27 KB

    ```
    shortlink      - 7 bytes (slug)
    expiration min - 4 bytes (expire)
    created_at     - 5 bytes (created)
    original url   - 255 bytes (destination)
    ...
    total          = ~1.27 KB
    ```

    Write 100M times = ~120 GB per month, 1.4 TB per year.

    Note: _The Short links expire after set duration. And expired links can be remove or delete, say after a month. (Purging or DB cleanup)_

## DB Schema:

    Table name - SHORT_LINK
    SLUG      VARCHAR(7)    NOT NULL
    EXP_MIN   INT(11)       NOT NULL
    CRT_TS    DATETIME      NOT NULL
    DEST_PATH VARCHAR(255)  NOT NULL

    PRIMARY KEY(shortlink)

## API Design:

Design the API endpoints (CRUD) to include the resource name, payload, and response code. For this part just design the interface, as documentation for developers or swagger output.

1. **Create** a new url, will auto generate the slug or the short link and create time.

   **Request:**

   ```
   ===>
   POST / HTTP/1.1
   Accept: application/json
   {
   "destination": "https://spokeo.com/abc",
   "expire": "1440"
   }
   ```

   PS: 60 _ 24 _ 10 = 1440 / 10 days in minutes

   **Success Response:**

   ```
   <===
   HTTP/1.1 200 OK
   Content-Type: application/json
   {
   "slug": "abcde",
   "destination": "https://spokeo.com/abc",
   "created": "2018-12-10T13:45:00.000Z"
   "expire": "1440"
   }
   ```

   **Application error reponse:**

   ```
   <===
   HTTP/1.1 400 Bad Request
   Content-Type: application/json
   {
   	"errors": [
   		{
   			"code":A123,
   			"message":"appl error details"
   		}
   	]
   }
   ```

   **System error reponse:**

   ```
   <===
   HTTP/1.1 500 Internal Server Error
   Content-Type: application/json
   {
   	"errors": [
   		{
   			"code":S123,
   			"message":"system error details"
   		}
   	]
   }
   ```

1. **Read or redirecting** user from the short link to destination url.

   ```
   ===>
   GET /{slug} HTTP/1.1
   Accept: application/json
   {}

   <===
   HTTP/1.1 302 Found
   Location: <<destination>> (header attribute)
   ```

1. **Update** the destination as user was to change to new url.

   ```
   ===>
   PUT /{slug} HTTP/1.1
   Accept: application/json
   {
   "destination": "https://spokeo.com/xyz",
   }

   <===
   HTTP/1.1 200 OK
   {
   "slug": "abcde",
   "destination": "https://spokeo.com/xyz",
   "created": "2018-12-10T13:45:00.000Z"
   "expire": "1440"
   }

   <===
   HTTP/1.1 404 NOT FOUND
   {
   	"errors": [
   		{
   			"code": "A1234",
   			"message": "slug resource not found."
   		}
   	]
   }
   ```

1. **Delete** the slug per user request.

   ```
   ===>
   DELETE /{slug} HTTP/1.1
   Accept: application/json
   {}

   <===
   HTTP/1.1 200 OK
   {
   	TRUE or FALSE
   }
   ```

### HTTP Response Code Reference:

#### Categories:

```
1XX 100-level (Informational) 	— Server acknowledges a request
2XX 200-level (Success) 	— Server completed the request as expected
3XX 300-level (Redirection) 	— Client needs to perform further actions to complete the request
4XX 400-level (Client error) 	— Client sent an invalid request
5XX 500-level (Server error) 	— Server failed to fulfill a valid request due to an error with server
```

#### Codes:

```
400 Bad Request 		— Client sent an invalid request — such as lacking required request body or parameter
401 Unauthorized 		— Client failed to authenticate with the server
403 Forbidden 			— Client authenticated but does not have permission to access the requested resource
404 Not Found 			— The requested resource does not exist
412 Precondition Failed 	— One or more conditions in the request header fields evaluated to false
500 Internal Server Error 	— A generic error occurred on the server
503 Service Unavailable 	— The requested service is not available
```

## Design the Algorith to generating Shortlink or Slug

1. How many unique slugs we need to create based on the system load?

   Per capacity above 100M writes / month. So, 1200M or 1.2B writes/year.
   If expired slugs are cleaned up then min of 2B unique slugs should be good.

1. How long should the slug be?

   Per current load we need ability to create 2B unique slugs.

   Slug Length : 5

   - If use only numbers - Base10: max `10^5` or `100000` or `100K` possible strings.
   - If use alpha + numbers (a-z, A-Z, 0-9) - Base62: max `62^5` or `~916M` possible strings.
   - If use (a-z, A-Z, 0-9, +, /) - Base64: max `64^5` or `~1.07B` possible strings.

   Slug Length : 6

   - If use only numbers - Base10: max `10^6` or `1000000` or `1M` possible strings.
   - If use alpha + numbers (a-z, A-Z, 0-9) - Base62: max `62^6` or `~56.8B` possible strings.
   - If use (a-z, A-Z, 0-9, +, /) - Base64: max `64^6` or `~68.7B` possible strings.

   Based on Base62 or Base64 encoding, using length of 6 we can generated > 2B unique slugs.

1. What is the algorithm to generate the slug?

   These are few options:

   1. SLUG = Base62(DB ID/Primary key)

   1. SLUG = Destination URL + Hash Algorthm

      Hash the destination URL with an algorithm like MD5 or SHA2 to generate unique hash value. Unfortunately these algorithm produce more than 8 characters so, we need to trucate 6. With this approach, we can end up with collisions. To avoid collision, check the slug is already used, if yes, pick the next 6 digits, etc.

   1. SLUG = Base62(UUID or rowid)

   1. SLUG = Base62()

   1. Generating Keys offline in separate service and manage its usage

   Regardless of the solution we choose, we need to understand how to manage collisions, latency impact on checking collisions, scalability and complexity of solution.

## Choosing the Database

Read are going to higher than the write...

NoSQL DB

MongoDB

## Scalability / Performance (assuming 1 Billion URLs and high traffic requests)

1. Ensure components are decoupled from one another and each service has 1 clear responsibility/domain.

2. Ensure there is no single point of failure in the system.

3. Dont solve the performance issue until you understand where is the bottleneck.

Initial Simplest solution:

Browser --> LB --> Webapp(H) --> DB

Knowing the system is READ heavy, quick benefits in cache incoming read requests.

Browser --> LB(+C) --> Webapp(H) --> DB

Cache on incoming request can be done in multiple ways:

1. Webapp add a Filter and validate incoming request has been cached, if yes, send cached response.
1. Add a proxy service between a LB and Webapp nodes to handle Cache or Rate limiting, etc.
1. Enable LB caching (disk based caching)
1. If using NGINX as LB, use NGINX Plus cache sharding that uses a consistent hashing algorithm.

If we the webapps are performing slower and more features are getting added, the webapp can be split as Read and Write service. Read service only serves redirection requests and Write service handles new shortlink / slug creations.

If we see the DB Reads and Write are slower. We can add a Caching layer before the DB. And use a simple write through cache or similar methods.

Browser --> LB(+C or +CD) --> Webapp(H) --> (+CD) -->DB

Cache Types: Inmemory cache, Distributed cache
Products: Redis, Ehcache, Hazelcast, etc.

And lastly instead of using one layer of LB, 2 layers can be used to ensure LB are no longer a single point of failures. Layer 1 LB can use simple algorithm to route to one of exising other LB layer2 nodes.

LB algorithms - Round robin, based on health, load of the target system.

Browser --> LB^2(+C or +CD) --> Webapp(H) --> (+CD) -->DB

# References

- https://github.com/donnemartin/system-design-primer/tree/master/solutions/system_design/pastebin
- https://www.educative.io/courses/grokking-the-system-design-interview/m2ygV4E81AR
- https://www.interviewcake.com/question/java/url-shortener
