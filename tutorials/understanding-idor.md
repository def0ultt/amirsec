# Deep Dive: Insecure Direct Object References (IDOR & BOLA)

Insecure Direct Object References (IDOR), or Broken Object Level Authorization (BOLA), occur when an application provides direct access to objects based on user-supplied input without properly validating that the requesting user owns or has permission to access that object.

---

## Why IDORs Happen
Most IDORs arise when developers perform **Authentication** (verifying *who* you are), but fail to perform **Authorization** (verifying *what* you can touch).

```text
User A (authenticated) ---> Requests /api/user/1002
                                  │
                                  ▼
                         Backend checks:
                         1. Is User A logged in? (YES)
                         2. Does User A own ID 1002? (MISSED!)
                                  │
                                  ▼
                         Returns Data for User B!
```

---

## High-Yield Testing Checklist

1. **Parameter Substitution**:
   - Swap your `id`, `user_id`, `uuid`, or `account_id` with another test account.
2. **HTTP Verb Tampering**:
   - If `GET /api/documents/50` is forbidden, test `PUT`, `DELETE`, or `PATCH`.
3. **Array / JSON Wrapping**:
   - `{"id": 50}` vs `{"id": [50]}` vs `{"id": {"id": 50}}`.
4. **Header Manipulation**:
   - Inspect custom headers like `X-User-ID`, `X-Account-ID`, `X-Tenant-ID`.
5. **Content-Type Switching**:
   - Switch from `application/json` to `application/xml` or `application/x-www-form-urlencoded`.

---

## Defensive Best Practice
Always resolve the object ID from the authenticated session context rather than trusting client-provided route parameters:

```python
# Insecure:
def get_profile(request, user_id):
    return db.query(User).filter_by(id=user_id).first()

# Secure:
def get_profile(request, user_id):
    current_user = request.auth.current_user
    return db.query(User).filter_by(id=user_id, organization_id=current_user.org_id).first()
```
