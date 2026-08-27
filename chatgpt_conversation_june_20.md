# MCP + Supabase Auth + OAuth Notes (June 20, 2026)

This document summarizes the MCP/Supabase/OAuth discussion. It excludes the initial Starlink question.

## 1. Core setup

The architecture being discussed is:

```txt
Claude / MCP client
        ↓
remote MCP endpoint on your backend
        ↓
MCP tool auth + tool permission checks
        ↓
shared backend service functions
        ↓
Supabase
```

Your MCP server is not a totally separate app. It is an endpoint on your backend server, such as:

```txt
https://backend.com/api/mcp
```

Your backend already has admin-protected endpoints that work like:

```txt
frontend sends Supabase JWT
        ↓
backend verifies JWT
        ↓
backend checks admins table
        ↓
backend allows or denies action
```

The key question was how Claude gets authorized to call MCP tools without Claude directly logging into Supabase.

## 2. MCP tools should call shared service functions, not your own API endpoints

Since the MCP server lives on the same backend as your API routes, it is cleaner for MCP tools to call shared backend functions directly.

Prefer this:

```txt
MCP tool
  ↓
updateProductService()
  ↓
Supabase
```

and:

```txt
Admin API route
  ↓
updateProductService()
  ↓
Supabase
```

Instead of this:

```txt
MCP tool
  ↓ fetch("https://same-backend.com/api/admin/update-product")
  ↓ API route
  ↓ Supabase
```

The better structure is:

```txt
src/
  app/
    api/
      mcp/
        route.ts
      admin/
        products/
          route.ts

  server/
    services/
      products.ts
    auth/
      requireAdmin.ts
    db/
      supabase.ts
```

Example:

```ts
// server/services/products.ts

export async function updateProductService({
  productId,
  updates,
  actor,
}: {
  productId: string;
  updates: {
    name?: string;
    priceCents?: number;
    stock?: number;
  };
  actor: {
    userId: string;
    role: "admin" | "user";
    scopes?: string[];
  };
}) {
  if (actor.role !== "admin") {
    throw new Error("Forbidden");
  }

  // update product in Supabase here
}
```

Then both the admin endpoint and the MCP tool call the same service:

```ts
// admin API route
const actor = await requireAdminFromSupabaseJwt(req);

return updateProductService({
  productId: body.productId,
  updates: body.updates,
  actor,
});
```

```ts
// MCP tool
const actor = await requireMcpAdmin(ctx);

return updateProductService({
  productId: args.productId,
  updates: args.updates,
  actor,
});
```

Important rule:

```txt
Do not put all security only in the API route.
The shared service should still require an actor/role/scopes.
```

Otherwise, an MCP tool could accidentally bypass permissions by calling the service directly.

## 3. Supabase JWT vs MCP access token

There are two possible designs.

### Option 1: Give Claude a Supabase JWT

Claude would send:

```http
Authorization: Bearer <supabase_access_token>
```

Then your MCP endpoint could verify it the same way your admin endpoints do.

This is simple, but less clean because Claude gets the user's real Supabase app session token.

### Option 2: Give Claude your own MCP access token

This is the better long-term design.

Flow:

```txt
User logs into your app with Supabase
        ↓
your app verifies user is in admins table
        ↓
your app creates a temporary OAuth authorization code
        ↓
Claude exchanges code for your MCP access token
        ↓
Claude sends MCP access token to /api/mcp
```

So there are two token types:

```txt
Supabase JWT:
  used between browser/frontend and your app/backend
  proves the user is logged into your app

MCP access token:
  created by your backend
  used by Claude to call MCP tools
  can have scopes like products:read, products:write, inventory:update
```

The clean design is:

```txt
/admin endpoint
  verifies Supabase JWT
  checks admins table
  calls shared service

/api/mcp endpoint
  verifies MCP access token
  checks scopes/admin role
  calls shared service
```

## 4. OAuth is the bridge/translator

A good mental model:

```txt
Supabase auth = proves who the user is to your app

/oauth/authorize = turns that logged-in app user into a temporary authorization code for Claude

/oauth/token = turns that temporary code into an MCP access token

MCP access token = what Claude sends when calling tools
```

