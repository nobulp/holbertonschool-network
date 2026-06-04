What Happens When You Type https://www.google.com and Press Enter?

This is a classic software engineering interview question. The answer touches almost every layer of the modern internet stack. Let's walk through it step by step.


1. DNS Request — Translating a Name into an Address

The moment you press Enter, your browser needs to find the actual IP address that corresponds to www.google.com. Computers communicate using IP addresses (like 142.250.74.68), not human-readable names. This translation is the job of the Domain Name System (DNS).

Here is the lookup chain:

1. Browser cache — the browser first checks whether it recently resolved this hostname and cached the result. If yes, it skips everything below.
2. OS cache — if the browser has no cached entry, it asks the operating system, which checks its own DNS cache and the local /etc/hosts file.
3. Recursive resolver — if neither cache has an answer, the OS forwards the query to a recursive DNS resolver (usually provided by your ISP or a public resolver like 8.8.8.8). This resolver does the heavy lifting on your behalf.
4. Root nameserver — the resolver asks a root nameserver which nameserver is authoritative for .com.
5. TLD nameserver — the .com TLD nameserver directs the resolver to Google's authoritative nameserver.
6. Authoritative nameserver — Google's nameserver returns the IP address for www.google.com.
7. The resolver hands the IP back to your browser, which caches it for future requests according to the DNS record's TTL (Time To Live).

The whole process typically takes tens of milliseconds.


2. TCP/IP — Opening a Connection

With the IP address in hand, your browser needs to establish a reliable communication channel. The internet uses TCP/IP for this.

IP (Internet Protocol) handles routing — it breaks data into packets and routes each one across networks until it reaches the destination IP address.

TCP (Transmission Control Protocol) provides reliability on top of IP. Before sending any real data, TCP performs a three-way handshake:

    Client → SYN      → Server   (I want to connect)
    Client ← SYN-ACK  ← Server   (Acknowledged, I'm ready)
    Client → ACK      → Server   (Connection established)

This handshake synchronises sequence numbers so that both sides can detect lost packets, reorder out-of-order data, and retransmit when needed. Once the handshake completes, a reliable TCP connection exists between your machine and Google's server on port 443 (HTTPS).


3. Firewall — The Traffic Checkpoint

Before your packets ever reach Google's servers, and before Google's responses reach you, they pass through firewalls — both on your side and on Google's side.

A firewall inspects packets and applies rules: allow or deny based on source IP, destination IP, port, and protocol.

Your side: your home router or corporate network firewall may block outbound traffic on certain ports or flag unusual connection patterns.

Google's side: Google operates large-scale network firewalls and DDoS mitigation systems (like Google Cloud Armor) at the edge of their infrastructure. These block malicious traffic, rate-limit suspicious sources, and allow only legitimate requests through to the internal servers.

A packet that passes all firewall rules continues on its journey; one that does not is silently dropped or rejected.


4. HTTPS/SSL — Encrypting the Conversation

Because you typed https://, your browser does not send plain HTTP. Instead, after the TCP handshake, it immediately initiates a TLS handshake (Transport Layer Security — the modern successor to SSL).

The TLS handshake:

1. ClientHello — your browser tells the server which TLS versions and cipher suites it supports.
2. ServerHello — the server picks a cipher suite and sends its SSL certificate, which contains Google's public key and is signed by a trusted Certificate Authority (CA) like DigiCert or Google Trust Services.
3. Certificate verification — your browser checks that the certificate is valid, not expired, and signed by a CA in its trusted store. This proves you are talking to the real Google, not an impostor.
4. Key exchange — both sides agree on a session key using asymmetric cryptography (e.g., ECDHE). From this point on, all data is encrypted with that symmetric session key.
5. Finished — both sides confirm the handshake succeeded.

Every byte of data sent after this — your search query, Google's response, cookies — is encrypted. Nobody on the network path can read or tamper with it.


5. Load Balancer — Distributing the Work

Google handles billions of requests per day. No single server could handle that. When your encrypted request reaches Google's network edge, it hits a load balancer.

A load balancer is a system that sits in front of a pool of servers and distributes incoming requests among them according to an algorithm:

Round Robin — requests go to each server in turn.
Least Connections — new requests go to the server currently handling the fewest active connections.
IP Hash — the client's IP is hashed to always send it to the same backend (useful for session affinity).

Google's load balancing is distributed globally. Anycast routing sends your TCP connection to the nearest Google Point of Presence (PoP) in the first place. Within a data centre, software load balancers (like Google's Maglev) forward requests to application servers.

The load balancer also performs health checks — if a backend server is down, it stops sending traffic to it automatically.


6. Web Server — Handling the HTTP Request

Once the load balancer picks a backend, the request reaches a web server. Google uses its own highly optimised HTTP infrastructure, but the concept is the same as Nginx or Apache in a typical stack.

The web server:
- Parses the HTTP request line (GET / HTTP/2), headers, and body.
- Handles TLS termination (if not already done by the load balancer).
- Serves static assets (CSS, JS, images) directly from a fast cache or CDN.
- For dynamic content, forwards the request to the application server layer.

The web server is optimised for handling a very large number of concurrent connections efficiently (event-driven, non-blocking I/O).


7. Application Server — Running the Business Logic

The application server is where the actual logic of the product runs. For Google Search, this is a massive distributed system, but conceptually it does the same thing any app server does:

1. Receives the parsed request from the web server.
2. Identifies what the user wants (a search query, in this case).
3. Runs business logic — in Google's case, this triggers the search pipeline: query parsing, spell correction, index lookups, ranking algorithms, personalisation, ad selection, etc.
4. Constructs a response (the search results page).

The application server is stateless in most modern architectures — it does not store the user's session on disk. Session data lives in a distributed cache (like Redis or Memcached) or is encoded in signed cookies.


8. Database — Storing and Retrieving Data

At some point in the application logic, data must be read from or written to persistent storage. For Google Search, this involves proprietary distributed storage systems (Bigtable, Spanner, Colossus), but the concept maps to any database layer.

In a typical web stack, the database is a relational database (like MySQL or PostgreSQL) or a NoSQL store (like MongoDB or Cassandra). The application server:

1. Opens a connection to the database (often via a connection pool for efficiency).
2. Executes a query (e.g., SELECT to fetch data, INSERT to write).
3. The database engine processes the query, reads from disk or its in-memory buffer, and returns the result set.
4. The application server uses that data to build the response.

For high-traffic systems, reads are often routed to read replicas to offload the primary node, and hot data is cached in memory before the database is queried at all.


Putting It All Together

From the moment you press Enter to the moment the page appears, all of this happens in under 300 milliseconds — often much less:

You press Enter
→ DNS lookup resolves the IP address
→ TCP three-way handshake on port 443
→ Firewall checks (your router + Google's edge)
→ TLS handshake establishes an encrypted channel
→ Load balancer selects a healthy backend server
→ Web server parses the HTTP request
→ Application server runs the business logic
→ Database queries and returns data
→ Response travels back through the same chain
→ Your browser renders the page

It is one of the most impressive feats of distributed engineering in human history, and it happens billions of times a day without most people ever thinking about it.

Written as part of the Holberton School curriculum — Systems Engineering & DevOps track.
