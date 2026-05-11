# Agentic Control Plane

**Description**: This github repository explore how you can use HashiCorp Vault and IdP in an OBO Token Exchange flow.

It aims to solve a few problems:

    1. Bootstrapping of Agent Identities

    2. Traceability, Accountability and Auditability of the Agent

## Architecture Diagram

![OBO token exchange flow with SPIFFE](img/obo-flow-spiffe.png)

## Vault Configuration

The steps below assume Vault Enterprise is already running and unsealed (see
[`vault/vault_setup.sh`](vault/vault_setup.sh) for the container bootstrap),
and that you authenticate via the **Kubernetes auth method** so each chatbot
pod proves its own identity — no AppRole secret-id on disk.

### Prerequisites — export environment

```bash
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=<root-or-admin-token>
```

## Bootstrapping the Agent Identity

These steps establish the chatbot agent's machine identity in Vault. The
agent is identified to Vault via Kubernetes auth (its projected
ServiceAccount token), and Vault then mints short-lived **SPIFFE JWTs**
that downstream services can verify via OIDC.

### 1. Write the chatbot policy

Grants the role permission to read/mint SPIFFE JWTs for `role-chatbot`.

```bash
cat <<EOF | vault policy write chatbot-policy -
# Read and mint SPIFFE JWTs
path "spiffe/role/role-chatbot/*" {
  capabilities = ["read", "list", "update"]
}
EOF
```

### 2. Enable & configure the Kubernetes auth method

The pod's projected ServiceAccount token (rotated
automatically by the kubelet) is what authenticates it to Vault.

```bash
vault auth enable kubernetes

vault write auth/kubernetes/config \
    kubernetes_host="https://kubernetes.default.svc" \
    kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

Bind a role to the chatbot ServiceAccount/namespace:

```bash
vault write auth/kubernetes/role/chatbot \
    bound_service_account_names=chatbot \
    bound_service_account_namespaces=chatbot \
    token_policies=chatbot-policy \
    token_ttl=1h \
    token_max_ttl=4h
```

Capture the mount accessor — the SPIFFE template needs it to read pod
metadata that Kubernetes auth attaches to the entity alias:

```bash
export K8S_ACCESSOR=$(vault auth list -format=json | jq -r '.["kubernetes/"].accessor')
```

### 3. Enable the SPIFFE secrets engine

```bash
vault secrets enable spiffe
```

Configure the trust domain. `issuer` is the public URL where Vault's SPIFFE
JWKS is reachable (e.g. an ngrok tunnel for local demos):

```bash
export ISSUER="https://<your-public-host>/v1/spiffe"

vault write spiffe/config \
    trust_domain=example.org \
    jwt_issuer_url="$ISSUER"
```

### 4. Create the SPIFFE role with pod-aware templating

The `sub` claim is rendered at mint time from the calling pod's alias
metadata, producing `spiffe://example.org/ns/<namespace>/<pod-name>`.

```bash
vault write spiffe/role/role-chatbot \
    template='{"sub": "spiffe://example.org/ns/{{identity.entity.aliases.'"$K8S_ACCESSOR"'.metadata.service_account_namespace}}/{{identity.entity.aliases.'"$K8S_ACCESSOR"'.metadata.pod_name}}"}' \
    ttl=1m \
    use_jti_claim=true
```

Vault publishes the SPIFFE OIDC discovery and JWKS endpoints automatically
once `jwt_issuer_url` is set in step 3 — no extra command is required.
Verify they're live:

```bash
curl -s "$ISSUER/.well-known/openid-configuration" | jq .
curl -s "$ISSUER/.well-known/keys"                 | jq .
```

At this point the agent identity is fully bootstrapped: any pod matching
the Kubernetes role can log in, mint a SPIFFE JWT, and that JWT is
verifiable by anything that trusts `$ISSUER`.

## Inbound JWT Auth (User OBO)

### 5. Configure the inbound JWT auth method

Lets Vault accept user identity tokens from the IdP (IBM Verify) so the
chatbot can perform on-behalf-of token exchange.

```bash
vault auth enable jwt

vault write auth/jwt/config \
    oidc_discovery_url="https://test-demo-2020.verify.ibm.com/oidc/endpoint/default" \
    bound_issuer="https://test-demo-2020.verify.ibm.com/oidc/endpoint/default"

cat <<'EOF' | vault write auth/jwt/role/chatbot-role -
{
  "role_type": "jwt",
  "policies": "chatbot-policy",
  "user_claim": "sub",
  "bound_audiences": "vault",
  "bound_claims_type": "glob",
  "bound_claims": { "sub": "spiffe://example.org/ns/chatbot/*" },
  "token_ttl": "1h",
  "token_max_ttl": "4h"
}
EOF
```

> **Note:** Vault's `bound_subject` is exact-match only and does not support
> wildcards. To match every pod under `ns/chatbot/`, use `bound_claims` with
> `bound_claims_type="glob"` as shown above. The role payload is sent as
> JSON via stdin because the CLI does not parse JSON object values supplied
> inline (e.g. `bound_claims='{...}'` fails with
> `expected a map, got 'string'`). Adjust the pattern (e.g. `chatbot-*`) to
> tighten matching.

