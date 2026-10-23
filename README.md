# Task Manager · Miraway

Task Manager runs on Sites with persistent D1 records and R2 avatars. Existing tasks, tickets, comments, activities, organization records and user IDs are preserved.

## Account access

The application uses local username/password authentication. The published origin can be reached without a ChatGPT account; application pages redirect unauthenticated visitors to `/login`, and APIs reject missing or invalid sessions. Native Sites source ownership and deployment permissions remain separate from application roles.

The existing `admin` record is initialized once as the super administrator from the secret runtime variable `BOOTSTRAP_ADMIN_HASH`. This value must contain a salted scrypt hash, never a plaintext password. The bootstrap marker prevents later logins from resetting a changed password. Old member records without a password cannot log in until Admin sets one.

Passwords use scrypt (N=16384, r=8, p=5) with a random salt. Sessions use random 256-bit tokens, store only token digests and expire after 12 hours. Session cookies are Secure, HttpOnly and SameSite=None. Legacy first-party cookies remain valid until expiry. Login attempts are limited by account and source IP. Login and mutation bodies require JSON.

## Cross-origin access

Every valid HTTP(S) origin is permitted to authenticate and use the API. Credentialed CORS reflects the caller's exact Origin and sends `Access-Control-Allow-Credentials: true`, `Vary: Origin`, supported methods and explicit requested header names. OPTIONS preflights do not require a session; success and error responses both carry CORS headers. Opaque `null` origins are not domain origins.

`https://hoptacxa.pages.dev` and `https://taskmanager-1610.exactlamp6.chatgpt.site` are the two owner-approved browser entry points. A session established by either may authenticate requests from the other when the browser/proxy carries its cookie. These entry points use a separate non-partitioned `__Secure-taskmanager_trusted_session` cookie so a top-level site change does not select a different cookie partition. The server sets no Domain attribute; a proxy may scope this Secure-prefixed cookie to its own host. Origin matching is exact; lookalike domains and subdomains do not inherit this trust.

Other domains retain origin-bound, partitioned v2 sessions and must log in independently. Cookie clients send `credentials: 'include'`. CORS does not register DNS names or create custom Site domains. A reverse proxy must forward all request methods, cookies, query strings and the Origin header unchanged, along with every Set-Cookie response header. Do not cache authentication responses or authenticated pages. After login the UI verifies `/api/auth/me` before navigation and shows a clear error if the session cookie was not retained, instead of repeatedly redirecting between login and dashboard.

Browsers may still block third-party cookies. Integration clients can POST JSON `{username,password,session_mode:'token'}` to `/api/auth/login` to obtain `access_token`, `token_type` and `expires_in`, then send `Authorization: Bearer <access_token>` on API requests. Token mode sets no cookie, uses the same RBAC/expiry/revocation rules and is also bound to its login origin. Keep tokens in application memory; never place them in URLs or logs. This mode is for API clients; normal Site pages use cookie login. Logout revokes only the authenticated session.

Admin can create lower-role accounts, edit profiles, reset passwords, require a password change, lock/unlock accounts and archive them while retaining history. Password changes and resets, account role changes, and account locking revoke sessions. Every authenticated request reads the current account and role state. The super administrator cannot be demoted, archived, locked or reset through other-account management endpoints.

## RBAC

| Role | Scope |
| --- | --- |
| Admin | All records, account and role management, organization, backups and protected maintenance. |
| Supervisor | All tasks/tickets, organization and announcements; no account administration or full JSON backup. |
| Leader | Assigned work and work of managed teams; team support routing, assignment and announcements. |
| User | Assigned/supported or authored work; task status, checklist and comments; own support requests. |
| Viewer | Read-only access to assigned/related business records; may change own password/avatar and read notifications. |

Legacy Management/Project Manager roles map to Supervisor; other legacy/custom roles use User permissions. A legacy Admin role does not grant super-administrator access unless the account has the protected super-admin flag. New accounts cannot be given the Admin role through the application.

## Features

Tasks, subtasks, progress, supporters, comments, activity history, support tickets, keyword routing, departments, teams, announcements, notifications, scoped CSV reports, Admin-only JSON backup and optional Telegram settings are retained. Backups exclude password hashes, session material, ChatGPT identity IDs and Telegram IDs.

Browser polling refreshes data while the page is visible. D1 is the system of record; browser localStorage is not used for authentication or business records.

## Business workspaces

Each workspace has an explicit authenticated route, its own heading and loading/error handling. Sidebar tabs and overview business links use full document navigation so their authenticated layouts load even through proxies that do not handle RSC transitions. A shared workspace catalog keeps tab names, descriptions and overview shortcuts aligned. Overview metrics open the matching task status/deadline filter; list and board views also support direct links. The overview clock is populated after hydration to avoid server/browser locale mismatches, and business dates use Vietnam time. The server supplies only an allowlisted public account profile; API authorization continues to enforce record scope and write permissions.

| Route | Responsibility |
| --- | --- |
| `/dashboard` | Summary, progress and quick links to business workspaces. |
| `/tasks` | Create, assign and follow authorized tasks; search, priority/status/assignee filters, deadline queues, list and status-board views, details, checklists, comments and history. |
| `/my-tasks` | Assigned and supported tasks; personal deadline queues and progress updates. |
| `/support` | Request intake, ticket assignment, status and collaboration. |
| `/notifications` | Company/team announcements and personal in-app notifications (`?view=personal`), read status and task links. |
| `/reports` | Scoped workload, completion and overdue metrics, support summary, CSV export; Admin-only JSON backup. |
| `/support/settings` | Departments, support teams, members and ticket routing rules. |
| `/settings` | Account/role administration where authorized and personal security settings. |

Vietnam calendar dates are used for today's task queue. Completed tasks are excluded from overdue queues. A failed optional Telegram settings request does not prevent account administration from loading.

## Reminders

`POST /api/system/reminders` writes idempotent in-app reminders and a `checked_at` timestamp. `GET` reads the last check. These endpoints require a fully signed-in super administrator or an independently configured `x-maintenance-token` matching the secret `MAINTENANCE_TOKEN_SHA256`. The maintenance secret is not currently configured. A Sites service bearer by itself is insufficient for this public-origin application.

The previous owner-private hourly schedule must remain paused until an unattended caller has an approved, retrievable credential for the new application boundary. Never put admin passwords or service tokens in schedule prompts, code, logs or files. Notification reads still check deadlines while the application is used.

## Validation

`node tests/api.test.mjs` exercises the API with an isolated SQLite database, applies all migrations and checks authentication, five-role boundaries, record scope, password changes, session revocation/expiry, rate limits, reminder idempotency and existing business flows. TypeScript and the production build are checked separately.

`node tests/workspaces.test.mjs` checks task filters, ordering, assigned/supporting queues and Vietnam date boundaries. After a production build, `node tests/routes.test.mjs` checks all eight compiled Worker routes and navigation payloads using an isolated in-memory D1 database and synthetic account. It also checks authentication redirects and verifies that password hashes and session tokens are not serialized into HTML.

Browser QA clicked all eight workspace tabs, task list/board and personal filters, announcement/personal feeds, routing rules and account role settings using an isolated preview with synthetic data. Production cookie handling through the external hoptacxa Pages proxy remains unverified; its missing-cookie issue is separate from workspace navigation.
