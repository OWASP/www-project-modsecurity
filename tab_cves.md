---
title: CVEs
layout:  col-sidebar
tab: true
order: 1
tags: example-tag
---

# CVEs

This page lists the official CVEs published since OWASP took over ModSecurity from Trustave on Jan 25, 2024, and started it anew as OWASP ModSecurity.

## CVE 2024-1019, 2024-01-30

### ModSecurity v3 WAF bypass (severity HIGH, 2024-01-30) 

### Advisory

ModSecurity / libModSecurity 3.0.0 to 3.0.11 is affected by a WAF bypass for path-based payloads submitted via specially crafted request URLs. ModSecurity v3 decodes percent-encoded characters present in request URLs before it separates the URL path component from the optional query string component. This results in an impedance mismatch versus RFC compliant back-end applications. The vulnerability hides an attack payload in the path component of the URL from WAF rules inspecting it. A back-end may be vulnerable if it uses the path component of request URLs to construct queries. Integrators and users are advised to upgrade to 3.0.12. The ModSecurity v2 release line is not affected by this vulnerability.

### Components affected with numbers

ModSecurity / libModSecurity v3.0.0 - v3.0.11

### Fixed version

ModSecurity / libModSecurity v3.0.12

ModSecurity v2.9.x is not affected by this vulnerability.

### Download

* Source code (tagged release): FIXME
* Source code repository: pending
* Pre-Compiled Packages: https://github.com/owasp-modsecurity/ModSecurity

### CVSS 3.1

* Score : 8.6 HIGH
* Attack Complexity: low,
* Attack Vector: network,
* Availability Impact: none,
* Confidentiality Impact: none,
* Integrity Impact: high,
* Privileges Required: none,
* scope: changed,
* userInteraction: none,

It can be argued wether the Integrity Impact should be lowered to "medium" instead of "high". After discussing this with our CNA NCSC Switzerland, we concluded to put it on "high". If set to "medium", the score drops to a "medium" level.


### CWE and CAPEC

* CWE-20: Improper Input Validation
* CAPEC-152: Inject Unexpected Items

In both taxonomies, CWE and CAPEC, we assigned very broad categories. None of the more detailed categories were really spot on.

### Longer description

