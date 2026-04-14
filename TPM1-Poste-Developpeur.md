# TP — Poste Développeur : VM Debian + Stack LAMP + SSH sécurisé + VirtualHost

> **Objectif** : Déployer une VM Debian 13, y installer une stack LAMP complète, sécuriser les accès SSH par clé publique, puis publier une application web via un VirtualHost Apache.

---

## Sommaire

1. [Création de la VM Debian 13](#1-création-de-la-vm-debian-13)
2. [Installation en mode CLI](#2-installation-en-mode-cli)
3. [Installation de la stack LAMP](#3-installation-de-la-stack-lamp)
4. [Sécurisation de MariaDB](#4-sécurisation-de-mariadb)
5. [Installation et configuration du serveur SSH](#5-installation-et-configuration-du-serveur-ssh)
6. [Génération d'une paire de clés SSH (PC hôte)](#6-génération-dune-paire-de-clés-ssh-pc-hôte)
7. [Copie de la clé publique vers la VM](#7-copie-de-la-clé-publique-vers-la-vm)
8. [Désactivation de l'authentification par mot de passe SSH](#8-désactivation-de-lauthentification-par-mot-de-passe-ssh)
9. [Autorisation sudo sans mot de passe pour Apache](#9-autorisation-sudo-sans-mot-de-passe-pour-apache)
10. [Création d'un VirtualHost Apache](#10-création-dun-virtualhost-apache)
11. [Déploiement de l'application qrcodem1](#11-déploiement-de-lapplication-qrcodem1)

---

## 1. Création de la VM Debian 13

### Spécifications matérielles

| Paramètre | Valeur |
|-----------|--------|
| CPU | 1 vCPU |
| RAM | 2 Go |
| Disque | 20 Go HDD |
| OS | Debian 13 (Trixie) |
| Mot de passe root | **Aucun** (laisser vide) |

### Paramétrage réseau recommandé

- **Mode réseau** : mode pont (Bridged) sur VirtualBox
- **Accès SSH depuis le PC hôte** : mode pont recommandé pour accéder à la VM par son IP locale

> ⚠️ **Important** : Lors de l'installation, laisser le champ mot de passe root **vide**. Cela désactivera le compte root et forcera l'utilisation de `sudo` via votre compte utilisateur.

---

## 2. Installation en mode CLI

Pendant l'assistant d'installation de Debian :

1. Choisir la langue, le pays et la disposition clavier
2. Configurer le réseau (DHCP automatique)
3. Créer un **utilisateur non-root** (ex : `devuser`) avec un mot de passe
4. Partitionnement : **Utiliser le disque entier** (méthode guidée)
5. Lors de la sélection des logiciels (**tasksel**), **décocher tout** sauf :
   - ✅ `Utilitaires standard du système`
   - ✅ `Serveur SSH` *(optionnel ici, on l'installe manuellement en étape 5)*

> 💡 **Pas d'environnement graphique** : on travaille exclusivement en ligne de commande (CLI).

---

## 3. Installation de la stack LAMP

**LAMP** = **L**inux + **A**pache + **M**ariaDB + **P**HP

Se connecter à la VM puis passer en mode superutilisateur :

```bash
su -
# ou, si sudo est disponible :
sudo -i
```

Mettre à jour les paquets :

```bash
apt update && apt upgrade -y
```

### 3.1 Installer Apache2

```bash
apt install apache2 -y
```

Vérifier qu'Apache fonctionne :

```bash
systemctl status apache2
```

Activer Apache au démarrage :

```bash
systemctl enable apache2
```

### 3.2 Installer MariaDB

```bash
apt install mariadb-server mariadb-client -y
```

Démarrer et activer le service :

```bash
systemctl start mariadb
systemctl enable mariadb
```

### 3.3 Installer PHP et les modules courants

```bash
apt install php libapache2-mod-php php-mysql php-cli php-curl php-mbstring php-xml php-zip -y
```

Vérifier la version installée :

```bash
php -v
```

Redémarrer Apache pour prendre en compte le module PHP :

```bash
systemctl restart apache2
```

### 3.4 Vérification de la stack

Créer un fichier de test PHP :

```bash
echo "<?php phpinfo(); ?>" > /var/www/html/info.php
```

Depuis votre navigateur (PC hôte), accéder à `http://<IP_VM>/info.php`. La page d'informations PHP doit s'afficher.

> 🔒 Supprimer ce fichier après vérification : `rm /var/www/html/info.php`

---

## 4. Sécurisation de MariaDB

Lancer le script de sécurisation interactif :

```bash
mariadb-secure-installation
```

Répondre aux questions comme suit :

| Question | Réponse recommandée |
|----------|---------------------|
| Entrer le mot de passe root MariaDB actuel | *(Appuyer sur Entrée — vide par défaut)* |
| Passer à l'authentification unix_socket ? | `n` |
| Définir un mot de passe root ? | `Y` → choisir un mot de passe fort |
| Supprimer les utilisateurs anonymes ? | `Y` |
| Désactiver la connexion root à distance ? | `Y` |
| Supprimer la base de test ? | `Y` |
| Recharger les tables de privilèges ? | `Y` |

---

## 5. Installation et configuration du serveur SSH

Installer le serveur OpenSSH :

```bash
apt install openssh-server -y
```

Démarrer et activer le service :

```bash
systemctl start ssh
systemctl enable ssh
```

Vérifier que SSH écoute bien sur le port 22 :

```bash
ss -tlnp | grep :22
```

Récupérer l'adresse IP de la VM (pour la suite) :

```bash
ip a
```

---

## 6. Génération d'une paire de clés SSH (PC hôte)

> ⚠️ Cette étape se fait **sur votre PC**, pas sur la VM.

Ouvrir un terminal sur votre machine locale et générer une paire de clés **ED25519** (algorithme moderne et sécurisé) :

```bash
ssh-keygen -t ed25519 -C "mon_poste_dev"
```

Options importantes :

| Paramètre | Valeur |
|-----------|--------|
| Fichier de destination | `~/.ssh/id_ed25519` (par défaut, appuyer sur Entrée) |
| Passphrase | Recommandé : entrer une phrase de passe robuste |

Deux fichiers sont créés :

- `~/.ssh/id_ed25519` → **clé privée** (à ne jamais partager)
- `~/.ssh/id_ed25519.pub` → **clé publique** (à déposer sur le serveur)

Vérifier la génération :

```bash
ls -la ~/.ssh/
```

---

## 7. Copie de la clé publique vers la VM

> Cette étape se fait **sur votre PC**.

Utiliser `ssh-copy-id` pour copier automatiquement la clé publique sur la VM :

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub devuser@<IP_VM>
```

Remplacer `devuser` par votre nom d'utilisateur sur la VM et `<IP_VM>` par l'adresse IP relevée à l'étape 5.

Le mot de passe de l'utilisateur sera demandé une dernière fois. La clé est alors ajoutée dans `~/.ssh/authorized_keys` sur la VM.

Tester la connexion par clé :

```bash
ssh devuser@<IP_VM>
```

La connexion doit s'établir **sans demande de mot de passe** (sauf si vous avez défini une passphrase sur la clé).

---

## 8. Désactivation de l'authentification par mot de passe SSH

> Cette étape se fait **sur la VM**, connecté en SSH.

Éditer le fichier de configuration SSH :

```bash
sudo nano /etc/ssh/sshd_config
```

Localiser et modifier (ou ajouter) les directives suivantes :

```
# Désactiver l'authentification par mot de passe
PasswordAuthentication no

# S'assurer que l'auth par clé publique est activée
PubkeyAuthentication yes

# Désactiver la connexion directe en root
PermitRootLogin no
```

> 💡 Utiliser `Ctrl+W` dans nano pour rechercher un terme.

Sauvegarder (`Ctrl+O`, `Entrée`) puis quitter (`Ctrl+X`).

Redémarrer le service SSH pour appliquer les changements :

```bash
sudo systemctl restart sshd
```

> ⚠️ **Ne pas fermer la session SSH actuelle** avant de tester la connexion depuis un autre terminal, afin d'éviter de vous bloquer hors de la VM.

Tester depuis un second terminal :

```bash
ssh devuser@<IP_VM>
```

La connexion doit fonctionner uniquement par clé. Une tentative de connexion avec mot de passe doit être refusée.

---

## 9. Autorisation sudo sans mot de passe pour Apache

L'objectif est de permettre à l'utilisateur `devuser` de redémarrer Apache (`apachectl restart`) **sans saisir de mot de passe**.

Éditer le fichier sudoers de manière sécurisée (avec `visudo`) :

```bash
sudo visudo
```

Ajouter la ligne suivante **à la fin du fichier** :

```
devuser ALL=(ALL) NOPASSWD: /usr/sbin/apachectl restart
```

> ⚠️ Toujours utiliser `visudo` pour éditer `/etc/sudoers` — il vérifie la syntaxe avant de sauvegarder et évite de rendre le système inaccessible en cas d'erreur.

Tester la commande :

```bash
sudo apachectl restart
```

Aucun mot de passe ne doit être demandé.

---

## 10. Création d'un VirtualHost Apache

### 10.1 Créer le répertoire du VirtualHost

```bash
sudo mkdir -p /var/www/qrcodem1
sudo chown -R devuser:www-data /var/www/qrcodem1
sudo chmod -R 755 /var/www/qrcodem1
```

### 10.2 Créer le fichier de configuration du VirtualHost

```bash
sudo nano /etc/apache2/sites-available/monapp.dev.conf
```

Contenu du fichier :

```apache
<VirtualHost *:80>
    ServerName monapp.dev
    ServerAlias www.monapp.dev

    DocumentRoot /var/www/qrcodem1

    <Directory /var/www/qrcodem1>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/monapp.dev_error.log
    CustomLog ${APACHE_LOG_DIR}/monapp.dev_access.log combined
</VirtualHost>
```

Sauvegarder et quitter.

### 10.3 Activer le VirtualHost

```bash
sudo a2ensite monapp.dev.conf
sudo a2enmod rewrite
sudo systemctl reload apache2
```

### 10.4 Modifier le fichier hosts sur le PC hôte

> Cette étape se fait **sur votre PC**.

Éditer le fichier hosts :

`sudo nano /etc/hosts`

Ajouter la ligne suivante :

```
<IP_VM>    monapp.dev www.monapp.dev
```

Remplacer `<IP_VM>` par l'adresse IP de votre VM.

Tester en accédant à `http://monapp.dev` depuis votre navigateur.

---

## 11. Déploiement de l'application qrcodem1

### 11.1 Installer Git

```bash
sudo apt install git -y
```

### 11.2 Cloner le dépôt

```bash
cd /tmp
git clone https://github.com/KevinCHAILLY/TPM1.git
```

> Remplacer `<utilisateur>` par le propriétaire du dépôt GitHub.

### 11.3 Déplacer le code dans le répertoire du VirtualHost

```bash
sudo mv /tmp/qrcodem1/* /var/www/qrcodem1/
```

> Si le dépôt contient un sous-répertoire `qrcodem1/` :
> ```bash
> sudo mv /tmp/qrcodem1/qrcodem1/* /var/www/qrcodem1/
> ```

### 11.4 Appliquer les bonnes permissions (chown)

Le propriétaire du répertoire doit être `www-data` (utilisateur Apache) pour que le serveur puisse lire les fichiers :

```bash
sudo chown -R www-data:www-data /var/www/qrcodem1
```

### 11.5 Vérifier les droits

```bash
ls -la /var/www/qrcodem1
```

La sortie attendue doit ressembler à :

```
drwxr-xr-x  ... www-data www-data  .
drwxr-xr-x  ... root     root      ..
-rw-r--r--  ... www-data www-data  index.php
...
```

Vérification récursive si nécessaire :

```bash
find /var/www/qrcodem1 -type f -ls
```

### 11.6 Redémarrer Apache

```bash
sudo apachectl restart
```

Accéder à l'application depuis le navigateur : `http://monapp.dev`

---

## Récapitulatif des commandes clés

| Action | Commande |
|--------|----------|
| Statut Apache | `sudo systemctl status apache2` |
| Redémarrer Apache | `sudo apachectl restart` |
| Statut MariaDB | `sudo systemctl status mariadb` |
| Statut SSH | `sudo systemctl status sshd` |
| Voir les logs Apache | `sudo tail -f /var/log/apache2/error.log` |
| Tester config Apache | `sudo apache2ctl configtest` |
| Lister les sites actifs | `sudo a2query -s` |

---

## Schéma de l'infrastructure

```
┌──────────────────────────────────────────────────────┐
│                      PC Hôte                         │
│                                                      │
│  ~/.ssh/id_ed25519  (clé privée)                     │
│  /etc/hosts  →  monapp.dev = <IP_VM>                 │
│                                                      │
│   ssh devuser@<IP_VM>  ──────────────────────────►  │
│   http://monapp.dev    ──────────────────────────►  │
└──────────────────────────────────────────────────────┘
                              │
                     Port 22 / Port 80
                              │
┌──────────────────────────────────────────────────────┐
│                   VM Debian 13                       │
│                                                      │
│  ┌──────────────┐   ┌──────────────┐                 │
│  │  OpenSSH     │   │    Apache2   │                 │
│  │  (port 22)   │   │   (port 80)  │                 │
│  │  Clé pub.    │   │  VirtualHost │                 │
│  │  uniquement  │   │  monapp.dev  │                 │
│  └──────────────┘   └──────┬───────┘                 │
│                            │                         │
│                   /var/www/qrcodem1/                  │
│                   (code de l'app)                    │
│                                                      │
│  ┌────────────────────────────────┐                  │
│  │  MariaDB  (sécurisé)           │                  │
│  │  Pas d'accès root distant      │                  │
│  └────────────────────────────────┘                  │
└──────────────────────────────────────────────────────┘
```

---

*TP réalisé dans le cadre du poste développeur — Debian 13 / LAMP / SSH / Apache VirtualHost*