# Kubernetes manifest template

A reusable baseline for deploying a containerized HTTP application to the
personal K3s cluster through Argo CD. Copy the `k8s/` directory into an
application repository, replace the example values, and point an Argo CD
`Application` at `path: k8s`.

The committed baseline is valid Kustomize configuration. Files under
`k8s/optional/` are examples and are **not** deployed unless explicitly added to
`k8s/kustomization.yaml`.

## Required Kubernetes files

The following files make up the standard browser-facing application baseline:

| File | Purpose | Required when |
| --- | --- | --- |
| `k8s/kustomization.yaml` | Declares the resources Argo CD renders and applies. | Always for this repository convention. |
| `k8s/namespace.yaml` | Creates an isolated namespace for the application. | Recommended for every application. It may be omitted when Argo CD owns namespace creation exclusively. |
| `k8s/deployment.yaml` | Defines the container image, replicas, ports, probes, resources, and security context. | Required for a stateless long-running application. |
| `k8s/service.yaml` | Gives the pods a stable in-cluster address and port. | Required when another workload or an Ingress must reach the application. |
| `k8s/ingress.yaml` | Routes a hostname through Traefik to the Service. | Required for a browser-facing HTTP application; remove it for cluster-internal workloads. |

Kubernetes does not require these exact filenames. The resource `kind` and the
references between resources are what matter. This layout is the convention used
by the homelab repositories.

## Values that must be changed

Before deploying, replace every use of the example values:

| Example | Replace with |
| --- | --- |
| `example-app` | The application and namespace name. |
| `ghcr.io/example/example-app:0.1.0` | The real registry image, preferably using an immutable tag or digest. |
| `8080` | The port the container actually listens on. |
| `example-app.home.arpa` | The internal DNS hostname, or the future public hostname. |
| `/` probe paths | Real health endpoints when the application provides them. |
| CPU and memory values | Limits observed to be appropriate for the workload. |

If the container must write to disk, either provide a writable `emptyDir` or
persistent volume mount, or deliberately adjust `readOnlyRootFilesystem`. Do not
disable the security settings without understanding why the application needs it.

## Optional Kubernetes files

Templates in `k8s/optional/` are starting points. Copy or edit the appropriate
file and add it to the `resources` list in `k8s/kustomization.yaml`.

| File or resource | Add it when |
| --- | --- |
| `configmap.yaml` | The application needs non-sensitive runtime configuration. Reference it with `envFrom`, individual `env` entries, or a volume mount. |
| `secret.example.yaml` | The application needs credentials or tokens. This file documents shape only; never commit real values. Use a secret manager, encrypted secret workflow, or create the Secret out of band. |
| `persistentvolumeclaim.yaml` | Data must survive pod replacement. K3s includes the `local-path` StorageClass, but its data remains tied to the node. |
| `serviceaccount.yaml` | The pod needs an explicit workload identity. Add Role/RoleBinding permissions separately and keep them minimal. |
| `horizontalpodautoscaler.yaml` | Replicas should scale from CPU or memory metrics. A metrics provider such as metrics-server must be installed. |
| `networkpolicy.yaml` | The cluster CNI enforces NetworkPolicy and traffic should be restricted. Test policy selectors before enabling it. |
| `poddisruptionbudget.yaml` | A multi-replica application needs availability protection during voluntary disruption. It offers little benefit to a single-replica workload. |
| `Job` or `CronJob` | Work runs once or on a schedule rather than continuously. |
| `StatefulSet` and headless `Service` | Pods need stable identities or per-replica persistent storage. Use instead of a Deployment where appropriate. |
| `Role` and `RoleBinding` | A workload must call the Kubernetes API. Grant only the verbs and resources it actually requires. |
| `Certificate` | cert-manager should obtain a TLS certificate for an Ingress. This is not needed for the current HTTP-only `.home.arpa` setup. |

The example NetworkPolicy assumes Traefik runs in `kube-system` and reaches the
application on port `8080`. Confirm the actual ingress-controller namespace and
pod traffic behavior before enabling it.

## Other repository files commonly required

These are not Kubernetes manifests, but most application repositories need them
to produce something Kubernetes can deploy:

| File | Purpose |
| --- | --- |
| `Dockerfile` | Builds the application into a container image. Use a small runtime stage and a non-root user where possible. |
| `.dockerignore` | Keeps Git metadata, dependencies, secrets, and build output out of the Docker context. |
| `.github/workflows/container.yml` | Builds the image and pushes immutable versions to a registry such as `ghcr.io`. |
| Application source and dependency lockfiles | Supply the code and reproducible dependencies used by the container build. |
| `.gitignore` | Prevents local secrets, environment files, generated output, and editor state from being committed. |
| `README.md` | Documents build commands, ports, health endpoints, configuration, and deployment behavior. |

The cluster GitOps repository also needs an Argo CD `Application` manifest that
references the application repository:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: example-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/OWNER/REPOSITORY.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: example-app
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ApplyOutOfSyncOnly=true
```

For private container images, create a registry pull Secret without committing
the credential and reference it through `spec.template.spec.imagePullSecrets` in
the Deployment. Public GHCR packages need no pull Secret.

## Validate before committing

Render the complete baseline:

```sh
kubectl kustomize k8s
```

Check it against a reachable cluster without creating resources:

```sh
kubectl --kubeconfig ~/.kube/home-k3s.yaml \
  apply --dry-run=server -k k8s
```

After customization, confirm that all names, selectors, container ports, Service
target ports, Ingress backends, health probes, and namespaces agree with one
another.

## Secret-handling rules

- Never commit kubeconfigs, tokens, passwords, TLS private keys, registry
  credentials, Terraform state, or populated `.env` files.
- A Kubernetes Secret manifest is only base64-encoded, not encrypted.
- Prefer immutable image tags or digests over `latest`.
- Keep Kubernetes API, SSH, Argo CD, and other administrative services private.
