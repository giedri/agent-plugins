# Authentication and Authorization

## Decision Tree

```
Is this a WebSocket API?
  YES -> Lambda Authorizer (REQUEST type only on $connect; TOKEN type not
         supported; cached policy applies for entire connection, must
         cover all routes) or IAM (SigV4)
  NO ->
Is the consumer an AWS service or resource?
  YES -> IAM Authorization (SigV4)
  NO -> Is the consumer a browser-based app?
    YES -> Do you use Cognito?
      YES -> REST API: Cognito User Pool Authorizer
             HTTP API: JWT Authorizer (Cognito issuer)
      NO  -> Do you use another OIDC provider?
        YES -> HTTP API: JWT Authorizer
               REST API: Lambda Authorizer (validate JWT)
        NO  -> Lambda Authorizer (custom logic)
    NO -> Is this machine-to-machine (M2M)?
      YES -> Do you need certificate-based auth?
        YES -> mTLS (Regional custom domain + S3 truststore)
        NO  -> OAuth 2.0 Client Credentials Grant (Cognito + JWT/Cognito authorizer)
      NO -> Lambda Authorizer (most flexible)
```

## IAM Authorization (SigV4)

- Works for REST, HTTP, and WebSocket APIs (WebSocket: evaluated on `$connect` only)
- Caller signs requests with AWS Signature Version 4
- Best for: AWS-to-AWS service calls, Cognito identity pools, resources already integrated with IAM
- **Cross-account REST API**: Requires BOTH IAM policy (caller account) AND resource policy (API account)
- **Cross-account HTTP API**: No resource policies; use `sts:AssumeRole` to assume a role in the API account
- **Multi-region**: SigV4 signatures are region-specific: the signing region must match the region receiving the request. In multi-region deployments with Route 53 failover or latency-based routing, clients signing for one region will get auth failures if routed to another. SigV4a (multi-region signing) is not supported by API Gateway. Workarounds: use a region-agnostic auth mechanism (Lambda authorizer, JWT) for multi-region APIs, or implement client-side retry logic that re-signs for the correct region on auth failure

## Lambda Authorizers

### REST API
- **TOKEN type**: Receives a single header value (typically `Authorization`) as input. Returns IAM policy document. If the identity source header is missing, API Gateway returns 401 immediately **without invoking the Lambda**; the authorizer function never gets the chance to handle missing tokens
- **REQUEST type**: Receives headers, query strings, stage variables, and context variables as input. Returns IAM policy document. When caching is enabled and identity sources are specified, a request missing any identity source returns 401 without invoking the Lambda
- Both types must return `principalId` (string identifying the caller) alongside the policy document. Missing `principalId` causes 500 Internal Server Error
- **Response limits**: IAM policy document max ~8 KB. Exceeding this or returning a malformed response causes 500 Internal Server Error (not 401/403), a common debugging pitfall
- **Caching**: TTL default 300s, max 3600s. Cache key is the token value (TOKEN type) or identity sources (REQUEST type). When caching is enabled, the IAM policy returned by the first request is reused for subsequent requests with the same cache key. If that policy only covers specific resources (e.g., the path of the initial request), subsequent requests to other paths will be denied by the cached partial policy, causing hard-to-troubleshoot failures where clients intermittently cannot access parts of the API. Always generate IAM policies that cover the entire API when caching is enabled

### HTTP API
- **Simple response format**: Returns `{isAuthorized: true/false, context: {...}}`, much simpler than IAM policy
- **IAM policy format**: Also supported for more complex authorization. When using IAM policy format with caching, the same full-API policy guidance from REST API applies; see REST API caching note above
- **Identity sources**: `$request.header.X`, `$request.querystring.X`, `$context.X`, `$stageVariables.X`
- **Caching**: Disabled by default (TTL=0), unlike REST API (TTL=300s). Add `$context.routeKey` to identity sources to cache per-route when enabling caching
- **Timeout**: 10,000ms max

### Common Patterns

**Multi-tenant per-tenant throttling**:
Dynamic API key injection for per-tenant rate limiting without exposing API keys to tenants:
1. **Onboarding**: For each tenant, create an API key in API Gateway, associate it with a tiered usage plan (bronze/silver/gold), and store the `tenantId → apiKey` mapping in DynamoDB
2. **Auth flow**: Tenant authenticates with external IdP (Auth0, Cognito, etc.), receives JWT with tenant ID in claims
3. **Lambda authorizer**: Validates JWT (structure, expiration, signature, audience), extracts tenant ID from claims, looks up the tenant's API key from DynamoDB, returns it in `usageIdentifierKey`
4. **API Gateway**: Uses the returned API key to enforce the associated usage plan's rate limit. Returns 429 if exceeded
5. **Caching layers**: (a) Lambda authorizer caches IdP token validation keys in memory (JWKS), avoiding per-request calls to IdP. (b) API Gateway authorizer caching (default 300s, max 3600s) avoids re-invoking the authorizer for the same token
6. **Key benefit**: Tenants never see or manage API keys; they only use their JWT. API key management is fully transparent and centralized
- Set `ApiKeySourceType: AUTHORIZER` on the REST API so API Gateway uses the key from the authorizer response rather than the `x-api-key` header
- Requires REST API (HTTP API does not support usage plans or API keys)

**HttpOnly cookie authentication**:
- Lambda authorizer extracts JWT from HttpOnly cookie, validates with `aws-jwt-verify` library. Prevents XSS token theft
- Identity source: `$request.header.cookie` (HTTP API) or `method.request.header.Cookie` (REST API REQUEST type)
- Create JWT verifier outside handler for JWKS caching across warm starts

