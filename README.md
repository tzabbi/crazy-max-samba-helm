# crazy-max-samba Helm Chart

This Helm chart deploys a Samba server based on the Docker image from [crazy-max/docker-samba](https://github.com/crazy-max/docker-samba).

## Project links

- OCI chart: `oci://ghcr.io/tzabbi/charts/crazy-max-samba`
- Docker image: `ghcr.io/crazy-max/samba`
- Upstream container image project: <https://github.com/crazy-max/docker-samba>
- Upstream Samba configuration documentation: <https://github.com/crazy-max/docker-samba?tab=readme-ov-file#configuration>

## OCI publishing

This chart is published as an OCI artifact to GitHub Container Registry (GHCR).

On every merge to `main`, the GitHub Actions workflow will:

- lint the chart
- bump the patch version in `Chart.yaml`
- package the chart
- push the chart to GHCR as an OCI artifact
- create a Git tag and GitHub Release for the new chart version

The published OCI reference is:

`oci://ghcr.io/tzabbi/charts/crazy-max-samba`

## What this chart creates

This chart creates the following resources:

- a `StatefulSet` for the Samba server
- a PVC for `/data`
- optionally one PVC per share defined in `share`
- optionally a Kubernetes `Service` when `service.enabled=true`
- a `ConfigMap` containing the Samba configuration

## Requirements

- a Kubernetes cluster
- Helm 3.8 or newer for OCI-based installs
- a working `StorageClass` or suitable statically provisioned volumes
- a free `445/TCP` port on the target node if you use the default configuration with `hostPort: 445`

## Quick start

### 1. Create a namespace

```bash
kubectl create namespace samba
```

### 2. Create password secret(s)

By default, the chart expects one Secret per user named `samba-user-<user>-password` with the key `password`.

Example for the user `backup`:

```bash
kubectl -n samba create secret generic samba-user-backup-password \
  --from-literal=password='MySecurePassword'
```

### 3. Create your own values file

Example `my-values.yaml`:

```yaml
service:
  enabled: true

envs:
  - TZ: "Europe/Berlin"
  - SAMBA_LOG_LEVEL: "1"

auth:
  - user: backup
    password: ${BACKUP_USER_PASSWORD}
    uid: 1001
    gid: 1001
    group: samba_users

share:
  - name: backup
    path: /mnt/backup
    readonly: "no"
    guestok: "no"
    validusers: backup
    size: 20Gi
    storageClassName: standard

global:
  - "map to guest = never"
```

## Installation

### Install from the OCI registry

If the GHCR package is private, authenticate first:

```bash
helm registry login ghcr.io -u <github-username>
```

Install the chart from GHCR:

```bash
helm upgrade --install samba oci://ghcr.io/tzabbi/charts/crazy-max-samba \
  --version <chart-version> \
  --namespace samba \
  --create-namespace \
  -f my-values.yaml
```

### Pull the chart locally from OCI

```bash
helm pull oci://ghcr.io/tzabbi/charts/crazy-max-samba \
  --version <chart-version> \
  --untar
```

### Local development install

If you are working directly in this repository, you can still install the local chart without using OCI:

```bash
helm upgrade --install samba . \
  --namespace samba \
  --create-namespace \
  -f my-values.yaml
```

## Important configuration notes

### Users and passwords

- Define Samba users in `auth`.
- The group should be `samba_users`.
- `gid` should be `1001` because the chart uses that ID for share permissions.
- The `password` field typically references an environment variable such as `${BACKUP_USER_PASSWORD}`.
- The actual password is then read from the Kubernetes Secret `samba-user-backup-password`.

### Shares

Each item in `share`:

- creates a Samba share
- mounts the path defined in `path` into the container
- creates a dedicated PVC with the configured `size`
- can optionally use its own `storageClassName`

### Network access

By default, the chart uses `hostPort: 445`. That means:

- the pod binds port `445` directly on the Kubernetes node
- port `445` must not already be in use on that node
- clients can access the share through the node IP address

If you also want an internal Kubernetes Service, enable:

```yaml
service:
  enabled: true
```

## Verify the deployment

```bash
kubectl -n samba get pods,pvc,svc
```

You can also run the Helm test:

```bash
helm test samba -n samba
```

## Access the Samba server

### List available shares

Example with `smbclient`:

```bash
smbclient -L //<NODE-IP> -U backup
```

### Mount a share

Linux example:

```bash
sudo mount -t cifs //<NODE-IP>/backup /mnt/backup \
  -o username=backup
```

Replace `<NODE-IP>` with the IP address of the Kubernetes node running the pod.

## Upgrade

Upgrade from the OCI registry with:

```bash
helm upgrade samba oci://ghcr.io/tzabbi/charts/crazy-max-samba \
  --version <chart-version> \
  --namespace samba \
  -f my-values.yaml
```

For local development, you can still use:

```bash
helm upgrade samba . \
  --namespace samba \
  -f my-values.yaml
```

## Uninstall

```bash
helm uninstall samba -n samba
```

Note: depending on your cluster and storage setup, PersistentVolumeClaims may remain and need to be deleted separately if required.

## Useful values

The most important settings are:

- `image.repository`: defaults to `ghcr.io/crazy-max/samba`
- `image.tag`: optionally overrides the image tag
- `service.enabled`: creates an internal ClusterIP Service
- `storage.size`: size of the PVC for `/data`
- `storage.storageClassName`: StorageClass for `/data`
- `auth`: user definitions
- `share`: Samba shares including PVC size
- `envs`: additional environment variables for the container
- `statefulSet.resources`: CPU and memory requests/limits
- `statefulSet.nodeSelector`, `statefulSet.affinity`, `statefulSet.tolerations`: pod scheduling

## Additional notes

For advanced Samba-specific configuration options, the upstream container project documentation is the primary reference:

<https://github.com/crazy-max/docker-samba>
