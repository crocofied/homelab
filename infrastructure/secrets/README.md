Sealed Secrets live here, one file per Secret, e.g. `karakeep-secret.yaml`.

Each file must set `metadata.namespace` explicitly (the `secrets` Application's
`destination.namespace` is only a fallback) and `metadata.name` must match
whatever `secretKeyRef.name` the consuming app's `values.yaml` expects — that's
what the SealedSecret decrypts into.

To add one:

```powershell
kubectl create secret generic <name> -n <namespace> `
  --from-literal=KEY="<value>" `
  --dry-run=client -o yaml > tmp-secret.yaml

kubeseal --format=yaml `
  --controller-name=sealed-secrets `
  --controller-namespace=sealed-secrets `
  < tmp-secret.yaml > infrastructure/secrets/<name>.yaml

Remove-Item tmp-secret.yaml
```

The output is ciphertext bound to this cluster's controller key — safe to commit.
