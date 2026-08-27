# Claude Code Safety, VS Code Debugging, and JWT Token Security Notes (June 25, 2026)

## 1. Can Claude Code delete your codebase?

Yes. Claude Code can potentially delete or damage your **local codebase** if it has terminal/file access and destructive commands are approved or allowed.

Examples of dangerous commands:

```bash
rm -rf .
rm -rf src
git clean -fd
git clean -fdx
git reset --hard
find . -delete
```

Claude Code is essentially an AI agent working inside your local project environment. If it can run shell commands, it can do whatever those commands are allowed to do.

---

## 2. Local repo vs remote GitHub repo

There are two different things:

```txt
local repo:
  the project folder on your laptop

remote repo:
  the GitHub copy in the cloud
```

Claude Code can mess up the **local repo** if it has file/terminal access.

It cannot delete the **remote GitHub repo itself** unless it has authenticated GitHub access and permission to do so.

---

## 3. What Claude Code could do locally

Claude could potentially:

```txt
delete files
rewrite local history
make a terrible commit
reset branches locally
remove local refs/pointers
delete untracked files
delete ignored files
```

Example local disaster:

```bash
rm -rf src
git add .
git commit -m "bad cleanup"
```

That would create a commit deleting source code locally.

---

## 4. Could Claude push bad changes to GitHub?

Only if it has access to your GitHub auth/session and runs `git push`.

If it can push, it could push a bad commit:

```bash
git push
```

That would not usually delete the GitHub repository itself. It would just make the remote branch include a bad commit.

Example:

```txt
good commit
↓
good commit
↓
bad Claude commit deleting src
```

The repo still exists. The history still exists. You can recover.

---

## 5. Can Claude delete the actual GitHub repo?

Not normally.

To delete the actual cloud GitHub repo, it would need GitHub-level authenticated permissions and a destructive command/API call.

Examples:

```bash
gh repo delete owner/repo
```

or a GitHub API delete request with a valid token.

Without your GitHub auth/token/session, Claude cannot delete the remote GitHub repo.

### Mental model

```txt
Local repo:
  Claude can wreck it if given terminal/file access.

Remote GitHub repo:
  Claude cannot delete it unless it has authenticated GitHub permissions.

Bad pushed commit:
  recoverable from history.

Actual remote repo deletion:
  requires GitHub-level auth and delete action.
```

---

## 6. Why committing often helps

If you commit often, your project is much safer.

Before using Claude Code for big changes:

```bash
git status
git add .
git commit -m "checkpoint before Claude"
git push
git checkout -b claude-work
```

Then Claude can work on a separate branch.

If it messes things up locally, you can recover from Git.

---

## 7. Recovering from a bad local change

If Claude changed files but did not commit:

```bash
git status
git diff
```

To discard changes:

```bash
git restore .
```

or:

```bash
git reset --hard HEAD
```

Be careful: this removes uncommitted changes.

---

## 8. Recovering from a bad local commit

If Claude made a bad commit locally:

```bash
git log --oneline
```

Find the good commit before the bad one.

Then:

```bash
git reset --hard <good_commit_hash>
```

If the bad commit was only local, that is enough.

---

## 9. Recovering from a bad pushed commit

If Claude pushed a bad commit to GitHub, the repo still usually exists.

Safer recovery:

```bash
git revert <bad_commit_hash>
git push
```

This creates a new commit that undoes the bad commit.

More forceful recovery:

```bash
git reset --hard <good_commit_hash>
git push --force-with-lease
```

Use `--force-with-lease` instead of plain `--force`.

---

## 10. Commands to be careful approving

Be very careful with:

```bash
rm -rf
git clean -fd
git clean -fdx
git reset --hard
git push --force
git push --force-with-lease
gh repo delete
```

Especially dangerous:

```bash
git clean -fdx
```

because it deletes ignored files too, such as build outputs, generated files, and sometimes local-only files.

Also protect:

```txt
.env
.env.local
private keys
credentials
uncommitted files
untracked files
```

---

# VS Code Debug Mode

## 11. What is debug mode?

Debug mode in VS Code lets you run code while being able to pause it, inspect it, and step through it line by line.

Normal run:

```txt
start → run everything → output/error
```

Debug mode:

```txt
start → pause at breakpoint → inspect variables → step line by line
```

Debug mode is basically a more powerful version of `console.log`.

