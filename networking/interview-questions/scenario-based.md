# Networking — Scenario Based

---

## 1. If one user reports errors but others are fine, how would you debug?

Single-user issues are the hardest category to debug because aggregate metrics look healthy. The problem is specific to something about that user's request path — their session, their location, their client, their data, or the specific pod they're hitting.

**Step 1 — Reproduce and isolate. Get specifics from the user.**

```
Questions to ask (or pull from logs):
- What error do they see? (screenshot, HTTP status code, error message)
- What URL / endpoint are they hitting?
- What browser / device / OS?
- What network are they on? (office, VPN, mobile carrier, home ISP)
- When did it start? (correlate with deployments or config changes)
- Is it every request or intermittent?
```

"It doesn't work" is not actionable. "I get a 403 on /api/orders from my laptop on the office Wi-Fi" is.

**Step 2 — Check if it's pod-specific (sticky sessions / session affinity).**

If your load balancer uses session affinity (sticky sessions), one user is pinned to one pod. If that pod is unhealthy, only users pinned to it see errors.

```bash
# Check which pod is serving this user's requests
# If using ALB with sticky sessions — check the AWSALB cookie in the user's browser
# Or check access logs for the user's IP:
kubectl logs -n prod -l app=order-service | grep "203.0.113.45"
# → All requests from this IP hitting pod order-service-7d9f8c-xkj2p

# Check if that specific pod is healthy
kubectl describe pod order-service-7d9f8c-xkj2p -n prod
kubectl logs order-service-7d9f8c-xkj2p -n prod --tail=50
```

If the pod shows errors or high resource usage — kill it. Kubernetes replaces it and the user gets a healthy pod:

```bash
kubectl delete pod order-service-7d9f8c-xkj2p -n prod
```

**Step 3 — Check if it's geographic / DNS related.**

One user in a specific region might resolve DNS to a different endpoint, hit a different CDN edge, or route through a different ISP path.

```bash
# Ask the user to run:
nslookup api.yourapp.com
traceroute api.yourapp.com
curl -v https://api.yourapp.com/health

# Compare their DNS resolution with yours
# If they resolve to a stale IP → DNS TTL issue or corporate DNS caching
```

We had a user on a corporate network whose DNS resolver cached a stale IP for 12 hours (corporate resolver had a forced high TTL). Every other user had the correct IP. The fix was the user's IT team flushing their DNS cache.

**Step 4 — Check if it's data-specific.**

The user's account, their specific data, or their specific request triggers a code path that fails for their data but works for everyone else.

```bash
# Search logs for the user's identifier
kubectl logs -n prod -l app=order-service | grep "user_id=12345"
# Look for: NullPointerException, validation error, specific field missing

# Or query structured logs:
# Loki: {app="order-service"} |= "user_id=12345" | level="ERROR"
```

We had a user whose name contained an emoji. The serialization layer didn't handle it — returned a 500 for every API call involving their profile. Every other user worked fine. The fix was UTF-8 encoding at the serialization boundary. Without searching by their specific user ID, this would have been invisible in aggregate metrics.

**Step 5 — Check if it's client-side.**

```bash
# Ask the user to:
# 1. Open DevTools → Network tab → reproduce the error
# 2. Look for: red requests, CORS errors, mixed content warnings, SSL errors
# 3. Try the same request in incognito (rules out browser extensions, cached auth)
# 4. Try from a different browser / device

# Common client-side causes:
# - Browser extension blocking requests (ad blockers, privacy extensions)
# - Expired or corrupted auth token in localStorage
# - Cached stale response (service worker caching old API response)
# - Corporate proxy modifying requests (stripping headers, injecting certificates)
# - Clock skew on user's machine (JWT validation fails if clock is >5 min off)
```

**Step 6 — Check for cache poisoning.**

If a CDN or reverse proxy cached an error response, one user gets a cached 500 while everyone else gets fresh 200s.

```bash
# Check CloudFront or CDN cache:
curl -H "Cache-Control: no-cache" https://api.yourapp.com/problematic-endpoint
# If this returns 200 but the user's browser gets 500 → cached error response

# Invalidate the CDN cache for that path:
aws cloudfront create-invalidation \
  --distribution-id E1234567890 \
  --paths "/api/problematic-endpoint"
```

**Step 7 — Check rate limiting or WAF rules.**

WAF or rate limiting might be blocking one user's IP while allowing others:

```bash
# Check WAF logs for the user's IP
aws wafv2 get-sampled-requests \
  --web-acl-arn arn:aws:wafv2:ap-south-1:123456:regional/webacl/prod-waf/xxx \
  --rule-metric-name BlockedRequests \
  --scope REGIONAL \
  --time-window StartTime=2024-01-15T14:00:00Z,EndTime=2024-01-15T15:00:00Z \
  --max-items 100

# If the user's IP is being rate-limited:
# → Check if they're behind a corporate NAT (many users share one IP, triggering rate limits)
# → Whitelist the corporate IP or increase the rate limit
```

---

**Quick diagnostic flowchart:**

```
One user reports errors
   ↓
Get: error message, URL, browser, network, timing
   ↓
Reproduce? → Check if pod-specific (session affinity → bad pod)
   ↓
Not pod-specific → Check DNS resolution (stale IP, corporate resolver)
   ↓
DNS is fine → Check logs for user-specific errors (data-related bug)
   ↓
No server errors → Client-side (DevTools, incognito, different browser)
   ↓
Still broken → Check WAF/rate limiting, CDN cache, corporate proxy
```

**Real scenario:** A user reported "your site is broken" with no other details. After getting a screenshot (403 Forbidden), we checked WAF logs — their corporate office had 200 employees behind one public IP. Our WAF rate limit was 100 requests per IP per minute. Their office hit 100 requests (from different employees), and the WAF blocked their shared IP for 5 minutes. One user's experience looked like "the site is broken" when it was actually rate limiting applied to their corporate NAT IP. We whitelisted their corporate IP range in WAF.

