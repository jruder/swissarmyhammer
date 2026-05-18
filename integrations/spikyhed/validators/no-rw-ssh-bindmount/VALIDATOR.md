---
name: no-rw-ssh-bindmount
description: Enforce SpikyHed Hard Rule §15 — never bind-mount ~/.ssh (or any directory containing SSH keys) read-write into a container. The 2026-04-09 incident locked the host user out of VM3 via chown propagation.
metadata:
  version: "1.0.0"
trigger: PreToolUse
match:
  files:
    - "**/Dockerfile*"
    - "**/docker-compose*.{yml,yaml}"
    - "**/compose*.{yml,yaml}"
    - "infra/k8s/**/*.{yml,yaml}"
    - "spikyhed-forge/**/*.{yml,yaml,sh,Dockerfile}"
    - "scripts/**/*.sh"
    - "tools/**/*.sh"
tags:
  - spikyhed-constitution
  - container-safety
  - incident-2026-04-09
severity: error
timeout: 30
---

# No RW Bind Mount of ~/.ssh into Containers

SpikyHed Hard Rule §15 (specific case of §14.5): **Never bind-mount `~/.ssh`
or any directory containing SSH keys read-write into a container.**

`chown`/`chmod` inside the container propagates through the rw bind mount to
host inodes and can lock the ubuntu user out of SSH. On 2026-04-09 this cost
3 hours, a full VM3 termination, and a rebuild.

## What to flag

### Docker Compose / Compose v2

A `volumes:` entry that targets `.ssh` as source and is NOT explicitly marked
read-only:

```yaml
# ❌ default (rw) bind mount of .ssh
- ~/.ssh:/root/.ssh
- /home/ubuntu/.ssh:/root/.ssh
- ${HOME}/.ssh:/root/.ssh
- $HOME/.ssh:/home/app/.ssh

# ❌ explicit rw
- ~/.ssh:/root/.ssh:rw

# ❌ long-form without read_only: true
- type: bind
  source: ~/.ssh
  target: /root/.ssh
```

### Dockerfile

```dockerfile
# ❌ mount from BuildKit secrets is fine, but a COPY ~/.ssh from a build arg
# pointing at the host's .ssh dir is also flagged.
COPY --from=host-ssh /root/.ssh /root/.ssh
```

### docker run / docker create in shell scripts

```bash
# ❌ rw bind mount
docker run -v ~/.ssh:/root/.ssh ...
docker run --mount type=bind,source=$HOME/.ssh,target=/root/.ssh ...

# ❌ no :ro suffix
docker run -v /home/ubuntu/.ssh:/root/.ssh
```

### Kubernetes manifests

```yaml
# ❌ hostPath volume mounting .ssh with readOnly: false (or missing readOnly)
volumes:
- name: ssh
  hostPath:
    path: /home/ubuntu/.ssh
volumeMounts:
- name: ssh
  mountPath: /root/.ssh
  # missing readOnly: true
```

## What NOT to flag

- **Single-file `:ro` mounts from a dedicated dir OUTSIDE `~/.ssh`**:
  ```yaml
  - /home/ubuntu/.ssh-keys/spikyhed_vm_admin:/root/.ssh/id_rsa:ro
  ```
  This is the documented safe pattern from `docs/forge/VM3-RUNBOOK.md`.

- **BuildKit secrets**:
  ```dockerfile
  RUN --mount=type=secret,id=ssh_key ssh -i /run/secrets/ssh_key ...
  ```
  Secrets are not bind mounts.

- **Mounts of dirs that aren't `.ssh`**: only `.ssh` and dirs whose names
  literally contain `ssh-key` or `ssh_key` are in scope.

## Why error, not warning

This rule exists because of a specific, documented incident. The cost was
hours of recovery. There is no acceptable rationale for an rw bind mount of
`~/.ssh`, so the rule blocks rather than warns.

## Remediation message

> SpikyHed Hard Rule §15 forbids rw bind-mounting `~/.ssh` (or any dir with
> SSH keys) into a container. The chown propagation can lock the host user
> out of SSH (2026-04-09 incident, VM3 termination).
>
> Use one of the two safe patterns:
>
> 1. **Copy a key via Dockerfile** so the image carries it (image stays
>    local):
>    ```dockerfile
>    COPY ssh_key /root/.ssh/id_rsa
>    RUN chmod 0600 /root/.ssh/id_rsa
>    ```
>
> 2. **Single-file read-only bind mount from outside `~/.ssh`**:
>    ```yaml
>    - /home/ubuntu/.ssh-keys/<keyname>:/root/.ssh/id_rsa:ro
>    ```
>
> Reference: `docs/forge/VM3-RUNBOOK.md`, working example in
> `spikyhed-forge/docker-compose.yml`.
