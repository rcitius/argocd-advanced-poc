# ArgoCD: One File, Many Environments

A working proof-of-concept showing that **a single Git repository — and a single file inside it — can drive per-environment Helm chart versions, configuration, environment variables and secret references across multiple Kubernetes clusters.**

No `values-dev.yaml` / `values-prod.yaml`. No branch-per-environment. No repo-per-environment. No hand-written `Application` manifest per environment.

---

## How this repo is organised

`main` holds this README and the shared Helm chart. **Each topology lives on its own branch**, because they are genuine alternatives rather than iterations — the later ones do not supersede the earlier ones.

| Branch | Topology | Description |
|---|---|---|
| `main` | — | Landing page and shared Helm chart. Explains the problem, the mechanism and which topology to read. Deliberately contains no ArgoCD manifests. |
| `argostage1` | Hub | First working cut. One ApplicationSet drives three environments across two clusters, each with its own chart version, namespace and log level. |
| `argostage2` | Hub | Extends stage 1 with per-environment environment variables and `secretKeyRef`. Credentials are named in Git, never stored in it. |
| `argostage3` | Per-cluster | ArgoCD on every cluster, one appset file per cluster, every destination `in-cluster`. No cross-cluster credentials exist at all. |

Read the topology comparison below to see which one fits which constraint. Everything else — the chart, the reconcile behaviour, the per-environment mechanics — is identical across all three.

---

## Why this exists

Adopting GitOps is easy to agree to and hard to lay out. The question that actually decides the design is mundane: *when dev needs `logLevel: debug` and test needs `logLevel: error`, where does that difference live?*

Every common answer fragments the configuration:

| Pattern | What goes wrong |
|---|---|
| `values-dev.yaml`, `values-test.yaml`, … | Drift. A change to one is invisible to the others, and nobody notices until an incident. |
| Branch per environment | Promotion becomes a merge conflict. History becomes unreadable. |
| Repo per environment | N copies of the same structure, kept in step by hand. |
| One `Application` manifest per environment | Boilerplate that grows linearly with environments. |

What you want instead is one place where every environment is readable side by side, and where changing exactly one of them is a one-line diff.

This repository demonstrates that, end to end, on real clusters.

---

## What it proves

Three environments, two clusters, driven from one file:

| Environment | Cluster | Chart version | Log level | Replicas |
|---|---|---|---|---|
| `dev` | cluster A (where ArgoCD runs) | 1.0.3 | info | 1 |
| `test` | cluster A | 1.0.3 | error | 2 |
| `uat` | **cluster B** (no ArgoCD installed) | 1.0.3 | warn | 3 |

Each of the following is set per environment, from the same file:

- **Which Helm chart version** to pull from the OCI registry
- **Configuration values** (log level, replica count)
- **Environment variables**, including ones that differ per environment
- **Secret references** — the file names the Secret and key; the value never enters Git
- **Which cluster** the environment is deployed to

Change one line on one environment, push, and only that environment reconciles. The other two keep their existing sync timestamps and do not redeploy.

---

## How it works

An **ApplicationSet with a list generator**. Each environment is one row of data; the `template` block is a stencil rendered once per row.

```yaml
generators:
  - list:
      elements:
        - env: dev
          cluster: in-cluster          # which cluster
          namespace: poc-dev
          chartVersion: "1.0.3"        # which chart version
          logLevel: info               # which config
          replicaCount: "1"
          envVars:                     # which environment variables
            API_URL:
              value: "https://dev.internal.example.com"
            MAX_WORKERS:
              value: "2"
            DB_PASSWORD:               # a secret reference, not a secret
              valueFrom:
                secretKeyRef:
                  name: argocd-poc-secret
                  key: db-password
```

There is only ever **one** `destination:` block written in the file. It is rendered once per row, so a single line routes each environment to its own cluster.

### The flow

```
Git repo                        OCI registry (ACR)         Clusters
--------                        ------------------         --------
charts/argocd-poc/  ──push───►  argocd-poc:1.0.0
                                argocd-poc:1.0.1
                                argocd-poc:1.0.3
                                        │
argocd/appsets/                         │ pulled per element
  poc-appset.yaml ──watched──►  ArgoCD  ├──────────────────►  cluster A (dev, test)
                                 (A)    └──────────────────►  cluster B (uat)
```

Chart **source** lives in Git; chart **artifacts** live in the registry. ArgoCD reads the artifact, never the source — so `chartVersion` is a genuine version selector, not a directory path.

### The reconcile loop

On every commit (or webhook), for each environment independently:

