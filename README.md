# uds-capability-gitlab-runner

Platform One Gitlab Runner deployed via flux

## Pre-req

- Minimum compute requirements for single node deployment are at LEAST 64 GB RAM and 32 virtual CPU threads (aws `m6i.8xlarge` instance type should do)
- k3d installed on machine

## Deploy

### Use zarf to login to the needed registries i.e. registry1.dso.mil and ghcr.io

```bash
# Download Zarf
make build/zarf

# Login to the registry
set +o history

# registry1.dso.mil (To access registry1 images needed during build time)
export REGISTRY1_USERNAME="YOUR-USERNAME-HERE"
export REGISTRY1_TOKEN="YOUR-TOKEN-HERE"
echo $REGISTRY1_TOKEN | build/zarf tools registry login registry1.dso.mil --username $REGISTRY1_USERNAME --password-stdin

# ghcr.io (To access oci packages needed)
export GH_USERNAME="YOUR-USERNAME-HERE"
export GH_TOKEN="YOUR-TOKEN-HERE"
echo $GH_TOKEN | build/zarf tools registry login ghcr.io --username $GH_USERNAME --password-stdin

set -o history
```

### Deploy Everything

```bash
# This will destroy and create a compatible k3d cluster then it will run make build/all and make deploy/all. Follow the breadcrumbs in the Makefile to see what and how its doing it.
make cluster/full
```

## Import Zarf Skeleton

Below is an example of how to import this projects zarf skeleton into your zarf.yaml. The [uds-package-sofware-factory](https://github.com/defenseunicorns/uds-package-software-factory.git) does this with a subset of the uds-capability projects.

```yaml
components:
  - name: values
    required: true
    files:
      - source: <path-to-the-values-you-want-to-use>
        target: values-gitlab-runner.yaml
  - name: gitlab-runner
    required: true
    import:
      name: gitlab-runner
      url: oci://ghcr.io/defenseunicorns/uds-capability/gitlab-runner:0.0.3-skeleton
```

## Prerequisites

### GitLab-Runner Capability

The Gitlab-Runner Capability expects the pieces listed below to exist in the cluster before being deployed.

#### General

- Create `gitlab-runner-sandbox` namespace
- Label `gitlab-runner-sandbox` namespace with `istio-injection: enabled` & `zarf.dev/agent: ignore`
- Create an `rbac` file for the `gitlab-runner` service account
- Replace zarf-created `ImagePullSecret` - See below

#### ImagePullSecret

By default Zarf will create an `ImagePullSecret` in any new namespace in the cluster called `private-registry`. Since
we have specified that the `gitlab-runner-sandbox` namespace will not be using the zarf registry that secret must be deleted.
However, the CI job pods will still require one that has the required credentials for where you expect your users to want to pull
CI images from.

- Delete the `secret` called `private-registry` in the `gitlab-runner-sandbox` namespace
- Create an `ImagePullSecret` type `secret` called `private-registry` in the `gitlab-runner-sandbox` with the credentials required
  - Example using kubectl:

```bash
kubectl create secret generic private-registry --from-file=$(printf ~/.docker/config.json) --type=kubernetes.io/dockerconfigjson -n gitlab-runner-sandbox
```

#### RBAC file

- The `rbac.yaml` should create a `ClusterRole` with the following values:

```yaml
rules:
  - apiGroups: [""]
    resources: ["configmaps", "pods", "pods/attach", "secrets", "services"]
    verbs: ["get", "list", "watch", "create", "patch", "update", "delete"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create", "patch", "delete"]
```

- The `ClusterRole` should then be bound using a `RoleBinding` in the `gitlab-runner-sandbox` namespace to the service account that `gitlab-runner` uses
example:

```yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: gitlab-runner-sandbox
  namespace: gitlab-runner-sandbox
subjects:
- kind: ServiceAccount
  name: default
  namespace: gitlab-runner
roleRef:
  kind: ClusterRole
  name: gitlab-runner-sandbox
```