---

## 12. What debug mode gives you

### Breakpoints

A breakpoint is a red dot next to a line number. VS Code pauses when execution reaches that line.

Example:

```ts
const brandSlug = req.nextUrl.searchParams.get("brandSlug");
```

### Step Over

Run the current line and move to the next line.

### Step Into

Go inside the function being called.

Example:

```ts
createAdminClient()
```

Step into lets you enter that function.

### Step Out

Finish the current function and return to the caller.

### Variables panel

Shows current values:

```txt
brandSlug = "bikershades"
slug = "foam-padded"
product = null
error = {...}
```

### Call stack

Shows how the code reached the current line.

Example:

```txt
GET route
apiFetch
ProductPage
```

### Debug console

Lets you evaluate expressions while paused.

Example:

```ts
product?.variations?.length
```

---

## 13. How to enter debug mode in VS Code

Open the project in VS Code.

Then:

```txt
Mac:
  Cmd + Shift + D

Windows:
  Ctrl + Shift + D
```

Or click the left sidebar icon that looks like:

```txt
play button + bug
```

Then press:

```txt
F5
```

or click the green play button.

---

## 14. Debugging a Next.js app

For a Next.js app, create:

```txt
.vscode/launch.json
```

Basic server-side debug config:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Next.js: debug server-side",
      "type": "node-terminal",
      "request": "launch",
      "command": "npm run dev"
    }
  ]
}
```

Then:

```txt
1. Open Run and Debug.
2. Select "Next.js: debug server-side".
3. Press the green play button.
4. Add a breakpoint in a server/API route.
5. Visit the route in the browser/Postman.
```

Example breakpoint location:

```ts
export async function GET(req: NextRequest) {
  const brandSlug = req.nextUrl.searchParams.get("brandSlug");
  const slug = req.nextUrl.searchParams.get("slug");
}
```

Put the breakpoint on a real executable line like:

```ts
const brandSlug = req.nextUrl.searchParams.get("brandSlug");
```

---

## 15. Server-side vs browser-side debugging

Next.js API routes run on the server.

So if you are debugging this:

```txt
app/api/.../route.ts
```

you need server-side Node debugging.

Browser/Chrome debugging is for client-side React code.

### Rule

```txt
API route / server component:
  server-side debugger

Client component:
  browser debugger
```

---

## 16. What does “unbound breakpoint” mean?

An unbound breakpoint means VS Code sees your breakpoint, but it has not connected that breakpoint to the code actually running.

Mental model:

```txt
VS Code:
  "I see where you want to pause."

Runtime:
  "I do not recognize that exact file/line as running code yet."
```

This commonly happens in Next.js because TypeScript files are compiled/bundled before Node runs them.

---

## 17. Common causes of unbound breakpoints

### App was not started through VS Code debug

If you just run:

```bash
npm run dev
```

in a normal terminal, VS Code may not attach to server code.

Start the app using the VS Code debug config.

### The route has not run yet

API route breakpoints may not bind/pause until the route is actually triggered.

Example:

```txt
http://localhost:3000/api/public/product-detail?brandSlug=bikershades&slug=some-product
```

### Breakpoint is on a non-runtime line

Do not put breakpoints on:

```ts
type RawAttr = ...
```

or imports, blank lines, or inside a SQL/select string.

Use real executable code:

```ts
const supabase = createAdminClient();
```

### Next cache/dev server weirdness

Sometimes restart the dev server or delete `.next`:

```bash
rm -rf .next
```

Then start debugging again.

---

## 18. Better Next.js debug config with inspector

On Mac/Linux:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Next.js: debug server-side",
      "type": "node-terminal",
      "request": "launch",
      "command": "NODE_OPTIONS='--inspect' npm run dev"
    }
  ]
}
```

On Windows:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Next.js: debug server-side",
      "type": "node-terminal",
      "request": "launch",
      "command": "set NODE_OPTIONS=--inspect && npm run dev"
    }
  ]
}
```

---

# JWT / Auth Token Security

## 19. What happens after login?

After login, auth systems usually give you:

```txt
access token:
  short-lived token used to prove the user is logged in

refresh token:
  longer-lived token used to get new access tokens
