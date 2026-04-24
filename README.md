# Google Gruyere Security Audit & Remediation
**Student:** Mutsaka Emmason  23/U/23689    2300723689  
**Course:** CSC3207 – Computer Security  
**University:** Makerere University  

## Project Overview
This repository contains a comprehensive security assessment of the Google Gruyere web application. The project involved migrating the legacy codebase to **Python 3.12** and fixing critical security vulnerabilities.

## Repository Map
To navigate this project, please explore the following directories:

* [**Vulnerable Version**](./vulnerable_version/): Contains the baseline application code used to demonstrate and document active exploits.
* [**Fixed Version**](./fixed_version/): Contains the hardened application featuring Python-level patches and CSS-based UI defenses.

## Vulnerabilities Addressed
1. **Stored XSS:** Fixed via output encoding with `html.escape`.
2. **Denial of Service (DoS):** Fixed by removing/protecting the administrative `/quit` endpoint.
3. **SQL Injection & Credential Exposure:** Remedied by transitioning to POST requests and implementing input validation.
4. **Buffer Overflow (Layout-Level):** Mitigated using CSS `word-break` and `overflow-wrap` properties.

---
*For a detailed breakdown of findings and methodology, please refer to the formal PDF report submitted via the student portal.*
