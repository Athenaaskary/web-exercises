# HTTP Practical Assignment

## Overview

This project implements a JWT-authenticated reverse proxy for the
`go-httpbin` service.

The legacy service does not provide authentication by itself. The proxy
receives client requests, validates the JWT in the `Authorization`
header, and forwards valid requests to the internal service. Requests
without a valid token are rejected.

## 1. Running a Service Only on an Internal Network

A service can be kept on an internal Docker network by placing it on a
network configured with `internal: true`.

In this project, `httpbin` is attached only to the internal network. It
does not publish a port to the host. The proxy is the entry point for
client requests and is connected to the network that accepts external
traffic as well as the internal network.

This means clients cannot access `httpbin` directly from the host or the
external network. They must send requests to the proxy. Docker
service-name discovery allows the proxy to reach the backend using
`http://httpbin:8080`.

The `expose` setting documents the port used by the service for
communication between containers. It does not publish that port on the
host.

## 2. TLS Termination

TLS Termination means that an incoming HTTPS connection is decrypted at
a trusted entry point, such as a reverse proxy or load balancer.

The client establishes an encrypted TLS connection with the proxy. The
proxy handles the TLS certificate and encryption, then forwards the
request to the internal backend. The connection between the proxy and
the backend can use the private internal network.

This approach centralizes certificate management and avoids configuring
TLS separately on every internal service. It also allows the backend
service to remain isolated from direct external access.

For production environments, the connection between the proxy and
backend should also be protected when the internal network cannot be
fully trusted.

## 3. JWT Authentication Proxy

The proxy is implemented in Go and uses `github.com/golang-jwt/jwt/v5`
to validate JSON Web Tokens.

For each incoming request, the proxy:

1.  Reads the `Authorization` header.
2.  Checks that the header uses the `Bearer` scheme.
3.  Parses the JWT using the configured secret.
4.  Accepts only the HS256 signing algorithm.
5.  Requires the token to contain an expiration claim and checks token
    validity.
6.  Returns `401 Unauthorized` if the token is missing, invalid, or
    expired.
7.  Forwards the request to `httpbin` if the token is valid.

The upstream service URL and JWT secret are provided through environment
variables:

-   `UPSTREAM_URL` specifies the internal backend address.
-   `JWT_SECRET` specifies the secret used to verify JWT signatures.

The proxy listens on port `8081` inside its container. Docker Compose
publishes the proxy on host port `80`.

## Project Structure

``` text
http-practical/
├── docker-compose.yml
├── .env
└── proxy/
    ├── Dockerfile
    ├── go.mod
    ├── go.sum
    └── main.go
```

## Running the Project

Create a `.env` file in the project root and set a secret:

``` env
JWT_SECRET=replace_with_a_long_random_secret
```

Start the services:

``` powershell
docker compose up -d --build
```

Check the running containers:

``` powershell
docker compose ps
```

View the proxy logs:

``` powershell
docker compose logs --tail=50 proxy
```

Stop the services:

``` powershell
docker compose down
```

## Testing

A request without an authorization token should be rejected:

``` powershell
curl.exe -i http://localhost/
```

Expected response:

``` text
HTTP/1.1 401 Unauthorized
```

A request with an invalid token should also be rejected:

``` powershell
curl.exe -i -H "Authorization: Bearer invalid-token" http://localhost/get
```

Expected response:

``` text
HTTP/1.1 401 Unauthorized
```

A request with a valid, unexpired HS256 token signed with the configured
`JWT_SECRET` should be forwarded to `go-httpbin`:

``` powershell
curl.exe -i -H "Authorization: Bearer <VALID_JWT>" http://localhost/get
```

The successful response comes from `go-httpbin`. Its `/get` endpoint
returns information about the received request.

## AI Tool Usage

AI tools were used only to check the correctness of the project and its
configuration.