OAuth is not literally converting the Supabase JWT into a Claude token. Instead:

```txt
/oauth/authorize reads the user's app login/session
        ↓
checks admins table
        ↓
creates a new temporary code
        ↓
Claude exchanges that code for your MCP token
```

The Supabase session stays between:

```txt
browser/frontend ↔ your app/backend
```

The OAuth code goes between:

```txt
your app/backend ↔ Claude callback
```

The MCP token goes between:

```txt
Claude ↔ your MCP endpoint
```

## 5. Authorization code vs access token

The authorization code is not the final credential.

It is a temporary, one-time-use claim ticket.

```txt
authorization code = temporary one-time pickup slip
access token = actual badge/keycard
```

The flow:

```txt
1. User logs into your app.
2. Your app confirms they are admin.
3. Your app creates a temporary authorization code.
4. Browser is redirected to Claude callback with ?code=...
5. Claude exchanges code at /oauth/token.
6. Your backend verifies code.
7. Your backend creates/returns MCP access token.
8. Claude uses MCP token in future tool requests.
```

Example authorization code record:

```ts
{
  code_hash: "hashed temporary code",
  user_id: "supabase-user-id",
  client_id: "claude-web",
  redirect_uri: "https://claude.ai/api/mcp/auth_callback",
  scopes: ["products:read", "products:write"],
  code_challenge: "...",
  code_challenge_method: "S256",
  expires_at: "5 minutes from now",
  used_at: null
}
```

Example MCP access token record:

```ts
{
  token_hash: "hashed MCP token",
  user_id: "supabase-user-id",
  scopes: ["products:read", "products:write"],
  expires_at: "1 hour from now",
  revoked_at: null
}
```

Claude sends the final token like:

```http
POST /api/mcp
Authorization: Bearer <mcp_access_token>
```

## 6. Expiring codes and tokens

You do not need a background process constantly watching tokens.

You just store `expires_at` and check it whenever a code/token is used.

Example check:

```sql
select *
from mcp_access_tokens
where token_hash = $1
  and expires_at > now()
  and revoked_at is null;
```

For authorization codes:

```sql
select *
from oauth_authorization_codes
where code_hash = $1
  and expires_at > now()
  and used_at is null;
```

Then mark the code as used:

```sql
update oauth_authorization_codes
set used_at = now()
where id = $1;
```

Expired rows can sit in the database. They are harmless if every query checks `expires_at > now()`.

Optional cleanup later:

```sql
delete from oauth_authorization_codes
where expires_at < now() - interval '1 day';

delete from mcp_access_tokens
where expires_at < now() - interval '7 days';
```

## 7. Hashing codes and tokens

Do not store raw authorization codes or raw access tokens in the DB.

Store only hashes.

```txt
raw token sent to Claude once
        ↓
hash stored in database
        ↓
future incoming token is hashed and compared
```

Example:

```ts
import crypto from "crypto";

function createRandomToken(prefix: string) {
  return `${prefix}_${crypto.randomBytes(32).toString("base64url")}`;
}

function hashToken(token: string) {
  return crypto.createHash("sha256").update(token).digest("hex");
}
```

When creating a token:

```ts
const rawToken = createRandomToken("mcp");
const tokenHash = hashToken(rawToken);

// Store tokenHash in DB.
// Return rawToken to Claude once.
```

When verifying:

```ts
const incomingHash = hashToken(incomingToken);

// Compare incomingHash to stored token_hash.
```

If someone steals your DB, they only see hashed tokens. If they try to use the hash as the token, your server hashes that hash again, which will not match.

### Passwords vs random tokens

For passwords:

```txt
Use Argon2id, bcrypt, or scrypt.
Do not use plain SHA-256.
```

For random MCP/API tokens:

```txt
Long random token + SHA-256 hash is usually fine.
```

Because a random 32-byte token is not guessable like a human password.

## 8. Hashing basics

A hash is one-way.

```txt
input → fixed-size fingerprint
```

