# ci-cd-helm

Bibliothèque de workflows GitHub Actions **réutilisables** pour applications MERN (React + Node.js + MongoDB) déployées sur Kubernetes via Helm.
 
Ces workflows sont conçus pour être appelés depuis n'importe quel repo applicatif via `workflow_call`, sans duplication de logique CI/CD.
 
---
 
## Vue d'ensemble
 
```
feature/** / develop          main                  tag vX.Y.Z-rc       tag vX.Y.Z
─────────────────────         ────────────────────  ─────────────────   ──────────────────
secret-scan                   secret-scan           secret-scan         secret-scan
lint-test                     lint-test             lint-test           lint-test
                              build-backend         build-backend       build-backend
                              build-frontend        build-frontend      build-frontend
                              ayur-smoke (k3d)
                                                    deploy → preprod    deploy → prod
                                                                        (approbation requise)
```
 
---
 
## Workflows disponibles
 
### `reusable-secret-scan.yml`
 
Scan de secrets dans l'historique Git complet avec [Gitleaks](https://github.com/gitleaks/gitleaks).
 
**Déclencheur :** `workflow_call` — aucun paramètre requis.
 
**Utilisation :**
```yaml
jobs:
  secret-scan:
    uses: roxane451/ci-cd-helm/.github/workflows/reusable-secret-scan.yml@main
    secrets: inherit
```
 
---
 
### `reusable-lint-test.yml`
 
TypeScript check, ESLint, tests Jest (backend + frontend) et `npm audit` sur les deux workspaces.
 
**Déclencheur :** `workflow_call` — aucun paramètre requis.
 
**Node.js :** version 22 LTS (alignée avec les images Docker de production).
 
**Utilisation :**
```yaml
jobs:
  lint-test:
    uses: roxane451/ci-cd-helm/.github/workflows/reusable-lint-test.yml@main
    secrets: inherit
```
 
> Le workflow installe les dépendances depuis la racine (`npm ci`) pour supporter les npm workspaces. Le `package-lock.json` doit être à la racine du repo applicatif.
 
---
 
### `reusable-build-push.yml`
 
Build Docker multi-stage, scan CVE Trivy et push vers GHCR.
 
**Déclencheur :** `workflow_call`
 
| Paramètre | Type | Description |
|-----------|------|-------------|
| `image-name` | `string` | Nom de l'image — ex: `backend`, `frontend` |
| `context` | `string` | Contexte Docker — ex: `./backend`, `./frontend` |
 
**Output :**
 
| Nom | Description |
|-----|-------------|
| `image-tag` | Tag principal poussé (`sha-<short>` sur branche, `vX.Y.Z` sur tag git) |
 
**Image produite :** `ghcr.io/<owner>/<image-name>:<tag>`
 
**Comportement :**
- Sur push de **branche** → tag `sha-<short>`
- Sur push de **tag git** → tag sémantique (`v1.2.3` ou `v1.2.3-rc`)
- Trivy bloque le build si une CVE `CRITICAL` ou `HIGH` avec un fix disponible est détectée
- Le résultat Trivy est uploadé en SARIF dans l'onglet Security du repo
- Trivy scanne le digest exact de l'image buildée localement (via `iidfile`) — pas un tag registry potentiellement stale
- Le cache BuildKit est partagé entre les runs via GitHub Actions cache
**Utilisation :**
```yaml
jobs:
  build-backend:
    needs: [secret-scan, lint-test]
    uses: roxane451/ci-cd-helm/.github/workflows/reusable-build-push.yml@main
    with:
      image-name: backend
      context: ./backend
    secrets: inherit
 
  build-frontend:
    needs: [secret-scan, lint-test]
    uses: roxane451/ci-cd-helm/.github/workflows/reusable-build-push.yml@main
    with:
      image-name: frontend
      context: ./frontend
    secrets: inherit
```
 
---
 
### `reusable-ayur-smoke-k3d.yml`
 
Déploiement dans un cluster k3d éphémère sur runner **self-hosted**, smoke test HTTP sur `/api/health`, puis suppression du cluster.
 
**Déclencheur :** `workflow_call`
 
| Paramètre | Type | Description |
|-----------|------|-------------|
| `image-tag` | `string` | Tag des images à déployer |
 
**Prérequis runner self-hosted :** `k3d`, `kubectl`, `helm` installés.
 
**Utilisation :**
```yaml
jobs:
  ayur-smoke:
    needs: [build-backend, build-frontend]
    uses: roxane451/ci-cd-helm/.github/workflows/reusable-ayur-smoke-k3d.yml@main
    with:
      image-tag: ${{ needs.build-backend.outputs.image-tag }}
    secrets: inherit
```
 
