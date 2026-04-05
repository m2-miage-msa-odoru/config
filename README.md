# Configuration Repository pour Odoru Microservices

Ce dépôt contient les configurations centralisées pour tous les microservices de l'architecture Odoru.

## Structure

```
config-repo/
├── application.yml              # Configuration commune à tous les services
├── user-service.yml             # Configuration spécifique au user-service
├── api-gateway.yml              # Configuration spécifique à l'API Gateway
├── application-dev.yml          # Configuration commune pour l'environnement dev
├── application-prod.yml         # Configuration commune pour l'environnement prod
└── user-service-prod.yml        # Configuration user-service en production
```

## Utilisation

1. **Pousser ces fichiers dans votre dépôt GitHub**
   ```bash
   git add config-repo/
   git commit -m "Add centralized configuration files"
   git push origin main
   ```

2. **Configurer le Config Server**
   - Modifier les variables d'environnement dans `docker-compose.yml` ou `.env`
   - `CONFIG_GIT_URI`: URL de votre dépôt GitHub
   - `CONFIG_SEARCH_PATHS`: `config-repo` (chemin vers ce dossier)
   - `CONFIG_GIT_USERNAME` et `CONFIG_GIT_PASSWORD`: Credentials GitHub si nécessaire

3. **Accès aux configurations**
   - Format: `http://config-server:8888/{application}/{profile}`
   - Exemple: `http://localhost:8888/user-service/default`
   - Exemple: `http://localhost:8888/user-service/prod`

## Priorité des configurations

1. `{application}-{profile}.yml` (le plus spécifique)
2. `{application}.yml`
3. `application-{profile}.yml`
4. `application.yml` (le moins spécifique)

## Sécurité

⚠️ **IMPORTANT**: Ne jamais commiter de secrets sensibles en clair !

Utilisez plutôt:
- Variables d'environnement
- Chiffrement avec Spring Cloud Config (encrypt/decrypt)
- Services de gestion de secrets (Vault, AWS Secrets Manager, etc.)
