# 🌐 Déploiement d'Open WebUI (Docker)

Ce document détaille la configuration et le déploiement d'**Open WebUI** sur le serveur `lnx`. Ce service sert d'interface graphique utilisateur (type ChatGPT) et se connecte directement au serveur d'inférence distant via l'API de `llama.cpp`.

---

## 📋 Prérequis

- Le réseau Docker `proxy-net` doit être créé (voir guide de configuration Traefik).
- Le reverse proxy Traefik doit être opérationnel.
- L'adresse IP ou le nom d'hôte du serveur d'inférence (`ubuntu-minimal`) exécutant `llama.cpp` doit être connu.

---

## 🏗️ Structure des dossiers

Pour conserver une architecture propre, Open WebUI est ajouté à l'arborescence existante dans `~/docker-apps/` :

```

~/docker-apps/
├── traefik/
├── open-webui/
│   └── data/              ← Données persistantes (utilisateurs, historiques)
└── docker-compose.yml     ← Fichier mis à jour incluant le service Open WebUI

```

Création du dossier pour la persistance des données :

```bash
mkdir -p ~/docker-apps/open-webui/data

```

---

## ⚙️ Configuration du `docker-compose.yml`

Voici le bloc à **ajouter** dans votre fichier `~/docker-apps/docker-compose.yml` actuel, juste en dessous du service Traefik.

```bash
nano ~/docker-apps/docker-compose.yml

```

```yaml
services:
  # ... (conserver le bloc du service traefik ici) ...

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    networks:
      - proxy-net
    volumes:
      - /home/${USER}/docker-apps/open-webui/data:/app/backend/data
    environment:
      # Connexion à l'API OpenAI compatible de llama.cpp
      - OPENAI_API_BASE_URL=http://<IP_SERVEUR_INFERENCE>:8080/v1
      - OPENAI_API_KEY=local_no_key_required
      # Désactiver l'envoi de statistiques anonymes (optionnel)
      - ANON_CHUNKS=false
      # Empêcher le téléchargement local de modèles (géré par le serveur d'inférence)
      - ENABLE_OLLAMA_API=false
    labels:
      - "traefik.enable=true"
      # Routage vers votre nom de domaine
      - "traefik.http.routers.openwebui.rule=Host(`openwebui.domain.com`)"
      - "traefik.http.routers.openwebui.entrypoints=websecure"
      - "traefik.http.routers.openwebui.tls=true"
      # Port interne par défaut d'Open WebUI
      - "traefik.http.services.openwebui.loadbalancer.server.port=8080"

networks:
  proxy-net:
    external: true
```

> [!IMPORTANT]
> Pensez à remplacer `<IP_SERVEUR_INFERENCE>` par l'adresse IP de votre serveur `ubuntu-minimal` et `openwebui.domain.com` par votre domaine configuré.

---

## 🚀 Lancement du service

Déployez le conteneur Open WebUI en tâche de fond :

```bash
cd ~/docker-apps/
docker compose up -d open-webui

```

---

## 🔍 Vérification et premier accès

### 1. Statut du conteneur

```bash
docker ps --filter "name=open-webui"

```

### 2. Accès à l'interface

Ouvrez votre navigateur et rendez-vous sur `https://openwebui.domain.com`.

> **NOTE** :
> Le tout premier compte enregistré sur l'interface devient automatiquement le compte **Administrateur**. C'est depuis ce compte que vous pourrez gérer les accès des futurs utilisateurs si nécessaire.

### 3. Validation de la liaison avec l'IA

Une fois connecté, cliquez sur votre profil en bas à gauche ➔ **Paramètres** ➔ **Modèles**. Vos modèles téléchargés sur le serveur d'inférence (**Qwen 2.5** et **Gemma 4**) doivent apparaître automatiquement dans le menu déroulant en haut de la page d'accueil.

---

## 🛠️ Maintenance & Logs

Pour suivre les requêtes et diagnostiquer les erreurs de connexion avec le serveur d'inférence :

```bash
docker logs -f open-webui

```
