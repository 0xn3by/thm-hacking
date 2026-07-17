# cmspit security assessment

- [scope](#scope)
- [reconnaissance and attack surface](#reconnaissance-and-attack-surface)
- [security findings and validation](#security-findings-and-validation)
- [impact and remediation](#impact-and-remediation)
- [lessons learned](#lessons-learned)
- [analyst notes](#analyst-notes)
- [references](#references)

## scope

### authorization and objective

Testing was conducted only against the TryHackMe CMSpit room machine in the authorized lab environment. The room objective is to assess a vulnerable CMS, identify the web attack path, and continue into host and privilege-escalation validation.

### room information

- Official room name: CMSpit
- Type: machine room
- Difficulty: medium
- Focus: web application hacking and privilege escalation
- Room duration: approximately 75 minutes

The room description states that the CMS exposes vulnerabilities that allow user enumeration and account-password changes. Those room objectives are recorded as scope context; only behavior supported by the supplied screenshots is treated as confirmed below.

Room link: [TryHackMe CMSpit](https://tryhackme.com/room/cmspit)

### target and testing environment

- Target address: _add the deployed VM IP here_
- Testing context: authorized TryHackMe lab environment
- Attacker environment: Kali/AttackBox terminal evidence
- Target operating context shown in the evidence: Ubuntu/Linux web server

The supplied screenshots are room evidence. Sensitive values visible in captures are not repeated in the narrative, commands, or code blocks.

## reconnaissance and attack surface

### host discovery and service identification

No host-discovery or port-scan output was included in the CMSplit screenshot set. Add the target IP, initial Nmap command, open ports, and service versions here when those results are available. The current report therefore starts at web-application and post-exploitation evidence rather than inferring network services that are not shown.

### client-side asset discovery

Page-source evidence referenced versioned application assets, including a stylesheet under `/assets/app/css/` and a JavaScript resource under a `/storage/tmp/` path.

![Versioned CSS and JavaScript assets in page source](Images/cmsplit/Screenshot%20From%202026-07-17%2000-07-17.png)

*Figure 1 — Page source exposes versioned static assets and a temporary-storage JavaScript path.*

The asset paths establish useful application structure for further enumeration. They do not, by themselves, prove a vulnerability or disclose the complete CMS version.

### vulnerability context and technology identification

An application reference page identified ExifTool as a relevant metadata-processing component and cited CVE-2021-22204 and CVE-2021-22205.

![ExifTool vulnerability context and referenced CVEs](Images/cmsplit/Screenshot%20From%202026-07-17%2014-59-27.png)

*Figure 2 — The application context identifies ExifTool and two relevant CVE references.*

This is vulnerability research context, not proof that the target is vulnerable. The target binary version, execution context, and applicable privilege boundary still require validation.

### authentication route discovery

The page source showed a login form using `POST` and submitting to `/auth/check`:

```http
POST /auth/check
```

![CMS authentication form action](Images/cmsplit/Screenshot%20From%202026-07-17%2016-15-53.png)

*Figure 3 — The login form submits credentials to the `/auth/check` route.*

The screenshot confirms the route and method, but does not establish authentication bypass, user enumeration, or password-change behavior. Those behaviors remain separate validation tasks.

## security findings and validation

### post-exploitation shell context

**Status:** confirmed observation; initial access vector not shown

The evidence begins with a listener receiving a connection and an interactive shell running as `www-data` on Ubuntu. A Python pseudo-terminal was then spawned to improve shell interactivity.

![Interactive shell running as www-data](Images/cmsplit/Screenshot%20From%202026-07-17%2015-25-57.png)

*Figure 4 — The captured session is an interactive `www-data` shell on the target host.*

The screenshot proves the shell context but does not show how the initial connection was obtained. The initial exploit or delivery mechanism must not be inferred from this post-exploitation capture alone.

### finding 1: readable user shell history exposes sensitive operations

**Status:** confirmed

From the `www-data` shell, `/home/stux` was enumerated. The directory listing showed a `.dbshell` history file readable by other users, alongside other user-home artifacts.

![stux home directory and readable shell-history file](Images/cmsplit/Screenshot%20From%202026-07-17%2015-26-01.png)

*Figure 5 — The `stux` home directory contains a readable `.dbshell` history file.*

Reading that history exposed MongoDB commands, database names, account-related values, and other sensitive operational material. The values are intentionally not reproduced here.

![MongoDB shell history and subsequent user switch](Images/cmsplit/Screenshot%20From%202026-07-17%2015-26-06.png)

*Figure 6 — The readable history contains database operations and sensitive account material; the session then switches to `stux`.*

This is a confirmed information-disclosure issue. A web-service account could read another user’s shell history, and the history contained material that enabled progression to the `stux` account.

### finding 2: user-level access and document-database exposure

**Status:** confirmed

After the account transition, the session accessed the `stux` home directory and read `user.txt`.

![User-level shell and user.txt retrieval](Images/cmsplit/Screenshot%20From%202026-07-17%2015-26-08.png)

*Figure 7 — The evidence confirms user-level access and retrieval of the user flag file.*

The same evidence set shows MongoDB shell-history commands referencing user and flag collections. This confirms that document-database access was part of the observed attack path, but the report intentionally does not reproduce flag values or credentials.

### privilege-escalation lead: ExifTool file-write capability

**Status:** investigation lead; target exploitation not confirmed

The ExifTool reference page documents a file-write capability and distinguishes unprivileged and `sudo` execution contexts.

![ExifTool file-write reference](Images/cmsplit/Screenshot%20From%202026-07-17%2016-13-56.png)

*Figure 8 — ExifTool file-write behavior is documented as a possible privilege-escalation technique.*

This screenshot is a technique reference, not target proof. To confirm exploitability, validate the target’s ExifTool version, binary path, `sudo -l` policy, permitted environment behavior, and whether a controlled proof of privilege change is possible. No root shell or `root.txt` retrieval is claimed from the current screenshots.

### pending validation

The following activities remain open:

- record the target IP and initial Nmap results;
- identify and version the CMS and confirm the relevant web routes;
- reproduce user enumeration and password-change behavior within the room scope;
- determine how the initial `www-data` shell was obtained;
- verify MongoDB authentication, authorization, and collection exposure;
- validate the target ExifTool version and effective `sudo` permissions; and
- confirm or rule out root-level impact with a controlled, non-destructive proof.

These are investigation items, not additional confirmed findings.

## impact and remediation

### impact

The confirmed evidence shows a meaningful post-compromise chain: a web-service context could access a user’s shell history, the history exposed sensitive database and account material, and that exposure enabled transition to the `stux` user. The evidence also demonstrates access to user-level challenge data. Root compromise is not confirmed.

### remediation

- Restrict home-directory and shell-history permissions so service accounts cannot read other users’ histories. Remove secrets from shell histories and rotate any exposed credentials.
- Run the web application with the minimum required filesystem permissions and isolate it from user home directories and database administration tools.
- Require authentication and least-privilege authorization for MongoDB. Do not store credentials or operational secrets in command history.
- Patch ExifTool to a supported version and remove unnecessary `sudo` permissions. If `sudo` access is required, allow only narrowly scoped commands and test the effective policy.
- Validate CMS user-enumeration and password-change endpoints with generic responses, authorization checks, rate limiting, and secure password-reset controls.
- Record and test the initial-access path separately so the full attack chain can be reproduced without relying on an unexplained shell state.

## lessons learned

- Page-source asset paths can reveal application structure and useful enumeration leads without proving a vulnerability.
- External vulnerability references must be matched to the target version and execution context before being treated as findings.
- Shell histories are sensitive files and can expose credentials, database commands, and lateral-movement paths.
- A captured `www-data` shell proves post-exploitation context, not the initial access vector.
- A privilege-escalation technique reference is not equivalent to a successful target compromise.
- Evidence should distinguish user-level access, database exposure, and root-level impact.

## references

- [TryHackMe CMSpit room](https://tryhackme.com/room/cmspit)
- [ExifTool official site](https://exiftool.org/)
- [NVD: CVE-2021-22204](https://nvd.nist.gov/vuln/detail/CVE-2021-22204)
- [NVD: CVE-2021-22205](https://nvd.nist.gov/vuln/detail/CVE-2021-22205)
- [GTFOBins: exiftool](https://gtfobins.github.io/gtfobins/exiftool/)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
