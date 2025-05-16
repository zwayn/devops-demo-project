#!/bin/bash
# Mise à jour des paquets et installation des dépendances

# Vérifier la connexion internet
check_internet() {
    if ! ping -c 1 google.com &> /dev/null; then
        echo "Erreur: Pas de connexion internet ou problèmes DNS"
        echo "Veuillez vérifier votre connexion réseau et les paramètres DNS"
        echo "Vous pouvez essayer: sudo systemctl restart systemd-resolved"
        exit 1
    fi
}

check_internet

# Mise à jour du système
echo "Mise à jour du système..."
sudo apt update -y && sudo apt upgrade -y || {
    echo "Échec de la mise à jour du système"
    exit 1
}

# Installation des dépendances
echo "Installation des dépendances..."
sudo apt install -y \
    curl \
    wget \
    git \
    apt-transport-https \
    ca-certificates \
    gnupg \
    software-properties-common \
    make \
    lsb-release || {
    echo "Échec de l'installation des dépendances"
    exit 1
}

# -------------------------------------------------------------------
# 1. Installation de Docker
# -------------------------------------------------------------------
echo "Vérification de Docker..."
if ! command -v docker &> /dev/null; then
    echo "Installation de Docker..."
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg || {
        echo "Échec de l'import de la clé Docker"
        exit 1
    }
    
    sudo chmod a+r /etc/apt/keyrings/docker.gpg
    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    
    sudo apt update -y
    sudo apt install -y docker-ce docker-ce-cli containerd.io || {
        echo "Échec de l'installation de Docker"
        exit 1
    }
    
    # Ajouter l'utilisateur courant au groupe Docker
    sudo usermod -aG docker $USER
    newgrp docker
else
    echo "Docker est déjà installé"
fi

# -------------------------------------------------------------------
# 2. Installation de Docker Compose
# -------------------------------------------------------------------
echo "Vérification de Docker Compose..."
if ! command -v docker-compose &> /dev/null; then
    echo "Installation de Docker Compose..."
    sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose || {
        echo "Échec du téléchargement de Docker Compose"
        exit 1
    }
    sudo chmod +x /usr/local/bin/docker-compose
else
    echo "Docker Compose est déjà installé"
fi

# -------------------------------------------------------------------
# 3. Installation de kubectl
# -------------------------------------------------------------------
echo "Vérification de kubectl..."
if ! command -v kubectl &> /dev/null; then
    echo "Installation de kubectl..."
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" || {
        echo "Échec du téléchargement de kubectl"
        exit 1
    }
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
    rm kubectl
else
    echo "kubectl est déjà installé"
fi

# -------------------------------------------------------------------
# 4. Installation de Minikube
# -------------------------------------------------------------------
echo "Vérification de Minikube..."
if ! command -v minikube &> /dev/null; then
    echo "Installation de Minikube..."
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 || {
        echo "Échec du téléchargement de Minikube"
        exit 1
    }
    sudo install minikube-linux-amd64 /usr/local/bin/minikube
    rm minikube-linux-amd64
    
    # Démarrer Minikube
    minikube start --driver=docker || {
        echo "Échec du démarrage de Minikube"
        exit 1
    }
else
    echo "Minikube est déjà installé"
fi

# -------------------------------------------------------------------
# 5. Installation de Helm
# -------------------------------------------------------------------
echo "Vérification de Helm..."
if ! command -v helm &> /dev/null; then
    echo "Installation de Helm..."
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash || {
        echo "Échec de l'installation de Helm"
        exit 1
    }
else
    echo "Helm est déjà installé"
fi

# -------------------------------------------------------------------
# 6. Installation d'ArgoCD CLI (optionnel)
# -------------------------------------------------------------------
read -p "Installer ArgoCD CLI? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    echo "Vérification d'ArgoCD CLI..."
    if ! command -v argocd &> /dev/null; then
        echo "Installation d'ArgoCD CLI..."
        curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 || {
            echo "Échec du téléchargement d'ArgoCD CLI"
            exit 1
        }
        sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
        rm argocd-linux-amd64
    else
        echo "ArgoCD CLI est déjà installé"
    fi
fi

# -------------------------------------------------------------------
# Vérification des versions installées
# -------------------------------------------------------------------
echo "Résumé des installations:"
echo -n "Docker: " && docker --version || echo "Non installé"
echo -n "Docker Compose: " && docker-compose --version || echo "Non installé"
echo -n "kubectl: " && kubectl version --client --short 2>/dev/null || echo "Non installé"
echo -n "Minikube: " && minikube version || echo "Non installé"
echo -n "Helm: " && helm version --short || echo "Non installé"
echo -n "Git: " && git --version || echo "Non installé"

if [[ $REPLY =~ ^[Yy]$ ]]; then
    echo -n "ArgoCD: " && argocd version --client --short 2>/dev/null || echo "Non installé"
fi

echo ""
echo "Remarques:"
echo "1. Si Docker a été installé, vous devez redémarrer votre session ou exécuter 'newgrp docker'"
echo "2. Un redémarrage du système est recommandé pour charger le nouveau noyau"
echo "3. Pour résoudre les problèmes de réseau, vérifiez:"
echo "   - La connexion internet"
echo "   - Les paramètres DNS (/etc/resolv.conf)"
echo "   - Le service systemd-resolved (sudo systemctl restart systemd-resolved)"
