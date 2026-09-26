---
description: "Security architecture for system designers: threat modeling, authentication and sessions, authorization models, service identity, secrets, encryption, and abuse."
---

# 07 · Security Architecture

Security is a property of the whole design, not a component you add at the end. Most
serious breaches come not from exotic cryptographic attacks but from ordinary design
gaps: an internal service trusted without authentication, an authorization check missing
on one endpoint, a secret committed to a repository, a storage bucket left public. This
lesson covers the architectural decisions that prevent those.

## Start with a threat model

Before choosing controls, ask four questions (a lightweight version of common
threat-modeling practice):

1. **What are we building?** Draw the data flow: clients, services, stores, third parties,
   and the **trust boundaries** between them (internet → edge, edge → internal, internal →
   database, you → third-party API).
2. **What can go wrong?** At each boundary, consider categories such as those in the STRIDE
   mnemonic: Spoofing identity, Tampering with data, Repudiation, Information disclosure,
   Denial of service, Elevation of privilege.
3. **What are we doing about it?** A control for each credible threat.
4. **Did we do a good job?** Review, test, revisit when the design changes.

## Authentication (who are you?)

**Users**

- Prefer delegating to a mature identity provider or well-reviewed library over
  hand-rolled password handling. If you store passwords, use a slow, salted password
  hashing function designed for the purpose (e.g. Argon2, scrypt, or bcrypt) — never a fast
  general-purpose hash.
- Support multi-factor authentication, ideally phishing-resistant forms such as
  passkeys/WebAuthn.
- **Sessions**: opaque session IDs in `HttpOnly`, `Secure`, `SameSite` cookies are easy to
  revoke (delete the server-side record). Self-contained signed tokens (JWTs) avoid a
  lookup but cannot be revoked before expiry without extra machinery — keep them short-lived
  and pair them with refresh tokens.
- Third-party login and API delegation use OAuth 2.0 / OpenID Connect flows; use the flows
  current guidance recommends (for example, authorization code with PKCE for public
  clients).

**Services**

Internal services should authenticate each other too — "it's on the internal network" is
not an identity. Common approaches: mutual TLS with short-lived certificates issued by an
internal authority (often automated by a service mesh), or signed service tokens.

## Authorization (what may you do?)

Authentication without authorization is the source of a very common vulnerability class:
**broken object-level authorization** — `GET /invoices/123` returns invoice 123 to any
logged-in user because the code checked *that* they were logged in, not *whose* invoice it
was.

Models:

- **RBAC** (role-based): users have roles; roles have permissions. Simple; coarse.
- **ABAC** (attribute-based): policies over attributes of user, resource, and context
  ("managers can approve expenses under $5,000 in their own department").