```

The access token is often a JWT.

A JWT may be valid for a short time, commonly around one hour depending on auth settings.

---

## 20. How the access token is used

The frontend may send the access token to the backend:

```http
Authorization: Bearer <access_token>
```

The backend verifies:

```txt
Is the token valid?
Is it expired?
Who is the user?
What permissions does the user have?
```

If valid, the backend accepts the request.

---

## 21. Can people steal a JWT from the browser?

Yes.

If the token is accessible to frontend JavaScript, then malicious JavaScript can potentially steal it.

This usually requires an XSS attack.

Example flow:

```txt
attacker gets malicious JS onto your site
↓
JS reads localStorage/sessionStorage/token
↓
JS sends token to attacker
↓
attacker uses token until it expires
```

---

## 22. Token storage options

### localStorage/sessionStorage

Easy to use.

Risk:

```txt
JavaScript can read it.
Malicious injected JavaScript can also read it.
```

### JS-readable cookies

Also readable by JavaScript unless `HttpOnly`.

Risk:

```txt
document.cookie can expose it.
```

### HttpOnly cookies

A cookie marked `HttpOnly` cannot be read by JavaScript.

That means this will not expose the token:

```js
document.cookie
```

The browser can still automatically send the cookie with requests.

This helps protect against direct token theft through XSS.

---

## 23. Cookies have CSRF concerns

HttpOnly cookies are safer against token theft through JavaScript, but they introduce CSRF concerns.

With cookie-based auth, you usually need protections like:

```txt
SameSite=Lax or SameSite=Strict
CSRF tokens for dangerous actions
Origin/Referer checks
```

So the tradeoff is:

```txt
JWT in localStorage:
  easier
  vulnerable if XSS happens

JWT in HttpOnly cookie:
  harder to steal with JS
  needs CSRF protection
```

---

## 24. Why access tokens are short-lived

If a JWT access token is stolen, the attacker can use it until it expires.

Short expiration limits the damage.

Example:

```txt
token valid for 1 hour
↓
attacker steals it
↓
attacker can use it for at most about 1 hour
```

The refresh token is more sensitive because it can create new access tokens.

Protect refresh tokens more carefully.

---

## 25. Your setup mental model

If your frontend does:

```ts
const { data } = await supabase.auth.getSession();

await fetch("/api/admin/thing", {
  headers: {
    Authorization: `Bearer ${data.session.access_token}`
  }
});
```

then the access token is available to frontend JavaScript.

That is a normal pattern, but you must take XSS seriously.

Important precautions:

```txt
avoid dangerouslySetInnerHTML
sanitize user-generated HTML
do not inject untrusted scripts
keep dependencies updated
do not expose service-role keys to the browser
keep secrets server-only
use short-lived access tokens
protect refresh tokens
```

---

## 26. Supabase admin/service role warning

Never expose the Supabase service role key to the browser.

This is okay on the server:

```ts
createAdminClient()
```

inside a server route.

This is not okay in client-side React code:

```txt
NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY
```

Anything prefixed with `NEXT_PUBLIC_` can be exposed to the browser.

### Rule

```txt
Anon key:
  okay in browser with RLS

Service role key:
  server only, never browser
```

---

## 27. Clean security mental model

```txt
Access token:
  short-lived proof of login

Refresh token:
  longer-lived and more sensitive

If token is stolen:
  attacker can use it until it expires

localStorage:
  easy but readable by JS

HttpOnly cookie:
  not readable by JS, but needs CSRF protection

XSS:
  main browser-token theft risk

CSRF:
  main cookie-auth risk

Server-only secrets:
  never expose to frontend
```

---

# Final Summary

## Claude Code

```txt
Can destroy local files:
  yes

Can push bad commits:
  yes, if it has git/GitHub auth and you approve it

Can delete remote GitHub repo:
  not unless it has GitHub delete permissions/auth

Best protection:
  commit often, push, use branches, review diffs
```

## VS Code Debugging

```txt
Debug mode:
  run code with breakpoints and variable inspection

Use it for:
  seeing actual runtime values

Unbound breakpoint:
  VS Code has not matched your breakpoint to running code yet

Fix:
  start through VS Code debug, trigger the route, use executable lines
```

## JWT Security

```txt
JWT access token:
  usually short-lived

Can be stolen:
  yes, if exposed to malicious JS/XSS

HttpOnly cookies:
  protect against JS reading tokens

Cookies:
  require CSRF protections

Main rule:
  protect tokens, avoid XSS, keep secrets server-only
```
