# Vagrantfile - Serveur Mediaschool (Debian Bookworm)
Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.vm.hostname = "serveur-mediaschool"

  # Réseau Host-Only (accès stable depuis l’hôte)
  config.vm.network "private_network", ip: "192.168.56.101"

  config.vm.provider "virtualbox" do |vb|
    vb.name  = "SERVEUR-MEDIASCHOOL"
    vb.memory = 4096
    vb.cpus   = 2
  end

  # Partage du dossier projet -> /vagrant
  config.vm.synced_folder ".", "/vagrant", disabled: false

  config.vm.provision "shell", inline: <<-'SHELL'
set -euo pipefail
export DEBIAN_FRONTEND=noninteractive

# ========== Variables modifiables ==========
LDAP_DOMAIN="mediaschool.local"
LDAP_ORG="MediaSchool"
LDAP_ADMIN_PASS="admin"      # à changer
STUDENT_PASS="eleve2025"     # à changer
PROF_PASS="prof2025"         # à changer
WG_PEERS=20                  # 10 SISR + 5 SLAM + 3 profs + 2 admins
HOST_IP="192.168.56.101"     # cohérent avec l'IP ci-dessus
export LDAP_DOMAIN LDAP_ORG LDAP_ADMIN_PASS STUDENT_PASS PROF_PASS WG_PEERS HOST_IP
# ==========================================

apt-get update -y
apt-get upgrade -y

# Outils + SSH + UFW + LDAP utils + ClamAV
apt-get install -y \
  sudo openssh-server ufw nftables \
  apt-transport-https ca-certificates curl gnupg lsb-release jq unzip iproute2 \
  software-properties-common ldap-utils clamav clamav-daemon

# Compte adminedj (sudo)
if ! id -u adminedj >/dev/null 2>&1; then useradd -m -s /bin/bash adminedj; fi
echo "adminedj:123456789" | chpasswd
usermod -aG sudo adminedj

# SSH: mot de passe ok, root interdit
sed -i 's/^#\?PasswordAuthentication .*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sed -i 's/^#\?PermitRootLogin .*/PermitRootLogin no/' /etc/ssh/sshd_config
systemctl enable ssh && systemctl restart ssh

# ClamAV DB
systemctl stop clamav-freshclam || true
freshclam || true
systemctl restart clamav-freshclam || true
systemctl enable clamav-daemon

# UFW (backend nftables)
ufw --force reset
ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow 80,443/tcp
ufw allow 8080/tcp
ufw allow 1389,1636/tcp
ufw allow 3000/tcp
ufw allow 9090/tcp
ufw allow 9093/tcp
ufw allow 9000/tcp
ufw allow 9100/tcp
ufw allow 51820/udp
sed -i 's/^DEFAULT_FORWARD_POLICY.*/DEFAULT_FORWARD_POLICY="ACCEPT"/' /etc/default/ufw
ufw --force enable
ufw reload

# Docker + compose v2
if ! command -v docker >/dev/null 2>&1; then
  curl -fsSL https://get.docker.com -o /tmp/get-docker.sh
  sh /tmp/get-docker.sh
  usermod -aG docker vagrant || true
  usermod -aG docker adminedj || true
fi
apt-get install -y docker-compose-plugin

# Arborescences
mkdir -p /vagrant/prometheus
mkdir -p /vagrant/alertmanager/config
mkdir -p /vagrant/ldap/data /vagrant/ldap/config
mkdir -p /vagrant/portainer/data
mkdir -p /vagrant/wireguard/config

# docker-compose.yml (sans clé 'version')
cat > /vagrant/docker-compose.yml <<'YAML'
services:

  openldap:
    image: osixia/openldap:1.5.0
    container_name: openldap
    environment:
      LDAP_ORGANISATION: "${LDAP_ORG}"
      LDAP_DOMAIN: "${LDAP_DOMAIN}"
      LDAP_ADMIN_PASSWORD: "${LDAP_ADMIN_PASS}"
    ports:
      - "1389:389"
      - "1636:636"
    volumes:
      - ./ldap/data:/var/lib/ldap
      - ./ldap/config:/etc/ldap/slapd.d
    restart: unless-stopped

  phpldapadmin:
    image: osixia/phpldapadmin:0.9.0
    container_name: phpldapadmin
    environment:
      PHPLDAPADMIN_LDAP_HOSTS: openldap
    ports:
      - "8080:80"
    depends_on:
      - openldap
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    volumes:
      - ./alertmanager/config:/etc/alertmanager
    ports:
      - "9093:9093"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: "admin"
    ports:
      - "3000:3000"
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:v1.10.1
    container_name: node-exporter
    ports:
      - "9100:9100"
    restart: unless-stopped

  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    command: -H unix:///var/run/docker.sock
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./portainer/data:/data
    ports:
      - "9000:9000"
    restart: unless-stopped

  wireguard:
    image: linuxserver/wireguard:latest
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/Paris
      - SERVERURL=${HOST_IP}
      - SERVERPORT=51820
      - PEERS=${WG_PEERS}
      - PEERDNS=auto
    volumes:
      - ./wireguard/config:/config
    ports:
      - "51820:51820/udp"
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
YAML

