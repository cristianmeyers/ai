# 🔁 Déploiement de n8n (Docker)

Ce document détaille la configuration et le déploiement de **n8n** sur le serveur `lnx`, utilisé pour l'automatisation de workflows.

---

## 📋 Prérequis

- Le réseau Docker `proxy-net` doit être créé (voir guide de configuration `lnx`/Traefik).
- Le reverse proxy Traefik doit être opérationnel.
- Une base de données MariaDB accessible (hôte, identifiants, base dédiée).

---

## 🏗️ Structure des dossiers

```text
~/docker-apps/
├── traefik/
└── n8n/
    ├── data/              ← Données persistantes (config, credentials chiffrés)
    └── docker-compose.yml
```

Création du dossier pour la persistance des données :

```bash
mkdir -p ~/docker-apps/n8n/data
```

---

## ⚙️ Configuration du `docker-compose.yml`

```bash
nano ~/docker-apps/n8n/docker-compose.yml
```

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    networks:
      - proxy-net
    volumes:
      - ./data:/home/node/.n8n
    environment:
      - GENERIC_TIMEZONE=Europe/Paris
      - TZ=Europe/Paris
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - WEBHOOK_URL=https://n8n.domain.com/
      - N8N_ENCRYPTION_KEY=votre_cle_secrete_aleatoire_tres_longue
      - N8N_DB_TYPE=mysqldb
      - N8N_DB_MARIADB_HOST=<IP_VM_MARIADB>
      - N8N_DB_MARIADB_PORT=3306
      - N8N_DB_MARIADB_DATABASE=n8n_db
      - N8N_DB_MARIADB_USER=n8n_user
      - N8N_DB_MARIADB_PASSWORD=votre_mot_de_passe_mariadb
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`n8n.domain.com`)"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.routers.n8n.tls=true"
      - "traefik.http.services.n8n.loadbalancer.server.port=5678"

networks:
  proxy-net:
    external: true
```

> [!IMPORTANT]
> Remplacez `<IP_VM_MARIADB>`, les identifiants de connexion, ainsi que `N8N_ENCRYPTION_KEY` par une valeur secrète longue et aléatoire (cette clé chiffre les credentials stockés par n8n — à conserver précieusement et à ne jamais perdre).

---

## 🚀 Lancement du service

```bash
cd ~/docker-apps/n8n
docker compose up -d
```

---

## 🔍 Vérification et premier accès

### 1. Statut du conteneur

```bash
docker ps --filter "name=n8n"
```

### 2. Accès à l'interface

Ouvrez votre navigateur et rendez-vous sur `https://n8n.domain.com`.

---

## 🛠️ Maintenance & Logs

```bash
docker logs -f n8n
```

| Action                      | Commande                                      |
| --------------------------- | --------------------------------------------- |
| **Voir les logs**           | `docker logs -f n8n`                          |
| **Redémarrer le conteneur** | `docker compose restart n8n`                  |
| **Mettre à jour l'image**   | `docker compose pull && docker compose up -d` |
