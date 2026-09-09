# Exception Record - ZAP Rule 10038

**Rule:** 10038 - Content Security Policy (CSP) Header Not Set
**Current policy:** FAIL (`.zap/rules.tsv`, commit bbc619d)
**Status:** Time-limited exception. Policy remains FAIL; the release is approved
despite the failing gate, and the gate is not weakened.

## URL scope

Narrow. Five instances on the lab target `http://127.0.0.1:3000`, confirmed in
`report_json.json` (`"count": "5"`):

- `http://127.0.0.1:3000`
- `http://127.0.0.1:3000/`
- `http://127.0.0.1:3000/ftp`
- `http://127.0.0.1:3000/ftp/coupons_2013.md.bak`
- `http://127.0.0.1:3000/sitemap.xml`

This exception applies to the ephemeral CI container only. It does not extend to
any deployed or staging environment.

## Evidence

- **Run 34304422950** - enforcement mode, `fail_action: true`. `FAIL-NEW: 1`
  (10038 x 5), red conclusion, ZAP exit code **1** ("a FAIL rule fired").
- **Run 34297893922** - observation mode, `fail_action: false`. Same finding,
  same policy applied, green conclusion. A green job carrying a FAIL finding.
- **Run 33993343018** - the same gate *before* the policy file was actually
  loaded: red, but `FAIL-NEW: 0` and exit code **2** ("warnings exist"). Retained
  because the exit-code difference is what distinguishes a policy-driven failure
  from a fail-on-any-alert failure.
- `report_json.json`, alert 10038: `riskdesc: "Medium (High)"`, `count: "5"`.
  The HTML and Markdown reports collapse this to "Systemic"; only the JSON
  carries the instance count and URIs.
- Target image digest:
  `sha256:73c53fbf442e8337b3ea3d98c7e8550308854701ebdfce4cc39768f36b75430e`

### Header absence verified directly

Local container started by the author (`docker run -d --rm --name juice-shop
-p 3000:3000 bkimminich/juice-shop:latest`), then:

```
$ curl -sSI http://127.0.0.1:3000/
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Feature-Policy: payment 'self'
X-Recruiting: /#/jobs
Accept-Ranges: bytes
Cache-Control: public, max-age=0
Content-Type: text/html; charset=UTF-8
Content-Length: 9393
Vary: Accept-Encoding

$ curl -sSI http://127.0.0.1:3000/ftp
HTTP/1.1 200 OK
Access-Control-Allow-Origin: *
X-Content-Type-Options: nosniff
X-Frame-Options: SAMEORIGIN
Feature-Policy: payment 'self'
X-Recruiting: /#/jobs
Content-Type: text/html; charset=utf-8
Content-Length: 11318
Vary: Accept-Encoding
```

No `Content-Security-Policy` header on either response. Checked across four
reachable paths (`/`, `/ftp`, `/ftp/coupons_2013.md.bak`, `/sitemap.xml`) -
absent on all four. The fifth reported instance is the bare origin, which
returns the same response as `/`.

This is an *absence* finding, so ZAP's evidence field is empty - there is no
matched string to quote, and verification can only be done by inspecting headers
directly.

**Note what the output shows beyond the finding itself:** the application does
set `X-Content-Type-Options`, `X-Frame-Options`, and `Feature-Policy`. This is
not an application that neglects response headers; it configures several and
omits this one. That also independently corroborates the scan, which returned
PASS for rules 10020 (Anti-clickjacking) and 10021 (X-Content-Type-Options) and
WARN for 10063 (Deprecated Feature Policy Header Set).

## Why remediation is not possible in this lab

The workflow scans `bkimminich/juice-shop:latest`, a prebuilt image pulled from
Docker Hub. It is not built from this repository. No commit to this fork can
change the headers that image serves, so there is no code change available here
that would clear this finding.

This is worth stating plainly because it is a property of the pipeline, not just
of this rule: the gate is attached to a branch while measuring an artifact that
branch cannot influence. Remediation requires either (a) rebuilding the image
from forked source inside the workflow, or (b) terminating TLS behind a proxy
that injects the header - both outside the scope of this lab.

## Compensating control

**None effective in the lab environment.** This is stated plainly rather than
invented. The target is a deliberately vulnerable training application on
loopback with no authentication, no reverse proxy, and no WAF. The honest
compensating control is environmental, not technical: the target is ephemeral,
is destroyed when the job ends, is not reachable from any network, and holds no
real data.

**What this exception actually accepts.** CSP is the mitigating control for rule
10110 (`bypassSecurityTrustHtml` in `/main.js`), and that sink has been
confirmed reachable, not merely suspected:

- Bundle inspection found **8** calls to `bypassSecurityTrustHtml` in
  `/main.js`, one of which binds the search term directly
  (`this.searchValue = this.sanitizer.bypassSecurityTrustHtml(e)`).
- Reproduction: searching `<b>hello</b>` at `http://127.0.0.1:3000/#/search`
  returned a results heading rendering **hello** in bold with the tags consumed,
  rather than displaying them as literal text. The input is interpolated as
  HTML, not escaped. A benign markup probe was used deliberately - it
  establishes the vulnerability class without executing script.

Accepting 10038 therefore accepts a **confirmed, reproducible injection sink
with no browser-side containment** - not a theoretical one. That is why this is
a time-limited exception with an expiry rather than a permanent IGNORE, and why
rule 10110 is tracked for remediation independently of this record.

## Owner / Approver / Expiration

- **Owner:** Michael Thompson
- **Approver:** Professor Dr. Ora Kenneth Melie
- **Granted:** 2026-09-08
- **Expires:** 2026-10-08
- **On expiry:** re-run the baseline. If 10038 still fails, either build the
  target from source in the workflow so remediation is possible, or escalate.
  This exception must not be renewed by default.