# Prometheus config
cat > /vagrant/prometheus/prometheus.yml <<'YAML'
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
YAML

# Alertmanager config
cat > /vagrant/alertmanager/config/alertmanager.yml <<'YAML'
global: {}
route:
  group_by: ['alertname']
  receiver: 'null'
receivers:
  - name: 'null'
YAML

# Démarrage des services
docker compose -f /vagrant/docker-compose.yml pull
docker compose -f /vagrant/docker-compose.yml up -d

# Attente OpenLDAP prêt (POSIX)
echo "Attente OpenLDAP..."
for i in $(seq 1 30); do
  docker logs openldap 2>&1 | grep -q "slapd starting" && break
  sleep 2
done

LDAP_URI="ldap://127.0.0.1:1389"
BASE_DN="dc=mediaschool,dc=local"
ADMIN_DN="cn=admin,${BASE_DN}"

# -- Bootstrap LDAP: exécuter une seule fois
if [ ! -f /vagrant/ldap/.bootstrapped ]; then
  echo "Bootstrap LDAP initial..."

  # LDIF : OU + Groupes (NE PAS créer ${BASE_DN} : déjà créé par le conteneur)
  cat > /vagrant/ldap/init.ldif <<'LDIF'
dn: ou=People,dc=mediaschool,dc=local
objectClass: organizationalUnit
ou: People

dn: ou=Groups,dc=mediaschool,dc=local
objectClass: organizationalUnit
ou: Groups

dn: cn=Admins,ou=Groups,dc=mediaschool,dc=local
objectClass: groupOfNames
cn: Admins
member: cn=admin,dc=mediaschool,dc=local

dn: cn=Profs,ou=Groups,dc=mediaschool,dc=local
objectClass: groupOfNames
cn: Profs
member: cn=admin,dc=mediaschool,dc=local

dn: cn=SISR,ou=Groups,dc=mediaschool,dc=local
objectClass: groupOfNames
cn: SISR
member: cn=admin,dc=mediaschool,dc=local

dn: cn=SLAM,ou=Groups,dc=mediaschool,dc=local
objectClass: groupOfNames
cn: SLAM
member: cn=admin,dc=mediaschool,dc=local
LDIF

  # Comptes
  for i in $(seq -f "%02g" 1 10); do uid="sisr${i}"; cat >> /vagrant/ldap/init.ldif <<EOD
dn: uid=${uid},ou=People,dc=mediaschool,dc=local
objectClass: inetOrgPerson
cn: ${uid}
sn: ${uid}
uid: ${uid}
userPassword: ${STUDENT_PASS}

EOD
  done
  for i in $(seq -f "%02g" 1 5); do uid="slam${i}"; cat >> /vagrant/ldap/init.ldif <<EOD
dn: uid=${uid},ou=People,dc=mediaschool,dc=local
objectClass: inetOrgPerson
cn: ${uid}
sn: ${uid}
uid: ${uid}
userPassword: ${STUDENT_PASS}

EOD
  done
  for i in $(seq 1 3); do uid="prof${i}"; cat >> /vagrant/ldap/init.ldif <<EOD
dn: uid=${uid},ou=People,dc=mediaschool,dc=local
objectClass: inetOrgPerson
cn: ${uid}
sn: ${uid}
uid: ${uid}
userPassword: ${PROF_PASS}

EOD
  done
  for i in $(seq 1 2); do uid="admin${i}"; cat >> /vagrant/ldap/init.ldif <<EOD
dn: uid=${uid},ou=People,dc=mediaschool,dc=local
objectClass: inetOrgPerson
cn: ${uid}
sn: ${uid}
uid: ${uid}
userPassword: ${PROF_PASS}