You can check whether an input matches a hash by hashing the input again and comparing.

But you cannot reverse the hash back into the original input.

Hashing is not encryption.

```txt
Encryption:
  message + key → ciphertext
  ciphertext + key → original message

Hashing:
  message → hash
  hash → cannot recover original message
```

For a given algorithm, the output is always the same size.

Examples:

```txt
SHA-256("hi")                    → 256-bit hash
SHA-256("entire movie file")     → 256-bit hash
SHA-256("whole database backup") → 256-bit hash
```

This is one reason hashes cannot be reversed. A tiny input and a huge input both become a fixed-size output.

### Collisions

Different inputs can theoretically produce the same hash. That is called a collision.

For random API/MCP tokens, add a unique constraint on the token hash:

```sql
create table mcp_access_tokens (
  id uuid primary key default gen_random_uuid(),
  token_hash text not null unique,
  user_id uuid not null,
  scopes text[] not null,
  expires_at timestamptz not null,
  revoked_at timestamptz,
  created_at timestamptz default now()
);
```

Then:

```txt
generate token
hash token
insert hash
if unique conflict happens, retry
```

In practice, with 32 random bytes and SHA-256, a collision is absurdly unlikely.

## 9. PKCE: code challenge and verifier

PKCE protects the OAuth code exchange.

Claude/the MCP client creates:

```txt
code_verifier = random secret string
code_challenge = base64url(sha256(code_verifier))
```

Claude sends the user to your authorization URL with:

```txt
code_challenge=...
code_challenge_method=S256
```

Your app stores the `code_challenge` with the temporary authorization code.

Later, Claude exchanges the code by sending:

```txt
code=temporary_code
code_verifier=original_secret
```

Your backend checks:

```txt
base64url(sha256(code_verifier)) == stored code_challenge
```

If it matches, the same client that started the flow is finishing it.

### Why send `code_challenge_method`?

Because the server needs to know how to transform the later `code_verifier` before comparing it to the stored `code_challenge`.

Modern implementations should require:

```txt
code_challenge_method = S256
```

and reject `plain`.

### Why PKCE matters

Without PKCE:

```txt
stolen authorization code = access token
```

With PKCE:

```txt
stolen authorization code alone = useless
```

The attacker also needs the original `code_verifier`.

The `code_challenge` may be visible. That is okay because it is only a hash/fingerprint of the secret.

The `code_verifier` should not pass through the browser redirect URL. It is sent later directly to `/oauth/token`.

## 10. Redirect URI validation

Login proves:

```txt
who the user is
```

Redirect URI validation proves:

```txt
where the authorization code is allowed to be sent
```

If you accept any redirect URI, an attacker can create a malicious OAuth link:

```txt
/oauth/authorize?
  client_id=claude-web
  redirect_uri=https://evil.com/callback
  code_challenge=attacker_challenge
  code_challenge_method=S256
```

If you click that link while logged in as admin, and your app accepts any redirect URL, your app could redirect to:

```txt
https://evil.com/callback?code=real_code&state=...
```

The attacker controls:

```txt
redirect_uri
code_challenge
code_verifier
```

You supply the logged-in admin approval.

Then the attacker can exchange the code with their verifier and get an MCP token.

So you must allowlist redirect URIs.

For Claude web / hosted Claude surfaces, the callback is:

```txt
https://claude.ai/api/mcp/auth_callback
```

Your client row can be simple:

```txt
client_id: claude-web
allowed_redirect_uris:
  - https://claude.ai/api/mcp/auth_callback
```

Then `/oauth/authorize` should check:

```ts
if (
  clientId !== "claude-web" ||
  redirectUri !== "https://claude.ai/api/mcp/auth_callback"
) {
  return new Response("Invalid redirect_uri", { status: 400 });
}
```

## 11. Claude runtime vs Claude model

There is a difference between the model and the MCP client runtime.

```txt
Claude model:
  decides when a tool is useful

Claude/MCP runtime:
  discovers tools
  handles OAuth redirect flow
  stores/sends access tokens
  executes tool calls
```

Your MCP server exposes tools with names, descriptions, and input schemas.

