# Application and ApplicationSet Finalizers

This page documents how the deletion finalizer works for the `Application` and
`ApplicationSet` custom resources (CRs), and which knobs exist to configure it.

## Short answer: can the finalizer be configured with an annotation?

No. There is **no annotation** that controls how the finalizer works, neither
for `Application` nor for `ApplicationSet`. The finalizer is controlled
exclusively by the entries in `.metadata.finalizers` (plus the flags/fields
that add or remove those entries, listed below).

The annotations Argo CD defines on these resources
(`argocd.argoproj.io/refresh`, `argocd.argoproj.io/manifest-generate-paths`,
`argocd.argoproj.io/application-set-refresh`, `argocd.argoproj.io/compare-options`,
etc.) have no effect on deletion/finalizer behavior.

## Application finalizer

### Finalizer values

The application-controller treats any of the following three
`.metadata.finalizers` entries as a "cascade deletion" finalizer
(see `isPropagationPolicyFinalizer` in
`pkg/apis/application/v1alpha1/types.go`, constants in
`pkg/apis/application/v1alpha1/application_defaults.go`):

| Finalizer | Meaning |
|---|---|
| `resources-finalizer.argocd.argoproj.io` | Cascade delete; managed live resources are deleted with the **foreground** propagation policy (this is also the default the CLI/API applies when no explicit propagation policy is given). |
| `resources-finalizer.argocd.argoproj.io/foreground` | Cascade delete with the **foreground** propagation policy. |
| `resources-finalizer.argocd.argoproj.io/background` | Cascade delete with the **background** propagation policy. |

Helpers on the `Application` type (`pkg/apis/application/v1alpha1/types.go`):

- `CascadedDeletion()` — true if any of the above finalizers is present.
- `GetPropagationPolicy()` — returns which of the finalizers is set.
- `SetCascadedDeletion(finalizer)` / `UnSetCascadedDeletion()` — add/remove
  the propagation-policy finalizers.
- `IsFinalizerPresent(finalizer)` — checks for a specific finalizer.

If **none** of these finalizers is present, deleting the `Application`
performs a *non-cascading* delete: only the `Application` CR is removed and
the live workload resources (`Deployments`, `Services`, …) are left orphaned
on the target cluster.

### How the application-controller finalizes deletion

When an `Application` with a `DeletionTimestamp` still carries one of the
finalizers, `processAppQueueItem` in `controller/appcontroller.go` calls
`finalizeApplicationDeletion`, which proceeds roughly as follows:

1. Resolve the destination cluster and the AppProject. If the destination
   cluster no longer exists (`validDestination == false`), live-object cleanup
   is skipped and the controller just drops its cache entries and removes the
   finalizer (this prevents deletions from getting stuck on a dangling
   cluster reference).
2. List the live objects the Application manages (`getPermittedAppLiveObjects`).
   Objects already pending deletion make the controller wait (`"<n> objects
   remaining for deletion"`) — deletion is re-attempted on subsequent
   reconciles until everything is gone.
3. Delete each remaining managed object with
   `metav1.DeletePropagationForeground` — or
   `metav1.DeletePropagationBackground` if the
   `.../background` finalizer variant is set.
4. Once no managed objects remain, clear the controller cache entries
   (`SetAppManagedResources`, `SetAppResourcesTree`) and remove the finalizer
   via a merge patch (`removeCascadeFinalizer` → `UnSetCascadedDeletion`).
   Removing the last finalizer lets Kubernetes garbage collection finally
   remove the `Application` object.

Deletion errors are surfaced as an `ApplicationCondition` of type
`DeletionError` on the Application status, plus an audit event.

### Foreground vs. background

The only behavioral difference between the finalizer variants is the
`PropagationPolicy` passed to the Kubernetes API when the controller deletes
each managed live object (`controller/appcontroller.go`):

- default / `/foreground` → `DeletePropagationForeground`
- `/background` → `DeletePropagationBackground`

### Setting and removing the Application finalizer

- **Argo CD CLI** (`cmd/argocd/commands/app.go`,
  `server/application/application.go`):
  - `argocd app delete APPNAME [--cascade]` — adds the finalizer
    automatically, then deletes (this is the default).
  - `argocd app delete APPNAME --cascade --propagation-policy background|foreground`
    — selects the `/background` or `/foreground` finalizer variant
    (`getPropagationPolicyFinalizer`).
  - `argocd app delete APPNAME --cascade=false` — removes the finalizers
    first, so only the `Application` CR is deleted.
  - `argocd app create ... --set-finalizer` — puts the finalizer on at
    creation time.
- **API**: the `ApplicationService.Delete` endpoint accepts `cascade` and
  `propagationPolicy` query parameters and patches `.metadata.finalizers`
  accordingly before issuing the delete.
