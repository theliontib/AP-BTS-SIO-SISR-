# Installation et Configuration de TrueNAS

---

## 1. Prérequis

- VMware Workstation Player (ou autre hyperviseur)
- ISO TrueNAS (version Core ou Scale)

---

## 2. Création de la VM

### Étapes
1. Créer une VM avec **un disque de 16 Go** pour l'installation
2. Après la création, **modifier la VM** pour ajouter un **second disque de 50 Go** (pour le stockage)
3. Lancer la VM et installer TrueNAS

### Installation de TrueNAS
- Langue / clavier : selon vos préférences
- Écran d'accueil : sélectionner **Install/Upgrade**
- Choisir le disque de 16 Go comme destination
- Définir le mot de passe administrateur :
  - **Identifiant :** `truenas_admin`
  - **Mot de passe :** `Serveur@)@@` (correspond à `Serveur2022` en azerty → qwerty)
- **EFI :** Yes
- Terminer l'installation puis **Reboot System**

---

## 3. Configuration réseau

### Depuis la console TrueNAS

#### Option 1 : Configurer l'interface réseau
- Aller dans **Option 1 > Configure network interfaces**
- Sélectionner `ens33` → **Entrée**
- `ipv4_dhcp` : **No**
- `aliases` : ajouter `192.168.10.40/24`
- **Save** → appuyer sur **A** pour appliquer

#### Option 2 : Configurer les paramètres réseau
- Aller dans **Option 2 > Configure network settings**

| Paramètre | Valeur |
|---|---|
| Hostname | `truenas` |
| Domain | `TECHNOLAB.local` |
| IPv4 Gateway | `192.168.10.1` |
| Name Server 1 | `192.168.10.10` |

### Synchronisation NTP (important pour Kerberos)

> Kerberos (protocole d'authentification AD) exige que l'heure soit synchronisée à moins de 5 minutes entre TrueNAS et le contrôleur de domaine.

Depuis l'interface web TrueNAS (depuis le navigateur du serveur Windows) :
[http://192.168.10.40](http://192.168.10.40)

Ou via la console, **Option 8 (Linux Shell)** :

```bash
# Forcer l'heure manuellement si chrony ne se synchronise pas
date -s "2026-06-05 10:26:00"

# Ou ajouter le serveur AD comme source NTP
echo "server 192.168.10.10 iburst prefer" >> /etc/chrony.conf
systemctl restart chronyd
```

---

## 4. Création du pool de stockage ZFS

1. Depuis l'interface web TrueNAS, aller dans **Storage > Create Pool**
2. Configurer :

| Champ | Valeur |
|---|---|
| General Info - Nom du pool | `datapool` |
| Allow non-unique disks | **Co cher** (pour VMware) |
| Data - Layout | **Stripe** (1 disque) |
| Disk Size | 50 GiB |
| Width | 1 |
| Number of VDEVs | 1 |

3. Cliquer sur **Create**, puis **Confirm** et **Continue**

---

## 5. Enregistrement DNS sur le serveur AD

1. Ouvrir le **Gestionnaire DNS** sur Windows Server (Outils > DNS)
2. Cliquer sur le DNS pour voir apparaître **"Zones de recherche directes"** et `TECHNOLAB.local`
3. Clic droit dans la zone > **Nouvel hôte (A)**

| Champ | Valeur |
|---|---|
| Nom | `truenas` |
| Adresse IP | `192.168.10.40` |

4. Cliquer sur **Ajouter un hôte** (ignorer l'avertissement PTR)

---

## 6. Jonction au domaine Active Directory

1. Dans l'interface web TrueNAS :
   - Menu **Identifiants > Services d'annuaire > Configurer les services d'annuaire**
2. Remplir les champs :

| Champ | Valeur |
|---|---|
| Type de configuration | **Active Directory** |
| Activer le service | **Co cher** |
| Realm Kerberos | *(laisser vide)* |
| Type d'identifiant | Kerberos User |
| Nom d'utilisateur | `Administrateur` |
| Mot de passe | `Serveur2022` |
| Nom d'hôte TrueNAS | `truenas` |
| Nom de domaine | `technolab.local` |

3. **Sauvegarder** — attendre la jonction au domaine

---

## 7. Création du partage SMB

### Création du dataset
1. Menu **Partages > Partages Windows (SMB) > Ajouter**
2. Cliquer sur le champ **Chemin** → sélectionner `datapool`
3. **Créer Dataset** → Nom : `partage`
4. Configurer :

| Champ | Valeur |
|---|---|
| Nom du partage | `Partage` |
| Description | `Partage commun technolab.local` |
| Activé | Coché |

### Configuration des ACL
Dans l'éditeur ACL :
1. Cliquer sur **+ Ajouter une entrée**

| Champ | Valeur |
|---|---|
| Qui | **Groupe** |
| Groupe | `DEVIDA\utilisateurs du domaine` |
| Type d'ACL | **Autoriser** |
| Autorisations | **Modifier** |
| Flags | **Hériter** |

2. **Sauvegarder** l'ACL

### Accès au partage depuis Windows
```
\\192.168.10.40\Partage
\\truenas\Partage
```

---

## 8. Mappage automatique via GPO

1. Sur le serveur AD, ouvrir `gpmc.msc` (Win+R → `gpmc.msc`)
2. Clic droit sur `technolab.local` → **Créer un objet GPO** → Nom : `Lecteur-TrueNAS` → OK
3. Clic droit sur `Lecteur-TrueNAS` → **Modifier**
4. Naviguer :

```
Configuration utilisateur > Préférences > Paramètres Windows > Mappages de lecteurs
```

5. Clic droit dans la zone → **Nouveau > Lecteur mappé**

| Champ | Valeur |
|---|---|
| Action | **Créer** |
| Emplacement | `\\192.168.10.40\Partage` |
| Reconnecter | **Co cher** |
| Lettre de lecteur | `Z:` |
| Libellé | `Partage TrueNAS` |

6. **OK**

### Application sur les clients
```cmd
gpupdate /force
```

---

## 9. Problèmes rencontrés et résolutions

### Problème 1 : Le lecteur Z: n'apparaît pas dans les PC

**Cause :** Par défaut, la GPO tente de monter le lecteur réseau avec le compte **SYSTEM** de la machine, et non avec la session de l'utilisateur.

**Solutions :**
1. Clic droit sur le lecteur Z: dans la GPO → **Propriétés**
2. Aller dans l'onglet **Commun**
3. Cocher **"Exécuter dans le contexte de sécurité de l'utilisateur connecté (option de stratégie utilisateur)"**
4. Appliquer puis OK

Si le lecteur Z existe déjà :
- Passer l'**Action** sur **Mettre à jour (Update)** au lieu de Créer

Redémarrer TrueNAS si nécessaire.

### Problème 2 : Le NAS est visible mais inaccessible

**Solution :** Vérifier que le service **SMB** est actif dans la page web de TrueNAS.

---

![Image 1](images/image1.png)
![Image 2](images/image2.png)
![Image 3](images/image3.png)
![Image 4](images/image4.png)
![Image 5](images/image5.png)
![Image 6](images/image6.png)
![Image 7](images/image7.png)
![Image 8](images/image8.png)
![Image 9](images/image9.png)
![Image 10](images/image10.png)
![Image 11](images/image11.png)
![Image 12](images/image12.png)
![Image 13](images/image13.png)
![Image 14](images/image14.png)
![Image 15](images/image15.png)
