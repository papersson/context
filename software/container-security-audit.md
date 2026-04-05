# Container Security Audit Reference

You are auditing a codebase for container security. Follow this document precisely.

## Step 1: Discovery

Locate all container-related configuration in the repository:

- **Dockerfiles**: `Dockerfile`, `Dockerfile.*`, `*.dockerfile`
- **Compose files**: `docker-compose*.yml`, `docker-compose*.yaml`, `compose*.yml`
- **Kubernetes manifests**: any YAML containing `kind: Deployment`, `kind: Pod`, `kind: StatefulSet`, `kind: DaemonSet`, `kind: CronJob`, `kind: Job`
- **Helm charts**: `Chart.yaml` (then inspect sibling `templates/` dir and `values.yaml`)
- **Kustomize**: `kustomization.yaml`, `kustomization.yml`
- **CI/CD pipelines**: `.github/workflows/*.yml`, `.gitlab-ci.yml`, `Jenkinsfile`, `cloudbuild.yaml`, `.circleci/config.yml`

Report the list of discovered files before proceeding. If no container-related files are found, stop and report that.

## Severity Scale

```
CRITICAL  Direct path to container escape or host compromise
HIGH      Significantly weakens container isolation
MEDIUM    Defense-in-depth gap, exploitable under specific conditions
LOW       Best practice violation, minimal direct risk
```

## Step 2: Audit

Run every check below against the relevant discovered files. Report every FAIL.

---

## Dockerfile Checks

Apply these checks to every Dockerfile found.

### D-01: Base image pinned to specific version

Look for `FROM` directives using `latest` tag, no tag, or mutable tags.

```
FAIL: FROM ubuntu
FAIL: FROM node:latest
FAIL: FROM python:3  (major-only tag, still mutable)
PASS: FROM python:3.12.4-slim-bookworm
PASS: FROM python@sha256:abcdef...  (digest pin, strongest)
```

**Severity:** MEDIUM  
**Fix:** Pin to a specific version tag. For maximum reproducibility, pin to a digest.

---

### D-02: Uses minimal base image

Check whether the base image is a full distribution.

```
FAIL: FROM ubuntu
FAIL: FROM debian
FAIL: FROM centos
FAIL: FROM node:20  (full variant)
PASS: FROM python:3.12-slim-bookworm
PASS: FROM node:20-alpine
PASS: FROM gcr.io/distroless/base
PASS: FROM scratch
```

**Severity:** MEDIUM  
**Fix:** Switch to `-slim`, `-alpine`, distroless, or `scratch` variants. Full distributions include hundreds of unnecessary packages that expand the attack surface and require patching.

---

### D-03: Runs as non-root user

Check for a `USER` directive. It must appear after the final `FROM` in multi-stage builds and must not be `root` or `0`.

```
FAIL: No USER directive present
FAIL: USER root
FAIL: USER 0
PASS: USER 1000
PASS: USER appuser
```

**Severity:** HIGH  
**Fix:** Add a non-root user and switch to it:
```dockerfile
RUN addgroup --system app && adduser --system --ingroup app app
USER app
```

---

### D-04: No secrets in Dockerfile

Scan the entire Dockerfile for secrets embedded via `ENV`, `ARG`, `COPY`, or `RUN` commands. Look for:

- `ENV` or `ARG` with names matching: `PASSWORD`, `SECRET`, `TOKEN`, `API_KEY`, `AWS_ACCESS_KEY`, `AWS_SECRET`, `PRIVATE_KEY`, `CREDENTIALS` (case-insensitive)
- `COPY` of files matching: `*.pem`, `*.key`, `.env`, `*credentials*`, `*secret*`, `id_rsa`, `*.p12`, `*.pfx`
- `RUN` commands containing inline passwords, `echo ... > ... password`, or `curl` with credentials in the URL