The [OWASP CRS](https://coreruleset.org) team has discovered an unexpected behavior that can lead to a bypass in the [ModSecurity](https://owasp.org/www-project-modsecurity/) v3 release line (also known as libModSecurity).

For a given HTTP request, ModSecurity3 performs URL decoding on the request URL. For example, the percent-encoded character sequence “%3F” is decoded into a question mark character, “?”. ModSecurity later separates the URL path component from the optional query component, allowing for subsequent rule-based inspection of the path in isolation. ModSecurity performs this separation at the first question mark character encountered in the decoded URL as “?” denotes the start of the query component. Crucially, ModSecurity3 performs the URL decoding step before the URL path separation step.

It is thus possible for an attacker to craft a request URL that contains a percent-encoded question mark character (“`%3F`”) to trick ModSecurity into separating the URL into a path component at a position of the attacker’s choosing. This allows an attacker to place a payload in the path of a request URL following a “`%3F`” character sequence in order to bypass the scrutiny of all rules intended to inspect the URL path component. This is further facilitated by the fact, that the part after the “`%3F`” is not appearing as part of the `QUERY_STRING` variable.

The ModSecurity variables `REQUEST_FILENAME` and `REQUEST_BASENAME` are affected as these variables are intended to hold the URL path, in full or in part, for inspection. ModSecurity rules that inspect or make use of these variables are affected and may be rendered ineffective against exploits of this vulnerability. This includes a significant number of rules in OWASP CRS.

### The fix: Following the correct order of operations

Referring to the relevant RFC governing the use of URIs ([RFC 3986](https://datatracker.ietf.org/doc/html/rfc3986)) reveals that this is a known hazard. The RFC [explicitly states](https://datatracker.ietf.org/doc/html/rfc3986#section-2.4) that a URI must be separated into its component parts before performing URL decoding in order to avoid mistaking an encoded character for a component delimiter.

The fix was to modify the ModSecurity3 function that processes a request URI. The URL is not separated into its components before URL decoding is performed, removing the possibility of ModSecurity3 mistakenly using an incorrect delimiter character.

### Example bypass

Consider sending an HTTP request with the following URL, which includes an SQL injection payload in the path:

```
example.com/admin/id/1%3F+OR+1=1--
```

ModSecurity would perform URL decoding, which gives:

```
example.com/admin/id/1?+OR+1=1--
```

The appearance of a newly decoded question mark character tricks ModSecurity3 into believing that a query component is now present. ModSecurity3 now separates the URL path component from what it believes is a query component and sets the path-related variables like so:

```
REQUEST_FILENAME: /admin/id/1
REQUEST_BASENAME: 1
```

The SQLi payload is no longer present, despite being a valid part of the URL path. The payload will bypass the detection of rules that rely on inspecting ModSecurity3’s URL path variables. Unfortunately, the payload is also not visible as part of the `QUERY_STRING` variable.

As demonstrated, this is a serious issue and ModSecurity v3 users are strongly encouraged to update to the fixed release (v3.0.12) as soon as is practical.

### Workaround

While upgrading to receive the fix is strongly encouraged, if upgrading the ModSecurity engine is impossible for some reason then a workaround can be implemented.

ModSecurity3’s `REQUEST_URI_RAW` variable contains the full URI and is unaffected by the URL decoding step. Comparing it to the path-related variables from the previous example, the raw URI variable looks like so:

```
REQUEST_URI_RAW: /admin/id/1%3F+OR+1=1--
REQUEST_FILENAME: /admin/id/1
REQUEST_BASENAME: 1
```

As such, it is possible to use the `REQUEST_URI_RAW` variable to derive all other required variables correctly, including performing any required URL decoding. Note that the raw URI variable also includes the query component, if present, and so can be cumbersome and difficult to use.

Implementing such a workaround is likely to generate many new false positives with existing rules. For ModSecurity3 users, performing an engine update to receive the fix is likely to be a much more preferable and simpler solution.

### Exploitability

As with every WAF bypass, it takes an additional vulnerability in the back-end application for a successful attack. If the backend application uses path components as query parameters as is likely the case for an application with a layout as in the example presented under “Example Bypass”, then the application may be affected. This can be the case for SQL queries (SQL injection), but also an exploit via a Remote Command Execution (RCE) is possible in a similar constellation.

Again, the back-end must neglect input validation in order for this exploit to work out, but the additional security usually gained from a well-configured WAF is gone with this bypass.

### Timeline

_The timeline comes with a twist. Trustwave transferred the ModSecurity project to OWASP during the window between report and release of the fix. This means that the reporting organization became the new steward of the open source software in question before a fix was released by Trustwave. More to the point: OWASP CRS reported the problem and OWASP recruited the new OWASP ModSecurity team out of the OWASP CRS team._

* 2023-11-13 : OWASP CRS submits report to Trustwave Spiderlabs, includes SQLi proof of concept
* 2023-11-14 : Trustwave Spiderlabs acknowledges report, promises investigation
* 2023-11-28 : OWASP CRS asks for update
* 2023-11-29 : Trustwave Spiderlabs rejects report, describes it as anomaly without security impact
* 2023-12-01 : OWASP CRS reiterates previously shared SQLi proof of concept
* 2023-12-01 : Trustwave Spiderlabs acknowledges security impact
* 2023-12-04 : OWASP CRS shares XSS proof of concept
* 2023-12-07 : Trustwave Spiderlabs promises security release early in the new year
* 2024-01-02 : OWASP CRS asks for update
* 2024-01-03 : Trustwave Spiderlabs announces preview patch by Jan 12, release in the week of Jan 22
* 2024-01-12 : Trustwave Spiderlabs shares preview patch with primary contact from OWASP CRS
* 2024-01-22 : OWASP CRS confirms preview patch fixes vulnerability
* 2024-01-24 : Trustwave Spiderlabs announces transfer of ModSecurity project to OWASP for 2023-01-25
* 2024-01-25 : Trustwave Spiderlabs transfers ModSecurity repository to OWASP
* 2024-01-25 : OWASP creates OWASP ModSecurity, assigns OWASP ModSecurity production level, primary contact from OWASP CRS becomes OWASP ModSecurity co-lead
* 2024-01-26 : OWASP ModSecurity leaders decide to release on 2023-01-30
* 2024-01-27 : OWASP ModSecurity creates GPG to sign upcoming release
* 2024-01-29 : NCSC-CH assigns CVE 2024-1019, advisory text and release notes are being prepared, planned release procedure is discussed with Trustwave Spiderlabs
* 2024-01-30 : OWASP ModSecurity Release 3.0.12

### Reporters

* Andrea Menin [@AndreaTheMiddle](https//twitter/AndreaTheMiddle) https://github.com/theMiddleBlue
* Matteo Pace [@M4tteoP](https//twitter/M4tteoP) https://github.com/M4tteoP
* Max Leske https://github.com/theseion
* Ervin Hegedüs [@Airween](https//twitter/airween) https://github.com/airween (primary contact)

### Acknowledgements

The OWASP ModSecurity team thanks the wider OWASP CRS team for their patience. Unfortunately, this fix took far longer than it should. We also thank Trustwave Spiderlabs, namely Martin Vierula, for the patch for this issue and the support with the release process. Harold Blankenship from OWASP Headquarters overlooked the transfer of ModSecurity and helped with the timely setup of a new OWASP project and GitHub organization. The OWASP project committee around Björn Kimminich fast tracked OWASP ModSecurity to production level. And finally NCSC-CH that supported us with the CVE process substantially. More than anything, Open Source Software is a team effort. Thank you all.
