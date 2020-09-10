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

## Design the Algorith to generate Short link or Slug

Create an Architecture design for the API described in part 1:

1. Algorithm for URL shortening

db row id

(or)

6 digit XXXXXX ==> 10\*\*6
take a random number....
make a string (charset -- alpha or.... 0-9, a-z, A-Z special char...)
validate the short is not used.

(or)

2 + 4 digit XXXXXX ==>
take a random number.... (db id)
make a string (charset -- alpha or.... 0-9, a-z, A-Z special char...)

short + long

======== Database choice/s ======

Read are going to higher than the write...

NoSQL DB

MongoDB

## Scalability / Performance (assuming 1 Billion URLs and high traffic requests)

1. Ensure components are decoupled from one another and each service has 1 clear responsibility/domain.

2. Ensure there is no single point of failure in the system.

Browser --> LB(M) (enable Distrubuted Memory Cache) --> Webapp (H) --> (DB Cache) DB

LB (?) -- > LB1, LB2, LB3 ..,,.. (Nginx -> Dist Cache)

LB Algorith -- Round robin, health, load, ...

LB1 .... lookup on Cache?

LB --> LB --> Proxy Service (Cache lookup : Memory ) (Cluster - of nodes -- share the memory )--> WebApp
