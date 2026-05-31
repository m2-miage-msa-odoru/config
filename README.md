# 🎵 Odoru / CallMe — Plateforme de gestion d'un club de danse

Projet **M2 MIAGE ITN — AMSC** (Applications Microservices sur Cloud).
Architecture microservices orchestrée avec **Spring Cloud** + **Docker Compose**.

Ce repo contient les **configurations centralisées** servies par Spring Cloud Config Server. Les instructions ci-dessous permettent à tout collaborateur de lancer la plateforme complète en local en quelques minutes.

---

## 📦 Architecture en un coup d'œil

```
                              ┌──────────────────┐
                Client ──────►│  Gateway :8080   │ ◄── seule porte d'entrée
                              └────────┬─────────┘
                                       │
                ┌──────────────────────┼─────────────────────┐
                ▼                      ▼                     ▼
        ┌─────────────┐        ┌─────────────┐       ┌──────────────┐
        │ user-service│        │cours-service│       │  composite-  │
        │  (Postgres) │        │  (MongoDB)  │       │   service    │
        └─────────────┘        └─────────────┘       └──────┬───────┘
                                                            │
                                              orchestre user + cours
                                                  via Feign

        Infra : Eureka (registry), Config-Server (configs),
                RabbitMQ (events), Postgres, MongoDB
```

---

## 🧩 Description des services

| Service | Rôle | Port | DB / Dépendances |
|---------|------|------|------------------|
| **eureka-server** | Annuaire / Service discovery — chaque microservice s'y enregistre | 8761 | — |
| **config-server** | Distribue les configs YAML depuis ce repo Git | 8888 | GitHub `m2-miage-msa-odoru/config` |
| **api-gateway** | Point d'entrée unique — route vers les services métier, CORS, circuit breaker | 8080 | Eureka, Config |
| **user-service** | Gestion des membres (CRUD, login, rôles : ADHERENT / ENSEIGNANT / SECRETAIRE / PRESIDENT) | interne | Postgres |
| **cours-service** | Gestion des créneaux de cours et stockage en MongoDB | interne | MongoDB |
| **composite-service** | Orchestrateur — appelle user + cours pour la création/consultation des cours avec vérification du rôle et du niveau | interne | user + cours |
| **postgres** | Base relationnelle pour user-service | 5432 | — |
| **mongo** | Base NoSQL pour cours-service | 27017 | — |
| **rabbitmq** | Bus d'événements (futur Mouvement) | 5672 / 15672 (UI) | — |

> 🔒 Seule la gateway (port 8080) est exposée pour les clients. Les services métier sont uniquement accessibles depuis l'intérieur du réseau Docker.

---

## 🚀 Lancer le projet en 5 étapes

### Prérequis

- **Docker Desktop** (Windows/Mac) ou **Docker Engine + Docker Compose v2** (Linux)
- 4 Go de RAM dispo pour Docker
- `git` et `curl` installés

### 1️⃣ Cloner les repos

Tous les composants sont dans l'organisation [`m2-miage-msa-odoru`](https://github.com/m2-miage-msa-odoru).

```bash
mkdir odoru && cd odoru

# Microservices
git clone https://github.com/m2-miage-msa-odoru/ms-user.git
git clone https://github.com/m2-miage-msa-odoru/ms-cours.git
git clone https://github.com/m2-miage-msa-odoru/ms-composite.git

# Infrastructure
git clone https://github.com/m2-miage-msa-odoru/infra-annuaire-eureka.git
git clone https://github.com/m2-miage-msa-odoru/infra-config-server.git
git clone https://github.com/m2-miage-msa-odoru/infra-apiGatway.git

# Orchestration Docker
git clone https://github.com/m2-miage-msa-odoru/docker-compose.git
```

Tu dois obtenir cette arborescence :
```
odoru/
├── docker-compose/         (contient docker-compose.yml et .env.example)
├── infra-annuaire-eureka/
├── infra-config-server/
├── infra-apiGatway/
├── ms-user/
├── ms-cours/
└── ms-composite/
```

### 2️⃣ Créer le fichier `.env`

À la racine de `docker-compose/`, copie le template et adapte les valeurs :

```bash
cd docker-compose
cp .env.example .env
```

Le contenu minimum nécessaire :
```env
# Config Server (repo public, pas de credentials)
CONFIG_GIT_URI=https://github.com/m2-miage-msa-odoru/config.git
CONFIG_SEARCH_PATHS=.
CONFIG_GIT_BRANCH=main
CONFIG_GIT_USERNAME=
CONFIG_GIT_PASSWORD=

# Postgres (admin du serveur + DB ms-user)
POSTGRES_USER=user
POSTGRES_PASSWORD=password
POSTGRES_DB=msuser

# Identifiants utilisés par ms-user pour se connecter à Postgres
MS_USER_DB=msuser
MS_USER_DB_USERNAME=user
MS_USER_DB_PASSWORD=password

# RabbitMQ
RABBITMQ_USER=msuser_admin
RABBITMQ_PASSWORD=msuser_mdp

# Eureka
EUREKA_URL=http://eureka-server:8761/eureka/
```

### 3️⃣ Lancer la stack

Toujours depuis `docker-compose/` :

```bash
docker compose up -d --build
```

⏱️ **Premier démarrage** : 5 à 10 min (téléchargement des images + build Maven).
Les fois suivantes : ~30 s.

