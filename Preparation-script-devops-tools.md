#!/bin/bash
# Script d'installation pour Docker, Kubernetes et outils associés avec gestion d'erreurs améliorée

# Couleurs pour les messages
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Fonction pour vérifier la connexion internet
check_internet() {
    echo -e "${YELLOW}Vérification de la connexion internet...${NC}"
    if ! ping -c 1 google.com &> /dev/null; then
        echo -e "${RED}Erreur: Pas de connexion internet ou problèmes DNS${NC}"
        echo "Veuillez vérifier:"
        echo "1. Votre connexion réseau"
        echo "2. Les paramètres DNS (/etc/resolv.conf)"
        echo "3. Le service systemd-resolved (sudo systemctl restart systemd-resolved)"
        exit 1
    fi
}

# Fonction pour vérifier et installer les paquets
install_package() {
    local pkg=$1
    echo -e "${YELLOW}Vérification de $pkg...${NC}"
    if ! dpkg -l | grep -q " $pkg "; then
        echo -e "${YELLOW}Installation de $pkg...${NC}"
        sudo apt install -y $pkg || {
            echo -e "${RED}Échec de l'installation de $pkg${NC}"
            exit 1
        }
    else
        echo -e "${GREEN}$pkg est déjà installé${NC}"
    fi
}

# -------------------------------------------------------------------
# Début de l'installation
# -------------------------------------------------------------------
clear
echo -e "${GREEN}Début de l'installation des outils Docker et Kubernetes...${NC}"

# 1. Vérification des prérequis
check_internet

# 2. Mise à jour du système
echo -e "${YELLOW}Mise à jour du système...${NC}"
sudo apt update -y && sudo apt upgrade -y || {
    echo -e "${RED}Échec de la mise à jour du système${NC}"
    exit 1
}

# 3. Installation des dépendances
echo -e "${YELLOW}Installation des dépendances...${NC}"
DEPENDENCIES=(
    curl
    wget
    git
    apt-transport-https
    ca-certificates
    gnupg
    software-properties-common
    make
    lsb-release
)

for pkg in "${DEPENDENCIES[@]}"; do
    install_package $pkg
done

# -------------------------------------------------------------------
# Installation de Docker
# -------------------------------------------------------------------
echo -e "${YELLOW}Vérification de Docker...${NC}"
if ! command -v docker &> /dev/null; then
    echo -e "${YELLOW}Installation de Docker...${NC}"
    
    # Ajout du dépôt Docker
    sudo install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg || {
        echo -e "${RED}Échec de l'import de la clé Docker${NC}"
        exit 1
    }
    
    sudo chmod a+r /etc/apt/keyrings/docker.gpg
    echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    
    sudo apt update -y
    sudo apt install -y docker-ce docker-ce-cli containerd.io || {
        echo -e "${RED}Échec de l'installation de Docker${NC}"
        exit 1
    }
    
    # Configuration des permissions Docker
    echo -e "${YELLOW}Configuration des permissions Docker...${NC}"
    sudo usermod -aG docker $USER || {
        echo -e "${RED}Échec de l'ajout de l'utilisateur au groupe docker${NC}"
        exit 1
    }
    
    # Test de l'installation Docker
    docker run hello-world && echo -e "${GREEN}Docker installé avec succès${NC}" || {
        echo -e "${RED}Échec de la vérification de Docker${NC}"
        exit 1
    }
else
    echo -e "${GREEN}Docker est déjà installé${NC}"
fi

# -------------------------------------------------------------------
# Installation de Docker Compose
# -------------------------------------------------------------------
echo -e "${YELLOW}Vérification de Docker Compose...${NC}"
if ! command -v docker-compose &> /dev/null; then
    echo -e "${YELLOW}Installation de Docker Compose...${NC}"
    sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
    -o /usr/local/bin/docker-compose || {
        echo -e "${RED}Échec du téléchargement de Docker Compose${NC}"
        exit 1
    }
    sudo chmod +x /usr/local/bin/docker-compose
    
    # Vérification de l'installation
    docker-compose --version && echo -e "${GREEN}Docker Compose installé avec succès${NC}" || {
        echo -e "${RED}Échec de l'installation de Docker Compose${NC}"
        exit 1
    }
else
    echo -e "${GREEN}Docker Compose est déjà installé${NC}"
fi

# -------------------------------------------------------------------
# Installation de kubectl
# -------------------------------------------------------------------
echo -e "${YELLOW}Vérification de kubectl...${NC}"
if ! command -v kubectl &> /dev/null; then
    echo -e "${YELLOW}Installation de kubectl...${NC}"
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" || {
        echo -e "${RED}Échec du téléchargement de kubectl${NC}"
        exit 1
    }
    sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
    rm kubectl
    
    # Vérification de l'installation
    kubectl version --client --short && echo -e "${GREEN}kubectl installé avec succès${NC}" || {
        echo -e "${RED}Échec de l'installation de kubectl${NC}"
        exit 1
    }
else
    echo -e "${GREEN}kubectl est déjà installé${NC}"
fi