1. The ApplicationSet re-renders and updates the affected `Application` objects
2. The repo-server pulls `argocd-poc:<chartVersion>` from the registry
3. Helm renders it with that element's values
4. ArgoCD diffs the result against live cluster state
5. Only the differences are applied

Unchanged environments produce an empty diff and do nothing. That is what makes single-file multi-environment safe rather than terrifying.

---

## Placement rules

Where the ArgoCD manifests sit matters more than it looks. The per-topology trees are in the comparison below; these three rules hold for all of them.

1. **ApplicationSets never live inside the chart path.** ArgoCD would render them as workload manifests and end up managing itself.
2. **`bootstrap.yaml` sits beside `appsets/`, not inside it.** It watches that directory — if it lived there, `prune: true` could delete it.
3. **One manual `kubectl apply`, ever.** `bootstrap` is an app-of-apps whose only job is to keep `argocd/appsets/` applied. After that single command, adding an environment is a commit.

---

## Two topologies, both built

The first is the **hub** model: one ArgoCD instance, on one cluster, deploying to all the others. It is the version that delivers the strongest form of the single-file claim — every environment on every cluster, readable in one place.

It has a cost: the hub holds a credential for every cluster it manages. A ServiceAccount token per target, stored as a Secret, plus network reachability from hub to every target API server. That is the first thing a security review will ask about.

So the same POC was also built the other way.

### Topology A — hub (branch `argostage2`)

```
argocd/
├── bootstrap.yaml              # applied once, on the hub
└── appsets/
    └── poc-appset.yaml         # every environment, every cluster
```

`destination.name` per element routes each environment to a registered cluster. Adding a cluster means a one-time ServiceAccount + labelled Secret, then one word of YAML.

- Single pane of glass across all clusters
- One installation to operate and upgrade
- Requires stored cross-cluster credentials

### Topology B — ArgoCD per cluster (branch `argostage3`)

```
argocd/
├── bootstrap/
│   ├── dev.yaml                # applied on the dev cluster
│   └── uat.yaml                # applied on the uat cluster
└── appsets/
    ├── dev/
    │   └── poc-appset.yaml     # environments on the dev cluster
    └── uat/
        └── poc-appset.yaml     # environments on the uat cluster
```

Each cluster runs its own ArgoCD, and each `bootstrap` watches only its own directory. Every `destination` is `in-cluster`. The directory boundary does the isolation — no filtering, no templating tricks, no cluster registration.

- **No cross-cluster credentials at all.** Nothing to leak, rotate, or scope.
- Blast radius contained per cluster
- N installations to operate
- Environment config now spans one file per cluster, so "show me every environment side by side" stops being a single `cat`

### Which to choose

The honest answer is that it depends on what your security posture will tolerate. Topology A is the better operator experience and the stronger version of the single-file property. Topology B removes an entire class of credential from the design.

Everything else — the chart, the per-environment chart versions, the env vars, the secret references, the reconcile behaviour — is identical between them. The single-file property survives in B too; it just becomes single-file-per-cluster.

---

## Environment variables and secrets

`envVars` is a **map** of name to spec, where the spec is either a plain value or a secret reference:

```yaml
envVars:
  API_URL:
    value: "https://test.internal.example.com"
  DB_PASSWORD:
    valueFrom:
      secretKeyRef:
        name: argocd-poc-secret
        key: db-password
```

A map rather than a list, deliberately:

- **Per-environment override works.** Helm deep-merges maps, so shared variables live in the chart's `values.yaml` and each element adds or overrides individual keys. With a list, an element's list replaces the default wholesale and you would restate every variable in every environment — exactly the duplication this design exists to remove.
- **Ordering is deterministic.** Helm sorts map keys, so no spurious diffs in ArgoCD.
- **Secrets work through the same mechanism.** The file records *which* Secret and *which* key. The value is created out of band and exists only in the cluster.

To remove an inherited default in one environment only, set that key to `null`.

The chart template handles both forms in a single loop:

```yaml
{{- range $name, $spec := .Values.envVars }}
- name: {{ $name }}
  {{- toYaml $spec | nindent 14 }}
{{- end }}
```

---

## Multi-cluster: what is in the file, and what is not

The `cluster:` field routes an environment to a cluster, and it lives in the one file. The **credential** to reach that cluster does not, and should not — it is a ServiceAccount token stored as a labelled Secret in the `argocd` namespace.

