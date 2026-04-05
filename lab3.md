```
# Remove all app config from flux-gitops (Flux prune deletes cluster resources)
git rm -r clusters/production/namespaces/
git rm -r clusters/production/sources/
git rm -r clusters/production/apps/
git commit -m "chore: tear down lab"
git push
```
```
flux reconcile source git flux-system
flux reconcile kustomization flux-system --with-source

```

```
kubectl get ns

```