## Testing

### 6. Test the JWT login

End-to-end smoke test: mint a SPIFFE JWT, decode its claims, then log in
to the JWT auth method and confirm the returned token carries
`chatbot-policy`.

> **Prerequisite:** `auth/jwt/config` must point at the **issuer that signed
> the JWT you're submitting**. For the actor (SPIFFE) flow this is the
> Vault SPIFFE engine's discovery URL, **not** IBM Verify. Re-run
> `vault write auth/jwt/config oidc_discovery_url="$ISSUER" bound_issuer="$ISSUER"`
> using the same `$ISSUER` from step 3 if you previously pointed it at the
> IdP. (See [Two-mount layout](#two-mount-layout) below for handling both
> issuers simultaneously.)

```bash
# 1. Mint a short-lived SPIFFE JWT
JWT=$(vault write -field=token spiffe/role/role-chatbot/mintjwt audience=vault)

# 2. Decode and inspect the claims (sub, aud, iss must match the role)
python3 -c "import sys,base64,json; p=sys.argv[1].split('.')[1]; \
  p+='='*(-len(p)%4); \
  print(json.dumps(json.loads(base64.urlsafe_b64decode(p)),indent=2))" "$JWT"

# 3. Log in
vault write auth/jwt/login role=chatbot-role jwt="$JWT"
```

A successful login returns:

```
Key                  Value
---                  -----
token                hvs.CAESI...
token_duration       1h
token_policies       ["chatbot-policy" "default"]
token_meta_role      chatbot-role
```

#### Negative tests (optional but recommended)

These should all **fail**, proving the role bindings are doing their job:

```bash
# Wrong audience: spiffe://example.org/ns/chatbot/test-pod with aud:not-vault
BAD_AUD=$(vault write -field=token spiffe/role/role-chatbot/mintjwt audience=not-vault)
vault write auth/jwt/login role=chatbot-role jwt="$BAD_AUD"
# → invalid audience (aud) claim

# Wrong namespace — create a second SPIFFE role outside ns/chatbot
vault write spiffe/role/role-other \
    template='{"sub": "spiffe://example.org/ns/other/test-pod"}' \
    ttl=1m \
    use_jti_claim=true

JWT_OTHER=$(vault write -field=token spiffe/role/role-other/mintjwt audience=vault)

# Inspect the claims to confirm sub is in ns/other/
python3 -c "import sys,base64,json; p=sys.argv[1].split('.')[1]; \
  p+='='*(-len(p)%4); \
  print(json.dumps(json.loads(base64.urlsafe_b64decode(p)),indent=2))" "$JWT_OTHER"

# Attempt login — must be rejected
vault write auth/jwt/login role=chatbot-role jwt="$JWT_OTHER"
# → claim "sub" does not match any associated bound claim values
```

Expected rejection:

```
Error writing data to auth/jwt/login: Error making API request.

URL: PUT http://127.0.0.1:8300/v1/auth/jwt/login
Code: 400. Errors:

* error validating claims: claim "sub" does not match any associated bound claim values
```

This confirms the glob binding `spiffe://example.org/ns/chatbot/*` rejects
any SPIFFE ID outside the `chatbot` namespace, even though the JWT itself
is correctly signed by the same Vault SPIFFE issuer.

#### Common failures

| Error | Fix |
|---|---|
| `failed to verify id token signature` | `auth/jwt/config` points at the wrong issuer (e.g. IBM Verify when you're submitting a SPIFFE JWT). Repoint to the SPIFFE `$ISSUER`. |
| `claim "sub" does not match associated bound claim values` | `sub` doesn't match the glob, **or** `bound_claims_type=glob` is missing. |
| `invalid audience (aud) claim` | `audience=` passed to `mintjwt` ≠ `bound_audiences` on the role. |
| `token is expired` | SPIFFE role TTL is 1m — re-mint. |

<a id="two-mount-layout"></a>
#### Handling both issuers (user OBO + actor SPIFFE)

The OBO flow uses **two** different JWT issuers — IBM Verify for the user
subject token and Vault SPIFFE for the chatbot actor token. A single
`auth/jwt` mount can only point at one OIDC discovery URL, so enable two:

```bash
vault auth enable -path=jwt-user  jwt
vault auth enable -path=jwt-actor jwt

# User subject (IBM Verify)
vault write auth/jwt-user/config \
    oidc_discovery_url="https://test-demo-2020.verify.ibm.com/oidc/endpoint/default" \
    bound_issuer="https://test-demo-2020.verify.ibm.com/oidc/endpoint/default"

# Actor (Vault SPIFFE)
vault write auth/jwt-actor/config \
    oidc_discovery_url="$ISSUER" \
    bound_issuer="$ISSUER"
```

Move `chatbot-role` under `jwt-actor/` and create a separate role under
`jwt-user/` for the user subject token.

