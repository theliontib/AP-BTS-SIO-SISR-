# Installation de GLPI avec Docker sur un serveur Linux

**Auteur :** Thibaut HUET  
**Date :** Juin 2026  
**Contexte :** Projet AP BTS SIO — Infrastructure GSB

---

## Table des matières

1. [Installation de la VM Linux (Ubuntu Server)](#1-installation-de-la-vm-linux-ubuntu-server)
2. [Configuration réseau](#2-configuration-réseau)
3. [Installation de Docker](#3-installation-de-docker)
4. [Déploiement de GLPI avec Docker Compose](#4-déploiement-de-glpi-avec-docker-compose)
5. [Configuration initiale de GLPI](#5-configuration-initiale-de-glpi)
6. [Liaison LDAP avec l'Active Directory](#6-liaison-ldap-avec-lactive-directory)
7. [Importation des utilisateurs AD](#7-importation-des-utilisateurs-ad)
8. [Intégration du serveur Linux à l'AD (Kerberos)](#8-intégration-du-serveur-linux-à-lad-kerberos)
9. [Tests et validation](#9-tests-et-validation)
10. [Problèmes rencontrés et solutions](#10-problèmes-rencontrés-et-solutions)
11. [Conclusion](#11-conclusion)

---

## 1. Installation de la VM Linux (Ubuntu Server)

### Création de la VM

- Utiliser l'ISO : `ubuntu-26.04-live-server-amd64` (ou version équivalente)
- Allouer les ressources suivantes :
  - **RAM :** 2 Go minimum
  - **CPU :** 2 vCPU
  - **Disque :** 20 Go
  - **Réseau :** NAT (même réseau que le contrôleur de domaine)

### Vérification du partitionnement

Lors de l'étape **Filesystem Summary**, vérifier que l'espace alloué est bien complet et non limité à 10 Go par défaut :

- Si la ligne indique `[ / 10.000G new ext4 nouveau LVM logical volume ]`
- Aller sur **ubuntu-lv** > **Edit** > Changer pour la valeur maximale (ex : `18.222G` pour une VM de 20 Go)
- **Save**

### Profils de configuration

| Champ | Valeur |
|---|---|
| Identifiant | `administrateur` |
| Mot de passe | `Serveur2022` |

> **Attention :** Utiliser la touche **Verrouillage Maj (Lock Maj)** et non **Shift** pour éviter les erreurs de mot de passe.

![Création du profil administrateur lors de l'installation d'Ubuntu Server](images/image1.png)

### Finalisation de l'installation

1. Terminer l'installation et redémarrer
2. Au message `Please remove the installation medium, then press ENTER` → appuyer sur **Entrée** (VMware démonte l'ISO automatiquement)
3. Se connecter avec : `administrateur` / `Serveur2022`

---

## 2. Configuration réseau

Lors de l'installation, configurer le réseau en manuel :

1. Sélectionner la carte réseau `ens33` avec les flèches et appuyer sur **Entrée**
2. Choisir **Edit IPv4**
3. Passer de `Automatic (DHCP)` à **Manual**
4. Remplir les champs selon le plan d'adressage :

| Champ | Valeur |
|---|---|
| Subnet | `192.168.10.0/24` |
| Address | `192.168.10.30` |
| Gateway | `192.168.10.1` |
| Name servers | `192.168.10.10,8.8.8.8` |

> **Note :** Le premier DNS (`192.168.10.10`) est l'IP du contrôleur de domaine Active Directory.

---

## 3. Installation de Docker

Depuis le serveur Windows, se connecter en SSH au serveur Ubuntu :

```bash
ssh administrateur@192.168.10.30
# Taper "yes" puis le mot de passe
```

![Connexion SSH au serveur Ubuntu pour l'installation de Docker](images/image2.png)

### Mise à jour du système

```bash
sudo apt update && sudo apt upgrade -y
```

### Installation des prérequis

```bash
sudo apt install -y ca-certificates curl gnupg
```

### Ajout de la clé GPG officielle de Docker

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### Ajout du dépôt Docker

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### Installation de Docker Engine et Docker Compose

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Ajout de l'utilisateur au groupe Docker

Pour éviter de taper `sudo` devant chaque commande Docker :

```bash
sudo usermod -aG docker administrateur
```

Se déconnecter puis se reconnecter pour que les changements prennent effet. Vérifier :

```bash
docker ps
```

---

## 4. Déploiement de GLPI avec Docker Compose

### Création du répertoire de travail

```bash
mkdir glpi && cd glpi
```

### Création du fichier docker-compose.yml

```bash
nano docker-compose.yml
```

**Contenu :**

```yaml
services:
  mariadb:
    image: mariadb:10.11
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: glpi
      MYSQL_USER: glpi
      MYSQL_PASSWORD: glpipass
    volumes:
      - mariadb_data:/var/lib/mysql

  glpi:
    image: diouxx/glpi
    ports:
      - "80:80"
    environment:
      MYSQL_HOST: mariadb
      MYSQL_DATABASE: glpi
      MYSQL_USER: glpi
      MYSQL_PASSWORD: glpipass
    depends_on:
      - mariadb

volumes:
  mariadb_data:
```

![Fichier docker-compose.yml pour le déploiement de GLPI](images/image3.png)

### Lancement des conteneurs

```bash
docker compose up -d
```

![Lancement des conteneurs Docker avec docker compose up -d](images/image4.png)

### Vérification

Sur le serveur Windows, ouvrir un navigateur et accéder à :

```
http://192.168.10.30
```

![Interface GLPI accessible depuis le navigateur](images/image5.png)

---

## 5. Configuration initiale de GLPI

1. Ouvrir `http://192.168.10.30` dans un navigateur
2. Se connecter avec les identifiants par défaut :
   - **Identifiant :** `glpi`
   - **Mot de passe :** `glpi`
3. Suivre l'assistant d'installation
4. Configurer la base de données MySQL avec les paramètres définis dans `docker-compose.yml`

---

## 6. Liaison LDAP avec l'Active Directory

### Ajout du serveur LDAP

1. Aller dans **Configuration** → **Authentification** → **Annuaires LDAP** → **Ajouter**
2. Utiliser la préconfiguration **Active Directory**

### Paramètres de connexion LDAP

| Champ | Valeur |
|---|---|
| Nom | `TECHNOLAB.local` |
| Serveur par défaut | Oui |
| Activé | Oui |
| Serveur | `192.168.10.10` |
| Port | `389` |
| BaseDN | `DC=TECHNOLAB,DC=local` |
| DN du compte | `CN=Administrateur,CN=Users,DC=TECHNOLAB,DC=local` |
| Mot de passe | `Serveur2022` |
| Champ identifiant | `samaccountname` |

![Configuration de l'annuaire LDAP dans GLPI](images/image6.png)

### Test de la connexion

Cliquer sur **Test de connexion** dans le menu de gauche pour valider que GLPI communique bien avec l'Active Directory.

![Test de connexion LDAP réussi](images/image7.png)

---

## 7. Importation des utilisateurs AD

1. Aller dans **Administration** → **Utilisateurs**
2. Cliquer sur **Liaison annuaire LDAP** (si absent, redémarrer GLPI)
3. Cliquer sur **Importer de nouveaux utilisateurs**
4. Activer le **mode simplifié**
5. Cliquer sur **Rechercher**
6. Sélectionner les utilisateurs dans la liste
7. Cliquer sur **Actions** → **Importer**

![Importation des utilisateurs depuis l'Active Directory](images/image8.png)

![Sélection et import des utilisateurs AD](images/image9.png)

---

## 8. Intégration du serveur Linux à l'AD (Kerberos)

Pour que le serveur Linux soit correctement intégré à l'Active Directory (résolution DNS, authentification Kerberos), il faut configurer le fichier Netplan et installer les outils d'intégration.

### Configuration du fichier Netplan

Modifier la configuration réseau pour pointer vers le serveur DNS de l'AD :

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

**Contenu du fichier :**

```yaml
network:
  ethernets:
    ens33:
      addresses:
        - 192.168.10.30/24
      nameservers:
        addresses:
          - 192.168.10.10
      routes:
        - to: default
          via: 192.168.10.1
  version: 2
```

Sauvegarder (`Ctrl+O`) et quitter (`Ctrl+X`).

Appliquer la configuration :

```bash
sudo netplan apply
```

![Configuration du fichier Netplan](images/image10.png)

Vider le cache DNS :

```bash
sudo systemctl restart systemd-resolved
```

### Vérification DNS

```bash
ping technolab.local
```

![Vérification de la résolution DNS du domaine](images/image11.png)

### Installation des outils d'intégration AD

```bash
sudo apt update && sudo apt install -y realmd sssd sssd-tools libnss-sss libpam-sss adcli samba-common-bin packagekit
```

### Découverte du domaine

```bash
sudo realm discover technolab.local
```

### Intégration au domaine

```bash
sudo realm join --user=Administrateur technolab.local
```

### Vérification

```bash
id Administrateur@technolab.local
```

---

## 9. Tests et validation

### Connexion à GLPI avec un compte AD

Depuis un poste client :

1. Ouvrir `http://192.168.10.30/`
2. Saisir les identifiants domaine :
   - **Identifiant :** `amartin`
   - **Mot de passe :** `Azerty01*`

### Création d'un ticket de test

1. Se connecter avec le compte `amartin`
2. Aller dans **Assistance** → **Créer un ticket**
3. Remplir les champs et valider

---

## 10. Problèmes rencontrés et solutions

### Problème : Impossible de se connecter à GLPI

**Problème :** Impossible d'accéder à l'interface GLPI après le déploiement Docker.

**Solution :** Le problème venait de la table de routage Docker. Redémarrer Docker et relancer le conteneur :

```bash
sudo systemctl restart docker
cd glpi && docker compose up -d
```

### Problème : Ping technolab.local sans réponse

**Problème :** Le serveur Linux ne résout pas le nom `technolab.local`.

**Solution :** Vérifier et corriger le fichier Netplan (`/etc/netplan/00-installer-config.yaml`) pour que les nameservers pointent vers `192.168.10.10`, puis :

```bash
sudo netplan apply
sudo systemctl restart systemd-resolved
ping technolab.local
```

---

## 11. Conclusion

GLPI, déployé via Docker sur un serveur Ubuntu, est désormais pleinement opérationnel et intégré à l'Active Directory de TechnoLab. Les utilisateurs peuvent s'authentifier avec leurs identifiants domaine et créer des tickets d'assistance.

### Points clés retenus

- **Docker Compose** permet un déploiement rapide et reproductible de GLPI
- **L'authentification LDAP** centralise la gestion des utilisateurs via l'AD
- **L'intégration Kerberos** sur Linux permet une authentification sécurisée
- La configuration **DNS** est cruciale pour le bon fonctionnement de l'infrastructure