```
FAIL: ENV DATABASE_PASSWORD=hunter2
FAIL: ARG AWS_SECRET_ACCESS_KEY
FAIL: COPY ./secrets/ /app/secrets/
FAIL: COPY .env /app/.env
PASS: No secrets found in Dockerfile
```

**Severity:** CRITICAL  
**Fix:** Remove all secrets from the Dockerfile. Use runtime secret injection (mounted volumes, k8s Secrets, Vault, cloud provider secret managers).

---

### D-05: COPY preferred over ADD

Check for `ADD` directives that are not extracting a local tar archive.

```
FAIL: ADD https://example.com/file.tar.gz /app/  (remote fetch)
FAIL: ADD . /app  (when COPY would suffice)
PASS: ADD local-archive.tar.gz /app/  (intentional extraction)
PASS: COPY . /app
```

**Severity:** LOW  
**Fix:** Replace `ADD` with `COPY` unless tar extraction is intended. For remote files, use `RUN curl` or `RUN wget` so the downloaded file can be verified and cleaned up in the same layer.

---

### D-06: Multi-stage build used

Check whether the Dockerfile has multiple `FROM` directives (multi-stage) when the image involves compilation, building, or dependency installation that produces build-time-only artifacts.

```
FAIL: Single FROM with build tools (gcc, make, cargo, go build, npm run build, pip install with gcc) remaining in final image
PASS: Multiple FROM stages where build tools are confined to earlier stages
PASS: Single FROM using a runtime-only base (no compilation needed)
```

**Severity:** LOW  
**Fix:** Use multi-stage builds to separate build and runtime environments:
```dockerfile
FROM node:20-alpine AS build
RUN npm ci && npm run build

FROM node:20-alpine
COPY --from=build /app/dist /app/dist
```

---

### D-07: No unnecessary EXPOSE

Check `EXPOSE` directives. Flag ports that seem unrelated to the application's purpose.

```
FAIL: EXPOSE 22  (SSH inside a container is almost always wrong)
FAIL: EXPOSE 3306 / 5432 / 6379  (database ports exposed from an app container)
PASS: EXPOSE 8080  (application port)
```

**Severity:** MEDIUM  
**Fix:** Remove EXPOSE directives for ports the application doesn't serve. SSH access to containers should use `kubectl exec` or equivalent.

---

### D-08: .dockerignore exists and is effective

Check for a `.dockerignore` file in the same directory as the Dockerfile. It should exclude at minimum:

```
FAIL: No .dockerignore file
FAIL: .dockerignore exists but does not exclude: .git, .env, *.pem, *.key, node_modules (where applicable)
PASS: .dockerignore excludes sensitive and unnecessary files
```

**Severity:** MEDIUM  
**Fix:** Create or update `.dockerignore`:
```
.git
.env
*.pem
*.key
*.p12
node_modules
__pycache__
.pytest_cache
```

---

### D-09: Package manager cache cleaned

Check for package install commands that don't clean up caches in the same layer.

```
FAIL: RUN apt-get update && apt-get install -y curl
FAIL: RUN apk add curl  (without --no-cache)
PASS: RUN apt-get update && apt-get install -y --no-install-recommends curl && rm -rf /var/lib/apt/lists/*
PASS: RUN apk add --no-cache curl
```

**Severity:** LOW  
**Fix:** Clean package manager caches in the same `RUN` layer to reduce image size and remove package metadata.

---

### D-10: No SUID/SGID binaries left in image

Check for `RUN` commands that remove SUID/SGID bits, or flag their absence in minimal/distroless images (where this is handled automatically).

```
FAIL: No step to strip SUID/SGID bits in a non-distroless image
PASS: RUN find / -perm /6000 -type f -exec chmod a-s {} + || true
PASS: Using distroless or scratch base image
```

**Severity:** MEDIUM  
**Fix:** Add before the final `USER` directive:
```dockerfile
RUN find / -perm /6000 -type f -exec chmod a-s {} + 2>/dev/null || true
```

---

## Kubernetes Manifest Checks

