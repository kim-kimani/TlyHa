# Cloud Metadata and SSRF to Cloud

The single most valuable thing an SSRF can reach is the cloud instance metadata service, because it hands out the machine's identity credentials. One SSRF plus one over-permissioned role equals control of cloud resources. This file is the bridge from a web bug to cloud compromise; the deep post-access cloud attacks live in the sibling `AWS/` track.

## The mental model

Every cloud VM/container can ask a link-local metadata endpoint "who am I and what are my credentials." That endpoint is reachable from the instance itself at a fixed internal IP. If you can make the server send a request there on your behalf (SSRF, see `../web/SSRF.md`), you read temporary credentials scoped to the instance's role, then use them against the cloud API. The blast radius is whatever that role can do, which is routinely far more than it should be.

## Where it hides / tell-tales

- **Any SSRF on a cloud-hosted app**: the metadata service is the first target after you confirm server-side fetch. Covered mechanically in `../web/SSRF.md`; this file is about what to do once you reach it.
- **Over-permissioned instance roles**: the credentials you steal carry the instance's IAM permissions. Enumerate them; roles are frequently granted `*` or broad policies "to make it work."
- **IMDSv1 still enabled**: AWS IMDSv1 is a simple GET, trivially reachable by SSRF. IMDSv2 requires a PUT to get a token then a header, which blocks naive GET-only SSRF but not SSRF that controls method and headers.
- **Container and orchestration metadata**: ECS task role endpoint, Kubernetes service account tokens mounted in the pod, GCP/Azure metadata. Containers add more credential sources.
- **SSRF-adjacent features**: webhooks, URL fetchers, PDF/screenshot generators, XXE, and `jku`/`jwk` in JWTs, all of which can reach the metadata IP.

## Real material

AWS credential theft via metadata:

```
# IMDSv1 (GET only):
http://169.254.169.254/latest/meta-data/iam/security-credentials/
http://169.254.169.254/latest/meta-data/iam/security-credentials/<ROLE-NAME>
-> returns temporary AccessKeyId, SecretAccessKey, Token

# IMDSv2 (needs a token first; requires method+header control in the SSRF):
PUT http://169.254.169.254/latest/api/token
    Header: X-aws-ec2-metadata-token-ttl-seconds: 21600
GET http://169.254.169.254/latest/meta-data/iam/security-credentials/<ROLE>
    Header: X-aws-ec2-metadata-token: <token>

# ECS task role (containers):
http://169.254.170.2${AWS_CONTAINER_CREDENTIALS_RELATIVE_URI}
```

GCP and Azure (note the required headers, which the SSRF must be able to set):

```
# GCP:
http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
    Header: Metadata-Flavor: Google
# Azure:
http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/
    Header: Metadata: true
```

Using stolen AWS creds (set them, then enumerate what you have):

```
export AWS_ACCESS_KEY_ID=...  AWS_SECRET_ACCESS_KEY=...  AWS_SESSION_TOKEN=...
aws sts get-caller-identity          # who am I
aws iam list-attached-role-policies --role-name <ROLE>   # what can I do (if allowed)
# then enumerate reachable services: s3 ls, ec2 describe-instances, etc.
```

Kubernetes service account token inside a pod (often reachable via SSRF/LFI to the mounted path):

```
/var/run/secrets/kubernetes.io/serviceaccount/token    # JWT for the API server
/var/run/secrets/kubernetes.io/serviceaccount/namespace
# use against the API server (often https://kubernetes.default.svc)
```

## Beyond the basics

- **IMDSv2 is a speed bump, not a wall**: it defeats GET-only SSRF but not SSRF that can issue a PUT and add a header (full webhooks, gopher-based SSRF, some XXE). Always assess whether your SSRF controls method and headers before declaring metadata unreachable. Also note the hop-limit defense (IMDSv2 default TTL of 1 means the packet should not traverse a container boundary), which blocks some container-to-metadata paths.
- **Enumerate the role before acting**: `sts get-caller-identity` is safe and tells you the account and role. Then determine permissions cautiously (read-only describe/list calls). The severity is defined by what the role can do; a role that can read S3 is a data breach, one that can `iam:PassRole` or create users is full account takeover. Do not perform destructive or privilege-changing actions on a live program; prove access minimally.
- **Credentials are temporary but long enough**: metadata creds have a TTL (often hours) and auto-rotate, but you can re-pull them via the same SSRF. The `Token`/session token is required for AWS calls; without it the keys fail.
- **Container credential sources multiply the surface**: ECS task metadata, EKS IRSA tokens, mounted Kubernetes service account tokens, and Docker socket exposure. A single SSRF/LFI inside a pod can reach the service account token and talk to the cluster API, which may let you read secrets, create pods, or escalate across the cluster.
- **Beyond credentials, metadata leaks more**: user-data scripts (often contain secrets and bootstrap credentials, `http://169.254.169.254/latest/user-data`), SSH keys, instance tags, and network config, all useful for pivoting.
- **This is the handoff to the AWS track**: once you hold cloud credentials, the post-access game (enumeration, privilege escalation via IAM, persistence, lateral movement, data exfiltration) is a deep topic covered in the sibling `AWS/` modules. SSRF-to-cloud is how you get the first credential; that track is what you do with it.

## What to watch for

- A reachable `169.254.169.254` is only critical if it returns usable credentials and the role has meaningful permissions; confirm `get-caller-identity` works and assess scope before claiming impact.
- IMDSv2, egress firewalls, and hop-limit may block the obvious path; try method/header-controlling SSRF and container credential paths before concluding it is dead.
- Stolen credentials access a real cloud account; use read-only identity/enumeration calls for proof, never destructive or persistence-creating actions, and get explicit authorization for anything beyond minimal confirmation.
- Credentials you retrieve must be treated as sensitive and reported for rotation; do not retain or reuse them.

## Hunting checklist

- On any cloud-hosted SSRF, target the metadata IP first; determine whether you control method and headers (for IMDSv2/GCP/Azure).
- Pull credentials and run `sts get-caller-identity` (or the cloud equivalent) to confirm and identify the role.
- Cautiously enumerate the role's permissions with read-only calls; map the blast radius.
- Check for container credential sources (ECS endpoint, mounted K8s service account token) and user-data secrets.
- Stop at minimal proof; hand off to the `AWS/` track for authorized deeper post-access work.
