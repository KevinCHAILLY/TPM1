# TP – Mise en place d’une infrastructure virtualisée et sécurisée

## Objectifs pédagogiques

- Déployer une infrastructure virtualisée avec Proxmox
- Mettre en place un Active Directory fonctionnel
- Déployer des services de supervision et de sécurité
- Structurer un réseau avec VLAN
- Configurer un pare-feu et une DMZ

---

# Partie 1 – Déploiement de l’infrastructure

## 1. Architecture globale

### Plan d’adressage (réseau principal)

| Équipement              | IP                | Description |
|------------------------|------------------|------------|
| Proxmox                | 172.16.10.10     | Hyperviseur |
| Active Directory       | 172.16.10.20     | Contrôleur de domaine |
| Point d’accès WiFi     | 172.16.10.30     | AP |
| Switch manageable      | 172.16.10.40     | Switch |
| LibreNMS               | 172.16.10.50     | Supervision |
| Wazuh                  | 172.16.10.60     | SIEM |
| Serveur LAMP           | 172.16.10.70     | Web |
| Routeur OPNsense (LAN) | 172.16.10.254    | Gateway |

### Paramètres réseau

- Réseau : 172.16.10.0/24
- Passerelle : 172.16.10.254
- DNS : 172.16.10.20

---

## 2. Installation de Proxmox

### Matériel

- CPU : i5
- RAM : 32 Go
- Stockage : SSD 240 Go

### Configuration réseau

- IP : 172.16.10.10/24
- Gateway : 172.16.10.254
- DNS : 172.16.10.20

### Travaux

- Installer Proxmox VE
- Configurer le bridge vmbr0
- Vérifier l’accès web

---

## 3. Active Directory

### Configuration VM

- OS : Windows Server
- CPU : 2 vCPU
- RAM : 4 Go
- IP : 172.16.10.20

### Étapes

1. Installer Windows Server
2. Configurer IP statique
3. Activer le bureau à distance
4. Installer AD DS
5. Promouvoir en contrôleur de domaine

Domaine : ad.m1.dev

### Vérifications

- Résolution DNS
- Connexion RDP
- Domaine accessible

---

## 4. Équipements réseau

### Point d’accès WiFi

- IP : 172.16.10.30
- Configuration :
  - SSID
  - WPA2/WPA3

### Switch manageable

- IP : 172.16.10.40
- Configuration :
  - Accès admin
  - Préparation VLAN

---

## 5. Machines virtuelles Linux

### LibreNMS (supervision)

- OS : Debian 13
- IP : 172.16.10.50

#### Travaux

- Installer Apache / MariaDB / PHP
- Installer LibreNMS
- Ajouter équipements réseau

---

### Wazuh (SIEM)

- OS : Debian 13
- IP : 172.16.10.60

#### Travaux

- Installer Wazuh manager
- Installer agents (AD, LAMP)
- Vérifier logs

---

### Serveur LAMP

- OS : Debian 13
- IP : 172.16.10.70

#### Travaux

- Installer Apache / MariaDB / PHP
- Déployer page web
- Tester HTTP

---

## 6. Routeur OPNsense

### Configuration

- WAN : DHCP
- LAN : 172.16.10.254/24

### Travaux

- Configurer NAT
- Tester accès Internet

---

# Partie 2 – Segmentation réseau

## 1. VLAN

### VLAN à créer

| VLAN | Nom      | Réseau          |
|------|----------|-----------------|
| 10   | Admin    | 172.16.10.0/24  |
| 20   | Clients  | 172.16.20.0/24  |
| 30   | DMZ      | 172.16.30.0/24  |

### Switch

- Créer VLAN
- Configurer ports access et trunk

---

## 2. Interfaces OPNsense

| Interface | VLAN | IP              |
|----------|------|-----------------|
| LAN      | 10   | 172.16.10.254   |
| VLAN20   | 20   | 172.16.20.254   |
| VLAN30   | 30   | 172.16.30.254   |

---

## 3. Firewall

### VLAN 10 (Admin)

- Autoriser tout

### VLAN 20 (Clients)

- Autoriser HTTP / HTTPS / DNS
- Bloquer accès VLAN 10

### VLAN 30 (DMZ)

- Autoriser accès web depuis Internet
- Bloquer accès LAN

---

## 4. DMZ

### Mise en place

- Déplacer serveur LAMP en VLAN 30
- Configurer NAT :
  - WAN → HTTP / HTTPS

---

## 5. Vérifications

### Réseau

- Ping inter-VLAN (selon règles)
- Accès Internet
- Accès web externe

### Supervision

- LibreNMS : équipements visibles
- Wazuh : logs remontent

---

## Livrables

- Schéma réseau
- Captures :
  - Proxmox
  - AD
  - LibreNMS
  - Wazuh
  - Firewall
- Plan d’adressage
- Explications sécurité

---

## Questions

1. Pourquoi utiliser des VLAN ?
2. À quoi sert une DMZ ?
3. Pourquoi isoler les clients ?
4. Risques sans firewall ?
5. Différence supervision / SIEM ?