```
Target cluster                          ArgoCD cluster
--------------                          --------------
ServiceAccount argocd-manager
  └─ token + CA  ────────copy─────────►  Secret (argocd.argoproj.io/secret-type=cluster)
                                                 │
                                        ArgoCD dials OUT to the target API ──► deploys
```

One-time per cluster. After that, pointing an environment at it is one word of YAML.

Two prerequisites that configuration cannot work around: ArgoCD must be able to **reach the target API server** over the network, and you need admin on the target long enough to create the ServiceAccount.

---

## Running it yourself

Prerequisites: `kubectl`, `helm` 3.x, an OCI-capable registry, and ArgoCD installed on one cluster.

```bash
# 1. publish two or more chart versions
helm package charts/argocd-poc
helm push argocd-poc-1.0.3.tgz oci://<your-registry>/helm

# 2. register the registry in ArgoCD as a repository (type: helm, enableOCI: true)

# 3. create any Secrets referenced by envVars, in each target namespace
kubectl -n poc-dev create secret generic argocd-poc-secret --from-literal=db-password='...'

# 4. bootstrap — the only manual apply in the whole design
kubectl apply -f argocd/bootstrap.yaml

# 5. verify
kubectl -n argocd get applications -o custom-columns=NAME:.metadata.name,CLUSTER:.spec.destination.name,CHART:.spec.source.targetRevision,SYNC:.status.sync.status,HEALTH:.status.health.status
```

### Proving it, rather than asserting it

Chart `1.0.1` contains a ConfigMap key that `1.0.0` does not. That makes the central claim falsifiable: if the chart version really is per-environment, the key appears in one environment and is absent in the other.

```console
$ kubectl -n poc-dev get cm argocd-poc-dev-info -o jsonpath='{.data}'
{"chartVersion":"1.0.0","environment":"poc-dev","logLevel":"info"}

$ kubectl --context <cluster-b> -n poc-test get cm argocd-poc-uat-info -o jsonpath='{.data}'
{"chartVersion":"1.0.1","environment":"poc-test","featureFlag":"enabled-in-1.0.1","logLevel":"warn"}
```

And on the second cluster:

```console
$ kubectl --context <cluster-b> get ns argocd
Error from server (NotFound): namespaces "argocd" not found
```

No ArgoCD there. Everything present was pushed from the ArgoCD instance on cluster A.

---

## Gotchas

Collected while building this. Every one cost real time.

| Symptom | Cause |
|---|---|
| Fewer Applications than elements | `missingkey=error` rejected an element missing a field. Intentional — it refuses to half-configure an environment. |
| `cluster not found` | `destination.name` takes a **registered cluster name**; `destination.server` takes an **API URL**. They are not interchangeable. |
| Local cluster missing from the cluster list | It is built in and always called `in-cluster`. It is not stored as a Secret. |
| `did not find expected node content` | A bare `{{ … }}` at the start of a YAML value. The manifest must parse as YAML *before* templating, so injecting structure needs `helm.values` (a string block), not `valuesObject`. |
| `CLUSTER` column empty | The cluster still has an older ApplicationSet — the commit was never pushed. |
| `Synced` but `Degraded` | Manifests applied fine; the pods are the problem. Usually an unpullable image. |
| `CreateContainerConfigError` | An `envVars` entry references a Secret that does not exist in that namespace. |
| Changes take up to 3 minutes | Default Git poll interval. Use a webhook in production; `argocd.argoproj.io/refresh=hard` for demos. |

---

## What would change for production

- **Webhook instead of polling.** Reconciliation defaults to 180s; a Git webhook makes it immediate.
- **Drop `prune: true`, or require PR review.** Deleting an element currently tears down that environment. That is correct behaviour, and it is not something to leave one keystroke away.
- **Scope the ServiceAccount.** `cluster-admin` is a POC convenience. A real deployment gets a role limited to the namespaces ArgoCD manages.
- **Replace static tokens with workload identity.** On AKS, ArgoCD can authenticate to target clusters via Entra ID (`execProviderConfig` with `kubelogin`), removing long-lived credentials from both cluster Secrets and Terraform state.
- **Consider per-cluster ArgoCD.** The single-file property survives: each instance reads the same file and filters to its own environments. It trades a single pane of glass for having no cross-cluster credentials at all.

---

## Why this design

The interesting property is not "ArgoCD can do multi-cluster" — it can, and that is documented. It is that **every environment change becomes a one-line, reviewable diff**, with the other environments visible immediately next to it in the same file.

Promotion is changing a version string in a pull request. Rollback is `git revert`. Adding an environment is four lines of YAML. That is the part that survives contact with a real team.
