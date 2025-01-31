# Authentication

Ambassador supports a highly flexible mechanism for authentication. An `AuthService` manifest configures Ambassador to use an external service to check authentication and authorization for incoming requests. Each incoming request is authenticated before routing to its destination.

There are currently two supported versions of the `AuthService` manifest:

### V1 (Ambassador 0.50.0 and higher):

`AuthService` V1, introduced in Ambassador 0.50, allows you to separately configure the headers that will be sent from the client to the auth service, and from the auth service to the upstream service. It also allows sending body data to the auth service.

You should use `AuthService` V1 for any new deployment of Ambassador 0.50 or higher.

```yaml
---
apiVersion: ambassador/v1
kind: AuthService
name: authentication
auth_service: "example-auth:3000"
path_prefix:  "/extauth"
proto: http
allowed_request_headers:
- "x-example-header"
allowed_authorization_headers:
- "x-qotm-session"
include_body:
  max_bytes: 4096
  allow_partial: true
status_on_error: 
  code: 503
failure_mode_allow: false
retry_policy:
  retry_on: "5xx"
  num_retries: 2
add_linkerd_headers: true
```

- `proto` (optional) specifies the protocol to use when communicating with the auth service. Valid options are `http` (default) or `grpc`.

- `allowed_request_headers` (optional) lists headers that will be sent from the client to the auth service. These headers are always included:
    * `Authorization`
    * `Cookie`
    * `From`
    * `Proxy-Authorization`
    * `User-Agent`
    * `X-Forwarded-For`
    * `X-Forwarded-Host`
    * `X-Forwarded-Proto`

- `allowed_authorization_headers` (optional) lists headers that will be sent from the auth service to the upstream service when the request is allowed, and also headers that will be sent from the auth service back to the client when the request is denied. These headers are always included:
    * `Location`
    * `Authorization`
    * `Proxy-Authenticate`
    * `Set-cookie`
    * `WWW-Authenticate`

- `include_body` (optional) controls how much of the request body to pass to the auth service, for use cases such as computing an HMAC or request signature:
    * `max_bytes` controls the amount of body data that will be passed to the auth service
    * `allow_partial` controls what happens to messages with bodies larger than `max_bytes`:
       * if `allow_partial` is `true`, the first `max_bytes` of the body are sent to the auth service
       * if false, the message is rejected. 

- `allow_request_body` is deprecated. It is exactly equivalent to `include_body` with `max_bytes` 4096 and `allow_partial` true.

- `status_on_error` (optional) status code returned when unable to communicate with auth service. 
    * `code` Defaults to 403

- `failure_mode_allow` (optional) if requests should be allowed on auth service failure. Defaults to false

- `add_linkerd_headers` (optional) when true, adds `l5d-dst-override` to the authorization request and set the hostname of the authorization server as the header value.

### v0 (Ambassador versions prior to 0.50.0)

`AuthService` V0 was current prior to Ambassador 0.50.0. It is deprecated and support for V0 will be removed in a future Ambassador release.

```yaml
---
apiVersion: ambassador/v0
kind:  AuthService
name:  authentication
auth_service: "example-auth:3000"
path_prefix: "/extauth"
allowed_headers:
- "x-qotm-session"
```

- `auth_service` gives the URL of the authentication service
- `path_prefix` (optional) gives a prefix prepended to every request going to the auth service
- `allowed_headers` (optional) gives an array of headers that will be incorporated into the upstream request if the auth service supplies them.

## Multiple AuthService resources

You may use multiple `AuthService` manifests to round-robin authentication requests among multiple services. **Note well that all services must use the same `path_prefix` and header definitions;** if you try to have different values, you'll see an error in the diagnostics service, telling you which value is being used.

## Using the AuthService API

By design, the AuthService interface is highly flexible. The authentication service is the first external service invoked on an incoming request (e.g., it runs before the rate limit filter). Because the logic of authentication is encapsulated in an external service, you can use this to support a wide variety of use cases. For example:

* Supporting traditional SSO authentication protocols, e.g., OAuth, OpenID Connect, etc.
* Support HTTP basic authentication (sample implementation available at:  https://github.com/datawire/ambassador-auth-httpbasic)
* Only authenticating requests that are under a rate limit, and rejecting authentication requests above the rate limit
* Authenticating specific services (URLs), and not others

## AuthService and TLS

You can tell Ambassador to use TLS to talk to your service by using an `auth_service` with an `https://` prefix. However, you may also provide a `tls` attribute: if `tls` is present and `true`, Ambassador will originate TLS even if the `service` does not have the `https://` prefix.

If `tls` is present with a value that is not `true`, the value is assumed to be the name of a defined TLS context, which will determine the certificate presented to the upstream service. TLS context handling is a beta feature of Ambassador at present; please [contact us on Slack](https://d6e.co/slack) if you need to specify TLS origination certificates.

## The External Authentication Service

The external auth service receives information about every request through Ambassador, and must indicate whether the request is to be allowed, or not. If not, the external auth service provides the response which is to be handed back to the client. The control flow for Authentication is shown below.

![Authentication flow](/doc-images/auth-flow.png)

### The Request

For every incoming request, the HTTP `method` and headers are forwarded to the auth service. Only two changes are made:

1. The `Content-Length` header is overwritten with `0`.
2. The body is removed.

So, for example, if the incoming request is

```
PUT /path/to/service HTTP/1.1
Host: myservice.example.com:8080
User-Agent: curl/7.54.0
Accept: */*
Content-Type: application/json
Content-Length: 27

{ "greeting": "hello world!", "spiders": "OMG no" }
```

then the request Ambassador will make of the auth service is:

```
PUT /path/to/service HTTP/1.1
Host: extauth.example.com:8080
User-Agent: curl/7.54.0
Accept: */*
Content-Type: application/json
Content-Length: 0
```

**ALL** request methods will be proxied, which implies that the auth service must be able to handle any request that any client could make. If desired, Ambassador can add a prefix to the path before forwarding it to the auth service; see the example below.

### Allowing the Request to Continue (HTTP status code 200)

To tell Ambassador that the request should be allowed, the external auth service must return an HTTP status of 200. **Note well** that **only** 200 indicates success; other 2yz status codes will prevent the request from continuing, as below.

The 200 response should not contain any body, but may contain arbitrary headers. Any header present in the response that is also listed in the `allow_headers` attribute of the `AuthService` resource will be copied from the external auth response into the request going upstream. This allows the external auth service to inject tokens or other information into the request, or to modify headers coming from the client.

### Preventing the Request from Continuing (any HTTP status code other than 200)

Any HTTP status code other than 200 from the external auth service tells Ambassador **not** to allow the request to continue. In this case, the entire response from the external auth service - including the status code, the headers, and the body - is handed back to the client verbatim. This gives the external auth service **complete** control over the entire response presented to the client.

Giving the external auth service control over the response on failure allows many different types of auth mechanisms, for example:

- The external auth service can simply return an error page with an HTTP 401 response.
- The external auth service can choose to include a `WWW-Authenticate` header in the 401 response, to ask the client to perform HTTP Basic Auth.
- The external auth service can issue a 301 `Redirect` to divert the client into an OAuth or OIDC authentication sequence.

Finally, if Ambassador cannot reach the auth service at all, it will return a HTTP 503 status code to the client.

## Configuring Public Mappings

Authentication can be disabled for a mapping by setting `bypass_auth` to `true`. This will tell Ambassador to allow all requests for that mapping through without interacting with the external auth service.

## Example

See [the Ambassador Authentication Tutorial](/user-guide/auth-tutorial) for an example.
