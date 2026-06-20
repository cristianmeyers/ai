# 🤖 Configuration du Serveur d'Inférence IA (`ubuntu-minimal`)

Ce document détaille la configuration complète du serveur dédié au calcul d'inférence. Ce système embarque deux cartes graphiques d'architecture **NVIDIA Pascal** (10 Go de VRAM totale) et fait tourner le moteur **`llama.cpp`** compilé pour optimiser la parallélisation sur les deux GPUs.

---

## 🏎️ Phase 1 : Installation des Pilotes NVIDIA et de CUDA 12

> [!CAUTION]
> L'architecture Pascal (série GTX 10xx, P40, P100, etc.) nécessite obligatoirement **CUDA 12**. Les versions supérieures comme CUDA 13 ont définitivement abandonné la compilation hors ligne pour cette architecture.

Mise à jour du système et détection des pilotes :

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y ubuntu-drivers-common

```

Installation du pilote recommandé et du toolkit de développement CUDA 12 :

```bash
# Installation du driver stable
sudo ubuntu-drivers install

# Installation des dépendances de compilation CUDA 12
sudo apt install -y nvidia-cuda-toolkit

```

Redémarrez le serveur pour appliquer les modifications :

```bash
sudo reboot

```

Vérifiez que vos deux cartes graphiques sont bien reconnues :

```bash
nvidia-smi

```

---

## 🛠️ Phase 2 : Compilation de `llama.cpp` (Multi-GPU CUDA)

Pour tirer parti de la totalité des 10 Go de VRAM en répartissant la charge sur les deux cartes Pascal, `llama.cpp` doit être compilé manuellement avec le flag d'architecture Compute Capability adapté (ex: `61` pour les GTX 1080/1070/P40).

### 2.1 Installation des outils de build

```bash
sudo apt install -y git cmake build-essential libcurl4-openssl-dev

```

### 2.2 Récupération et compilation du code source

```bash
git clone [https://github.com/ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp)
cd llama.cpp

# Génération des fichiers de build ciblant l'architecture Pascal (61)
cmake -B build -DGGML_CUDA=ON -DCMAKE_CUDA_ARCHITECTURES=61 -DCMAKE_BUILD_TYPE=Release

# Compilation en parallélisant sur tous les cœurs du processeur
cmake --build build --config Release -j$(nproc)

```

Le binaire exécutable du serveur se trouve désormais dans `./build/bin/llama-server`.

---

## 📦 Phase 3 : Configuration de Hugging Face CLI (`hf`)

Pour récupérer les versions quantifiées GGUF requises par `llama.cpp`, nous installons l'outil officiel Hugging Face en ligne de commande.

### 3.1 Création de l'environnement Python

```bash
sudo apt install -y python3-pip python3-venv
mkdir -p ~/ai-models
cd ~/ai-models
python3 -m venv venv
source venv/bin/activate

```

### 3.2 Installation de l'utilitaire CLI

```bash
pip install --upgrade pip
pip install "huggingface_hub[cli]"

```

Vérification de l'installation :

```bash
huggingface-cli --help

```

---

## 📥 Phase 4 : Téléchargement des modèles (Gemma 4 & Qwen 2.5)

Nous allons rapatrier les deux modèles choisis directement dans notre dossier local.

### 4.1 Modèle Rapide : Qwen 2.5 7B Instruct (Quantifié Q4)

```bash
huggingface-cli download Qwen/Qwen2.5-7B-Instruct-GGUF \
  qwen2.5-7b-instruct-q4_k_m.gguf \
  --local-dir . \
  --local-dir-use-symlinks False

```

### 4.2 Modèle Multimodal : Gemma 4 E4B Vision (Quantifié Q4)

_Gemma 4 requiert à la fois le fichier du modèle de langage et le fichier projecteur de vision (`mmproj`) pour traiter les images._

```bash
# Téléchargement du fichier de poids GGUF principal
huggingface-cli download google/gemma-4-e4b-it \
  gemma-4-e4b-it-q4_k_m.gguf \
  --local-dir . \
  --local-dir-use-symlinks False

# Téléchargement du composant d'analyse d'image (mmproj)
huggingface-cli download google/gemma-4-e4b-it \
  gemma-4-e4b-it-mmproj.bin \
  --local-dir . \
  --local-dir-use-symlinks False

```

---

## 🚀 Phase 5 : Lancement et allocation Multi-GPU

Pour exécuter un modèle en mode Multi-GPU, passez l'argument `-ngl 99` (`--n-gpu-layers`). `llama.cpp` va automatiquement détecter vos deux cartes graphiques et équilibrer le chargement des couches du modèle sur la VRAM disponible.

### Exemple pour exécuter Qwen 2.5 7B :

```bash
~/llama.cpp/build/bin/llama-server \
  --model ~/ai-models/qwen2.5-7b-instruct-q4_k_m.gguf \
  --host 0.0.0.0 \
  --port 8080 \
  --n-gpu-layers 99

```

### Exemple pour exécuter Gemma 4 E4B (Multimodal) :

```bash
~/llama.cpp/build/bin/llama-server \
  --model ~/ai-models/gemma-4-e4b-it-q4_k_m.gguf \
  --mmproj ~/ai-models/gemma-4-e4b-it-mmproj.bin \
  --host 0.0.0.0 \
  --port 8080 \
  --n-gpu-layers 99

```
