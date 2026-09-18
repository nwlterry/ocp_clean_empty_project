## Layout

Runbook repository: PackageManifest finalizer commands stay in this README.

Got it — the blocking resources are **PackageManifest** objects from `packages.operators.coreos.com`.

This is a very common issue in OpenShift. `PackageManifest` is provided by the **Operator Lifecycle Manager (OLM)** and is treated as a namespaced resource (even though it's mostly read-only/cache data from catalogs). When a project is being deleted, these lingering `PackageManifest`s can hold a finalizer and prevent the namespace from terminating.

### Step-by-step fix (for OpenShift 4.18)

1. **List the exact PackageManifests** still present in your project:

```bash
NAMESPACE=<your-project-name>   # e.g. my-stuck-project

oc get packagemanifest.packages.operators.coreos.com -n $NAMESPACE
```

   (If the command fails with a discovery error, that's normal — we'll handle it.)

2. **Remove finalizers from all of them** (this is the key step):

```bash
NAMESPACE=<your-project-name>

for pm in $(oc get packagemanifest.packages.operators.coreos.com -n $NAMESPACE -o name 2>/dev/null || echo ""); do
    echo "Patching $pm ..."
    oc patch -n $NAMESPACE $pm --type=merge -p '{"metadata":{"finalizers":null}}' || true
done
```

   This loops through every `PackageManifest` and clears its finalizer(s).

3. **Wait 30–60 seconds**, then check if the project is gone:

```bash
oc get project $NAMESPACE
```

   It should either disappear or change from `Terminating` to gone.

### Alternative one-liner (if the loop feels heavy)

```bash
NAMESPACE=<your-project-name>

oc get packagemanifest.packages.operators.coreos.com -n $NAMESPACE -o name 2>/dev/null | \
xargs -I {} oc patch -n $NAMESPACE {} --type=merge -p '{"metadata":{"finalizers":null}}' || true
```

### If it still doesn't delete after the above

Force finalize the namespace itself:

```bash
NAMESPACE=<your-project-name>

oc get namespace $NAMESPACE -o json | jq '.spec.finalizers = []' | \
oc replace --raw /api/v1/namespaces/$NAMESPACE/finalize -f -
```

Or the simpler patch:

```bash
oc patch project $NAMESPACE --type=merge -p '{"metadata":{"finalizers":null}}'
```

### Quick tip
You can combine everything safely. Many admins run the `api-resources` list again after patching the PackageManifests to confirm nothing else is left.

Would you like me to adjust the commands with your exact project name?  
Just tell me the project name (or paste the output of `oc get packagemanifest.packages.operators.coreos.com -n <project>`) and I’ll give you the ready-to-run commands.

Let me know what happens after you run the patch!

---

See [GROUP.md](GROUP.md) for sibling repositories. Catalog: https://github.com/nwlterry/nwlterry
