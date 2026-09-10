# Replace Argo CD Community Operator with the official OpenShift GitOps operator — without deleting or syncing anything

> **Goal:** completely remove the community operator **without cascade deletion**, install the Red Hat
> `openshift-gitops-operator` from the **latest stable channel**, then restore every `Application` and
> `ApplicationSet` manifest **without triggering any sync** — so developers return to all their apps present
> and nothing changed on the clusters.
>
> Target: **OCP 4.20** · GitOps: **1.20** via channel **`latest`** · Package: **`openshift-gitops-operator`**
> · Source: **`redhat-operators`** · Risk level: **high — follow in order**
>
> HTML twin: `migrate-community-to-openshift-gitops.html` (same directory).

## Contents

1. [Why this order matters (finalizers)](#why-this-order-matters--the-finalizer-in-one-minute)
2. [0 · Freeze & prerequisites](#0--freeze--prerequisites)
3. [1 · Inventory & backup everything](#1--inventory--backup-everything)
4. [2 · Neutralize sync (no-sync restore prep)](#2--neutralize-sync--prepare-a-no-sync-restore)
5. [3 · Strip cascade finalizers (no-cascade removal)](#3--strip-cascade-finalizers--the-no-cascade-removal)
6. [4 · Remove the community operator](#4--remove-the-community-operator-completely-orphaned-not-cascaded)
7. [5 · Install openshift-gitops from channel latest](#5--install-the-official-openshift-gitops-operator-channel-latest-ocp-420)
8. [6 · Align the new instance](#6--align-the-new-instance--namespaces-rbac-repos-before-any-app-returns)
9. [7 · Restore manifests without triggering sync](#7--restore-manifests--present-but-inert-no-sync)
10. [8 · Verify](#8--verify-everything-present-nothing-changed--then-hand-back)
11. [Rollback & troubleshooting](#rollback--troubleshooting)
12. [Appendix](#appendix--copy-paste-scripts)

## Why this order matters — the finalizer in one minute

Everything in this runbook follows from one fact (see `application-finalizers.md`):

> **Cascade deletion is caused by the finalizer, not by the operator itself.**
> An `Application` carrying `resources-finalizer.argocd.argoproj.io` (or its `/foreground` / `/background`
> variants) tells the application-controller to delete all live workload objects (`Deployments`, `Services`, …)
> before the `Application` object may disappear. Remove the finalizer and the `Application` deletes (or is
> re-created) **without touching workloads**.

| Object | What deletes what | What saves you |
|---|---|---|
| `Application` | Its finalizer cascade-deletes **workloads** on the target cluster. | Strip `.metadata.finalizers` before any delete; restore them only after the new controller is healthy and sync is off. |
| `ApplicationSet` | Has **no** Argo CD finalizer. Deleting it garbage-collects generated `Application`s via `ownerReference` — and *their* finalizers then delete workloads. | Strip the generated apps' finalizers first, or delete sets with `--cascade=orphan`. |
| CRDs (`applications.argoproj.io` …) | Deleting a CRD deletes **every** CR of that type in etcd, finalizers or not. | **Never delete the CRDs during this migration.** Both operators use the same `argoproj.io/v1alpha1` API, so CRs survive the swap if CRDs stay. |

Flow: `1 · backup → 2 · sync OFF → 3 · finalizers OFF → 4 · remove old operator → 5 · install official operator → 6 · restore, sync still OFF → 7 · verify, hand back`

## 0 · Freeze & prerequisites

1. **Announce a change freeze.** No git pushes to app repos, no manual `argocd app sync`, no ApplicationSet/generator changes. Record the freeze window.
2. **You need:** `cluster-admin`, `oc` + `jq` (+ `yq` for file edits), access to the namespaces hosting the community operator (commonly `argocd` or `openshift-operators`) and to every namespace containing `Application`/`ApplicationSet` CRs.
3. **Identify what is installed.** Record operator namespace, subscription/channel, ArgoCD CRs and app namespaces — you will compare against this at the end:

   ```bash
   oc get subscriptions -A | grep -i -E "argocd|gitops"
   oc get argocds -A
   oc get applications.argoproj.io -A --no-headers | awk '{print $1}' | sort -u
   oc get applicationsets.argoproj.io -A --no-headers | awk '{print $1}' | sort -u
   oc get appprojects.argoproj.io -A
   oc version && oc get clusterversion
   ```

4. **Snapshot workload state** (your "nothing changed" proof). Save pod ages, deployment revisions and Argo CD sync state:

   ```bash
   oc get applications.argoproj.io -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{" sync="}{.status.sync.status}{" health="}{.status.health.status}{"\n"}{end}' | sort | tee sync-before.txt
   kubectl get deployments -A -o wide > deployments-before.txt   # repeat per workload ns as needed
   ```

## 1 · Inventory & backup everything

Back up **before** touching anything. Export each object individually (easier to restore selectively) with `status`, `operation` and server-side metadata stripped — the restore section reuses these files.

```bash
export BACKUP_DIR=argocd-migration-$(date +%F)
mkdir -p $BACKUP_DIR/{applications,applicationsets,appprojects,argocds,secrets,subscriptions}

# One file per Application / ApplicationSet / AppProject (status + operation stripped)
for KIND in applications applicationsets appprojects; do
  oc get $KIND.argoproj.io -A -o json | jq -c '.items[] | {ns: .metadata.namespace, name: .metadata.name, obj: .}' |
  while read -r row; do
    ns=$(echo "$row" | jq -r .ns); name=$(echo "$row" | jq -r .name)
    echo "$row" | jq '.obj
      | del(.status, .operation, .metadata.resourceVersion, .metadata.uid,
            .metadata.creationTimestamp, .metadata.generation, .metadata.managedFields)' \
      > "$BACKUP_DIR/${KIND}/${ns}__${name}.json"
  done
done

# ArgoCD CRs (to replicate instance config later), subscriptions, repo/TLS secrets
oc get argocds -A -o yaml > $BACKUP_DIR/argocds/argocds.yaml
oc get subscriptions -A -o yaml > $BACKUP_DIR/subscriptions/subscriptions.yaml
oc get secrets -A -l argocd.argoproj.io/secret-type=repository -o yaml > $BACKUP_DIR/secrets/repo-secrets.yaml
oc get secrets -A -l argocd.argoproj.io/secret-type=repo-creds -o yaml > $BACKUP_DIR/secrets/repo-creds.yaml

ls -R $BACKUP_DIR | head -30
echo "Applications backed up: $(ls $BACKUP_DIR/applications | wc -l)"
echo "ApplicationSets backed up: $(ls $BACKUP_DIR/applicationsets | wc -l)"
```

> **Keep two copies.** One in version control / object storage off-cluster, one local. Verify the counts match
> `oc get` output. Also back up any `argocd-cm`, `argocd-rbac-cm`, repo certificates and SSO config you customized —
> the new operator will need them in step 6.

> **Generated Applications are owned by ApplicationSets — but back them up anyway.**
> You will restore *sets first and let them regenerate apps*; the per-app backups are your safety net and your sync-policy reference.

## 2 · Neutralize sync — prepare a no-sync restore

Sync (not the finalizer) is what would change workloads on restore. Create **restore copies** of your backups with automated sync, self-heal and stray operations removed. Keep the originals untouched — you re-apply original sync policy only after verification.

```bash
mkdir -p $BACKUP_DIR/restore && cp -r $BACKUP_DIR/applications $BACKUP_DIR/restore/ \
  && cp -r $BACKUP_DIR/applicationsets $BACKUP_DIR/restore/ && cp -r $BACKUP_DIR/appprojects $BACKUP_DIR/restore/

# Strip automated sync + operation from every restore copy
for f in $BACKUP_DIR/restore/applications/*.json $BACKUP_DIR/restore/applicationsets/*.json; do
  tmp="$f.tmp" && jq 'del(.spec.syncPolicy.automated, .operation)' "$f" > "$tmp" && mv "$tmp" "$f"
done

# For ApplicationSets, also neutralize the *template's* sync policy so generated apps come up manual
for f in $BACKUP_DIR/restore/applicationsets/*.json; do
  tmp="$f.tmp" && jq 'del(.spec.template.spec.syncPolicy.automated, .spec.template.spec.operation)' "$f" > "$tmp" && mv "$tmp" "$f"
done
echo "Restore copies are manual-sync only."
```

> **Why files, not live patching?** Editing the restore copies (rather than the live cluster) means the old
> controller keeps running unchanged until removal, and the new controller only ever sees manual-sync manifests.
> Optionally also pause the old apps live
> (`oc patch application … --type=merge -p '{"spec":{"syncPolicy":{}}}'`), but the file prep above is the mandatory part.

## 3 · Strip cascade finalizers — the no-cascade removal

With sync neutralized on disk, now defuse deletion on the live cluster: remove every Argo CD cascade finalizer from every `Application`. After this, deleting an app/operator can only remove CRs — **workloads are orphaned, never deleted**.

```bash
# Audit first: which apps would cascade?
oc get applications.argoproj.io -A -o json \
  | jq -r '.items[] | [.metadata.namespace, .metadata.name, (.metadata.finalizers // [] | join(","))] | @tsv' \
  | column -t

# Remove ALL finalizers from every Application (repeat until the audit shows empty)
for ns_name in $(oc get applications.argoproj.io -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'); do
  ns=${ns_name%/*}; name=${ns_name#*/}
  oc patch applications.argoproj.io "$name" -n "$ns" --type=merge -p '{"metadata":{"finalizers":[]}}'
done

# Confirm: nothing left to cascade
oc get applications.argoproj.io -A -o json | jq -r '.items[].metadata.finalizers // empty' | sort -u
# ^ must print nothing
```

> **ApplicationSets need no finalizer stripping** (they carry none), but their generated apps did — covered above.
> From now on, delete any `ApplicationSet` only with `oc delete applicationset NAME --cascade=orphan` until the migration is done.

> **Do not "clean up" CRDs.** Never run
> `oc delete crd applications.argoproj.io applicationsets.argoproj.io appprojects.argoproj.io` —
> that deletes all CRs instantly, bypassing finalizers entirely. Leave CRDs in place; the new operator reuses the same API group.

## 4 · Remove the community operator completely (orphaned, not cascaded)

1. **Delete the community ArgoCD CRs.** This removes operator-managed Deployments/StatefulSets (server, controller, repo-server, redis) in the operator namespace. Workloads are safe: app finalizers are gone (step 3). If the community operator namespace *is* also hosting user apps, move or orphan them first.

   ```bash
   oc get argocds -A --no-headers
   # delete each one, e.g.:
   oc delete argocd -n <community-ns> <instance-name> --wait=true
   oc get pods -n <community-ns> -w   # wait until only the operator pod (or nothing) remains
   ```

2. **Uninstall the community operator** (Subscription → CSV → OperatorGroup), then its namespace *only if it contains no user CRs*:

   ```bash
   oc get subscriptions -A | grep -i argocd
   oc delete subscription -n <community-ns> <community-sub-name>
   oc get clusterserviceversions -n <community-ns> | grep -i argocd
   oc delete clusterserviceversion -n <community-ns> <community-csv>
   # OperatorGroup only if you created it for the community operator:
   oc get operatorgroup -n <community-ns>
   oc delete operatorgroup -n <community-ns> <name>  # only the community one
   # Delete the namespace only when empty of apps/appprojects you still need:
   oc get applications.argoproj.io,applicationsets.argoproj.io,appprojects.argoproj.io -n <community-ns>
   oc delete namespace <community-ns>  # ONLY if the line above is empty / "No resources found"
   ```

3. **Leave behind:** the CRDs, your backed-up `Application`/`ApplicationSet`/`AppProject` CRs (finalizer-free, harmless without a controller), and cluster-scoped leftovers you will let the new operator adopt (ClusterRoles/RoleBindings are reconciled on install).

> **Checkpoint.** Workloads still running? `kubectl get deployments -A` should match `deployments-before.txt`.
> App CRs still present but inert? `oc get applications -A` lists them (no controller = no sync, no cascade possible).

## 5 · Install the official openshift-gitops operator (channel `latest`, OCP 4.20)

On OCP 4.20 the supported GitOps release is **1.20** (supports OCP 4.14–4.21), published on the
`redhat-operators` catalog under package `openshift-gitops-operator`. The **`latest` channel always tracks the
newest stable GitOps release** — that is the channel to subscribe to. Confirm it resolves before installing:

```bash
oc get packagemanifest openshift-gitops-operator -n openshift-marketplace \
  -o jsonpath='{range .status.channels[*]}{.name}{" -> "}{.currentCSV}{"\n"}{end}'
# expect a line like:  latest -> openshift-gitops-operator.v1.20.x
```

**Console path:** Operators → OperatorHub → *Red Hat OpenShift GitOps* → Install → Update channel `latest`, Installation mode *All namespaces*, Installed Namespace `openshift-gitops` (created for you), Approval *Automatic*.

**CLI path (pinned to the same choices):**

```bash
oc create namespace openshift-gitops --dry-run=client -o yaml | oc apply -f -
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-gitops-operator-group
  namespace: openshift-gitops
spec:
  upgradeStrategy: Default
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-gitops
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
# Wait for success:
oc get subscription openshift-gitops-operator -n openshift-gitops -w
oc get csv -n openshift-gitops -w   # PHASE → Succeeded
oc get pods -n openshift-gitops     # server, controller, repo-server, redis, applicationset-controller
oc get argocd openshift-gitops -n openshift-gitops   # default instance auto-created by the operator
```

## 6 · Align the new instance — namespaces, RBAC, repos (before any app returns)

1. **Application namespaces.** If your apps live outside `openshift-gitops`, allow those namespaces on the new instance (compare with the old ArgoCD CR you backed up in step 1):

   ```bash
   oc get -n openshift-gitops argocd openshift-gitops -o yaml | grep -A5 sourceNamespaces
   # e.g. patch:  spec.sourceNamespaces: ["*"]  (or the explicit list you used before)
   oc patch argocd openshift-gitops -n openshift-gitops --type=merge \
     -p '{"spec":{"sourceNamespaces":["*"]}}'
   ```

2. **RBAC / SSO / TLS.** Re-apply your saved `argocd-rbac-cm` settings, OIDC/dex config, custom TLS and the repository / repo-cred secrets from `$BACKUP_DIR/secrets` into `openshift-gitops`. Verify repo connectivity in the new UI before restoring apps.
3. **Keep global sync off.** Do not enable auto-sync, auto-prune or self-heal at instance level yet. The per-manifest neutralization in step 2 is what guarantees the restore stays inert.

## 7 · Restore manifests — present but inert (no sync)

> **Order matters.** AppProjects → ApplicationSets (let them generate apps) → standalone Applications only.
> Never apply the original (auto-sync) files until step 8 is green. Every apply below uses the
> **`$BACKUP_DIR/restore` manual-sync copies** from step 2.

1. **AppProjects first** (apps referencing a missing project go `Unknown`/error):

   ```bash
   for f in $BACKUP_DIR/restore/appprojects/*.json; do oc apply -f "$f"; done
   oc get appprojects -A
   ```

2. **ApplicationSets next — and stop there.** The applicationset-controller regenerates the owned `Application`s itself; do *not* also apply the backed-up generated apps (that fights the controller):

   ```bash
   for f in $BACKUP_DIR/restore/applicationsets/*.json; do oc apply -f "$f"; done
   sleep 30; oc get applicationsets -A
   oc get applications -A -l app.kubernetes.io/managed-by=openshift-gitops  # generated apps reappearing
   # Confirm every (re)generated app is manual-sync:
   oc get applications -A -o json | jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name) automated=\(.spec.syncPolicy.automated // "none")"'
   ```

3. **Standalone Applications only** (skip any app whose `ownerReferences` points at an ApplicationSet — the controller owns those):

   ```bash
   for f in $BACKUP_DIR/restore/applications/*.json; do
     if ! jq -e '.metadata.ownerReferences // [] | map(select(.kind=="ApplicationSet")) | length > 0' "$f" >/dev/null; then
       oc apply -f "$f"
     else
       echo "skip (owned by ApplicationSet): $f"
     fi
   done
   ```

4. **Restore the cascade finalizers — but still no sync.** Now that the new controller is healthy and everything is manual, re-arm future cascade behavior by re-adding the finalizer exactly as your originals had it (compare each file with its backup twin; default is the plain finalizer):

   ```bash
   # Example: re-arm the default cascade finalizer on a restored app (repeat per app or script from backup):
   oc patch applications.argoproj.io APPNAME -n APPNS --type=merge \
     -p '{"metadata":{"finalizers":["resources-finalizer.argocd.argoproj.io"]}}'
   # Finalizers only act on *deletion* — adding them triggers no sync.
   ```

## 8 · Verify: everything present, nothing changed — then hand back

- [ ] **Counts match.** App / ApplicationSet / AppProject counts equal step-1 inventory and the backup file counts.
- [ ] **No sync happened.** No app shows a fresh `operation`; sync state is `OutOfSync`/`Unknown`, never a new successful auto-sync revision:

  ```bash
  oc get applications -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{" sync="}{.status.sync.status}{" rev="}{.status.sync.revision}{"\n"}{end}' | sort | tee sync-after.txt
  diff sync-before.txt sync-after.txt   # expect only status-shape noise, no new synced revisions
  oc get applications -A -o json | jq -r '.items[] | select(.operation != null) | "\(.metadata.namespace)/\(.metadata.name) HAS OPERATION!"'
  # ^ must print nothing
  ```

- [ ] **Workloads untouched.** Deployments/pods predate the migration (compare `deployments-before.txt`; pod `AGE` older than the install window).
- [ ] **Spot-check diffs are advertised, not applied:** `argocd app diff APPNAME` may show drift — that is the *pending* sync, correctly waiting for developers.

Only when all of the above hold: announce return, and re-enable each team's original sync policy deliberately (re-apply the **original** backup files' `spec.syncPolicy`, or re-run the team's GitOps pipeline) — app by app, watching the first syncs.

## Rollback & troubleshooting

| Symptom | Cause / fix |
|---|---|
| An `Application` is stuck `Terminating` | A cascade finalizer survived step 3. `oc patch applications … -p '{"metadata":{"finalizers":[]}}'` orphans workloads immediately; then investigate the blocking live object. |
| Generated apps don't reappear after step 7 | Wrong `sourceNamespaces` on the new ArgoCD CR, or generators can't reach git/clusters. Check applicationset-controller logs and repo secrets in `openshift-gitops`. |
| An app synced on restore | A non-neutralized manifest slipped in (original file applied, or template `automated` left in an ApplicationSet). Delete nothing — set `spec.syncPolicy.automated` to null, assess drift with `app diff`, and re-freeze. |
| CRDs accidentally deleted | Stop. Re-create CRDs (reinstall either operator), then re-apply backups: AppProjects → ApplicationSets → standalone Applications (manual copies first). Workloads themselves were never touched by a CRD delete — only the CRs need rebuilding. |
| Need full rollback | Uninstall openshift-gitops (Subscription/CSV, keep CRDs + CRs), reinstall the community operator, re-apply original manifests. Your two backup copies make this a repeat of steps 5–7 in reverse. |

## Appendix — copy-paste scripts

### A · Audit finalizers (run before step 4 and after step 7)

```bash
oc get applications.argoproj.io -A -o json \
  | jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name) finalizers=\((.metadata.finalizers // []) | join(",")) sync=\(.spec.syncPolicy.automated // "manual")"' | sort
```

### B · Emergency orphan (unstick anything without touching workloads)

```bash
ns=APPNS; name=APPNAME
oc patch applications.argoproj.io "$name" -n "$ns" --type=merge -p '{"metadata":{"finalizers":[]}}'
oc delete applicationset -n "$ns" SETNAME --cascade=orphan   # only if a set itself must go
```

### C · Post-restore "nothing changed" evidence pack

```bash
oc get applications.argoproj.io -A > evidence-apps.txt
oc get applicationsets.argoproj.io -A > evidence-appsets.txt
oc get applications -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{" "}{.status.sync.status}{" "}{.status.health.status}{"\n"}{end}' | sort > evidence-sync.txt
kubectl get pods -A -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,AGE:.metadata.creationTimestamp | sort > evidence-pods.txt
tar czf migration-evidence-$(date +%F).tar.gz $BACKUP_DIR sync-before.txt sync-after.txt evidence-*.txt deployments-before.txt
```

---
Sources: `docs/operator-manual/application-finalizers.md` · Code: `controller/appcontroller.go`,
`applicationset/controllers/applicationset_controller.go`, `applicationset/utils/utils.go` ·
Platform: OCP 4.20, OpenShift GitOps 1.20 (`latest` channel, `redhat-operators`) ·
Related: `docs/operator-manual/application-finalizers.html`
