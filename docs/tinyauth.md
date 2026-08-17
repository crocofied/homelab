Ich hab mir dein Repo und die aktuellen Tinyauth-Docs angeschaut. Kurz vorweg: Tinyauth ist seit v5.1.0 ein OpenID-Connect-zertifizierter Server — du kannst also beides mit einer Instanz machen (OIDC-Provider + Forward-Auth). Wichtig dabei: für den OIDC-Server ist HTTPS auf der App-URL zwingend, weil das die Issuer-URL wird, und Sessions und Keys liegen im `/data`-Verzeichnis — du brauchst also ein PVC.

Das offizielle Helm-Chart (`helm.tinyauth.app`, aktuell 2.1.3 / appVersion v5.1.3) passt gut zu deinem Multi-Source-Pattern.

## 1. Secret für den User-Hash

Hash erzeugen:

```bash
docker run -i -t --rm ghcr.io/tinyauthapp/tinyauth:v5 user create --interactive
```

**Wichtig:** *nicht* „format for docker" wählen — das `$$`-Escaping brauchst du nur in Compose. Im K8s-Secret kommt der rohe bcrypt-Hash mit einfachen `$` rein.

Dann nach deinem README-Pattern (`infrastructure/secrets/tinyauth-secret.yaml`):

```powershell
$sealed = kubectl create secret generic tinyauth-secret -n tools `
  --from-literal=USERS='damian:$2a$10$...' `
  --dry-run=client -o yaml |
  kubeseal --format=yaml --controller-name=sealed-secrets --controller-namespace=sealed-secrets |
  Out-String

[System.IO.File]::WriteAllText("$PWD\infrastructure\secrets\tinyauth-secret.yaml", $sealed.TrimEnd() + "`n", [System.Text.Encoding]::ASCII)
```

## 2. ArgoCD Application

`apps/tinyauth.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: tinyauth
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://helm.tinyauth.app
      chart: tinyauth
      targetRevision: 2.1.3
      helm:
        valueFiles:
          - $values/apps-manifests/tools/tinyauth/values.yaml
    - repoURL: https://github.com/crocofied/homelab.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: tools
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

## 3. Values

`apps-manifests/tools/tinyauth/values.yaml`:

```yaml
tinyauth:
  appUrl: https://auth.damians.cloud
  # "auto" würde versuchen, ACLs aus Ingress-Annotations zu lesen -> braucht
  # ClusterRole auf ingresses (legt das Chart NICHT an). Erstmal aus.
  labelProvider: none

  database:
    driver: sqlite
    path: /data/tinyauth.db

  auth:
    secureCookie: true
    subdomainsEnabled: true        # muss true bleiben, sonst kein Forward-Auth
    trustedProxies: 10.244.0.0/16  # Pod-CIDR anpassen (Talos default flannel)
    sessionExpiry: 604800
    # OIDC-Claims für den User - viele Apps brauchen zwingend eine E-Mail
    userAttributes:
      - username: damian
        name: Damian
        email: damian@damians.cloud

  oidc:
    privateKeyPath: /data/oidc/key.pem
    publicKeyPath: /data/oidc/key.pub

  ui:
    title: damians.cloud

service:
  port: 3000

persistence:
  enabled: true
  storageClass: longhorn
  accessMode: ReadWriteOnce
  size: 1Gi
  mountPath: /data

ingress:
  enabled: true
  className: traefik
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-cloudflare
  hosts:
    - host: auth.damians.cloud
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: tinyauth-tls
      hosts:
        - auth.damians.cloud

env:
  - name: TINYAUTH_AUTH_USERS
    valueFrom:
      secretKeyRef:
        name: tinyauth-secret
        key: USERS

resources:
  requests:
    cpu: 50m
    memory: 128Mi
```

Die RSA-Keys generiert Tinyauth beim ersten Start selbst unter dem konfigurierten Pfad — du musst nichts vorbereiten, das PVC muss nur da sein, sonst sind nach jedem Restart alle OIDC-Sessions tot.

