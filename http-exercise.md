# HTTP Assignment — Web Course — Athena Askary

> An AI model (Claude Sonnet 5.5 by Anthropic) was used to draft, review, and refine the answers.

## Question 1 — HTTP Versions

### Part A: Key differences between HTTP versions

**HTTP/0.9** *(additional background, not covered in the lecture notes)*
- Supports only the `GET` method.
- Has no HTTP headers or status codes in the message format.
- Was designed for simple HTML document transfer.

**HTTP/1.0**
- Introduced headers, status codes, and additional methods such as `POST` and `HEAD`.
- Uses non-persistent connections by default: a new TCP connection is established for each object.
- Repeated connection setup increases network overhead, latency, and resource usage.
- Parallel connections can be used to improve loading speed.

**HTTP/1.1**
- Supports persistent connections, allowing multiple objects to be transferred over one TCP connection.
- Supports pipelining: multiple requests can be sent without waiting for each response.
- Pipelining can suffer from **Head-of-Line (HOL) Blocking**: a slow or large response can delay later responses.
- Requires the `Host` header to identify the target host.
- Supports methods such as `PUT`, `DELETE`, `OPTIONS`, and `TRACE`.
- Supports caching mechanisms such as `Cache-Control` and validation with `ETag`.

**HTTP/2**
- Aims to improve web transfer performance.
- Supports **multiplexing**, allowing multiple request/response streams to be active over a single connection.
- Reduces HTTP/1.1's response-ordering limitation, but packet loss on the shared TCP connection can still affect multiple streams.
- Introduced Server Push, which was intended to send resources before the client explicitly requested them.
- Server Push could waste bandwidth by sending resources already present in the browser cache; Chrome removed support for it in 2022.
- `103 Early Hints` lets a server indicate potentially needed resources early, allowing the browser to decide whether to start fetching them.

**HTTP/3**
- Uses QUIC, which runs over UDP *(additional background, not covered in the lecture notes)*.
- Handles streams more independently, reducing the impact of packet loss in one stream on other streams.
- Server Push is present in the HTTP/3 specification but was not practically deployed in browsers.
- HTTP semantics and caching rules are independent of the transport version.

### Part B: Why does HTTP/3 use UDP?

HTTP/3 uses **QUIC**, which runs over UDP. UDP alone does not guarantee reliable or ordered delivery; QUIC provides the necessary reliability features at its own layer.

- **Reducing Head-of-Line Blocking:** In HTTP/2, multiple streams share a TCP connection. If a packet is lost, TCP may delay delivery of subsequent data until the missing packet is retransmitted. QUIC manages streams independently, so packet loss in one stream does not necessarily block the others.
- **Implementation flexibility:** TCP is generally implemented in the operating system kernel, making changes harder to deploy. QUIC can be implemented and updated in user space.
- **Faster connection setup:** QUIC can combine connection establishment with TLS 1.3, reducing the time needed to start a secure connection. In suitable circumstances, setup can take one round trip or avoid an additional round trip.
- **Connection migration:** In suitable circumstances, QUIC can maintain a connection when the client changes networks or IP addresses.

**Important:** HTTP/3 does not use raw UDP as a reliable transport. It uses QUIC over UDP; QUIC provides features such as retransmission of lost packets and congestion control.

---

## Question 2 — Stateless and Stateful Communication

### A. Difference between stateless and stateful communication

**Stateless:** The server does not retain the state of previous client interactions between requests and processes each request independently. This simplifies request handling, but features such as user login need a separate mechanism to preserve state.

**Stateful:** The server retains information about previous client interactions, such as a user's session or shopping cart. This supports session management but requires the server to store and manage state.

### B. Is HTTP stateless or stateful, and how can an application preserve user state?

HTTP is a **stateless** protocol: by itself, the server does not retain the state of previous client requests. However, applications such as login systems and shopping carts need to associate multiple requests with the same user.

Using an IP address as a user identifier is unreliable. Multiple users may share one IP address behind NAT, and an IP address may change when a device changes networks or receives a new address. Therefore, an IP address alone is not a reliable identifier for a person.

Two common approaches are:

**1. Cookies**

The server sends a value to the browser using the `Set-Cookie` header. For subsequent requests that match the cookie's scope, the browser sends it back using the `Cookie` header. Commonly, the cookie contains a random, hard-to-guess session identifier, while the actual session data is stored on the server. The server should not trust client-controlled data without appropriate validation.

Example:

