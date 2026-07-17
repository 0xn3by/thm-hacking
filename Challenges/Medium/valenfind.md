# valenfind

- [scope](#scope)
- [reconnaissance and attack surface](#reconnaissance-and-attack-surface)
- [security findings and validation](#security-findings-and-validation)
- [impact and remediation](#impact-and-remediation)
- [lessons learned](#lessons-learned)
- [analyst notes](#analyst-notes)
- [references](#references)

## scope

### authorization and objective

Testing was conducted only against the TryHackMe ValenFind room instance provided for this assessment. The objective was to identify and safely validate web-application weaknesses within the authorized lab environment.

### room information

- Type: challenge
- Difficulty: medium
- Category: web
- Subscription: free

The room describes a newly created dating application that may contain security weaknesses.

Room link: [TryHackMe ValenFind](https://tryhackme.com/room/lafb2026e10)

### target and testing environment

- Target: `10.48.170.144`
- Primary web application: `http://10.48.170.144:5000`
- Additional exposed service: SSH on TCP/22
- Testing context: authorized TryHackMe lab environment

The screenshots below are room evidence. Sensitive values visible in captures are not repeated in the narrative, commands, or code blocks.

## reconnaissance and attack surface

### host discovery and initial port scan

The initial scan used `-Pn`, treating the target as available without relying on ICMP host discovery:

```text
nmap -Pn 10.48.170.144
```

Relevant output:

```text
22/tcp   open  ssh
5000/tcp open  upnp
```

Port 5000 was initially labelled `upnp` by Nmap's service-name mapping. Because that label was not sufficient to identify the application, a targeted version and default-script scan was performed next.

### service identification

```text
nmap -Pn -sC -sV -p22,5000 10.48.170.144
```

Relevant results:

```text
22/tcp   open  ssh   OpenSSH 9.6p1 Ubuntu
5000/tcp open  http  Werkzeug HTTP server 3.0.1
                     Python 3.12.3
                     HTTP title: ValenFind - Secure Dating
```

Version detection confirmed that TCP/5000 hosts the Python/Werkzeug web application rather than the initial UPnP classification. SSH remains a secondary attack surface; no SSH authentication testing was performed.

### web route discovery

Web content discovery with Gobuster identified the `/login` route. This established the initial authentication surface for manual request analysis.

The route accepts a login form and submits data using `POST` with the `application/x-www-form-urlencoded` content type. No additional public routes are asserted here without supporting enumeration output.

### authentication flow analysis

The login request was reviewed manually in an intercepting proxy. The request shape was:

```http
POST /login HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=<test-username>&password=<redacted-test-value>
```

The form parameters are `username` and `password`. For an unsuccessful authentication attempt, the application returned an HTTP success status with a generic failure message:

```text
HTTP/1.1 200 OK
Invalid credentials
```

![Burp Repeater login request and generic authentication failure](Images/Screenshot%20From%202026-07-16%2021-15-11.png)

*Figure 1 — Burp Repeater shows the `POST /login` request and the application's generic `Invalid credentials` response.*

The response did not distinguish an unknown username from an incorrect password. Username enumeration through distinct error messages was therefore not confirmed. The HTTP 200 status was interpreted together with the response body; it was not treated as evidence of successful authentication. No credential brute-force attack was performed.

## security findings and validation

### finding 1: arbitrary local file read through the layout parameter

**Status:** confirmed

The `/api/fetch_layout` endpoint accepted traversal sequences in its `layout` parameter and returned files outside the intended template location. The captured responses demonstrated access to:

- application source code;
- the running process command line;
- process environment data; and
- `/etc/passwd`.

![Application source code returned through layout traversal](Images/Screenshot%20From%202026-07-16%2021-36-32.png)

*Figure 2 — Application source code is returned through the `layout` parameter.*

![Process command line returned through layout traversal](Images/Screenshot%20From%202026-07-16%2021-36-39.png)

*Figure 3 — The process command line is readable through the same file-fetch behavior.*

![Process environment returned through layout traversal](Images/Screenshot%20From%202026-07-16%2021-36-41.png)

*Figure 4 — Process environment data is exposed by the file-read behavior.*

![System account file returned through layout traversal](Images/Screenshot%20From%202026-07-16%2021-36-44.png)

*Figure 5 — `/etc/passwd` is readable through the vulnerable file path.*

This confirms a path-traversal/local-file-read vulnerability and source disclosure. The response content also exposed implementation and host details that should not be available to an unauthenticated web client.

### finding 2: administrative database export protected by disclosed authorization material

**Status:** confirmed

The disclosed application source contained an administrative database-export route, `/api/admin/export_db`, that checked a custom request header against a hard-coded authorization value. A subsequent request to the route returned a database backup, confirming that the export function was reachable with the disclosed lab value. The value is intentionally not reproduced here.

![Administrative database export route in application source](Images/Screenshot%20From%202026-07-16%2021-58-40.png)

*Figure 6 — The source defines the administrative database-export route and its custom-header check.*

![Retrieved database backup from administrative export route](Images/Screenshot%20From%202026-07-16%2021-58-58.png)

*Figure 7 — A request to the export route successfully retrieves the database backup.*

![SQLite database backup file identification](Images/Screenshot%20From%202026-07-16%2021-59-06.png)

*Figure 8 — The retrieved backup is identified as an SQLite database.*

The issue is not merely the presence of an administrative endpoint: authorization material was embedded in application source and could be used to access a bulk data export. This creates a direct confidentiality risk for all records in the database.

### finding 3: plaintext credentials and personal data in the exported database

**Status:** confirmed

Inspection of the exported `users` table showed a password column containing plaintext values alongside user profile data such as names, addresses, and other contact or biography fields.

![Exported user records with plaintext password and profile fields](Images/Screenshot%20From%202026-07-16%2021-57-07.png)

*Figure 9 — The exported user table contains plaintext password values and profile data.*

Storing passwords in plaintext means a database disclosure immediately exposes reusable authentication material. The profile data also increases privacy impact. A record containing HTML-like input was observed, but script execution was not validated; stored cross-site scripting remains an investigation item rather than a confirmed finding.

### pending validation

The following areas should be tested before closing the assessment:

- whether the file-read behavior is reachable without authentication in a fresh session;
- whether any other endpoints expose or execute the disclosed source and environment data;
- session creation, cookie attributes, logout behavior, and session invalidation;
- rendering of user-controlled profile fields to validate or rule out stored XSS; and
- rate limiting, account lockout, and other authentication abuse controls.

## impact and remediation

### impact

The confirmed issues combine into a high-impact data-exposure chain: arbitrary local file read discloses source and host information; the source discloses administrative authorization material; and that material enables a database export containing plaintext credentials and personal data.

### remediation

- Replace file-path construction with an allowlist of fixed layout identifiers. Resolve paths server-side, reject traversal and absolute paths, and keep source code, process data, and system files outside any user-controlled file-fetch operation.
- Remove hard-coded authorization values from source control and rotate the exposed value. Enforce authentication and authorization server-side for administrative exports, apply least privilege, and audit export access.
- Store passwords using a modern adaptive password-hashing scheme such as Argon2id, bcrypt, or scrypt. Treat the exposed values as compromised and rotate affected lab credentials.
- Minimize exported fields and protect sensitive personal data. Escape or sanitize user-controlled content at the correct output context before rendering it.
- Add regression tests for traversal payloads, unauthenticated access to administrative routes, secret scanning, password storage, and profile-field rendering.

## lessons learned

- Service labels in an initial Nmap scan require version verification before conclusions are drawn.
- An HTTP 200 response does not necessarily indicate successful authentication.
- Generic authentication errors reduce direct username-enumeration risk.
- Manual request analysis should precede brute-force testing.
- Source disclosure can turn a lower-level file-read issue into a direct administrative-access path.
- Sensitive data exposure should be assessed across the full chain, not endpoint by endpoint only.

## references

- [TryHackMe ValenFind room](https://tryhackme.com/room/lafb2026e10)
- [Nmap Reference Guide](https://nmap.org/book/man.html)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
- [OWASP Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [Werkzeug documentation](https://werkzeug.palletsprojects.com/en/stable/)