- **ReBAC** (relationship-based): permissions derived from relationships in a graph ("can
  view a document if member of a folder that contains it") — the model popularized by
  Google's published Zanzibar paper and implemented by several open-source systems.

Architectural principles:

- **Deny by default**; check on every request, on the server, for the specific object.
- **Centralize policy logic** (a library or policy service) instead of scattering ad-hoc
  checks — scattered checks are where one gets forgotten.
- Include the **tenant ID in every data access path** of a multi-tenant system, enforced at
  the data layer (e.g. row-level security or a mandatory query scope), not just in UI code.

## Worked example: object-level authorization done centrally

```python
# authz.py — a central policy check that every handler must call
from dataclasses import dataclass

@dataclass
class User:
    id: str
    tenant: str
    roles: frozenset

INVOICES = {
    "inv-1": {"tenant": "acme",   "owner": "u1", "amount": 1200},
    "inv-2": {"tenant": "globex", "owner": "u9", "amount": 800},
}

class Forbidden(Exception):
    pass

def authorize(user, action, resource):
    if resource["tenant"] != user.tenant:              # hard tenant isolation first
        raise Forbidden("cross-tenant access")
    if action == "read" and ("finance" in user.roles or resource["owner"] == user.id):
        return
    if action == "approve" and "finance" in user.roles and resource["owner"] != user.id:
        return                                          # no self-approval
    raise Forbidden(f"{action} denied")

def get_invoice(user, invoice_id):
    inv = INVOICES.get(invoice_id)
    if inv is None:
        raise KeyError("not found")
    authorize(user, "read", inv)                       # object-level check
    return inv

alice = User("u1", "acme", frozenset({"employee"}))
fin = User("u2", "acme", frozenset({"finance"}))
for who, inv, act in [(alice, "inv-1", "read"), (alice, "inv-2", "read"),
                      (fin, "inv-1", "approve"), (alice, "inv-1", "approve")]:
    try:
        authorize(who, act, INVOICES[inv])
        print(f"{who.id} {act} {inv}: allowed")
    except Forbidden as e:
        print(f"{who.id} {act} {inv}: DENIED ({e})")
```

In an API, a cross-tenant or unauthorized lookup should usually return `404` rather than
`403`, so that attackers cannot probe which IDs exist in other tenants.

## Secrets

- Never in source code, images, or client apps. Use a secrets manager; inject at runtime;
  grant each service access only to its own secrets.
- Prefer **short-lived, automatically issued credentials** (workload identity, dynamic
  database credentials) over long-lived static keys.
- Rotate, and design so rotation needs no downtime (accept old and new during a window).
- Scan repositories and logs for leaked secrets; redact secrets from logs.

## Encryption

- **In transit**: TLS everywhere, including internal traffic.
- **At rest**: storage-level encryption is widely available and often on by default in
  managed services; for sensitive fields, consider application-level encryption with keys
  held in a key management service, and **envelope encryption** (data encrypted with a data
  key, which is encrypted by a master key that never leaves the KMS) so that rotation and
  access control happen at the key level.
- Encryption does not replace authorization: an attacker who can call your API as a user
  gets decrypted data.

## Defense in depth at the edge

- Edge protections: TLS termination, a web application firewall for common attack patterns,
  DDoS absorption (often by the CDN), rate limits (Level 2, lesson 6), bot detection for
  abuse-prone flows (signup, login, checkout).
- Input validation and output encoding in services; parameterized queries to prevent
  injection.
- Least-privilege network paths: databases reachable only from the services that need them.
- Audit logs for sensitive actions (who did what, when, from where), stored so that
  attackers who compromise a service cannot erase them.

## How It Actually Works

The **principle of least privilege** is the architectural idea behind most of these
controls, and it works by limiting **blast radius**. Compromise is assumed to happen
somewhere eventually: a vulnerable dependency, a phished credential. What decides whether
that becomes a minor incident or a catastrophe is what the compromised piece can reach. A
service with credentials only for its own tables, a network path only to its own
dependencies, and short-lived tokens that expire in minutes gives an attacker little to
work with. A service with a shared admin database password and flat network access gives
them everything.

Mutual TLS illustrates how identity is built into the transport: both sides present
certificates signed by a trusted internal authority; each verifies the other's certificate
chain and the identity encoded in it (for example, a service name) before any application
data flows. Because certificates are short-lived and issued automatically, a stolen
certificate is useful only briefly, and policy can say "only the orders service may call
the payments service" in terms of verified identities rather than IP addresses.

## Common mistakes

- **Trusting the internal network** — no service-to-service authentication.
- **Missing object-level authorization checks.**
- **Long-lived shared credentials**, secrets in code or environment dumps.
- **JWTs with long expiry and no revocation strategy.**
- **Tenant isolation only in the UI.**
- **Security review only at the end**, when changing the design is expensive.

## Exercise

1. Extend `authz.py` with a "support agent" role that can read any invoice in any tenant
   *only* with a ticket ID recorded in an audit log. Where should that audit entry be
   written so it cannot be silently deleted?
2. Draw the trust boundaries for the payment system from lesson 4 and list one STRIDE threat
   and its control for each boundary.
3. Design secret handling for a service that needs a third-party API key and a database
   password: storage, delivery to the process, rotation, and what happens if one leaks.
