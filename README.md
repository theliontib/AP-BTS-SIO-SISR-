# Dossier de Présentation de Projet — Projet Infrastructure GSB
**Déploiement d'une infrastructure système, réseau, de stockage et de ticketing conteneurisée**

* **Cadre :** Projet de Laboratoire GSB (Galaxy Swiss Bourdin)
* **Environnement :** Maquettage de Production / Virtualisation

---

## 1. Présentation Générale du Projet GSB

Dans le cadre du projet GSB (Galaxy Swiss Bourdin), l'objectif principal a été de concevoir, déployer et sécuriser une infrastructure réseau et système complète. 

Cette maquette fonctionnelle répond aux besoins métiers des utilisateurs de la structure en leur mettant à disposition un ensemble de services interconnectés, centralisés et hautement contrôlés.

### 1.1 Périmètre Fonctionnel et Objectifs de l'Infrastructure
La mise en place de cette maquette informatique s'articule autour de 4 piliers majeurs :

* **Gestion centralisée des identités :** Configuration d'un annuaire d'entreprise pour manager les utilisateurs, les ordinateurs et l'application des droits d'accès.
* **Support et gestion des incidents :** Déploiement d'une solution de helpdesk moderne pour l'assistance technique.
* **Sécurité et conformité :** Application de politiques de restriction strictes par le biais de stratégies de groupe ciblées.
* **Stockage et partage de données :** Mise à disposition d'un espace de stockage réseau centralisé, sécurisé et segmenté.

---

## 2. Architecture Technique et Solutions Retenues

L'intégralité de l'infrastructure a été virtualisée sur un poste hôte unique à l'aide d'un hyperviseur de type 2, permettant de simuler fidèlement un environnement d'entreprise de production.

### 2.1 Services d'Infrastructure et Systèmes Déployés

* **Active Directory Domain Services (AD DS) :** Hébergé sur Windows Server, il constitue le cœur de la forêt de l'infrastructure, orchestrant l'authentification et l'arborescence des Unités Organisationnelles (UO).
* **Solution de Ticketing (GLPI) via Docker :** Déployée sous forme de conteneur sur un serveur Linux Ubuntu. L'utilisation de Docker garantit l'isolation du service de gestion des tickets, la légèreté des processus et la rapidité de maintenance/mise à niveau.
  > **Note d'orientation :** Le choix de l’utilisation de *Docker* plutôt que *WampServer* est un choix personnel alimenté par la volonté d’apprentissage de l’outil, dans un but de continuer des études en DevOps.
* **Stockage Centralisé (TrueNAS) :** Serveur de stockage réseau (NAS) dédié permettant le partage de fichiers au sein de l'organisation avec gestion affinée des privilèges.
* **Poste Client de Validation :** Un environnement Windows 10 Professionnel a été intégré au domaine afin de valider l'application effective des règles de sécurité et de simuler le poste de travail d'un collaborateur.

> 💡 **Note Méthodologique & Ressources Documentaires :**
> * Exploitation rigoureuse de scénarios de Travaux Pratiques guidés.
> * Utilisation de l'Intelligence Artificielle comme assistance technique pour l'optimisation des scripts et procédures de configuration.

---

## 3. Stratégies de Sécurité Globale (GPO)

Afin de centraliser l'administration des postes de travail et d'élever le niveau de sécurité du système d'information conformément aux exigences du laboratoire GSB, plusieurs Stratégies de Groupe (GPO) ont été déployées et appliquées aux Unités Organisationnelles correspondantes.

### 3.1 Politique de Mots de Passe (Applicable à l'ensemble du domaine)
* **Longueur minimale :** 12 caractères requis afin de prémunir le domaine contre les attaques par force brute.
* **Durée de vie maximale :** 45 jours, obligeant un renouvellement fréquent des secrets d'authentification.

### 3.2 Restrictions de Sécurité Ciblées par Service
Des restrictions spécifiques ont été appliquées selon le profil métier de l'utilisateur :

* **Service Comptabilité :** Blocage complet de l'Invite de Commande (CMD) pour empêcher l'exécution de scripts non autorisés ou l'élévation locale de privilèges.
* **Membres de la Direction :** Accès interdit au Panneau de Configuration afin de figer la configuration système des postes sensibles et d'éviter les modifications accidentelles.
* **Service Informatique (IT) :** Attribution exclusive des droits d'administration locale pour assurer le support et la maintenance des machines du parc.

---

## 4. Spécifications Réseau et Plan d'Adressage IP

L'infrastructure est isolée au sein d'un sous-réseau privé unique de classe C, utilisant le mode de connexion NAT (VMnet8) de l'hyperviseur, exploitant la machine hôte physique comme passerelle d'accès internet.

### 4.1 Matrice d'Adressage Statique des Équipements

| Équipement / Rôle | Système d'Exploitation | Adresse IP Statique | Identifiants par Défaut |
| :--- | :--- | :--- | :--- |
| **Hyperviseur / Hôte (Passerelle)** | *N/A* | 192.168.10.1 | *N/A* |
| **Contrôleur de Domaine** | Windows Server 2022 | 192.168.10.10 | Administrateur / `Serveur2022` |
| **Poste Client Standard** | Windows 10 Pro | 192.168.10.20 | amartin / `Azerty01*` |
| **Serveur Application (GLPI)** | Ubuntu Server (Docker) | 192.168.10.30 | administrateur / `Serveur2022` |
| **Serveur de Stockage (NAS)** | TrueNAS | 192.168.10.40 | administrateur / `Serveur@)@@` |

### 4.2 Règle d'Imputation des Comptes Utilisateurs
Pour l'ensemble de la population de l'annuaire, la convention de nommage standardisée adoptée est le format suivant : 
**Première lettre du prénom accolée au nom de famille complet** (Exemple : `amartin` pour Albert Martin).

Un mot de passe initial générique unique (`Azerty01*`) est attribué lors de la création pour la première connexion.

---

## 5. Environnement Matériel d'Hébergement

Le maquettage et l'exécution fluide et simultanée de l'ensemble des machines virtuelles (Windows Server, Ubuntu, TrueNAS, Client Windows 10) reposent sur l'allocation des ressources physiques du poste hôte décrit ci-après :

* **Modèle de la machine :** Dell Latitude 7420 (Gamme Professionnelle)
* **Processeur (CPU) :** Intel Core i7 de 11ème Génération cadencé à 3.00 GHz (Gestion optimisée de la virtualisation VT-x)
* **Mémoire Vive (RAM) :** 16 Go de mémoire vive, dimensionnée pour la répartition équitable des allocations mémoires des VM
* **Espace de Stockage :** Disque SSD de 1 To (Garantissant des taux de transfert IOPS élevés pour l'exécution conjointe des OS)

---

## 6. Livrables et Documentations de Synthèse

Dans le but d'assurer la pérennité de la solution, son exploitation future ou sa parfaite reproductibilité technique, le projet est accompagné de deux livrables documentaires majeurs :

1. **Documentation Générale :** Ce présent document.
2. **Documentation Technique Détaillée :** Manuel de procédures pas-à-pas regroupant l'ensemble des étapes d'installation, de configuration et d'interconnexion de chaque brique applicative (Active Directory, GPO, Docker, GLPI, TrueNAS).

📥 **Accès aux livrables numériques :** Dossier Partagé Google Drive GSB (Également disponibles au format PDF au sein du dossier de fichiers joint remis au jury).
