# Reading and Analyzing Logs

## The Challenge

When you first see logs in Splunk, they look overwhelming:
```
192.168.56.1 - admin [15/Nov/2025:11:55:26.597 -0500] "GET /en-US/splunkd/__raw/services/search/shelper?output_mode=json&snippet=true... HTTP/1.1" 200 5657 "-" "Mozilla/5.0..." - 5a4cd46829949bc277bd975a8f1d22a6 4ms
```

**The secret:** You don't read every character. You scan for patterns.

## Log Anatomy (Web Access Log)

### Zone 1: WHO (Source)
```
192.168.56.1 - admin
```
- IP address of requester
- Username making the request

**Analysis question:** Is this IP/user expected or suspicious?

### Zone 2: WHEN (Timestamp)
```
[15/Nov/2025:11:55:26.597 -0500]
```
- Date, time, timezone
- Critical for attack timeline correlation

### Zone 3: WHAT (Action)
```
"GET /en-US/splunkd/__raw/services/search/shelper?..."
```
- **GET/POST** - Method (reading vs writing)
- **Path** - What resource was accessed
- **Parameters** (after `?`) - Usually can skim unless deep investigation

**Translation:** "Admin user requested search help from Splunk web interface"

### Zone 4: RESULT (Status Code)
```
200 5657
```
- **200** = Success
- **5657** = Bytes returned

**Key status codes:**
- `200` = Success ✅
- `401` = Unauthorized (authentication failed) 🚨
- `403` = Forbidden (no permissions) 🚨
- `404` = Not found
- `500` = Server error 🚨

### Zone 5: METADATA (Often Ignore)
```
"-" "Mozilla/5.0..." - 5a4cd... 4ms
```
- Referrer, user agent, session ID, response time
- Usually not relevant for initial triage

## Scanning Technique

### Step 1: Status Codes First
Look for `401`, `403`, `500` - these indicate problems

### Step 2: Action (Method + Path)
- `GET /search` = Normal reading
- `POST /admin/delete` = Modification (verify it's authorized)
- `GET /../../../../etc/passwd` = Path traversal attack! 🚨

### Step 3: Who and When
- Internal or external IP?
- Work hours or 3 AM?
- Expected user?

### Step 4: Ignore the Noise
- Long query parameters - skim
- User agent strings - ignore unless investigating bots
- Session IDs - not relevant for triage

## Practice Examples

### Normal Activity
```
192.168.56.1 - admin [15/Nov/2025:14:30:00 -0500] "GET /search/jobs HTTP/1.1" 200 1024
```
✅ Internal IP  
✅ Admin user  
✅ GET request  
✅ Status 200  
✅ Daytime hours

**Verdict:** Normal

### Suspicious Activity
```
203.0.113.45 - guest [15/Nov/2025:03:47:00 -0500] "POST /admin/user/delete HTTP/1.1" 401 0
```
🚨 External IP  
🚨 Guest attempting admin action  
🚨 POST to delete user  
✅ 401 (failed - good!)  
🚨 3:47 AM

**Verdict:** Attempted unauthorized access - investigate

### Active Attack
```
203.0.113.45 - admin [15/Nov/2025:03:47:15 -0500] "GET /../../../../etc/passwd HTTP/1.1" 200 2048
```
🚨🚨🚨 Path traversal attack  
🚨🚨🚨 Status 200 (succeeded!)

**Verdict:** Active compromise - immediate response needed

## Key Takeaways

1. **Scan, don't read** - Focus on key zones
2. **Status codes tell the story** - 200 vs 401/403/500
3. **Context matters** - Same log can be normal or suspicious based on who/when/where
4. **Pattern recognition** - You'll get faster with practice

## Lab Observations

From `index=_internal sourcetype=splunkd_access`:
- Mostly 200 status codes (healthy system)
- Some 404s (Splunk checking for optional components - normal)
- Mostly GET requests (reading operations)
- Internal IPs (127.0.0.1) = system operations

**Analysis conclusion:** Normal operational activity, no threats detected.

---

**Next:** [Basic Search Commands](../03-fundamentals/basic-searches.md)