EOD
  done

  echo "Import LDIF (idempotent) ..."
  ldapadd -c -x -H "${LDAP_URI}" -D "${ADMIN_DN}" -w "${LDAP_ADMIN_PASS}" -f /vagrant/ldap/init.ldif || true

  # Filet de sécurité : crée le groupe s'il manque
  ensure_group () {
    local cn="$1"
    if ! ldapsearch -x -H "${LDAP_URI}" -b "cn=${cn},ou=Groups,${BASE_DN}" -s base dn >/dev/null 2>&1; then
      cat <<EOF | ldapadd -c -x -H "${LDAP_URI}" -D "${ADMIN_DN}" -w "${LDAP_ADMIN_PASS}" || true
dn: cn=${cn},ou=Groups,${BASE_DN}
objectClass: groupOfNames
cn: ${cn}
member: cn=admin,${BASE_DN}
EOF
    fi
  }
  ensure_group "Admins"; ensure_group "Profs"; ensure_group "SISR"; ensure_group "SLAM"

  # Ajout membres (tolérant)
  for i in $(seq -f "%02g" 1 10); do uid="sisr${i}"; ldapmodify -c -x -H "${LDAP_URI}" -D "${ADMIN_DN}" -w "${LDAP_ADMIN_PASS}" <<EOF || true
dn: cn=SISR,ou=Groups,${BASE_DN}
changetype: modify
add: member
member: uid=${uid},ou=People,${BASE_DN}
EOF
  done
  for i in $(seq -f "%02g" 1 5); do uid="slam${i}"; ldapmodify -c -x -H "${LDAP_URI}" -D "${ADMIN_DN}" -w "${LDAP_ADMIN_PASS}" <<EOF || true
dn: cn=SLAM,ou=Groups,${BASE_DN}
changetype: modify
add: member
member: uid=${uid},ou=People,${BASE_DN}
EOF
  done
  for i in $(seq 1 3); do
    uid="prof${i}"
    ldapmodify -c -x -H "${LDAP_URI}" -D "${ADMIN_DN}" -w "${LDAP_ADMIN_PASS}" <<EOF || true
dn: cn=Profs,ou=Groups,${BASE_DN}
changetype: modify
add: member
member: uid=${uid},ou=People,${BASE_DN}
EOF
    ldapmodify -c -x -H "${LDAP_URI}" -D "${ADMIN_DN}" -w "${LDAP_ADMIN_PASS}" <<EOF || true
dn: cn=Admins,ou=Groups,${BASE_DN}
changetype: modify
add: member
member: uid=${uid},ou=People,${BASE_DN}
EOF
  done
  for i in $(seq 1 2); do uid="admin${i}"; ldapmodify -c -x -H "${LDAP_URI}" -D "${ADMIN_DN}" -w "${LDAP_ADMIN_PASS}" <<EOF || true
dn: cn=Admins,ou=Groups,${BASE_DN}
changetype: modify
add: member
member: uid=${uid},ou=People,${BASE_DN}
EOF
  done

  touch /vagrant/ldap/.bootstrapped
  echo "LDAP initialisé (bootstrap)."
else
  echo "Bootstrap LDAP déjà fait — on saute la création d'entrées."
fi

# WireGuard: attendre les peers (POSIX) et exporter les .conf
echo "Attente WireGuard (génération des peers)..."
for i in $(seq 1 60); do
  [ -d /vagrant/wireguard/config/peer1 ] && break || true
  sleep 2
done
mkdir -p /vagrant/wireguard_clients
rm -f /vagrant/wireguard_clients/names.txt
for i in $(seq -f "%02g" 1 10); do echo "sisr${i}" >> /vagrant/wireguard_clients/names.txt; done
for i in $(seq -f "%02g" 1 5); do echo "slam${i}" >> /vagrant/wireguard_clients/names.txt; done
for i in $(seq 1 3); do echo "prof${i}" >> /vagrant/wireguard_clients/names.txt; done
for i in $(seq 1 2); do echo "admin${i}" >> /vagrant/wireguard_clients/names.txt; done
idx=1
while read name; do
  peerpath="/vagrant/wireguard/config/peer${idx}"
  conf=$(ls "${peerpath}"/*.conf 2>/dev/null | head -n1 || true)
  [ -n "$conf" ] && cp "$conf" "/vagrant/wireguard_clients/${name}.conf" || true
  idx=$((idx+1))
done < /vagrant/wireguard_clients/names.txt || true

echo "WireGuard clients copiés dans /vagrant/wireguard_clients."
echo "Déploiement terminé. Services disponibles sur ${HOST_IP}."
SHELL
end