Example:

```ts
server.tool(
  "update_inventory",
  "Update product inventory for an admin user",
  updateInventorySchema,
  async (args, ctx) => {
    // verify MCP token + scopes
    // call service function
  }
);
```

Claude sees the tool metadata and can choose it when relevant.

But your backend remains the authority:

```txt
Claude chooses tool
        ↓
MCP request hits /api/mcp
        ↓
backend checks Authorization token
        ↓
backend checks scopes/admin
        ↓
tool runs
```

## 12. `/oauth/authorize` and redirects do not wait

A redirect is not `await redirect()`.

It is just an HTTP response.

```txt
Request comes in
        ↓
server returns 302 Location: /login
        ↓
that request is done
        ↓
browser follows redirect by making a new request
```

Example:

```ts
export async function GET(req: Request) {
  const user = await getUserFromCookie(req);

  if (!user) {
    return redirect(`/login?next=${encodeURIComponent(req.url)}`);
  }

  const code = await createAuthCode(user.id);

  return redirect(`${redirectUri}?code=${code}&state=${state}`);
}
```

`await` is for DB/API work like checking user or creating code. The redirect ends the current request.

The flow continues because the browser makes another request later.

```txt
/oauth/authorize
  no user
  redirect to login

/login
  user logs in
  redirect back to /oauth/authorize

/oauth/authorize
  user exists now
  create code
  redirect to Claude
```

## 13. Browser redirects cannot send custom Authorization headers

This is a major issue in your setup.

A `fetch()` request can send an auth header:

```ts
await fetch("https://backend.com/api/admin/products", {
  headers: {
    Authorization: `Bearer ${session.access_token}`,
  },
});
```

But top-level browser navigation cannot:

```ts
window.location.href = "https://backend.com/oauth/authorize?...";
```

There is no way to attach:

```http
Authorization: Bearer <token>
```

to that navigation.

Browser navigations automatically send cookies, not custom headers.

So:

```txt
Authorization header:
  good for fetch/API requests

Cookies:
  good for browser navigation/redirect flows
```

This matters because `/oauth/authorize` is a browser navigation endpoint, not just a normal API endpoint.

## 14. Your split-domain problem

Your frontend and backend are on different domains.

Example:

```txt
frontend:
https://frontend.com

backend:
https://backend.com
```

If Supabase auth lives only in the frontend/browser client, then your backend `/oauth/authorize` cannot automatically see it.

This is why a direct redirect to `/oauth/authorize` is painful.

Your current dashboard pattern is:

```txt
frontend uses Supabase client
        ↓
frontend gets session.access_token
        ↓
frontend sends Authorization: Bearer <token> to backend
        ↓
backend verifies with Supabase getUser(token)
        ↓
backend checks admins table
```

This works well for `fetch` requests, but not for top-level browser redirects.

## 15. The backend session bridge

To make standard OAuth redirects work with a separate backend domain, use a session bridge.

Flow:

```txt
1. Claude opens:
   https://backend.com/oauth/authorize?client_id=...&redirect_uri=...&code_challenge=...

2. Backend sees no backend_session cookie.

3. Backend redirects to frontend login:
   https://frontend.com/login?next=<original oauth authorize url>

4. User logs in with Supabase on frontend.

5. Frontend gets session.access_token.

6. Frontend sends one bridge fetch:

   POST https://backend.com/auth/session
   Authorization: Bearer <supabase_access_token>
   credentials: include

7. Backend verifies Supabase JWT with getUser(token).

8. Backend checks admins table.

9. Backend sets backend_session cookie.

10. Frontend redirects browser to original next URL:
    https://backend.com/oauth/authorize?...

11. Browser automatically sends backend_session cookie to backend.

12. /oauth/authorize creates code.

13. Backend redirects browser to Claude callback.

14. Claude exchanges code at /oauth/token.

15. Claude gets MCP access token.

16. Claude sends MCP token to /api/mcp.
```

The first bridge request needs both:

```ts
await fetch("https://backend.com/auth/session", {
  method: "POST",
  credentials: "include",
  headers: {
    Authorization: `Bearer ${session.access_token}`,
    "Content-Type": "application/json",
  },
});
```