- **kubectl / declarative GitOps** (`docs/user-guide/app_deletion.md`):
  cascade delete by declaring the finalizer on the resource, e.g.
  `metadata.finalizers: ["resources-finalizer.argocd.argoproj.io"]`, then
  `kubectl delete applications.argoproj.io APPNAME`. Omitting the finalizer
  means a non-cascading delete.

## ApplicationSet finalizer behavior

Unlike `Application`, the **`ApplicationSet` CR itself has no Argo CD
finalizer and no custom deletion handler**: `Reconcile` in
`applicationset/controllers/applicationset_controller.go` returns immediately
when the `ApplicationSet` carries a `DeletionTimestamp`. Deleting an
`ApplicationSet` cascades to its generated `Application`s purely through
standard Kubernetes garbage collection, because every generated `Application`
gets an `ownerReference` (controller reference) pointing back at the parent
`ApplicationSet` (set via `controllerutil.SetControllerReference` in
`createOrUpdateInCluster`).

Consequences:

- `kubectl delete applicationset NAME` deletes the generated `Application`
  CRs via owner-reference cascading. Each `Application`'s **own** finalizer
  then decides, as described above, whether its live workload resources are
  deleted too (handled by the application-controller, via
  [the deletion finalizer](../user-guide/app_deletion.md#about-the-deletion-finalizer)).
- `kubectl delete applicationset NAME --cascade=orphan` keeps the generated
  `Application` CRs (and their workloads) around.
- Deleting the `ApplicationSet` CR never deletes anything directly; the
  applicationset-controller is not involved in that path at all.

### Finalizers on generated Applications

What finalizer a generated `Application` carries is decided at render time:

1. If `spec.template.metadata.finalizers` in the `ApplicationSet` declares
   finalizers, they are copied verbatim onto every generated `Application`
   (`getTempApplication`), and the update path
   (`found.ObjectMeta.Finalizers = generatedApp.Finalizers`) keeps them in
   sync — manual finalizer edits on a generated `Application` are reverted on
   the next reconcile unless reflected in the template.
2. Otherwise, `RenderTemplateParams` in `applicationset/utils/utils.go`
   injects the default `resources-finalizer.argocd.argoproj.io` finalizer
   **unless** `spec.syncPolicy.preserveResourcesOnDeletion` is `true`.
   With `preserveResourcesOnDeletion: true`, generated `Application`s get no
   finalizer, so deleting them later is non-cascading (workloads are
   preserved). See also
   [Application Pruning & Resource Deletion](applicationset/Application-Deletion.md).

### Pruning of generated Applications

When generator output changes so an `Application` is no longer desired,
`deleteInCluster` issues a plain `Delete` for it; the finalizer already on
the `Application` governs whether its workloads are cascade-deleted. One
special case: `removeFinalizerOnInvalidDestination` strips the
`resources-finalizer.argocd.argoproj.io` finalizer (only that exact value)
before deletion if the `Application`'s destination no longer matches a known
cluster, so pruning cannot get stuck behind a finalizer the
application-controller is unable to honor (Argo CD issue #5817).

## Configuration summary

| Goal | How |
|---|---|
| Cascade-delete an `Application` (app + workloads) | Put one of the three `resources-finalizer.argocd.argoproj.io…` finalizers in `.metadata.finalizers` (CLI `--cascade` does this for you). |
| Non-cascading `Application` delete (keep workloads) | Ensure no finalizer is set (`--cascade=false` removes them). |
| Choose foreground/background deletion | Use the `/foreground` or `/background` finalizer variant (`--propagation-policy`). |
| Cascade-delete workloads of apps generated by an `ApplicationSet` | Leave the default: declare no finalizers in the template and keep `preserveResourcesOnDeletion` unset/`false`. |
| Preserve workloads when generated `Application`s are deleted | Set `spec.syncPolicy.preserveResourcesOnDeletion: true`, or declare `spec.template.metadata.finalizers: []`… via an explicit empty/finalizer-free template (explicit template finalizers win over the default injection). |
| Keep generated `Application`s when deleting the `ApplicationSet` | `kubectl delete applicationset NAME --cascade=orphan`. |
| Configure any of this via annotation | Not supported — no such annotation exists. |

## Troubleshooting stuck deletions

If an `Application` stays in `Terminating`, it is almost always the cascade
finalizer waiting on live objects (or an unreachable cluster). Options:

- Fix or remove the blocking live resources and let the controller finish.
- Drop the cascade requirement by removing the finalizer, e.g.
  `kubectl patch app APPNAME -p '{"metadata":{"finalizers":[]}}' --type merge`
  (workloads are then orphaned, not deleted).
- For pruned `ApplicationSet` children pointing at deleted clusters, the
  applicationset-controller already strips the finalizer automatically (see
  above); if an `Application` still hangs, check the same live-object causes.