Nach dem Sync: DNS-Record für `auth.damians.cloud` in Technitium/Cloudflare setzen, einloggen, testen ob `https://auth.damians.cloud/.well-known/openid-configuration` sauber JSON liefert. Das ist dein Gate für Schritt 4.

## 4. OIDC-Clients

## 4.1 Client erzeugen

```bash
docker run -i -t --rm ghcr.io/tinyauthapp/tinyauth:v5 oidc create karakeep
```

Output notieren:
```
Client ID: <client-id>
Client Secret: ta-<secret>
```

## 4.2 SealedSecret anlegen

Nur das Secret muss verschlüsselt werden — die Client-ID ist nicht geheim und kann plain in die values. Beide Apps liegen im Namespace `tools`, also reicht **ein** Secret für beide Seiten.

**Neue Datei:** `infrastructure/secrets/tinyauth-oidc.yaml`

```powershell
$sealed = kubectl create secret generic tinyauth-oidc -n tools `
  --from-literal=KARAKEEP='ta-<secret>' `
  --dry-run=client -o yaml |
  kubeseal --format=yaml --controller-name=sealed-secrets --controller-namespace=sealed-secrets |
  Out-String

[System.IO.File]::WriteAllText("$PWD\infrastructure\secrets\tinyauth-oidc.yaml", $sealed.TrimEnd() + "`n", [System.Text.Encoding]::ASCII)


kubectl create secret generic tinyauth-oidc -n tools `
  --from-literal=OPENWEBUI='ta-<secret>' `
  --dry-run=client -o json |
  kubeseal --format=yaml `
    --controller-name=sealed-secrets --controller-namespace=sealed-secrets `
    --merge-into infrastructure/secrets/tinyauth-oidc.yaml
```

Für spätere Apps: neue Keys dazu (`--from-literal=OPENWEBUI=...`) und die Datei komplett neu sealen — SealedSecret ersetzt immer das ganze Secret, du brauchst also alle Klartext-Werte auf einmal. Deshalb die Client-Secrets beim Erzeugen irgendwo zwischenspeichern.

## 4.3 Tinyauth-Seite

**Datei:** `apps-manifests/tools/tinyauth/values.yaml` — unter `tinyauth:` ergänzen:

```yaml
  oidc:
    privateKeyPath: /data/oidc/key.pem
    publicKeyPath: /data/oidc/key.pub
    clients:
      - id: karakeep
        clientId: <client-id>
        clientSecretSecretRef:
          name: tinyauth-oidc
          key: KARAKEEP
        trustedRedirectUris: https://karakeep.damians.cloud/api/auth/callback/custom
        name: Karakeep
```

`id` wird intern zum Env-Var-Namen hochgecased (`TINYAUTH_OIDC_CLIENTS_KARAKEEP_*`) — nur Buchstaben, Zahlen, Bindestriche.

## 4.4 Karakeep-Seite

**Datei:** `apps-manifests/tools/karakeep/values.yaml` — im Block `controllers.karakeep.containers.app.env` ergänzen:

```yaml
          OAUTH_WELLKNOWN_URL: https://auth.damians.cloud/.well-known/openid-configuration
          OAUTH_PROVIDER_NAME: damians.cloud
          OAUTH_SCOPE: "openid email profile"
          OAUTH_ALLOW_DANGEROUS_EMAIL_ACCOUNT_LINKING: "true"
          OAUTH_CLIENT_ID: <client-id>
          OAUTH_CLIENT_SECRET:
            valueFrom:
              secretKeyRef:
                name: tinyauth-oidc
                key: KARAKEEP
