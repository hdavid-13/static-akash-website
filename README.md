# ZestNet — Guide de déploiement complet

## Structure du projet
```
zestnet/
├── site/
│   ├── index.html       ← ton site web (modifie ici)
│   └── style.css        ← le style du site
├── Dockerfile
├── deploy.yaml          ← déploiement Akash
└── .github/
    └── workflows/
        └── deploy.yml   ← build automatique
```

---

## ÉTAPE 1 — Créer les comptes nécessaires

1. **GitHub** → https://github.com (pour stocker ton code)
2. **Docker Hub** → https://hub.docker.com (pour stocker l'image)
3. **Akash Console** → https://console.akash.network (pour déployer)

---

## ÉTAPE 2 — Mettre le projet sur GitHub

```bash
# Dans le dossier zestnet/
git init
git add .
git commit -m "premier commit"
git branch -M main
git remote add origin https://github.com/TON_USERNAME/zestnet.git
git push -u origin main
```

---

## ÉTAPE 3 — Ajouter les secrets GitHub

Dans ton repo GitHub → Settings → Secrets and variables → Actions → New repository secret :

| Nom | Valeur |
|-----|--------|
| `DOCKER_USERNAME` | ton username Docker Hub |
| `DOCKER_PASSWORD` | ton mot de passe Docker Hub |

---

## ÉTAPE 4 — Modifier deploy.yaml

Dans `deploy.yaml`, remplace :
```
TON_USERNAME_DOCKERHUB/zestnet:latest
```
par ton vrai username Docker Hub, ex :
```
johndoe/zestnet:latest
```

---

## ÉTAPE 5 — Déployer sur Akash

1. Va sur https://console.akash.network
2. Connecte ton wallet Keplr (avec des AKT)
3. Clique sur "Deploy" → "From SDL"
4. Colle le contenu de `deploy.yaml`
5. Lance le déploiement
6. Note l'URL publique donnée par Akash (ex: `xyz.provider.akash.network`)

---

## ÉTAPE 6 — Configurer Cloudflare

Dans ton dashboard Cloudflare pour `zestnet.org` :

| Type | Nom | Valeur | Proxy |
|------|-----|--------|-------|
| CNAME | `@` | `xyz.provider.akash.network` | ✅ Activé |
| CNAME | `www` | `xyz.provider.akash.network` | ✅ Activé |

---

## MODIFIER TON SITE

Pour mettre à jour ton site :
1. Modifie `site/index.html` ou `site/style.css`
2. Commit & push sur GitHub
3. GitHub Actions rebuild et push l'image automatiquement
4. Sur Akash Console → redémarre le déploiement pour pull la nouvelle image

```bash
git add .
git commit -m "mise à jour du site"
git push
```

C'est tout ! ✅
