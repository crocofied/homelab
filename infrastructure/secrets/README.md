Sealed Secrets live here, one file per Secret, e.g. `karakeep-secret.yaml`.

Each file must set `metadata.namespace` explicitly (the `secrets` Application's
`destination.namespace` is only a fallback) and `metadata.name` must match
whatever `secretKeyRef.name` the consuming app's `values.yaml` expects — that's
what the SealedSecret decrypts into.

To add one (PowerShell — pipe straight through, don't use `>`/`<` for the
intermediate files: native-command output redirected with `>` is written as
UTF-16 by default in Windows PowerShell, which `kubeseal` can't parse, and
PowerShell has no `<` input redirection for native exes at all):

```powershell
$sealed = kubectl create secret generic <name> -n <namespace> `
  --from-literal=KEY="<value>" `
  --dry-run=client -o yaml |
  kubeseal --format=yaml `
    --controller-name=sealed-secrets `
    --controller-namespace=sealed-secrets |
  Out-String

[System.IO.File]::WriteAllText(
  "$PWD\infrastructure\secrets\<name>.yaml",
  $sealed.TrimEnd() + "`n",
  [System.Text.Encoding]::ASCII)
```

`WriteAllText` with `ASCII` encoding is deliberate: `kubeseal`'s output is
plain ASCII (YAML + base64 ciphertext), and this is the only reliable way in
Windows PowerShell 5.1 to write it without a stray BOM that would break
ArgoCD's YAML parsing.

The output is ciphertext bound to this cluster's controller key — safe to commit.