### 4️⃣ Vérifier que tout tourne

```bash
docker compose ps
```

Tu dois voir **tous les services en `(healthy)`**. Ouvre aussi dans le navigateur :

| URL | Ce que tu dois voir |
|-----|---------------------|
| http://localhost:8761 | Dashboard **Eureka** — liste de tous les services enregistrés |
| http://localhost:8888/user-service/default | Config YAML de user-service en JSON |
| http://localhost:15672 | Console **RabbitMQ** (login : `msuser_admin` / `msuser_mdp`) |

### 5️⃣ Tester un endpoint

Tous les appels client passent par la gateway (`http://localhost:8080`).

```bash
# Créer un membre
curl -X POST "http://localhost:8080/api/users/api/v1/membres/" \
  -H "Content-Type: application/json" \
  -d '{
    "email":"alice@callme.fr","nom":"Dupont","prenom":"Alice","motDePasse":"azerty",
    "adresse":{"pays":"France","ville":"Toulouse","codePostal":31000,"numero_et_voie":"12 rue de la Danse"}
  }'

# Lister tous les membres
curl -s "http://localhost:8080/api/users/api/v1/membres/" | jq .
```

🎉 Si tu vois Alice, c'est gagné.

---

## 🔌 Endpoints principaux (via gateway)

| Service | Méthode | URL | Description |
|---------|---------|-----|-------------|
| user | POST | `/api/users/api/v1/membres/` | Créer un membre |
| user | PATCH | `/api/users/api/v1/membres/{email}?roleMembre=X&niveau_expertise=N` | Mettre à jour rôle + niveau |
| user | GET | `/api/users/api/v1/membres/` | Lister tous |
| user | GET | `/api/users/api/v1/membres/{email}` | Récupérer par email |
| user | POST | `/api/users/api/v1/membres/login` | Authentification |
| cours | POST | `/api/cours/api/v1/cours/create` | Créer cours (batch) |
| cours | GET | `/api/cours/api/v1/cours/by-enseignant/{email}` | Cours d'un enseignant |
| cours | GET | `/api/cours/api/v1/cours/by-niveau/{niveau}` | Cours par niveau |
| composite | POST | `/api/composite/api/v1/composite/cours/` | Créer cours (avec check rôle + niveau) |
| composite | GET | `/api/composite/api/v1/composite/cours/by-enseignant/{email}` | Cours d'un enseignant |
| composite | GET | `/api/composite/api/v1/composite/cours/by-eleve/{email}` | Cours du niveau de l'élève |

---

## 🛠️ Commandes utiles

```bash
# Suivre les logs en live d'un service
docker compose logs -f composite-service

# Voir les 50 dernières lignes d'erreur
docker compose logs --tail=50 user-service

# Tout arrêter (sans perdre les données)
docker compose down

# Tout arrêter ET reset des volumes (perd DB Postgres + Mongo)
docker compose down -v

# Rebuild + relance d'un seul service après modif du code
docker compose up -d --build user-service

# Inspecter le contenu de Postgres
docker exec -it odoru-postgres psql -U user -d msuser \
  -c "SELECT email, role FROM user_entity;"

# Inspecter le contenu de MongoDB
docker exec -it odoru-mongo mongosh cours_db \
  --eval "db.cours.find().pretty()"

# Lister les routes actives sur la gateway
curl -s http://localhost:8080/actuator/gateway/routes | jq '.[].id'
```

---

## 📂 Contenu de ce repo (config-repo)

| Fichier | Sert à |
|---------|--------|
| `api-gateway.yml` | Config Spring Cloud Gateway (routes minimales) |
| `user-service.yml` | Datasource Postgres + RabbitMQ pour user-service |
| `cours-service` | Mongo + RabbitMQ pour cours-service |
| `composite-service.yml` | Eureka URL pour composite-service |

> 🔁 Toute modification poussée ici est récupérée par les services à leur prochain démarrage (`docker compose restart <service>`).

---

## ⚠️ Dépannage rapide

| Symptôme | Cause probable | Fix |
|----------|---------------|-----|
| Postgres crashe au boot avec `Is a directory` | Le mount `init.sql` cible un dossier | `touch init.sql` à la racine puis `docker compose down -v && up -d` |
| RabbitMQ ne démarre pas (pas de logs) | Variables `RABBITMQ_USER` / `PASSWORD` vides dans `.env` | Vérifier `.env` (pas d'espaces avant `=`) |
| `composite-service` 503 `No servers available` | Cache Eureka pas encore rafraîchi | Attendre 30s ou `docker compose restart composite-service` |
| `Connection refused` sur `localhost:8081/8082/8083` | C'est normal : ports non exposés à l'extérieur (seule la gateway l'est) | Utiliser `http://localhost:8080/api/...` |
| Build Maven coince sur `mockito-mock-maker` en lançant les tests | JDK 21 bloque l'agent attachment | Le fichier `mockito-extensions/org.mockito.plugins.MockMaker` est déjà fourni dans `src/test/resources` |

---

## 👥 Équipe

Projet réalisé en trinôme dans le cadre du M2 MIAGE ITN — Université de Toulouse.
Encadrants : Jonathan Detrier, Cédric Teyssié, Patrice Torguet.

Soutenance : **17 juin 2026**.