Why both?

```txt
Authorization header:
  proves the Supabase user to your backend

credentials: "include":
  allows browser to accept/store Set-Cookie from backend response
```

Backend response needs something like:

```http
Access-Control-Allow-Origin: https://frontend.com
Access-Control-Allow-Credentials: true
Set-Cookie: backend_session=abc; HttpOnly; Secure; SameSite=None; Path=/; Max-Age=600
```

Then later:

```ts
window.location.href = next;
```

The browser automatically sends:

```http
Cookie: backend_session=abc
```

to `backend.com`.

## 16. `credentials: "include"`

For cross-origin `fetch`, `credentials: "include"` does two things:

```txt
1. Sends existing cookies for that backend domain, if any.
2. Allows the browser to accept/store Set-Cookie from the backend response.
```

So on the bridge request:

```ts
await fetch("https://backend.com/auth/session", {
  method: "POST",
  credentials: "include",
  headers: {
    Authorization: `Bearer ${session.access_token}`,
  },
});
```

The auth header proves the user.

The `credentials: "include"` lets the backend set the session cookie.

After that, redirects do not need the auth header.

```txt
Auth header = one-time bridge from Supabase frontend auth to backend auth

Backend cookie = what makes browser redirects work afterward
```

## 17. Cross-domain cookies

People usually avoid truly unrelated cross-domain cookies.

They use one of these patterns.

### Same parent domain/subdomains

Example:

```txt
frontend:
https://admin.bikershades.com

backend:
https://api.bikershades.com
```

Set cookie with:

```http
Set-Cookie: session=abc;
  Domain=.bikershades.com;
  Path=/;
  HttpOnly;
  Secure;
  SameSite=Lax
```

Then both subdomains can receive the cookie.

For frontend `fetch` to backend:

```ts
await fetch("https://api.bikershades.com/api/admin/products", {
  method: "POST",
  credentials: "include",
});
```

Backend CORS:

```http
Access-Control-Allow-Origin: https://admin.bikershades.com
Access-Control-Allow-Credentials: true
```

### Same-origin proxy

Browser sees:

```txt
https://admin.bikershades.com/api/admin/products
```

Internally, the request proxies to:

```txt
https://backend-server.com/api/admin/products
```

To the browser, this is same-origin, so cookies and CORS are much easier.

### Truly different domains

Example:

```txt
frontend:
https://bikershades.com

backend:
https://my-backend.vercel.app
```

Then the backend cookie usually needs:

```http
Set-Cookie: session=abc; HttpOnly; Secure; SameSite=None; Path=/
```

Frontend fetch needs:

```ts
fetch("https://my-backend.vercel.app/...", {
  credentials: "include",
});
```

Backend CORS needs:

```http
Access-Control-Allow-Origin: https://bikershades.com
Access-Control-Allow-Credentials: true
```

But totally different domains can still be annoying because browser privacy features increasingly restrict third-party cookies.

So for a split frontend/backend setup, bearer tokens are often simpler for normal API calls.

## 18. Option A vs Option B for your OAuth flow

Two options were discussed.

### Option A: standard OAuth redirect flow with backend session cookie

This is cleaner and more standard.

```txt
Claude opens /oauth/authorize
        ↓
backend redirects to frontend login if needed
        ↓
frontend logs in with Supabase
        ↓
frontend does bridge fetch to /auth/session
        ↓
backend sets backend_session cookie
        ↓
frontend redirects back to /oauth/authorize
        ↓
backend reads cookie
        ↓
backend creates code
        ↓
backend redirects browser to Claude callback
```

This keeps `/oauth/authorize` as the route that performs the final redirect to Claude.

### Option B: backend returns Claude callback URL to frontend

This is practical but less standard.

```txt
Claude opens /oauth/authorize
        ↓
backend redirects to frontend login
        ↓
frontend logs in
        ↓
frontend POSTs Supabase token to backend /oauth/continue
        ↓
backend verifies admin
        ↓
backend creates code
        ↓
backend returns Claude callback URL
        ↓
frontend does window.location.href = callbackUrl
```