# -------------------------------------------------------------------
# Installation de Minikube
# -------------------------------------------------------------------
echo -e "${YELLOW}Vérification de Minikube...${NC}"
if ! command -v minikube &> /dev/null; then
    echo -e "${YELLOW}Installation de Minikube...${NC}"
    curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 || {
        echo -e "${RED}Échec du téléchargement de Minikube${NC}"
        exit 1
    }
    sudo install minikube-linux-amd64 /usr/local/bin/minikube
    rm minikube-linux-amd64
    
    # Démarrer Minikube
    echo -e "${YELLOW}Démarrage de Minikube...${NC}"
    if [ "$(id -u)" -eq 0 ]; then
        echo -e "${RED}Attention : Minikube ne peut pas être démarré en tant que root${NC}"
        echo -e "${YELLOW}Utilisez un utilisateur normal avec la commande suivante :${NC}"
        echo "  minikube start --driver=docker"
    else
        # Vérifier que l'utilisateur est dans le groupe docker
        if ! groups $USER | grep -q '\bdocker\b'; then
            echo -e "${RED}L'utilisateur doit être dans le groupe docker pour utiliser Minikube${NC}"
            echo "Exécutez la commande suivante puis reconnectez-vous :"
            echo "  sudo usermod -aG docker $USER"
            exit 1
        fi
        
        minikube start --driver=docker || {
            echo -e "${RED}Échec du démarrage de Minikube${NC}"
            echo -e "${YELLOW}Solutions alternatives :${NC}"
            echo "1. Essayez avec un autre driver : minikube start --driver=podman"
            echo "2. Ou installez un driver comme kvm2 : https://minikube.sigs.k8s.io/docs/drivers/"
            exit 1
        }
        echo -e "${GREEN}Minikube démarré avec succès${NC}"
    fi
else
    echo -e "${GREEN}Minikube est déjà installé${NC}"
fi

# -------------------------------------------------------------------
# Installation de Helm
# -------------------------------------------------------------------
echo -e "${YELLOW}Vérification de Helm...${NC}"
if ! command -v helm &> /dev/null; then
    echo -e "${YELLOW}Installation de Helm...${NC}"
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash || {
        echo -e "${RED}Échec de l'installation de Helm${NC}"
        exit 1
    }
    
    # Vérification de l'installation
    helm version --short && echo -e "${GREEN}Helm installé avec succès${NC}" || {
        echo -e "${RED}Échec de l'installation de Helm${NC}"
        exit 1
    }
else
    echo -e "${GREEN}Helm est déjà installé${NC}"
fi

# -------------------------------------------------------------------
# Installation optionnelle d'ArgoCD CLI
# -------------------------------------------------------------------
read -p "Installer ArgoCD CLI? (y/n) " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    echo -e "${YELLOW}Vérification d'ArgoCD CLI...${NC}"
    if ! command -v argocd &> /dev/null; then
        echo -e "${YELLOW}Installation d'ArgoCD CLI...${NC}"
        curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 || {
            echo -e "${RED}Échec du téléchargement d'ArgoCD CLI${NC}"
            exit 1
        }
        sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
        rm argocd-linux-amd64
        
        # Vérification de l'installation
        argocd version --client --short && echo -e "${GREEN}ArgoCD CLI installé avec succès${NC}" || {
            echo -e "${RED}Échec de l'installation d'ArgoCD CLI${NC}"
            exit 1
        }
    else
        echo -e "${GREEN}ArgoCD CLI est déjà installé${NC}"
    fi
fi

# -------------------------------------------------------------------
# Résumé de l'installation
# -------------------------------------------------------------------
echo -e "${GREEN}\nInstallation terminée. Résumé :${NC}"
echo "----------------------------------------"
echo -n "Docker:          " && docker --version 2>/dev/null || echo "Non installé"
echo -n "Docker Compose:  " && docker-compose --version 2>/dev/null || echo "Non installé"
echo -n "kubectl:         " && kubectl version --client --short 2>/dev/null || echo "Non installé"
echo -n "Minikube:        " && minikube version 2>/dev/null || echo "Non installé"
echo -n "Helm:            " && helm version --short 2>/dev/null || echo "Non installé"
echo -n "Git:             " && git --version 2>/dev/null || echo "Non installé"

if [[ $REPLY =~ ^[Yy]$ ]]; then
    echo -n "ArgoCD:          " && argocd version --client --short 2>/dev/null || echo "Non installé"
fi

echo "----------------------------------------"
echo -e "${YELLOW}Remarques importantes :${NC}"
echo "1. Pour que les permissions Docker soient effectives, vous devez :"
echo "   - Soit vous déconnecter et vous reconnecter"
echo "   - Soit exécuter la commande : newgrp docker"
echo ""
echo "2. Si Minikube n'a pas pu démarrer (surtout si exécuté en root) :"
echo "   - Exécutez en tant qu'utilisateur normal : minikube start --driver=docker"
echo "   - Ou choisissez un autre driver : minikube start --driver=podman"
echo ""
echo "3. Pour vérifier le cluster Minikube : kubectl get nodes"
echo ""
echo -e "${GREEN}Tous les outils ont été installés avec succès!${NC}"
