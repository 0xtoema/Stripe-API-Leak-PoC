🔐 Development Environment Secret Exposure → Stripe API Account Compromise
Executive Summary

While performing structured reconnaissance on a development environment of a large SaaS platform, I identified a critical secret management failure that resulted in the exposure of a Stripe Secret API key within a publicly accessible frontend bundle.
Although the environment was protected by a WAF at the application layer, the exposed key enabled direct API-level interaction with Stripe, effectively rendering perimeter protections irrelevant.
The issue demonstrates a complete breakdown in secret isolation between backend infrastructure and client-side assets.
All identifiers have been intentionally redacted.
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Context & Initial Observation

The target development subdomain was protected behind an edge-based WAF.
Attempts to access sensitive internal routes such as authentication or CSRF-related endpoints resulted in:

403 Forbidden

This confirmed that direct backend interaction was restricted.
Rather than continuing endpoint fuzzing, I pivoted toward static asset analysis.
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Static Bundle Analysis

Publicly accessible JavaScript bundles were retrieved and inspected manually.
Example pattern:

/static/js/main.[hash].js

The asset was served successfully (200 OK) via object storage.
During review of the compiled bundle, I identified embedded environment variables, including:

REACT_APP_STRIPE_KEY: "sk_test_********************************"

This immediately raised concern for the following reasons:
Prefix sk_ indicates a Stripe Secret Key
Secret keys must never be exposed client-side
They grant full privileged API access

This was not a publishable key (pk_), but a backend secret.
----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Technical Pivot – Why the WAF Became Irrelevant

The application’s WAF protects its own backend routes.
However, the exposed key belongs to Stripe.
By interacting directly with:

https://api.stripe.com

the application infrastructure, WAF, and rate-limiting controls are completely bypassed.
At this stage, the trust boundary shifted from:
Client → Application → Stripe
to:

Attacker → Stripe (Direct)

This is a fundamental architectural break.
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Proof of Access

Using the leaked key in an Authorization header:

Authorization: Bearer sk_test_********************************

I successfully performed controlled API requests.
Validated capabilities included:
Listing recent charges
Retrieving customer objects
Accessing payment metadata
Creating new customer records (write validation)
Returned objects contained:
Customer identifiers
Email addresses
Payment status data
Card brand and last four digits
Transaction amounts and timestamps
This confirms full read and write privileges within the associated Stripe account (test mode).

No destructive actions were performed beyond minimal validation.
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Security Impact Analysis

Even in a development context, the exposure of a Stripe Secret Key introduces:
1️: Financial Account Compromise
Any actor can perform API operations as the account owner.

2️: Data Confidentiality Risk
Customer and transaction metadata become accessible.
ك
3️: Infrastructure Bypass
Application-layer defenses become meaningless once secrets are leaked.

4️: Lateral Risk
If similar CI/CD practices are used in production pipelines, escalation becomes plausible.

5️: Compliance Exposure
Embedding payment processor secrets in client-side code violates secure key management principles and potentially PCI-related standards.
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Root Cause

The issue likely originated from:
Injecting environment variables at build time into frontend artifacts
Improper separation of secret and public configuration variables
Lack of automated secret scanning during CI/CD
Absence of static asset inspection before deployment
This is not a WAF failure.
This is a secret lifecycle failure.
Remediation Strategy
Immediate
Revoke and rotate the exposed key.
Audit Stripe logs for anomalous API usage.
Verify no production keys are similarly exposed.
Structural
Enforce backend-only usage of sk_ keys.
Proxy all Stripe interactions through authenticated server endpoints.
Integrate automated secret detection tools in CI pipelines.
Apply strict environment segregation controls.
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Research Methodology

This finding was discovered through:
Controlled reconnaissance
Static asset inspection
Manual bundle analysis
API-level validation with minimal impact testing
No production systems were targeted.
No real financial transactions were executed.
All sensitive identifiers have been redacted.
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Closing Note

Perimeter defenses are often strong.
But once secrets cross the trust boundary into client-side code, the architecture collapses from within.
This issue was not about bypassing a firewall.
It was about bypassing the design assumptions behind it.
