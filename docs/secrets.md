 cloudflare-api-token · cert-manager · Cloudflare-Token (Zone-DNS-Edit, damians.cloud) · CREATE IN cf
 kubectl create secret generic cloudflare-api-token -n cert-manager --from-literal=api-token=

 kubectl create secret generic paperless-secret `
>>   -n media `
>>   --from-literal=PAPERLESS_SECRET_KEY=""

 karakeep-secret · tools · NEXTAUTH_SECRET, MEILI_MASTER_KEY · sealed, see below (not manual kubectl anymore)

---

## Sealed Secrets (new secrets go here, not manual kubectl)

Controller: `apps/sealed-secrets.yaml` (Bitnami chart, namespace `sealed-secrets`).
Encrypted output lives in `infrastructure/secrets/`, applied by `apps/secrets.yaml`.
See `infrastructure/secrets/README.md` for the exact `kubeseal` command.

kubeseal only needs the controller running (bootstrap it before sealing anything):

```powershell
kubeseal --fetch-cert `
  --controller-name=sealed-secrets `
  --controller-namespace=sealed-secrets > pub-sealed-secrets.pem
```

That `.pem` is a public key — safe to keep around locally, no need to commit it
(kubeseal can always re-fetch it from the live controller).