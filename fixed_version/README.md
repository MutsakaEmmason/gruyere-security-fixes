# Gruyere Security Fixes
### CSC3207 – Computer Security | Module 4 Assignment
**Student:** Mutsaka Emmanuel (EMMASON)  
**Course:** CSC3207 – Computer Security  
**Application:** [Google Gruyere](https://google-gruyere.appspot.com/) – A Deliberately Vulnerable Web Application

---

## Overview

This repository contains the **patched source code** for Google Gruyere after a hands-on penetration testing exercise. Four critical vulnerability classes were identified, exploited, documented, and fixed.

| # | Vulnerability | Severity | Files Changed |
|---|---------------|----------|---------------|
| 1 | Cross-Site Scripting (XSS) | 🔴 HIGH | `gruyere.py`, `sanitize.py`, `resources/newsnippet.gtl` |
| 2 | Denial of Service (DoS) | 🔴 CRITICAL | `gruyere.py` |
| 3 | SQL Injection / Credential Exposure | 🔴 HIGH | `gruyere.py`, `resources/login.gtl` |
| 4 | Buffer Overflow / Input Length | 🟡 MEDIUM | `gruyere.py`, `resources/newsnippet.gtl`, `resources/base.css` |

---

## How to Run

### Requirements
- Python 3.x

### Start the server
```bash
git clone https://github.com/EMMASON/gruyere-security-fixes.git
cd gruyere-security-fixes
python gruyere.py
```

Open your browser at the URL printed in the terminal (e.g. `http://127.0.0.1:8008/<unique_id>/`).

---

## Vulnerability 1 – Cross-Site Scripting (XSS)

### What was vulnerable
The New Snippet form accepted arbitrary HTML and stored it without sanitisation. The snippet template rendered stored content as raw HTML using `{{_this:html}}`. The browser's XSS protection was also explicitly **disabled** in the response header (`X-XSS-Protection: 0`).

### Exploit used
```html
<img src="x" onerror="alert('SECURITY_AUDIT:__XSS_VULNERABILITY_CONFIRMED__BY_MUTSAKA');">
```
This payload was submitted as a snippet. Every user who visited the My Snippets page triggered the JavaScript alert — a **Stored XSS** attack.

### Root cause
- `_DoNewsnippet2` stored snippets with no output encoding
- `sanitize.py` blacklist was missing `onerror` (and many other event handlers)
- `_SendHtmlResponse` set `X-XSS-Protection: 0`, actively disabling browser protection
- No Content Security Policy header

### Fix applied
**`gruyere.py` – `_DoNewsnippet2`:**
```python
# BEFORE
snippets.insert(0, snippet)

# AFTER — html.escape() encodes <, >, ", ' and & as safe HTML entities
snippet = html.escape(snippet)
snippets.insert(0, snippet)
```

**`gruyere.py` – `_SendHtmlResponse`:**
```python
# BEFORE
self.send_header('X-XSS-Protection', '0')

# AFTER
self.send_header('X-XSS-Protection', '1; mode=block')
self.send_header('Content-Security-Policy',
    "default-src 'self'; script-src 'self'; object-src 'none';")
```

**`sanitize.py` – `_SanitizeTag`:** Added `onerror`, `onmouseover`, `onmouseenter`, `onpaste`, `oninput`, and 10 other missing event handlers to the disallowed list.

---

## Vulnerability 2 – Denial of Service (DoS)

### What was vulnerable
The `/quitserver` endpoint — which immediately shuts down the Python server process — was **not listed in `_PROTECTED_URLS`**. This meant any user (even unauthenticated) could call it and take the entire application offline.

### Exploit used
Navigating to:
```
http://127.0.0.1:8008/<instance_id>/quitserver
```
Immediately terminated the server. Subsequent requests returned `ERR_CONNECTION_REFUSED`.

### Root cause
`_PROTECTED_URLS` only listed `/quit` and `/reset`, but the actual shutdown handler was mapped to `/quitserver`, which had no access control.

### Fix applied
**`gruyere.py` – `_PROTECTED_URLS`:**
```python
# BEFORE
_PROTECTED_URLS = ['/quit', '/reset']

# AFTER — added /quitserver to require admin role
_PROTECTED_URLS = ['/quit', '/quitserver', '/reset']
```

**`gruyere.py` – `_DoQuitserver`:** Added a secondary admin check inside the handler as defence-in-depth:
```python
def _DoQuitserver(self, cookie, specials, params):
    if not cookie.get(COOKIE_ADMIN):
        self._SendError('Forbidden: administrator access required.',
                        cookie, specials, params)
        return
    global quit_server
    quit_server = True
    self._SendTextResponse('Server quit.', None)
```

---

## Vulnerability 3 – SQL Injection / Credential Exposure

### What was vulnerable
The login form used `method='get'`, sending the username and password as plaintext URL parameters:
```
/login?uid=Emmason&pw=emason
```
This exposed credentials in:
- The browser address bar (visible to anyone looking at the screen)
- Browser history
- Server access logs
- HTTP Referer headers sent to third parties

Additionally, the `uid` field accepted any characters including SQL metacharacters (`'`, `--`, `;`), enabling injection attempts.

### Exploit used
Inspecting the URL bar after login revealed credentials in plaintext. Injecting `'` or `' OR '1'='1` into the uid field was attempted.

### Root cause
- `login.gtl` used `method='get'` on the login form
- `_DoLogin` performed no input validation on the uid field
- Password comparison used plain `==` with no timing-attack protection

### Fix applied
**`resources/login.gtl`:**
```html
<!-- BEFORE -->
<form method='get' action='/{{_unique_id}}/login'>

<!-- AFTER — POST keeps credentials out of the URL -->
<form method='post' action='/{{_unique_id}}/login'>
```

**`gruyere.py` – `_DoLogin`:**
```python
import re, hmac as _hmac

UID_PATTERN = re.compile(r'^[a-zA-Z0-9_\-]{1,32}$')

# Validate uid format — blocks SQL metacharacters
if not uid or not UID_PATTERN.match(uid):
    message = 'Invalid user name or password.'
    
# Constant-time comparison prevents timing-based attacks
if _hmac.compare_digest(str(stored_pw), str(pw)):
    ...
```

---

## Vulnerability 4 – Buffer Overflow / Input Length

### What was vulnerable
The New Snippet form had no maximum length limit. Submitting thousands of characters caused the string to overflow the page layout, breaking the UI for every user. In a production environment, unbounded inputs can also exhaust server memory and degrade database performance.

### Exploit used
A snippet of thousands of repeated `A` characters was submitted. The page rendered it as a single unbroken horizontal line that overflowed all content boundaries.

### Root cause
- `_DoNewsnippet2` stored snippets with no length check
- The HTML textarea had no `maxlength` attribute
- No CSS overflow rules on snippet display containers

### Fix applied
**`gruyere.py` – `_DoNewsnippet2`:**
```python
MAX_SNIPPET_LENGTH = 500  # characters

if len(snippet) > self.MAX_SNIPPET_LENGTH:
    self._SendError(
        'Snippet is too long (max %d characters).' % self.MAX_SNIPPET_LENGTH,
        cookie, specials, params)
    return
```

**`resources/newsnippet.gtl`:**
```html
<textarea name='snippet' rows='5' style='width:100%' maxlength='500'></textarea>
```

**`resources/base.css`:**
```css
.content td, .content div {
  word-break: break-all;
  overflow-wrap: break-word;
  max-width: 100%;
  overflow: hidden;
}
```

---

## File Change Summary

| File | Changes |
|------|---------|
| `gruyere.py` | XSS escape in `_DoNewsnippet2`; CSP + XSS headers in `_SendHtmlResponse`; DoS fix in `_PROTECTED_URLS` and `_DoQuitserver`; UID validation + constant-time compare in `_DoLogin`; length limit in `_DoNewsnippet2` |
| `sanitize.py` | Extended `disallowed_attributes` with 10+ missing event handlers including `onerror` |
| `resources/login.gtl` | Changed form method from `get` to `post` |
| `resources/newsnippet.gtl` | Changed form method from `get` to `post`; added `maxlength='500'` |
| `resources/base.css` | Added `word-break`, `overflow-wrap`, and `overflow: hidden` to content containers |

---

## References
- [Google Gruyere Codelab](https://google-gruyere.appspot.com/)
- [OWASP Top 10 (2021)](https://owasp.org/www-project-top-ten/)
- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- CWE-79: Cross-Site Scripting
- CWE-400: Uncontrolled Resource Consumption
- CWE-89: SQL Injection
- CWE-120: Buffer Copy Without Checking Size of Input