Apply to every container spec found in Deployments, StatefulSets, DaemonSets, Pods, CronJobs, and Jobs. Also check Helm `values.yaml` and templates for these fields.

### K-01: runAsNonRoot is set

```yaml
# Look in: spec.containers[*].securityContext OR spec.securityContext (pod level)
FAIL: runAsNonRoot missing or false
PASS: runAsNonRoot: true
```

**Severity:** HIGH  
**Fix:** Add to container or pod securityContext:
```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
```

---

### K-02: Privileged mode disabled

```yaml
FAIL: securityContext.privileged: true
FAIL: securityContext.privileged missing (implicit false, but should be explicit)
PASS: securityContext.privileged: false
```

**Severity:** CRITICAL  
**Fix:** Set `privileged: false` explicitly. There is almost never a legitimate reason for a workload container to run privileged.

---

### K-03: Privilege escalation blocked

```yaml
FAIL: securityContext.allowPrivilegeEscalation: true
FAIL: securityContext.allowPrivilegeEscalation missing
PASS: securityContext.allowPrivilegeEscalation: false
```

**Severity:** HIGH  
**Fix:** Set `allowPrivilegeEscalation: false` on every container.

---

### K-04: All capabilities dropped

```yaml
FAIL: securityContext.capabilities missing entirely
FAIL: capabilities.drop does not include "ALL"
PASS:
  capabilities:
    drop: ["ALL"]
```

If specific capabilities are added back via `capabilities.add`, flag each one with its risk:

```
CRITICAL if adding: SYS_ADMIN, SYS_PTRACE, SYS_MODULE, DAC_READ_SEARCH, SYS_RAWIO
HIGH if adding:     NET_ADMIN, NET_RAW, SYS_CHROOT, MAC_ADMIN, MAC_OVERRIDE
MEDIUM if adding:   SETUID, SETGID, FOWNER, CHOWN, MKNOD
LOW if adding:      NET_BIND_SERVICE (common for port <1024)
```

**Fix:** Set `drop: ["ALL"]` and only add back capabilities with documented justification.

---

### K-05: Read-only root filesystem

```yaml
FAIL: securityContext.readOnlyRootFilesystem missing or false
PASS: securityContext.readOnlyRootFilesystem: true
```

If set to true, check that required writable paths use `emptyDir` volume mounts (e.g., `/tmp`, `/var/run`).

**Severity:** MEDIUM  
**Fix:**
```yaml
securityContext:
  readOnlyRootFilesystem: true
volumeMounts:
  - name: tmp
    mountPath: /tmp
volumes:
  - name: tmp
    emptyDir: {}
```

---

### K-06: Resource limits defined

```yaml
FAIL: resources.limits missing
FAIL: resources.limits.memory missing
FAIL: resources.limits.cpu missing
FAIL: resources.requests missing
PASS:
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 256Mi
```

**Severity:** MEDIUM  
**Fix:** Set both requests and limits. Missing memory limits allow a container to OOM-kill the node. Missing CPU limits allow a single container to starve others.

---

### K-07: No hostPath volumes

```yaml
FAIL: volumes[*].hostPath present
PASS: No hostPath volumes
```

Exceptions: logging agents, monitoring agents (must be documented).

**Severity:** HIGH  
**Fix:** Replace hostPath with `emptyDir`, `configMap`, `secret`, or persistent volume claims.

---

### K-08: No host namespace sharing

```yaml
FAIL: spec.hostNetwork: true
FAIL: spec.hostPID: true
FAIL: spec.hostIPC: true
PASS: All three absent or false
```

**Severity:** CRITICAL  
**Fix:** Remove host namespace sharing. `hostNetwork` exposes the container to all host network interfaces and services. `hostPID` allows signaling and inspecting host processes.

---

### K-09: Service account token not auto-mounted (when unused)

```yaml
FAIL: automountServiceAccountToken missing (defaults to true) on a workload that does not call the k8s API
PASS: automountServiceAccountToken: false
PASS: automountServiceAccountToken: true with documented k8s API usage
```

