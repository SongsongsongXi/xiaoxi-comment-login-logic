---
name: xiaoxi-comment-login-logic
description: Design, implement, review, or troubleshoot a passwordless comment-login system built around friend-link identity, ordinary email verification, server-issued trusted-device tokens, email security alerts, device management, and explicit user-controlled login popups. Use this skill when replacing third-party comment services, designing comment authentication flows, preventing cache-clearing bypasses, adding trusted-device controls, or integrating comment identity with a website's link directory.
---

# 小曦的园子评论登录逻辑

Use this skill to design a low-friction, revocable identity system for a website comment area. Keep the comment system independent from third-party platforms. Treat friend links, email verification, browser sessions, trusted devices, security emails, and comment moderation as one coordinated system.

## Core design goals

- Avoid passwords and account registration for ordinary commenters.
- Let a friend-link owner appear as their website identity: cached logo, site name, and site URL.
- Let ordinary visitors authenticate with a short-lived email code.
- Make normal friend-link login quiet and fast after the first trusted login.
- Keep every trust decision revocable from email, a device list, an admin reset, or server-side token invalidation.
- Never treat a browser fingerprint, IP address, user agent, canvas output, or random client UUID as an authentication credential.
- Never open the login modal merely because the page loaded, a token expired, or a user logged out.

## Identity model

Maintain these conceptual records, regardless of the chosen database or framework:

1. **Comment access session** — the current browser credential used for reading, posting, replying, and liking comments.
2. **Trusted device** — a server-issued cryptographically random token bound to an email and friend link. Store only its hash on the server. Keep creation time, last-used time, device label, IP, user agent, and revocation time.
3. **Friend-link identity** — website name, URL, logo, and owner email. Resolve the owner email case-insensitively and normalize it before every lookup.
4. **Email login verification** — one-time code records for users without a matching friend link, with expiry, attempt limits, and send cooldowns.
5. **Security action record** — a single-use deny token for login alerts or comment alerts. Store only its hash and make it expire.

Do not use the friend-link viewing URL, an email address, or a browser fingerprint as a substitute for a revocable device token. Keep comment history separate from login state so logout or device removal does not accidentally erase legitimate history.

## Login decision flow

### 1. Restore an existing session

On page load:

- Read the current comment session token.
- If it is valid, restore the user without showing a modal.
- If it is invalid, clear it silently.
- Read the saved trusted-device token only to attempt silent restoration; do not expose raw token details in the UI.
- If silent restoration fails, keep the page logged out and do not open a modal or show an internal token error.

Record an explicit client-side manual-logout marker when the user clicks logout. While that marker exists, do not silently restore the trusted device on refresh. Clear the marker only after the user actively logs in again.

### 2. Handle an explicit login action

Open the login modal only from an intentional user action, such as clicking “点击登录”. Keep overlay clicks and unrelated page loads from closing or opening the modal.

After the user enters an email, debounce an identity lookup:

- If the email matches a friend-link owner, show the site logo and name and offer “以该网站身份登录评论区”.
- If it does not match, show the ordinary email-code route and explain that the address is not in the friend-link directory.

### 3. Friend-link login

Use the following server-side decision tree:

- If a valid trusted-device token is supplied, update its last-used metadata and log in directly.
- If a token is supplied but is revoked, reject the attempt and let the user start email authorization.
- If no token is supplied and the email has never had a trusted-device record, allow the first login without authorization and create a new trusted-device token.
- If no token is supplied but the email has any historical trusted-device record, require email authorization. This rule prevents changing browsers or clearing site storage from becoming an authorization bypass.
- After authorization succeeds, create or refresh the current session and create a new trusted-device record when appropriate.

The existence of historical records matters: a revoked or removed device must not make the email look like a first-time visitor again. Keep revoked records instead of deleting them when historical trust must be enforced.

### 4. Ordinary email login

For an email without a friend link:

- Send a short-lived numeric code.
- Enforce a resend cooldown, expiry, attempt limit, and reasonable per-email and per-IP rate limits.
- Consume the code after successful use.
- Create an ordinary email comment identity with privacy-preserving display rules.