---
 
### `reusable-helm-deploy.yml`
 
Déploiement Helm sur cluster K3s via `helm upgrade --install` avec `--atomic` (rollback automatique en cas d'échec).
 
**Déclencheur :** `workflow_call`
 
| Paramètre | Type | Description |
|-----------|------|-------------|
| `environment` | `string` | `preprod` ou `prod` |
| `image-tag` | `string` | Tag des images à déployer |
| `helm-values-file` | `string` | Chemin du fichier values dans `ayur-veda-helm` — ex: `helm/values-preprod.yaml` |
 
| Secret | Description |
|--------|-------------|
| `KUBECONFIG_B64` | Kubeconfig encodé en base64 |
 
Le workflow checkout automatiquement le repo `ayur-veda-helm` pour récupérer les values files.
 
**Utilisation :**
```yaml
jobs:
  deploy-preprod:
    needs: [build-backend, build-frontend]
    uses: roxane451/ci-cd-helm/.github/workflows/reusable-helm-deploy.yml@main
    with:
      environment: preprod
      image-tag: ${{ github.ref_name }}
      helm-values-file: helm/values-preprod.yaml
    secrets: inherit
 
  deploy-prod:
    needs: [build-backend, build-frontend]
    uses: roxane451/ci-cd-helm/.github/workflows/reusable-helm-deploy.yml@main
    with:
      environment: prod
      image-tag: ${{ github.ref_name }}
      helm-values-file: helm/values-production.yaml
    secrets: inherit
```
 
> L'approbation manuelle en production est configurée via **GitHub Environments** — le job attend une validation humaine avant d'exécuter le déploiement.
 
---
 
## Intégration dans un repo applicatif
 
Exemple de workflow complet (`ci-cd.yml`) reproduisant la stratégie de branchement ci-dessus :
 
```yaml
name: CI/CD
 
on:
  push:
    branches:
      - "feature/**"
      - develop
      - main
    tags:
      - "v[0-9]+.[0-9]+.[0-9]+-rc"
      - "v[0-9]+.[0-9]+.[0-9]+"
 
jobs:
  secret-scan:
    uses: roxane451/ci-cd-helm/.github/workflows/reusable-secret-scan.yml@main
    secrets: inherit
 
  lint-test:
    uses: roxane451/ci-cd-helm/.github/workflows/reusable-lint-test.yml@main
    secrets: inherit
 
  build-backend:
    if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/')
    needs: [secret-scan, lint-test]
    uses: roxane451/ci-cd-helm/.github/workflows/reusable-build-push.yml@main
    with:
      image-name: backend
      context: ./backend
    secrets: inherit
 
  build-frontend:
    if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/')
    needs: [secret-scan, lint-test]
    uses: roxane451/ci-cd-helm/.github/workflows/reusable-build-push.yml@main
    with:
      image-name: frontend
      context: ./frontend
    secrets: inherit
 
  ayur-smoke:
    if: github.ref == 'refs/heads/main'
    needs: [build-backend, build-frontend]
    uses: roxane451/ci-cd-helm/.github/workflows/reusable-ayur-smoke-k3d.yml@main
    with:
      image-tag: ${{ needs.build-backend.outputs.image-tag }}
    secrets: inherit
 
  deploy-preprod:
    if: startsWith(github.ref, 'refs/tags/') && contains(github.ref, '-rc')
    needs: [build-backend, build-frontend]
    uses: roxane451/ci-cd-helm/.github/workflows/reusable-helm-deploy.yml@main
    with:
      environment: preprod
      image-tag: ${{ github.ref_name }}
      helm-values-file: helm/values-preprod.yaml
    secrets: inherit
 
  deploy-prod:
    if: startsWith(github.ref, 'refs/tags/') && !contains(github.ref, '-rc')
    needs: [build-backend, build-frontend]
    uses: roxane451/ci-cd-helm/.github/workflows/reusable-helm-deploy.yml@main
    with:
      environment: prod
      image-tag: ${{ github.ref_name }}
      helm-values-file: helm/values-production.yaml
    secrets: inherit
```
 
---
 
## Secrets requis
 
Les secrets sont transmis via `secrets: inherit`. Ils doivent être configurés dans le repo applicatif ou au niveau de l'organisation.
 
| Secret | Utilisé par | Description |
|--------|-------------|-------------|
| `PAT_HELM_REPO` | tous | Automatiquement disponible — push GHCR, upload SARIF |
| `KUBECONFIG_B64` | `reusable-helm-deploy` | Kubeconfig du cluster cible encodé en base64 |
 
Pour générer `KUBECONFIG_B64` :
```bash
cat ~/.kube/config | base64 -w 0
```
 