**Severity:** MEDIUM  
**Fix:** Unless the pod needs to talk to the Kubernetes API, set:
```yaml
automountServiceAccountToken: false
```

---

### K-10: No Docker socket mounted

```yaml
FAIL: Any volume or volumeMount referencing /var/run/docker.sock
PASS: No Docker socket mount
```

**Severity:** CRITICAL  
**Fix:** Remove the Docker socket mount. Access to the Docker socket is equivalent to root on the host.

---

### K-11: Seccomp profile applied

```yaml
FAIL: No seccomp annotation or securityContext.seccompProfile
PASS:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
```

**Severity:** MEDIUM  
**Fix:** Apply the runtime default seccomp profile:
```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

---

### K-12: Image pull policy prevents stale images

```yaml
FAIL: imagePullPolicy: Never (in production manifests)
FAIL: imagePullPolicy missing with a :latest tag (defaults to Always, but tag itself is mutable)
PASS: imagePullPolicy: Always with tagged/digest images
PASS: Image pinned to digest (pull policy less relevant)
```

**Severity:** LOW  
**Fix:** Use immutable image references (digests or specific tags) and `imagePullPolicy: Always`.

---

## Compose File Checks

Apply to `docker-compose*.yml` / `compose*.yml` files.

### C-01: No privileged containers

```yaml
FAIL: privileged: true
PASS: privileged absent or false
```

**Severity:** CRITICAL

---

### C-02: No host network mode

```yaml
FAIL: network_mode: "host"
PASS: network_mode absent or set to a named network
```

**Severity:** HIGH

---

### C-03: No Docker socket mounts

```yaml
FAIL: volumes contains /var/run/docker.sock
PASS: No Docker socket mount
```

**Severity:** CRITICAL

---

### C-04: Resource limits set

```yaml
FAIL: No deploy.resources.limits or mem_limit/cpus
PASS:
  deploy:
    resources:
      limits:
        memory: 256M
        cpus: "0.5"
```

**Severity:** MEDIUM

---

### C-05: No secrets in environment

```yaml
FAIL: environment or env_file contains hardcoded secret values
PASS: Secrets referenced via external secret management or are clearly placeholders in a dev-only compose file
```

**Severity:** CRITICAL

---

## CI/CD Pipeline Checks

Apply to any pipeline file that builds or deploys container images.

### P-01: Image scanning step present

```
FAIL: No step running a vulnerability scanner (trivy, grype, snyk, clair, docker scout)
PASS: Pipeline includes an image scan step before push/deploy
```

**Severity:** MEDIUM  
**Fix:** Add a scanning step:
```yaml
- name: Scan image
  run: trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE
```

---

### P-02: No --privileged in build/run

```
FAIL: docker run --privileged or docker build with --privileged
PASS: No privileged flags in pipeline
```

**Severity:** HIGH

---

### P-03: Image pushed to authenticated registry

```
FAIL: Image pushed over HTTP or to unauthenticated registry
FAIL: Registry credentials hardcoded in pipeline
PASS: Push to authenticated registry using CI/CD secret management
```

**Severity:** HIGH

---

### P-04: Build secrets not baked into image

```
FAIL: docker build --build-arg SECRET_KEY=...
PASS: Using --secret flag with BuildKit, or no secrets needed at build time
```

**Severity:** CRITICAL

---

## Step 3: Report

For each FAIL, report in this format:

```
### [CHECK-ID]: Check Name
- **Severity:** CRITICAL | HIGH | MEDIUM | LOW
- **File:** path/to/file:line_number
- **Current:** what was found (quote the offending line)
- **Required:** what it should be
- **Fix:** concrete code change (diff or replacement)
```

Group findings by severity (CRITICAL first). End with a summary count:

```
## Summary
- CRITICAL: N
- HIGH: N
- MEDIUM: N
- LOW: N
- Total checks: N passed, N failed
```