Do not route a friend-link email through the ordinary code flow unless that is an intentional fallback policy.

## Email security design

Send a friend-link login security email after every successful friend-link login, including the first direct login and later trusted-device logins. Include the site name, friend-link identity, login time, approximate device label, and observed IP.

Explain the two possible actions plainly:

- If the recipient performed the login, they may ignore the message.
- If they did not perform it, they can click “不是我” to revoke the specific device token and invalidate the related comment session.

For every new comment, optionally send a separate comment identity email. If the recipient rejects it, delete the targeted comment and revoke the email’s relevant comment credentials according to the site’s policy. Do not require a confirmation click for every normal comment when the user has already authenticated; use the email as a low-friction recovery and abuse-reversal mechanism.

Make security links single-use, opaque, signed or random, stored as hashes, time-limited, and safe to revisit. Return a simple confirmation page that does not reveal sensitive data.

## Device management

Provide a “current login devices” view through the user’s protected comment-access page. Show:

- Device label derived from the user agent for human readability.
- Latest login time.
- Latest observed IP.
- A clear remove action.

On removal, mark the trusted-device token revoked and invalidate the associated current comment session. Do not delete historical device rows if the system uses their existence to force authorization on future browsers. Restrict device-list and removal operations to the authenticated friend-link owner represented by the access token, not to the admin permission middleware.

## Logout behavior

Separate “logout this browser session” from “revoke this device”.

Manual logout should:

- Remove the current comment session from browser storage.
- Set a local manual-logout marker to prevent immediate silent re-login after refresh.
- Preserve the trusted-device token.
- Avoid sending the user through email authorization next time they intentionally click login.

Device removal, a security-email “不是我” action, friend-link reset, or an administrative security action should revoke the trusted-device token. These are stronger actions than ordinary logout.

## Privacy and security boundaries

- Use normalized, lowercased emails for lookup but never display the full email in public comments when a shorter name is sufficient.
- Use server-generated cryptographically random tokens; hash them before persistence.
- Do not use `Math.random()`, a 32-bit string hash, canvas fingerprints, fonts, WebGL renderer data, or screen dimensions as proof of identity.
- Treat IP and user-agent data as audit metadata, not authentication factors.
- Apply rate limits to code sending, login attempts, comment creation, and security actions.
- Avoid leaking whether an arbitrary email belongs to a friend link unless the product explicitly accepts that disclosure.
- Ensure token invalidation is checked on every protected comment operation, not only during page load.

## UI behavior requirements

- Match the site’s existing modal design instead of inventing a separate visual system.
- Keep the modal closed until an explicit login action.
- Do not close the modal from accidental overlay clicks when the flow is in progress.
- During login or comment submission, blur only the active operation area and show a clear spinner.
- Support light and dark themes, mobile keyboard behavior, focus stability, and accessible labels.
- After successful login, show the resolved avatar and display name briefly before closing when that matches the surrounding UI.
- After a silent-login failure, remain logged out without an automatic popup or internal error text.

## Implementation workflow

1. Inspect the existing comment API, storage keys, database tables, modal component, theme selectors, and permission middleware.
2. Draw the state machine before editing code: anonymous, first friend login, trusted restore, revoked-token authorization, ordinary email login, manual logout, and security revocation.
3. Add or migrate persistence tables without deleting historical trust records.
4. Implement backend token checks and revocation first.
5. Implement the frontend restore logic and explicit-login boundary second.
6. Add email templates and single-use security actions.
7. Add device listing and removal through public authenticated comment routes, not admin-only routes.
8. Test these cases explicitly:
   - First friend-link login.
   - Refresh with a valid trusted token.
   - Manual logout followed by refresh.
   - Manual logout followed by explicit login.
   - Clear storage or use a new browser after a previous login.
   - Removed device token.
   - Email “不是我” action.
   - Ordinary email code cooldown and expiry.
   - Dark-mode and mobile modal input.
9. Run backend syntax checks, frontend build checks, and a protected-route smoke test before packaging.

When adapting this workflow to another stack, preserve the behavioral guarantees rather than copying endpoint names or framework-specific implementation details.
