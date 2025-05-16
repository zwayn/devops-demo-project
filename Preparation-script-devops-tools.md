#!/bin/bash
# Mise à jour des paquets et installation des dépendances
sudo apt update -y
sudo apt upgrade -y
sudo apt install -y \
    curl \
    wget \
    git \
    apt-transport-https \
    ca-certificates \
    gnupg \
    software-properties-common \
    make \
    lsb-release

# -------------------------------------------------------------------
# 1. Installation de Docker
# -------------------------------------------------------------------
echo "Installing Docker..."
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update -y
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Ajouter l'utilisateur courant au groupe Docker
sudo usermod -aG docker $USER
newgrp docker  # Recharger les groupes sans redémarrer la session

# -------------------------------------------------------------------
# 2. Installation de Docker Compose
# -------------------------------------------------------------------
echo "Installing Docker Compose..."
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# -------------------------------------------------------------------
# 3. Installation de kubectl
# -------------------------------------------------------------------
echo "Installing kubectl..."
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

# -------------------------------------------------------------------
# 4. Installation de Minikube
# -------------------------------------------------------------------
echo "Installing Minikube..."
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64

# Démarrer Minikube (nécessite Docker)
minikube start --driver=docker

# -------------------------------------------------------------------
# 5. Installation de Helm
# -------------------------------------------------------------------
echo "Installing Helm..."
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# -------------------------------------------------------------------
# 6. Installation d'ArgoCD CLI (optionnel)
# -------------------------------------------------------------------
read -p "Install ArgoCD CLI? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    echo "Installing ArgoCD CLI..."
    curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
    sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
    rm argocd-linux-amd64
fi

# -------------------------------------------------------------------
# Vérification des versions installées
# -------------------------------------------------------------------
echo "Installation completed! Versions:"
docker --version
docker-compose --version
kubectl version --client --short
minikube version
helm version --short
git --version
if [[ $REPLY =~ ^[Yy]$ ]]; then
    argocd version --client --short
fi

echo "Reboot or run 'newgrp docker' to apply Docker group permissions."