Example:

```ts
const res = await fetch("https://backend.com/oauth/continue", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${session.access_token}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    originalAuthorizeUrl: next,
  }),
});

const { redirectUrl } = await res.json();

window.location.href = redirectUrl;
```

This works because the browser still eventually lands on:

```txt
https://claude.ai/api/mcp/auth_callback?code=...&state=...
```

But it is less standard because the frontend JS sees the code-containing URL and participates in completing OAuth.

### Final decision

Option A is probably the move.

Even though it still needs one initial `fetch`, that fetch is only to create the backend session cookie.

After that, the OAuth flow behaves normally.

## 19. Supabase `getSession()` and `getUser()`

Frontend gets the Supabase access token with:

```ts
const {
  data: { session },
} = await supabase.auth.getSession();

const accessToken = session?.access_token;
```

Then sends it to backend:

```ts
await fetch("https://backend.com/api/admin/products", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: `Bearer ${accessToken}`,
  },
  body: JSON.stringify(data),
});
```

Backend should not trust the frontend saying it already verified the user.

Backend must verify the token itself:

```ts
const authHeader = req.headers.get("authorization");
const token = authHeader?.replace("Bearer ", "");

const { data, error } = await supabase.auth.getUser(token);

if (error || !data.user) {
  return new Response("Unauthorized", { status: 401 });
}

const isAdmin = await checkAdminsTable(data.user.id);

if (!isAdmin) {
  return new Response("Forbidden", { status: 403 });
}
```

Frontend `getUser()` can be used for UI confidence, but backend must still verify.

## 20. Verifying JWT vs using RLS

There are two different things:

```txt
1. Verifying the user
2. Querying Supabase with RLS applied
```

### `getUser(token)`

This verifies the JWT and returns a trusted user.

```ts
const { data: { user } } = await supabase.auth.getUser(token);
```

This does not automatically make future DB queries RLS-scoped.

If you then use a service-role client, RLS is bypassed.

Flow:

```txt
JWT proves identity
        ↓
backend checks admins table
        ↓
service role performs action
        ↓
protection is your backend admin check, not RLS
```

### Supabase client with `Authorization: Bearer token`

If you create a Supabase client with the user's token:

```ts
const userSupabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_ANON_KEY!,
  {
    global: {
      headers: {
        Authorization: `Bearer ${token}`,
      },
    },
  }
);
```

Then DB queries run as that user and RLS applies.

Flow:

```txt
JWT proves identity
        ↓
Supabase DB sees auth.uid()
        ↓
RLS decides allowed rows
```

### Service role client

A service role client bypasses RLS.

Use it only after your backend verifies the user and checks permissions.

Simple rule:

```txt
Need to verify identity?
  use supabase.auth.getUser(token)

Need to make RLS-protected queries as that user?
  create Supabase client with Authorization: Bearer token

Need backend/admin power?
  verify user first, check admins table, then use service role
```

For your admin endpoints, the current pattern is valid:

```txt
get token from Authorization header
        ↓
getUser(token)
        ↓
check admins table
        ↓
use service-role Supabase client
```

## 21. Cookies vs localStorage

Server/SSR cookie auth is good when backend routes need to know who the user is.

```txt
User logs in
        ↓
session stored in cookies
        ↓
browser requests /oauth/authorize
        ↓
cookies automatically sent
        ↓
server reads session
```

Browser localStorage auth is okay for pure client apps, but the server cannot naturally read localStorage.

```txt
browser has token in localStorage
        ↓
server route /oauth/authorize cannot see it
```

For normal fetch requests, localStorage/browser-session token can still work because the frontend can manually attach:

```http
Authorization: Bearer <token>
```

But for browser redirects, you need cookies or a handoff strategy.

## 22. Final recommended design for this setup

Use Option A with a backend session bridge.

### Routes

```txt
GET  /oauth/authorize
POST /oauth/token
POST /auth/session
POST /api/mcp
```

Potential frontend page:

```txt
/frontend-login?next=<original authorize url>
```

