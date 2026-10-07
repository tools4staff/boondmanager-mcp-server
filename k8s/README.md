# Déploiement Kubernetes (fork In Fine)

Calqué sur `tools4staff/contract` : même cluster, namespace `tools4staff`,
image sur ghcr.io, ingress Traefik avec certificat Let's Encrypt
(cert-manager), hôte **`boond.infine.com`**.

Le workflow `.github/workflows/deploy.yml` se déclenche à chaque push sur
`infine_main` (ou à la main, via *Run workflow*). Il :

1. teste (lint, typecheck, build, tests) ;
2. construit et pousse `ghcr.io/tools4staff/boondmanager-mcp-server:sha-<commit>`
   (et le tag mobile `:infine`) ;
3. remplace l'image dans `deployment.yaml`, crée `boond-mcp-secrets` si les
   secrets GitHub existent, applique les manifestes et attend le rollout ;
4. vérifie `https://boond.infine.com/healthz`.

Les workflows de publication de l'upstream (`release.yml` → npm / MCP Registry /
Docker Hub) ne se déclenchent plus sur les tags ; `docker-publish.yml` et
`dockerhub-description-sync-manual.yml` étaient déjà manuels.

## Ce qui tourne

| Objet | Nom |
|-------|-----|
| Deployment (1 réplique) + Service `:3000` | `boond-mcp` / `boond-mcp-service` |
| ConfigMap | `boond-mcp-config` |
| Secret (relais d'upload, optionnel) | `boond-mcp-secrets` |
| Ingress + certificat | `boond-mcp-ingress` / `boond-mcp-tls` |
| Pull d'image (secret partagé de l'org) | `ghcr-credentials` |

Endpoints publics : `https://boond.infine.com/mcp` (MCP Streamable HTTP),
`/.well-known/oauth-protected-resource` (découverte OAuth, RFC 9728),
`/healthz` (sonde, non authentifiée).

Une seule réplique : les slots d'upload du relais SharePoint vivent en mémoire
du pod. Le transport est sans état (`MCP_HTTP_STATEFUL=false`), donc aucun volume.

## Authentification

Le serveur est une **ressource protégée OAuth2** : il ne détient aucun
secret BoondManager. Chaque client MCP s'authentifie auprès de BoondManager
et envoie `Authorization: Bearer <token Boond>` à chaque requête ; le
serveur le transmet tel quel à l'API. Chaque utilisateur agit donc avec ses
propres droits Boond. Avec `MCP_HTTP_VALIDATE_TOKEN=true`, un token expiré
reçoit un `401 invalid_token`, ce qui déclenche une nouvelle autorisation
côté client. Détails : [`docs/oauth.md`](../docs/oauth.md).

Côté BoondManager : **Administration → Apps → Sécurité**, activer OAuth2,
déclarer les URL de retour des clients MCP utilisés et les API autorisées.

## Prérequis (une fois)

1. **Environnement GitHub `prod`** sur ce dépôt, avec le secret
   `KUBE_CONFIG` (le même kubeconfig que pour `tools4staff/contract`).

2. **Secret d'accès à ghcr.io** : l'image est privée. Le déploiement réutilise
   `ghcr-credentials`, déjà présent dans le namespace et partagé avec launcher,
   cockpit, tenderly et mycv (token `read:packages` sur les packages de l'org).
   Rien à créer tant qu'il existe ; sinon :

   ```bash
   kubectl create secret docker-registry ghcr-credentials \
     --namespace tools4staff --docker-server=ghcr.io \
     --docker-username=<utilisateur> --docker-password=<token read:packages>
   ```

   Un pod en `ImagePullBackOff` signifie que ce token ne couvre pas le package
   `tools4staff/boondmanager-mcp-server`.

3. **Secrets du relais d'upload SharePoint** (`boond-mcp-secrets`), au choix :
   - **à la main** : `cp k8s/secrets.yaml.template k8s/secrets.yaml`, remplir
     (ne pas commiter, le fichier est ignoré), puis
     `kubectl apply -f k8s/secrets.yaml` ;
   - **par la CI** : définir `BOOND_MCP_SHAREPOINT_CLIENT_ID`,
     `BOOND_MCP_SHAREPOINT_CLIENT_SECRET` et `BOOND_MCP_SHAREPOINT_SITE` dans
     les secrets de l'environnement `prod`. Dès que
     `BOOND_MCP_SHAREPOINT_CLIENT_SECRET` y est défini, chaque déploiement
     recrée `boond-mcp-secrets` (et écrase le secret manuel).

   Le tenant Entra ID et la bibliothèque (`Documents`) sont dans
   `configmap.yaml`. L'app Entra ID a besoin de la permission Graph
   `Sites.Selected`, avec le rôle `write` accordé sur le site de transit.
   Sans ce secret le serveur démarre normalement : seul
   `boond_documents_upload_slot` répond que le relais n'est pas configuré (le
   déploiement affiche alors un avertissement).

4. **DNS** : `boond.infine.com` vers l'ingress du cluster (comme
   `contrat.infine.com`).

## Brancher un client MCP

URL du serveur : `https://boond.infine.com/mcp`. Le client découvre le
serveur d'autorisation BoondManager via
`https://boond.infine.com/.well-known/oauth-protected-resource`.

## Déploiement manuel

```bash
cp k8s/secrets.yaml.template k8s/secrets.yaml   # remplir, ne pas commiter
kubectl apply -f k8s/secrets.yaml
sed "s|image: boondmanager-mcp-server:latest|image: ghcr.io/tools4staff/boondmanager-mcp-server:infine|" \
  k8s/deployment.yaml | kubectl apply -f -
kubectl apply -f k8s/namespace.yaml -f k8s/configmap.yaml -f k8s/ingress.yaml
kubectl rollout restart deployment/boond-mcp -n tools4staff
```

## Diagnostic

```bash
kubectl -n tools4staff get pods -l app=boond-mcp
kubectl -n tools4staff logs deploy/boond-mcp        # logs JSON (pino), un corrId par requête
curl -s https://boond.infine.com/healthz
curl -s https://boond.infine.com/.well-known/oauth-protected-resource
```
