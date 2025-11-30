# SRV IRIS — Version Propre (Sécurité Systèmes & Réseau)

Cette branche **`srviris`** contient une version totalement épurée et sécurisée de l’infrastructure du serveur pédagogique.
Elle sert de **base propre avant la mise en production** sur le vrai serveur de l’école.

L’objectif :
Mettre en place un environnement **simple, stable et sécurisé**, centré uniquement sur les services essentiels à l’enseignement **SISR / cybersécurité**.

---

## 🚀 Objectif de cette version

* Repartir sur un serveur **neuf**, propre et documenté
* Préparer un environnement 100% dédié à la **sécurité et la gestion réseau**
* N'installer **que les services nécessaires** avant la mise en place réelle de l'infrastructure finale
* Permettre des tests, audits, TP et démonstrations
* Servir de base propre pour évoluer plus tard (monitoring, dashboard, supervision…)

---

## 🧩 Services installés dans cette version

Cette version inclut uniquement les services essentiels :

| Service                  | Description                                 | Accès                                                    |
| ------------------------ | ------------------------------------------- | -------------------------------------------------------- |
| **WireGuard (wg-easy)**  | VPN sécurisé + interface web de gestion     | [http://192.168.56.10:51821](http://192.168.56.10:51821) |
| **OpenLDAP**             | Annuaire LDAP pour gestion des utilisateurs | Port 389 / 636                                           |
| **phpLDAPadmin**         | Interface web pour gérer OpenLDAP           | [http://192.168.56.10:8080](http://192.168.56.10:8080)   |
| **ClamAV**               | Antivirus serveurs + service de scan        | Port 3310                                                |
| **UFW + nftables**       | Pare-feu sécurisé                           | Configuré automatiquement                                |
| **Utilisateur adminedj** | Utilisateur administrateur (sudo)           | Login dans VM                                            |

➡️ Pas de monitoring, pas de containers inutiles, pas de services superflus.

---

## 🏗️ Architecture technique

### 🖥️ VM Vagrant (VirtualBox)

* **OS** : Ubuntu 22.04 LTS (jammy)
* **RAM** : 2 Go
* **CPU** : 2 vCore
* **IP** : `192.168.56.10`
* **Accès SSH** :

  ```bash
  vagrant ssh
  ```

### 📦 Docker + Docker Compose

Les services sont déployés via `docker compose`, dans :

```
/vagrant
```

### 📁 Arborescence des données

```
SRV_Mediaschool_IRIS/
│── Vagrantfile
│── docker-compose.yml
│── .env
│
├── ldap/
│   ├── config/
│   ├── database/
│   └── certs/
│
├── wireguard/
│   └── data/
│
├── wireguard_clients/
│
└── clamav/
    └── a_scanner/
```

---

## 🔐 Accès aux interfaces web

Depuis le PC hôte (192.168.56.1) :

### ➤ WireGuard (wg-easy)

👉 [http://192.168.56.10:51821](http://192.168.56.10:51821)
Mot de passe : défini dans `.env`

### ➤ phpLDAPadmin

👉 [http://192.168.56.10:8080](http://192.168.56.10:8080)
Identifiant :

```
cn=admin,dc=ldap,dc=local
```

Mot de passe : celui de `LDAP_ADMIN_PASS` dans `.env`

---

## ⚙️ Commandes utiles

### Démarrer la VM

```powershell
vagrant up
```

### Arrêter la VM

```powershell
vagrant halt
```

### Détruire la VM (pour repartir propre)

```powershell
vagrant destroy -f
```

### Se connecter en SSH

```powershell
vagrant ssh
```

---

## 🐳 Contrôle des conteneurs (dans la VM)

```bash
cd /vagrant
docker compose ps
docker compose up -d
docker compose down
```

---

## 🔥 Pare-feu UFW

Ports autorisés :

* SSH → 22
* LDAP → 389 / 636
* phpLDAPadmin → 8080
* WireGuard interface → 51821
* WireGuard VPN → 51820/udp
* ClamAV → 3310

---

## 👤 Utilisateur administrateur local

Un utilisateur **adminedj** est créé automatiquement :

| Utilisateur | Mot de passe | Droits |
| ----------- | ------------ | ------ |
| `adminedj`  | `123456789`  | sudo   |

---

## 🎯 Pourquoi cette branche existe ?

La branche **`srviris`** est une version **épurée**, dédiée aux TP, démonstrations et tests de l’infrastructure de sécurité **avant** le déploiement sur le vrai serveur pédagogique.

Elle sert de :

* base propre
* environnement de test
* référence stable
* support pédagogique

---