### Flow

```txt
1. Claude starts OAuth:
   GET https://backend.com/oauth/authorize?client_id=claude-web&redirect_uri=https://claude.ai/api/mcp/auth_callback&state=...&code_challenge=...&code_challenge_method=S256

2. Backend validates:
   client_id
   redirect_uri allowlist
   code_challenge
   code_challenge_method=S256

3. Backend sees no backend_session cookie.

4. Backend redirects to frontend login with next URL.

5. User logs in with Supabase on frontend.

6. Frontend gets Supabase session.access_token.

7. Frontend POSTs access token to backend /auth/session using:
   Authorization: Bearer <supabase_access_token>
   credentials: include

8. Backend verifies Supabase token with getUser(token).

9. Backend checks admins table.

10. Backend sets short-lived backend_session cookie.

11. Frontend redirects browser back to original /oauth/authorize URL.

12. Browser sends backend_session cookie.

13. /oauth/authorize recognizes logged-in admin.

14. /oauth/authorize creates authorization code:
    short-lived
    one-time-use
    stored hashed
    linked to user_id, client_id, redirect_uri, scopes, code_challenge

15. /oauth/authorize redirects browser to Claude callback:
    https://claude.ai/api/mcp/auth_callback?code=...&state=...

16. Claude exchanges code:
    POST https://backend.com/oauth/token
    code=...
    code_verifier=...

17. /oauth/token verifies:
    code exists
    code not expired
    code not used
    redirect_uri/client_id match
    PKCE verifier matches challenge
    user still allowed/admin if desired

18. Backend creates MCP access token:
    stored hashed
    scoped
    expires in about 1 hour

19. Claude stores MCP access token.

20. Claude calls tools:
    POST https://backend.com/api/mcp
    Authorization: Bearer <mcp_access_token>

21. /api/mcp verifies token, scopes, and actor.

22. MCP tool calls shared service function.
```

## 23. Practical security checklist

For `/oauth/authorize`:

```txt
Validate client_id.
Validate redirect_uri exactly against allowlist.
Require code_challenge.
Require code_challenge_method=S256.
Check backend session cookie.
Check admins table.
Create short-lived one-time code.
Store only code hash.
Preserve state.
Redirect to Claude callback with code and state.
```

For `/oauth/token`:

```txt
Accept code and code_verifier.
Hash incoming code and look it up.
Reject expired/used code.
Validate PKCE.
Mark code as used.
Create MCP access token.
Store only token hash.
Return raw token once.
```

For `/auth/session`:

```txt
Accept Supabase access token in Authorization header.
Verify with Supabase getUser(token).
Check admins table.
Set httpOnly Secure session cookie.
Use CORS with exact frontend origin.
Use Access-Control-Allow-Credentials: true.
Require frontend fetch credentials: include.
```

For `/api/mcp`:

```txt
Require Authorization: Bearer <mcp_access_token>.
Hash incoming token.
Find valid unexpired non-revoked token.
Load actor/user/scopes.
Check tool-level permissions.
Call shared service function.
Audit dangerous actions.
```

## 24. Mental model summary

```txt
Supabase access token:
  frontend proves login to backend

/auth/session:
  turns Supabase login into backend session cookie

backend_session cookie:
  lets browser redirects authenticate to backend

/oauth/authorize:
  turns logged-in backend session into temporary code

authorization code:
  safe-ish browser redirect value, short-lived and one-time-use

/oauth/token:
  turns code + PKCE verifier into MCP token

MCP access token:
  what Claude sends to call tools

/api/mcp:
  verifies MCP token and executes authorized tools
```

## 25. The big lesson

This feels painful because browser redirects and API requests carry credentials differently.

```txt
fetch:
  can send Authorization headers
  can include credentials
  does not naturally move the top-level browser window

browser redirect:
  moves the top-level browser window
  cannot send custom Authorization headers
  can send cookies automatically
```

So the clean solution is:

```txt
Use fetch once to create a backend cookie.
Then use browser redirects for OAuth.
Then use MCP bearer tokens for tool calls.
```
