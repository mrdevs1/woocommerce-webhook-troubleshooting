# WooCommerce Webhook Troubleshooting Guide

A practical guide to diagnosing WooCommerce webhook failures, delivery errors, duplicate events, signature verification problems, and server-side integration issues.

This repository is intended for WooCommerce developers, WordPress administrators, hosting support engineers, and anyone maintaining e-commerce integrations.

## Table of Contents

- [Overview](#overview)
- [Common Webhook Problems](#common-webhook-problems)
- [1. Understand WooCommerce Webhooks](#1-understand-woocommerce-webhooks)
- [2. Check Webhook Configuration](#2-check-webhook-configuration)
- [3. Review Delivery Logs](#3-review-delivery-logs)
- [4. Diagnose HTTP Response Codes](#4-diagnose-http-response-codes)
- [5. Test the Webhook Endpoint](#5-test-the-webhook-endpoint)
- [6. Troubleshoot Signature Verification](#6-troubleshoot-signature-verification)
- [7. Investigate Cloudflare and Security Plugins](#7-investigate-cloudflare-and-security-plugins)
- [8. Diagnose Duplicate Events](#8-diagnose-duplicate-events)
- [9. Investigate PHP and Server Errors](#9-investigate-php-and-server-errors)
- [10. Retry Failed Deliveries Safely](#10-retry-failed-deliveries-safely)
- [11. Security Best Practices](#11-security-best-practices)
- [Troubleshooting Checklist](#troubleshooting-checklist)
- [Useful References](#useful-references)
- [Contributing](#contributing)
- [License](#license)

## Overview

WooCommerce webhooks send event notifications from a store to an external URL when configured events occur.

Examples include:

- An order is created or updated.
- A product is created or modified.
- A customer record changes.
- An order is deleted.
- A supported WooCommerce action triggers an integration.

Webhooks allow WooCommerce to communicate with external applications, inventory systems, fulfillment services, and custom integrations.

When a webhook fails, the receiving application may not process the event. The cause can be an incorrect destination URL, authentication failure, server timeout, blocked request, invalid signature, or an error in the receiving application.

The correct troubleshooting process is to identify the failing component before changing production settings.

## Common Webhook Problems

| Symptom | Possible cause |
|---|---|
| Webhook delivery fails | Unreachable destination or non-success response |
| HTTP 400 | Invalid request or payload |
| HTTP 401 | Missing or invalid authentication |
| HTTP 403 | Firewall, permission, or security restriction |
| HTTP 404 | Incorrect endpoint URL |
| HTTP 429 | Rate limiting |
| HTTP 500 | Receiving application or server error |
| HTTP 502 or 504 | Upstream failure or timeout |
| Duplicate orders or updates | Retries, repeated events, or non-idempotent processing |
| Invalid signature | Incorrect secret or request-body verification |
| Webhook remains pending or fails repeatedly | Receiver unavailable or delivery processing issue |

These are common possibilities, not definitive diagnoses. Always compare the delivery response with logs from the receiving application and server.

## 1. Understand WooCommerce Webhooks

WooCommerce provides webhook management through the WordPress administration interface.

Navigate to:

**WooCommerce → Settings → Advanced → Webhooks**

The exact interface may differ by WooCommerce version.

A webhook generally includes:

- **Name:** A label identifying the integration.
- **Status:** Whether the webhook is active.
- **Topic:** The event that triggers delivery.
- **Delivery URL:** The destination receiving the request.
- **Secret:** A shared secret used to generate the request signature.
- **API version:** The configured webhook payload version.

Before troubleshooting, identify the specific webhook, its topic, the destination URL, and the time of the failed delivery.

Do not publish webhook secrets, authentication tokens, or complete customer payloads in support requests.

## 2. Check Webhook Configuration

Incorrect configuration can prevent successful delivery even when both servers are operational.

Verify the following:

1. The webhook is enabled.
2. The topic matches the intended event.
3. The delivery URL is correct.
4. The destination uses the expected protocol, normally HTTPS.
5. The receiver accepts the HTTP method and content type used by WooCommerce.
6. The endpoint is accessible from the WooCommerce server.
7. The receiver has the expected authentication and signature-verification configuration.

### Check the destination URL

Confirm the URL does not contain:

- A misspelled hostname.
- An incorrect path.
- An outdated endpoint.
- An unexpected redirect.
- A missing required route or API version.

If the endpoint redirects from HTTP to HTTPS or from one hostname to another, update the configured URL to the intended final HTTPS endpoint where appropriate.

Do not assume a redirect is harmless: some integrations or clients may handle redirects differently.

## 3. Review Delivery Logs

WooCommerce records webhook delivery attempts in its logging system. Depending on the version and interface, delivery logs may be available from the webhook's details or the WooCommerce logs area.

Check the available logs before retrying a failed delivery.

Look for:

- Request timestamp.
- Destination URL.
- HTTP response status.
- Response body.
- Connection or timeout errors.
- Repeated attempts.
- Changes in behavior after a deployment.

### Investigate the timeline

Compare the webhook failure timestamp against:

- WordPress and WooCommerce logs.
- PHP error logs.
- Web-server error logs.
- Cloudflare security events, if enabled.
- Receiving application logs.
- Database errors and resource alerts.

A response generated by the receiver may contain useful diagnostics, but redact personal information, authentication data, and internal details before sharing it.

## 4. Diagnose HTTP Response Codes

### HTTP 400 — Bad Request

Possible causes:

- Invalid or incomplete payload.
- Incorrect content handling.
- Unexpected data types.
- Receiver-side validation failure.

**Recommended action:** Inspect the receiver's validation errors and compare the incoming payload with the expected schema.

### HTTP 401 — Unauthorized

Possible causes:

- Required authentication is missing.
- Credentials are incorrect.
- The receiver requires authentication not configured for the integration.

**Recommended action:** Verify the receiver's authentication requirements. Do not place secret credentials in public URLs or logs.

### HTTP 403 — Forbidden

Possible causes:

- Web application firewall rules.
- Security-plugin restrictions.
- Receiver authorization rules.
- IP allowlisting.
- Access restrictions at the web server or reverse proxy.

**Recommended action:** Check security logs and the receiver's authorization rules. Make only a narrow, verified exception if a legitimate request is being blocked.

### HTTP 404 — Not Found

Possible causes:

- Incorrect delivery URL.
- Deleted endpoint.
- Changed API route.
- Incorrect application routing.

**Recommended action:** Verify the route and test it using the expected HTTP method.

### HTTP 429 — Too Many Requests

Possible causes:

- Rate limits.
- Excessive retry traffic.
- A receiver unable to process events quickly enough.

**Recommended action:** Check the receiver's rate-limit policy and processing capacity. Use a controlled retry strategy instead of repeatedly sending requests.

### HTTP 500 — Internal Server Error

Possible causes:

- Application exceptions.
- PHP errors.
- Database failures.
- Invalid assumptions in the receiver's processing code.

**Recommended action:** Check the receiving application's logs at the exact delivery timestamp.

### HTTP 502 or 504

Possible causes:

- An unavailable upstream application.
- Reverse-proxy errors.
- PHP worker exhaustion.
- Slow database queries.
- Processing that exceeds a timeout.

**Recommended action:** Check the receiver's web server, application service, PHP handler, and database. Do not increase timeouts without investigating the underlying delay.

## 5. Test the Webhook Endpoint

Use a controlled test endpoint or an endpoint you own and are authorized to test.

A basic connectivity check:

```bash
curl -sS -o /dev/null \
  -w 'HTTP status: %{http_code}\nTotal time: %{time_total}s\n' \
  https://example.com/webhook
```

Replace the example URL with your actual endpoint.

This command sends a GET request. It checks basic reachability but does not reproduce a WooCommerce webhook, which normally uses POST.

### Test the expected HTTP method

For an endpoint that accepts a simple test payload, use:

```bash
curl -sS -X POST \
  -H 'Content-Type: application/json' \
  --data '{"event":"diagnostic_test"}' \
  -w '\nHTTP status: %{http_code}\nTotal time: %{time_total}s\n' \
  https://example.com/webhook
```

Only use this against a test endpoint or a production endpoint where the payload is known to be safe.

This example does not include WooCommerce's actual webhook signature. A receiver that validates signatures may reject it, which is expected.

A successful synthetic request proves only that the endpoint handled that particular request. It does not prove that WooCommerce's real webhook delivery will succeed.

### Check redirects

```bash
curl -sS -I https://example.com/webhook
```

The `-I` option sends a HEAD request, which some endpoints do not support. An unsupported HEAD request does not necessarily mean that POST delivery will fail.

For accurate diagnostics, inspect the actual webhook delivery response and test with the correct method.

## 6. Troubleshoot Signature Verification

WooCommerce webhook requests can include an `X-WC-Webhook-Signature` header. The signature is calculated using the configured webhook secret and the request body.

Signature verification helps the receiver confirm that the request was generated by a party possessing the shared secret and that the body matches the signed content.

Common causes of signature failures include:

- Different secrets configured on the sender and receiver.
- The receiver verifying a parsed and re-encoded JSON body instead of the original request bytes.
- Incorrect Base64 handling.
- A custom implementation using the wrong hashing algorithm.
- Middleware modifying the request body before verification.

### Verification principles

1. Use the correct webhook secret.
2. Verify the signature against the exact raw request body.
3. Use the expected HMAC-SHA256 algorithm.
4. Decode and compare signatures according to the documented format.
5. Use a constant-time comparison function.
6. Never log or expose the webhook secret.

Do not disable signature verification merely to make deliveries succeed. Investigate the sender and receiver configuration instead.

Official documentation:

https://woocommerce.github.io/woocommerce-rest-api-docs/#webhooks

## 7. Investigate Cloudflare and Security Plugins

A firewall, CDN, or WordPress security plugin may block legitimate webhook requests.

Potential sources include:

- Cloudflare WAF or custom security rules.
- WordPress security plugins.
- Server-level ModSecurity rules.
- IP restrictions.
- Reverse-proxy access controls.
- Rate-limiting rules.

### Diagnostic procedure

1. Record the exact time of a failed delivery.
2. Review the relevant firewall or security events.
3. Determine whether the request reached the origin server.
4. Identify the rule that blocked or challenged the request.
5. Confirm that the request belongs to the expected integration.
6. Apply a narrow exception only when justified.
7. Retest and confirm that other security protections remain active.

Do not disable an entire firewall or security plugin to troubleshoot one endpoint.

Do not trust a request solely because it claims to originate from WooCommerce. Verify the webhook signature and use appropriate access controls.

## 8. Diagnose Duplicate Events

Webhook delivery should be designed to tolerate repeated events. A sender may retry a delivery after a timeout or failed response, and a receiver may process an event successfully even if its response never reaches the sender.

Without duplicate protection, this can create:

- Duplicate orders in an external system.
- Repeated inventory adjustments.
- Duplicate fulfillment requests.
- Multiple notifications.
- Repeated financial or business operations.

### Recommended receiver design

- Validate the request before processing it.
- Use a stable event identifier or delivery identifier when available.
- Store processed-event records durably.
- Enforce uniqueness with database constraints where appropriate.
- Make processing idempotent.
- Return a successful response after the event has been safely accepted or processed, according to the integration's design.
- Separate event acceptance from lengthy background processing when appropriate.

Do not assume that two events with identical payloads are always duplicates; legitimate events can contain the same data. Use an appropriate identifier and business rules.

## 9. Investigate PHP and Server Errors

If the receiver is a PHP application or WordPress site, inspect the logs for errors at the time of delivery.

Typical issues include:

- PHP fatal errors.
- Memory exhaustion.
- Database connection failures.
- Slow SQL queries.
- PHP-FPM worker exhaustion.
- File permission problems.
- Uncaught application exceptions.
- Upstream timeouts.

### Check PHP-FPM service status

On systems using PHP-FPM, replace the service name with the one installed on the server:

```bash
systemctl status php-fpm
```

For a version-specific service, the name may resemble:

```bash
systemctl status php8.2-fpm
```

### Inspect recent service logs

```bash
journalctl -u php-fpm --since "30 minutes ago" --no-pager
```

### Check overall memory and load

```bash
free -h
uptime
```

### Review the web-server logs

The log location depends on the web server, hosting panel, and virtual-host configuration. Review the relevant site's error log rather than assuming one universal path.

For cPanel, CloudLinux, and LiteSpeed environments, use the supported management interface and confirm which PHP handler the domain actually uses before applying PHP-FPM-specific changes.

## 10. Retry Failed Deliveries Safely

Before retrying a failed webhook:

1. Identify the failure's root cause.
2. Confirm that the receiving endpoint is healthy.
3. Check whether the event was already processed.
4. Confirm whether the integration supports duplicate protection.
5. Retry a small number of events first.
6. Verify the outcome in the receiver's logs and application data.

Avoid repeatedly retrying every failed event while the destination remains unavailable. This can overload the receiver and create a backlog.

For integrations that require reliable processing, consider a durable queue, bounded retry attempts, exponential backoff, and a dead-letter process for events that cannot be processed automatically.

Preserve the original event details securely so that failures can be investigated without exposing customer information.

## 11. Security Best Practices

- Use HTTPS for webhook destinations.
- Protect webhook secrets and rotate them when compromise is suspected.
- Validate webhook signatures using the original request body.
- Use constant-time signature comparison.
- Validate incoming data before processing it.
- Prevent duplicate processing.
- Apply request-size and rate limits appropriate to the integration.
- Keep WooCommerce, WordPress, PHP, and dependencies updated.
- Avoid logging secrets, authentication headers, or unnecessary personal data.
- Restrict access to delivery logs and diagnostic endpoints.
- Never expose private PHP-FPM status endpoints to the public.
- Use narrowly scoped firewall exceptions.
- Do not execute arbitrary commands or trust payload fields without validation.

Webhook signatures authenticate possession of a shared secret; they do not replace input validation, authorization, or application-level security controls.

## Troubleshooting Checklist

Use this checklist when investigating a webhook delivery failure.

- [ ] Identify the affected webhook and event topic.
- [ ] Verify the delivery URL and active status.
- [ ] Record the delivery timestamp and response code.
- [ ] Inspect WooCommerce delivery logs.
- [ ] Confirm whether the request reached the destination.
- [ ] Review receiver, PHP, and web-server logs.
- [ ] Investigate firewall, CDN, and security-plugin events.
- [ ] Validate signature configuration and raw-body handling.
- [ ] Check database and application performance.
- [ ] Determine whether duplicate processing occurred.
- [ ] Fix the underlying issue before retrying.
- [ ] Verify successful processing at the destination.
- [ ] Document the cause and preventive action.

## Contributing

Contributions are welcome.

You can help by:

- Adding verified troubleshooting examples.
- Documenting version-specific WooCommerce behavior.
- Improving diagnostic commands.
- Sharing redacted error messages and safe test procedures.
- Adding guidance for payment, inventory, and fulfillment integrations.
- Correcting inaccurate or outdated documentation.

When submitting a contribution, include the affected environment, observed behavior, diagnostic evidence, and the proposed fix.

Do not include credentials, webhook secrets, customer information, private URLs, or unredacted production logs.

## License

This project is available under the MIT License.