**mTLS client identity propagation**:
- REST API: Lambda authorizer extracts client certificate from `event.requestContext.identity.clientCert.clientCertPem`
- HTTP API (payload format 2.0): Client certificate is at `event.requestContext.authentication.clientCert.clientCertPem`
- Return extracted subject/fields in `context` for injection into backend request headers via `RequestParameters`

## JWT Authorizers (HTTP API Only)

- **Validates**: `iss`, `aud`/`client_id`, `exp`, `nbf` (must be before current time), `iat` (must be before current time), `scope`/`scp` (against route-configured scopes). Uses `kid` for JWKS key lookup. Request is denied if any validation fails
- Only RSA-based algorithms supported (RS256, RS384, RS512). ECDSA (ES256, ES384, ES512) is not supported. If your IdP signs tokens with ECDSA, use a Lambda authorizer instead
- Public key cached for 2 hours; account for this in key rotation
- Token validation runs on every request (no result caching); only the JWKS public keys are cached (2 hours). This differs from REST API Cognito authorizer which caches the validation result
- JWKS endpoint timeout: 1,500ms
- Max audiences per authorizer: 50. Max scopes per route: 10
- Use access tokens with scopes for authorization. ID tokens also work when no scopes are configured on the route, but access tokens are preferred for API authorization
- Only supports self-contained JWTs; opaque access tokens are not supported. If your IdP issues opaque tokens by default, use a Lambda authorizer instead
- Works natively with Cognito, Auth0, Okta, and any OIDC-compliant provider

## Cognito User Pools (REST API)

- Native authorizer type for REST APIs
- When no OAuth scopes configured on the method: use **ID token**
- When scopes configured: use **access token**
- Set up: Create user pool, app client, configure scopes on resource server
- **Token revocation not enforced**: The Cognito authorizer validates tokens locally (signature + claims) and does not check revocation status with Cognito. Revoked tokens (`GlobalSignOut`, `AdminUserGlobalSignOut`) are accepted until the token's `exp` time, as revocation is invisible to local validation regardless of caching. Separately, caching (default TTL 300s) means expired tokens may be accepted for up to the TTL duration after `exp`. For immediate revocation, use a Lambda authorizer with token introspection instead
- **M2M auth**: OAuth 2.0 Client Credentials Grant (confidential app client with client ID + secret, custom resource server scopes). Also works with HTTP API JWT authorizer using Cognito as issuer

## Resource Policies (REST API Only)

Four key use cases:
1. **Cross-account access**: Allow specific AWS accounts by specifying the account principal in the `Principal` field
2. **IP filtering**: Allow/deny CIDR ranges via `aws:SourceIp` (public) or `aws:VpcSourceIp` (private/VPC)
3. **VPC restriction**: Restrict to specific VPCs via `aws:SourceVpc`
4. **VPC endpoint restriction**: Restrict to specific VPC endpoints via `aws:SourceVpce`

### Policy Evaluation
Evaluation depends on which auth type is combined with the resource policy:
- **Same account + IAM or Lambda authorizer**: OR logic. If the auth mechanism allows, access is granted even if the resource policy has no matching statement (silent). An explicit Deny in the resource policy still wins
- **Same account + Cognito**: AND logic. Both the Cognito authorizer and the resource policy must allow
- **Resource policy alone** (no other auth): Must explicitly allow, otherwise request is denied
- **Cross-account**: AND logic. BOTH resource policy AND caller auth must explicitly allow. A silent resource policy results in implicit deny. This applies regardless of auth type (IAM, Cognito, Lambda authorizer)
- An explicit Deny always wins regardless of combination
- **Always redeploy the API after changing the resource policy**

## Mutual TLS (mTLS)

- Truststore in S3 (PEM-encoded, max 1,000 certs, max 1 MB). Certificate chain max 4 levels deep; minimum SHA-256 signature, RSA-2048 or ECDSA-256 key strength
- S3 bucket must be in the same region as API Gateway; enable versioning for rollback
- Works with **Regional** custom domain names for REST and HTTP APIs. Edge-optimized custom domains do not support mTLS
- WebSocket APIs do not support native mTLS; use CloudFront viewer mTLS instead (see `references/security.md`)
- ACM certificate required for the API Gateway domain (ACM-issued or imported) for server-side TLS. Truststore accepts CA certificates from any source (ACM Private CA, commercial CA, self-signed root); just needs PEM format
- **Private APIs do not natively support mTLS**. Use ALB as a reverse proxy in front: Client → ALB (mTLS verify with trust store) → VPC endpoint → Private API Gateway → backend. The ALB terminates the mTLS handshake, validates the client certificate, and forwards the request to the private API via the execute-api VPC endpoint
- **Disable default endpoint**: Always set `disableExecuteApiEndpoint: true` when using mTLS; otherwise clients can bypass mTLS entirely by calling the default `execute-api` URL directly
- **CRL checks**: API Gateway does not check Certificate Revocation Lists. Implement via Lambda authorizer checking against CRL in DynamoDB/S3
- **Certificate propagation to backend**: Use Lambda authorizer to extract subject, return in context, inject as custom header via `RequestParameters`

## API Keys

- **Not a primary authorization mechanism** (easily shared/exposed)
- Use with usage plans for throttling/quota enforcement only
- Max 10,000 API keys per region (adjustable). Imported key values must be 20-128 characters
- Key source: `HEADER` (default, `x-api-key`) or `AUTHORIZER` (Lambda returns key in `usageIdentifierKey`)
- REST API only. HTTP API does not support API keys or usage plans

## Access Control Combinations

Layer multiple mechanisms for defense in depth:
- Resource policy (network/account restriction) + IAM auth (identity verification)
- mTLS (client certificate) + Lambda authorizer (business logic authorization)
- WAF (web exploit protection) + Cognito (user authentication) + resource policy (IP restriction)
- VPC endpoint policy (network isolation) + resource policy (API-level control)