```http
Set-Cookie: session=RANDOM_ID; Path=/; HttpOnly; Secure; SameSite=Lax
```

**2. JSON Web Token (JWT)**

A JWT carries claims about identity or permissions in a token. It can be digitally signed so that tampering can be detected. In a common usage pattern, the client sends the token in the `Authorization` header using the `Bearer` scheme. Unlike cookies, a bearer token is not automatically attached by the browser to every matching request; client-side code generally sends it explicitly.

**Conclusion:** HTTP remains stateless, while an application can maintain user identity and state across requests using cookies and server-side sessions, or by using JWTs.

---

## Question 3 — The Limitation That Clients Initiate HTTP Communication

### A. Explain the challenge

HTTP normally follows a **request–response** model: the client sends a request, and the server sends data in response. In a regular HTTP exchange, the server cannot spontaneously send new data whenever it wants unless a request or a previously established communication channel is available.

This creates difficulties for applications such as chat, alerts, and live event updates, because the client may need to send repeated requests to learn about changes. Repeated requests create overhead, including headers and request processing. However, each HTTP request does not necessarily require a new connection; persistent connections can be reused.

### B. Possible solutions

**1. Polling**

The client sends requests at fixed intervals to check whether new data is available. This is easy to implement and uses ordinary HTTP request–response exchanges. However, repeated requests create overhead, and many may return no new data. If an event occurs just after a check, the client will not learn about it until the next request. The polling interval therefore affects both latency and request volume. Polling is suitable for simple periodic updates.

**2. Long Polling** *(additional background, not covered in the lecture notes)*

The client sends a request, and the server keeps the response open until a new event occurs or a timeout is reached. After receiving the response, the client immediately sends another request. Compared with ordinary polling, this can reduce empty responses and event-delivery delay. However, it still relies on request–response cycles and requires management of timeouts, pending requests, and reconnections.

**3. Server-Sent Events (SSE)**

The client establishes an HTTP connection, and the server sends new events over the same streaming response. In browsers, SSE can be used through the `EventSource` interface, and the stream uses the `text/event-stream` content type. SSE is suitable for alerts, live status updates, and event streams because a separate request is not needed for every event. Its main limitation is that the event channel is one-way, from server to client. Client-to-server data must use separate requests or another mechanism.

**4. WebSocket**

WebSocket establishes a persistent, full-duplex connection. Once it is established, both client and server can send messages without a separate HTTP request for each message. In the common HTTP/1.1 setup, the client starts with a request containing `Upgrade: websocket`; if accepted, the server responds with `101 Switching Protocols`. After the upgrade, messages are exchanged using the WebSocket protocol. Addresses begin with `ws://`, or `wss://` for secure connections.

WebSocket is suitable for real-time chat, online games, collaborative editing, and applications that require continuous two-way data exchange. In production, use `wss://`; the server should validate the `Origin` header and apply appropriate authentication and authorization.

**5. Webhook** *(additional background, not covered in the lecture notes)*

When an event occurs, the service producing the event sends an HTTP request, usually a `POST`, to a URL registered by the receiving service. For example, a payment service may notify an online store after a transaction succeeds. Webhooks are useful for event-driven communication between services. The receiver must have a reachable endpoint, and a webhook does not by itself create a channel for sending messages directly to a user's browser.

**6. HTTP/2 Server Push and 103 Early Hints**

HTTP/2 Server Push was designed to send page resources, such as CSS files, before the client explicitly requested them. In practice, it could send resources already in the browser cache, reducing efficiency; Chrome removed support for Server Push in 2022.

`103 Early Hints` works differently: the server can announce potentially needed resources before the final response, allowing the browser to start fetching them earlier. This helps page loading and is not a general replacement for real-time communication, chat, or event notifications.

**Summary:** Polling is simple but can create extra requests and delays. Long Polling can reduce empty responses and delay. SSE is suitable for one-way server-to-client events, while WebSocket is suitable for real-time two-way communication. Webhooks are mainly used for event notifications between services, and `103 Early Hints` helps browsers begin fetching page resources earlier.

---

## Question 4 — URL Components

**Given URL:**

```text
ss://asghar:1234!!@ss.myproxy.com:1234#shadowSocks1
```

A general URL structure is:

```text
<scheme>://<user>:<pass>@<host>:<port>/<path>?<query>#<fragment>
```

The components of the given URL are:

