# Q14–Q15. Bootstrap Tokens — Create, List, and Delete

## Q14 Task
Generate a new bootstrap token valid for 24 hours with description "Worker node join token".

## Q15 Task
List all kubeadm tokens with expiration times. Delete any expired tokens.

---

## Q14: Create a Bootstrap Token

```bash
kubeadm token create \
  --ttl 24h \
  --description "Worker node join token" \
  --print-join-command
```

Output:
```
kubeadm join 192.168.1.100:6443 --token abc123.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:<hash>
```

To create a token without printing the join command:
```bash
kubeadm token create --ttl 24h --description "Worker node join token"
# abc123.0123456789abcdef
```

---

## Q15: List All Tokens and Delete Expired Ones

### List tokens with expiry

```bash
kubeadm token list
```

```
TOKEN                     TTL       EXPIRES                USAGES                   DESCRIPTION
abc123.0123456789abcdef   23h       2024-01-17T10:00:00Z   authentication,signing   Worker node join token
xyz789.abcdef0123456789   <invalid> 2024-01-16T08:00:00Z   authentication,signing   old token
```

> Tokens with `<invalid>` TTL are expired.

---

### Delete a Specific Expired Token

```bash
kubeadm token delete xyz789.abcdef0123456789
```

### Delete All Expired Tokens (Automated)

```bash
# List expired tokens and delete each
kubeadm token list -o json | \
  python3 -c "
import json, sys, subprocess
from datetime import datetime, timezone

tokens = json.load(sys.stdin)
now = datetime.now(timezone.utc)

for t in tokens.get('items', []):
    expiry = t.get('expirationTimestamp')
    if expiry:
        exp = datetime.fromisoformat(expiry.replace('Z', '+00:00'))
        if exp < now:
            name = t['metadata']['name']
            print(f'Deleting expired token: {name}')
            subprocess.run(['kubeadm', 'token', 'delete', name])
"
```

Or simpler using bash:
```bash
# Show only expired (past TTL)
kubeadm token list | grep '<invalid>'
# Then delete manually:
kubeadm token delete <token>
```

---

## Token Reference

| Flag | Description |
|---|---|
| `--ttl` | Duration before expiry (e.g. `24h`, `2h`, `0` = never expire) |
| `--description` | Human-readable label |
| `--usages` | What the token can be used for (default: `authentication,signing`) |
| `--groups` | Groups the token authenticates as (default: `system:bootstrappers:kubeadm:default-node-token`) |
