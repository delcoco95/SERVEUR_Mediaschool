# 🧭 Documentation du Serveur de Supervision — Projet Mediaschool Nice

## 1. Présentation du projet
Le serveur Mediaschool a pour objectif d’offrir un environnement complet de supervision, administration et collaboration aux étudiants du BTS SIO (spécialité IRIS) du campus Mediaschool Nice.

Ce serveur permet :
- la surveillance des performances et de la sécurité du réseau via Grafana, Prometheus et Alertmanager,
- la gestion centralisée des utilisateurs grâce à OpenLDAP et phpLDAPadmin,
- la visualisation et la gestion des conteneurs Docker via Portainer,
- la connexion sécurisée à distance à travers WireGuard,
- le déploiement reproductible et simplifié via Vagrant et Docker.

L’ensemble est conçu pour être pédagogique, collaboratif et reproductible sur tout autre campus.

## 2. Environnement technique
| Élément | Description |
|----------|--------------|
| Système hôte | Windows / Linux avec VirtualBox et Vagrant |
| VM principale | Ubuntu Server 22.04 LTS |
| Gestionnaire de conteneurs | Docker & Docker Compose |
| Surveillance | Grafana, Prometheus, Node Exporter, Alertmanager |
| Administration | Portainer |
| Annuaire LDAP | OpenLDAP + phpLDAPadmin |
| VPN sécurisé | WireGuard |

## 3. Installation avec Vagrant
Structure du projet :
Serveur_Mediaschool/
├── Vagrantfile
├── docker-compose.yml
├── prometheus.yml
├── alertmanager.yml
└── ldap/
    ├── bootstrap.ldif
    └── config/

### Prérequis
- VirtualBox ≥ 7.x
- Vagrant ≥ 2.3
- Accès internet
- 8 Go RAM / 2 CPU minimum

### Déploiement
```
vagrant up
vagrant ssh
```

## 4. Installation et configuration Docker
```
sudo apt update && sudo apt install -y docker.io docker-compose
sudo systemctl enable docker --now
```

## 5. Services déployés
Services : Prometheus, Node Exporter, Alertmanager, Grafana, Portainer, OpenLDAP, phpLDAPadmin, WireGuard.
Démarrage :
```
docker compose up -d
```

## 6. Supervision : Grafana & Prometheus
Accès Grafana : http://192.168.56.101:3000 (admin/admin)

## 7. Gestion LDAP
phpLDAPadmin : https://192.168.56.101:6443  
Admin : cn=admin,dc=mediaschool,dc=local

## 8. Sécurité VPN : WireGuard
Fichiers clients dans ./wireguard/config/

## 9. Alertmanager
Notifications vers mail ou Discord webhook.

## 10. Commandes utiles
| Action | Commande |
|--------|-----------|
| Lister conteneurs | docker ps |
| Redémarrer services | docker compose restart |
| Logs service | docker logs <nom> |
| Stopper | docker compose down |

## 11. Objectifs pédagogiques
Apprentissage du déploiement automatisé, de la supervision complète, de la centralisation LDAP, de la sécurité VPN et des pratiques DevOps.

## 12. Sauvegarde & maintenance
```
docker compose down
tar -czf sauvegarde_mediaschool_$(date +%F).tar.gz ./wireguard ./portainer_data ./ldap
docker compose pull
docker compose up -d
```

## 13. Auteurs & contact
Équipe IRIS – BTS SIO Mediaschool Nice  
Encadré par : Nedj (Administrateur du serveur)  
Année : 2025