1. **Scheme:** `ss`. The scheme indicates how the address should be interpreted. In this example, `ss` refers to the scheme used for Shadowsocks; this is additional background beyond the lecture notes.
2. **User:** `asghar`. This is the username portion of the user information, which is separated from the host by `@`.
3. **Password:** `1234!!`. This follows the `:` in the user information and precedes `@`.
4. **Host:** `ss.myproxy.com`. This identifies the destination host. Domain names are not case-sensitive.
5. **Port:** `1234`. The port number follows the host and is separated by `:`.
6. **Path:** Absent; the path is empty. In ordinary web URLs, the path identifies a resource on the host.
7. **Query:** Absent; the query is empty. A query begins with `?` and can provide parameters for the destination application.
8. **Fragment:** `shadowSocks1`. This follows `#`. In ordinary HTTP URLs, a fragment is not sent to the server and is handled client-side. For the `ss` scheme, its exact meaning depends on the application parsing the address; it may be used as a label or display name.

**Note about special characters:** Some characters may need percent-encoding when used as ordinary URL data. For example, spaces and the `%` character may need encoding. The `!` characters in this example's password are permitted in this position and do not need to be encoded here.

**Conclusion:** This URL contains a scheme, username, password, host, port, and fragment. It has no path or query.

---

## Question 5 — Appropriate HTTP Status Code for Each Scenario

### 1. The website's database has a problem, and login requests are failing.

- **Suggested code: `500 Internal Server Error`**
- **Reason:** The problem is on the server side, and the server cannot process the login request because of an internal error.
- If the database is temporarily unavailable, `503 Service Unavailable` may be more appropriate. For a general internal server failure, `500` is the standard choice.

### 2. The username entered by the user does not exist, and login fails.

- **Suggested code: `401 Unauthorized`**
- **Reason:** Authentication has failed; the server has not successfully verified the user's identity.
- **Security note:** It is better to return the same response for a nonexistent username and an incorrect password to avoid user enumeration. If the input itself is malformed, such as a required field being empty, `400 Bad Request` may be appropriate.

### 3. The website has moved to another domain.

- **Suggested code: `301 Moved Permanently`**
- **Reason:** This indicates that the resource has permanently moved to a new URL. The destination is specified in the `Location` header.
- If the redirect is for an API and the request method and body must be preserved, `308 Permanent Redirect` is a better choice.

### 4. The number of requests sent by a user exceeds the permitted rate.

- **Suggested code: `429 Too Many Requests`**
- **Reason:** This status code is specifically used when a client sends too many requests within a given period (rate limiting).
- **Note:** This code is not included in the lecture notes' explicit status-code list. It is correct under the HTTP standard, but it is additional information beyond the listed lecture material.

### 5. A JWT refresh request succeeds, and the refreshed token is returned in a response header.

- **Suggested code: `200 OK`**
- **Reason:** The request succeeded, and the server returns the successful result. Returning the new token in a response header does not, by itself, require a different status code.

### 6. A user tries to access the service from a prohibited IP range.

- **Suggested code: `403 Forbidden`**
- **Reason:** The server understands the request but denies access according to its access policy. This is an access restriction and does not necessarily mean authentication failed.
- **Alternative:** `451 Unavailable For Legal Reasons` is appropriate only when the restriction is specifically due to legal reasons. For an ordinary IP-range block, `403` is the more general choice.

---

## Question 6 — Proxy Type for Each Scenario

### 1. Using university-internal servers instead of central servers to download package-manager packages, such as Ubuntu `apt` packages

- **Proxy type: Forward Proxy (Caching Proxy)**
- **Reason:** The proxy sends requests to external servers, such as Ubuntu repositories, on behalf of clients inside the university. This is the role of a Forward Proxy. It also caches downloaded packages so that later requests can be served from the internal cache when the stored version is still valid. This reduces network traffic and avoids repeatedly downloading the same data.

### 2. Distributing requests based on the geographical origin of the requests

- **Proxy type: Reverse Proxy (Load Balancing)**
- **Reason:** The proxy sits on the server side and receives user requests. It then routes requests to an appropriate backend server according to the client's geographic location or a configured routing policy. This can reduce latency and improve load distribution among servers.

### 3. Adding authentication to a very old service

- **Proxy type: Reverse Proxy (Authentication)**
- **Reason:** A Reverse Proxy can authenticate clients before forwarding requests to the legacy server. Only authorized requests are passed to the original service. This adds authentication without requiring changes to or a rewrite of the legacy service.