```

`NEXTAUTH_URL` steht schon richtig drin — daraus leitet Karakeep die Callback-URL ab: die erlaubte Redirect-URL beim Provider muss `<KARAKEEP_ADDRESS>/api/auth/callback/custom` sein. Muss also exakt mit `trustedRedirectUris` übereinstimmen.

`OAUTH_ALLOW_DANGEROUS_EMAIL_ACCOUNT_LINKING` brauchst du, damit dein bestehender Karakeep-Account per E-Mail verknüpft wird statt einen zweiten anzulegen. Setzt voraus, dass in den Tinyauth-values `auth.userAttributes[].email` für `damian` gesetzt ist.

## 4.5 Ausrollen und testen

```bash
git add . && git commit -m "feat: tinyauth oidc for karakeep" && git push
argocd app sync secrets tinyauth karakeep     # oder warten
```

Reihenfolge der Checks:

```bash
curl -s https://auth.damians.cloud/.well-known/openid-configuration | jq .issuer
kubectl -n tools logs deploy/tinyauth | tail -20
```

Sobald der Endpoint 200 liefert: Karakeep aufrufen → „Sign in with damians.cloud" → Tinyauth-Login → Consent-Screen → zurück. Klappt das, kannst du `DISABLE_PASSWORD_AUTH: "true"` bei Karakeep setzen.

Wenn's hakt, sind die zwei üblichen Fehler `redirect_uri mismatch` (Tippfehler in `trustedRedirectUris`) und `invalid_client` (Secret im SealedSecret ≠ Secret in Tinyauth, z.B. weil du zwischendurch `oidc create` nochmal ausgeführt hast).

## 5. Forward-Auth vor die ungeschützten Apps

Traefik erlaubt Middleware-Referenzen über Namespace-Grenzen hinweg standardmäßig **nicht** — dafür müsste `allowCrossNamespace` im Kubernetes-CRD-Provider aktiviert sein, alternativ definiert man die Middleware im selben Namespace. Ich würde letzteres nehmen: pro Namespace eine Middleware, über `extraObjects` im Chart, damit alles in einer Datei bleibt:

```yaml
extraObjects:
  - apiVersion: traefik.io/v1alpha1
    kind: Middleware
    metadata:
      name: tinyauth
      namespace: tools
    spec:
      forwardAuth:
        address: http://tinyauth.tools.svc.cluster.local:3000/api/auth/traefik
        authResponseHeaders:
          - remote-user
          - remote-name
          - remote-email
          - remote-groups
          - remote-sub
          - authorization
  - apiVersion: traefik.io/v1alpha1
    kind: Middleware
    metadata:
      name: tinyauth
      namespace: media
    spec:
      forwardAuth:
        address: http://tinyauth.tools.svc.cluster.local:3000/api/auth/traefik
        authResponseHeaders:
          - remote-user
          - remote-name
          - remote-email
          - remote-groups
          - remote-sub
          - authorization
```

Und dann in `apps-manifests/media/sonarr/values.yaml` nur eine Zeile dazu:

```yaml
ingress:
  app:
    annotations:
      cert-manager.io/cluster-issuer: letsencrypt-cloudflare
      traefik.ingress.kubernetes.io/router.middlewares: media-tinyauth@kubernetescrd
```

Format ist immer `namespace-middlewarename@kubernetescrd`.

## Worauf du bei den *arr-Apps achten musst

Seerr, SABnzbd und die *arrs reden untereinander über Cluster-Service-Namen, nicht über Traefik — die laufen also weiter, auch wenn der Ingress hinter Forward-Auth liegt. Was aber bricht: externe API-Clients und Mobile Apps, die auf `sonarr.damians.cloud` gehen. Falls du sowas nutzt, brauchst du eine Path-ACL (`/api` freigeben) oder eine IP-Bypass-Regel — und dafür musst du dann doch `labelProvider: kubernetes` plus ClusterRole auf `ingresses` nachrüsten, oder die App-ACLs statisch über `tinyauth.apps` in den values definieren. Letzteres ist bei dir GitOps-mäßig sowieso sauberer.

Reihenfolge zum Abarbeiten: Secret → Application + values → Login testen → `.well-known` prüfen → eine App auf OIDC (Karakeep als Testkandidat, dort ist es am unkompliziertesten) → dann Middleware + Sonarr.