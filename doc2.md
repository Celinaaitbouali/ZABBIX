sudo su
bashvi /etc/netplan/01-network-manager-all.yaml
yamlnetwork:
  version: 2
  renderer: NetworkManager
  ethernets:
    ens192:
      dhcp4: no
      addresses:
        - 192.168.40.40/24
      routes:
        - to: default
          via: 192.168.40.254
      nameservers:
        search:
          - bts.lan
        addresses:
          - 192.168.20.10
bashnetplan apply
ping 192.168.20.254
ping 192.168.40.10
nslookup bts.lan
apt update -y && apt upgrade -y

1.2 Intégration dans le domaine Active Directory
Installer les paquets nécessaires
bashsudo apt update
sudo apt install -y \
  realmd sssd sssd-tools \
  libnss-sss libpam-sss \
  adcli samba-common-bin \
  oddjob oddjob-mkhomedir \
  packagekit
Découvrir le domaine AD
bashrealm discover bts.lan
Joindre la machine au domaine
bashsudo realm join bts.lan -U administrateur
Vérification
bashrealm list


Aller dans le contrôleur de domaine
Outils > Utilisateurs et ordinateurs Active Directory
Computers → on voit la VM Zabbix



1.3 Correspondance DNS
> Outils
> DNS
> Déplier Zone de recherche directe
> Clic droit sur BTS.LAN
> Nouvel hôte (A ou AAAA)
> Nom : zabbix
> Adresse IP : 192.168.40.40
> Cocher "Créer un pointeur d'enregistrement PTR associé"

1.4 Installer et configurer Zabbix
bashsudo su
apt update -y && apt upgrade -y
Installer le dépôt Zabbix
bashwget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu22.04_all.deb
dpkg -i zabbix-release_latest_7.4+ubuntu22.04_all.deb
apt update
Installer le serveur Zabbix, l'interface web et l'agent
bashapt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
Créer la base de données initiale
bashapt install mariadb-server mariadb-client
mysql -uroot -p
sqlCREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER zabbix@localhost IDENTIFIED BY 'P@ssword';
GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;
SET GLOBAL log_bin_trust_function_creators = 1;
QUIT;
Importer le schéma initial
bashzcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
Désactiver l'option après import
bashmysql -uroot -p
sqlSET GLOBAL log_bin_trust_function_creators = 0;
QUIT;
Configurer la base de données pour le serveur Zabbix
bashnano /etc/zabbix/zabbix_server.conf
Ajouter/modifier :
DBPassword=P@ssword
Démarrer les services Zabbix
bashsystemctl restart zabbix-server zabbix-agent apache2
systemctl enable zabbix-server zabbix-agent apache2
systemctl status zabbix-server zabbix-agent apache2 mariadb
Accéder à l'interface web Zabbix
http://zabbix/zabbix

1.5 Mise en place du HTTPS (CA locale)
Création des répertoires
bashmkdir -p /etc/ssl/ca /etc/ssl/apache
1.5.1 Création de la CA
bashcd /etc/ssl/ca
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes \
  -key ca.key \
  -sha256 \
  -days 3650 \
  -out ca.crt \
  -subj "/C=FR/O=bts/CN=bts-CA"
1.5.2 Création du certificat serveur Zabbix
bashcd /etc/ssl/apache
vi openssl-zabbix.cnf
ini[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = req_ext

[ dn ]
C = FR
O = bts
CN = zabbix.bts.lan

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = zabbix.bts.lan
IP.1  = 192.168.40.40
bashopenssl genrsa -out zabbix.key 2048
openssl req -new -key zabbix.key -out zabbix.csr -config openssl-zabbix.cnf
openssl x509 -req \
  -in zabbix.csr \
  -CA /etc/ssl/ca/ca.crt \
  -CAkey /etc/ssl/ca/ca.key \
  -CAcreateserial \
  -out zabbix.crt \
  -days 365 \
  -sha256 \
  -extensions req_ext \
  -extfile openssl-zabbix.cnf
Vérification :
bashopenssl x509 -in zabbix.crt -noout -subject -issuer
1.5.3 Configuration Apache HTTPS
VirtualHost HTTPS :
bashvi /etc/apache2/sites-available/secure.conf
apache<VirtualHost *:443>
    ServerName zabbix1.bts.lan
    DocumentRoot /usr/share/zabbix/ui

    SSLEngine on
    SSLCertificateFile /etc/ssl/apache/zabbix.crt
    SSLCertificateKeyFile /etc/ssl/apache/zabbix.key

    <Directory /usr/share/zabbix/ui>
        Require all granted
    </Directory>
</VirtualHost>
Redirection HTTP → HTTPS :
bashvi /etc/apache2/sites-available/zabbix-http.conf
apache<VirtualHost *:80>
    ServerName zabbix1.bts.lan
    Redirect permanent / https://zabbix1.bts.lan/
</VirtualHost>
1.5.4 Activation Apache
basha2enmod ssl
a2ensite secure.conf
a2ensite zabbix-http.conf
a2dissite 000-default.conf
ServerName global :
bashvi /etc/apache2/conf-available/servername.conf
ServerName zabbix.bts.lan
basha2enconf servername
apachectl configtest
systemctl reload apache2 && systemctl status apache2
Vérification finale :
bashapt install curl -y
curl -I http://zabbix1.bts.lan
Résultat attendu :
HTTP/1.1 301 Moved Permanently
Location: https://zabbix1.bts.lan/
1.5.5 Importer le certificat dans Firefox
> Paramètres
> Vie privée et sécurité
> Dans la zone Sécurité, cliquer sur "Afficher les certificats"
> Importer
> Aller dans /etc/ssl/ca, sélectionner le certificat et cliquer sur Importer
> Cocher la case
> Valider

1.6 Accès à l'interface Zabbix
URL : https://zabbix1.bts.lan
Identifiants par défaut :
ChampValeurUtilisateurAdminMot de passezabbix

1.7 Configuration de Zabbix
1.7.1 Modification des identifiants de connexion par défaut
> Utilisateurs > Utilisateurs
> Cliquer sur Admin
> Changer le mot de passe
> Mot de passe actuel : zabbix
> Nouveau mot de passe : P@ssword*
> Confirmer le mot de passe : P@ssword*
> Actualiser
> Se connecter avec les nouveaux identifiants

1.7.2 Configuration LDAP et JIT
Création de l'OU ZabbixAdmin (sur le contrôleur de domaine)
> Outils > Utilisateurs et ordinateurs Active Directory
> Clic sur bts
> Clic sur l'icône "Créer une nouvelle unité d'organisation dans le conteneur actuel"
  > Nom : ZabbixAdmin
  > OK
Création du groupe zabbix_Administrateurs dans l'OU
> Clic droit sur ZabbixAdmin
> Nouveau > Groupe
  > Nom du groupe : zabbix_Administrateurs
  > Étendue du groupe : Globale
  > Type du groupe : Sécurité
  > OK
> Double-clic sur zabbix_Administrateurs
> Onglet Membres > Ajouter
  > Saisir les noms des utilisateurs à ajouter
  > Vérifier les noms > Ajouter
Création d'un utilisateur Zabbix Bind dans l'Active Directory
> Gestionnaire de serveur > Outils
> Utilisateurs et ordinateurs Active Directory
> Action > Nouveau > Utilisateur
  > Prénom : Zabbix
  > Nom : Bind
  > Nom d'ouverture de session : zbind
  > Suivant
  > Saisir le mot de passe x2
  > Décocher "L'utilisateur doit changer le mot de passe à la prochaine session"
  > Suivant > Terminer
Création de groupe d'utilisateurs dans Zabbix
> Utilisateurs > Groupes d'utilisateurs
> Utiliser "Zabbix administrators" par défaut pour les administrateurs Zabbix
Configuration de l'authentification LDAP dans Zabbix
> Utilisateurs > Authentification
> Onglet Authentification :
    > Authentification par défaut : LDAP
    > Groupe d'utilisateurs déprovisionnés : Disabled
> Onglet Paramètres LDAP :
    > Cocher "Activer l'authentification LDAP"
    > Cocher "Activer le provisionnement JIT"
    > Dans l'onglet Serveurs, cliquer sur Ajouter :
        > Nom : AD_bts
        > Hôte : 192.168.20.10
        > Port : 389
        > DN de base : OU=ZabbixAdmin,DC=bts,DC=lan
        > Attribut recherché : sAMAccountName
        > DN de lien : zbind@bts.lan
        > Mot de passe de lien : P@ssword*
        > Cocher "Configurer le provisionnement JIT"
        > Sélectionner "Configuration du groupe"
        > Attribut du nom du groupe : cn
        > Attribut d'appartenance au groupe : memberOf
        > Attribut du nom d'utilisateur : sAMAccountName
        > Attribut du nom de famille : sn
        > Correspondance des groupes d'utilisateurs > Ajouter :
            > Modèle de groupe LDAP : ZABBIX_Administrateurs
            > Groupes d'utilisateurs : Zabbix administrators
            > Rôle utilisateur : Super admin role
            > Actualiser
        > Cliquer sur Ajouter x2
    > Actualiser

1.8 Enregistrement automatique des agents
1.8.1 Agents Linux
Création de l'action d'enregistrement automatique
> Alertes > Actions > Actions d'enregistrement automatique
> Créer une action
> Nom : Linux - Enregistrement automatique des agents
> Conditions :
    > Ajouter
    > Type : Métadonnées de l'hôte
    > Valeur : Linux
    > Ajouter
> Onglet Opérations > Ajouter :
    > Sélectionner "Ajouter hôte" > Ajouter
    > Sélectionner "Ajouter au groupe d'hôtes" > Discovered hosts > Ajouter
    > Sélectionner "Lier le modèle" > Linux By Zabbix Agent > Ajouter
> Ajouter x2
Installation de l'agent Zabbix sur les clients Linux
bashapt update
apt install zabbix-agent -y
vi /etc/zabbix/zabbix_agentd.conf
Modifier les paramètres suivants :
Server=192.168.40.40
ServerActive=192.168.40.40
Hostname=clientlinux1
HostMetadata=Linux
bashsystemctl restart zabbix-agent
systemctl enable zabbix-agent
systemctl status zabbix-agent

1.8.2 Agents Windows
Création de l'action d'enregistrement automatique
> Alertes > Actions > Actions d'enregistrement automatique
> Créer une action
> Nom : Windows - Enregistrement automatique des agents
> Conditions :
    > Ajouter
    > Type : Métadonnées de l'hôte
    > Valeur : Windows
    > Ajouter
> Onglet Opérations > Ajouter :
    > Sélectionner "Ajouter hôte" > Actualiser
    > Sélectionner "Ajouter au groupe d'hôtes" > Discovered hosts > Actualiser
    > Sélectionner "Lier le modèle" > Windows By Zabbix Agent > Actualiser
> Ajouter x2
Installation de l'agent Zabbix sur les clients Windows
> Aller sur https://www.zabbix.com/download_agents
> Télécharger l'agent Zabbix (version Windows / MSI / amd64)
> Exécuter l'installeur :
    > Next
    > Accepter la licence > Next x2
    > Zabbix server IP/DNS : 192.168.40.40
    > Server or Proxy for active checks : 192.168.40.40
    > Cocher "Add agent location to the PATH"
    > Next > Install > Finish
> Ouvrir Notepad++ en tant qu'administrateur
> Ouvrir C:\Program Files\Zabbix Agent\zabbix_agentd.conf
> Décommenter et modifier : HostMetadata=Windows
> Enregistrer et fermer
> Dans PowerShell :
    Stop-Service "Zabbix Agent"
    Start-Service "Zabbix Agent"

1.9 Système d'alertes par email
1.9.1 Création d'un mot de passe d'application Gmail
Afin de sécuriser l'envoi des emails, un mot de passe spécifique doit être généré.
> Accéder à : https://myaccount.google.com/security
> Activer la validation en 2 étapes (si ce n'est pas déjà fait)
> Accéder à : https://myaccount.google.com/apppasswords
> Créer une application :
    > Nom : Zabbix
> Un mot de passe à 16 caractères est généré → le copier
1.9.2 Configuration du type de média dans Zabbix
> Alertes > Types de média > Email
Paramètre
    > Valeur : SMTP 
    > server : smtp.gmail.com
    > Port :587
    > Sécurité: STARTTLS 
    > SMTP helo : gmail.com
    > Authentification Nom d'utilisateur et mot de passe
    > Nom utilisateur
    > adresse Gmail
    > Mot de passe: mot de passe d'application généré

1.9.3 Configuration de l'adresse email de l'administrateur
> Utilisateurs > Utilisateurs > Admin
> Onglet Média > Ajouter
    > Type : Email
    > Adresse : adresse Gmail
    > Période : 24/7
    > Gravité : toutes
> Ajouter > Actualiser

Test a faire :
Desactiver l'agent zabbix sur une des VM 


Mise en place d'une alerte via Discord

Sur Discord, il faut :


> créer un compte sur Discord
> Créer un serveur nommé Zabbix
> Créer un salon nommé Alertes
> Créer un webhook Discord
  > Paramètres du salon
  > Intégrations
  > Webhooks
  > Créer un webhook
  > Cliquer sur le webhook
  > Cliquer sur Copier l'URL du webhook


Sur Zabbix

Configuration du type de média Discord (Webhook) dans Zabbix


> Alertes
> Type de media
> Cliquer sur Discord
> Nom : Alerte via DIscord
> Type : Webhook
> Dans Paramètres, aller sur le champs Zabbix_url et copier sa valeur {$ZABBIX.URL}
> Actualiser


Définition de la macro globale {$ZABBIX.URL} dans Zabbix


> Administration
> Macro
> Ajouter
> Dans Macro Coller {$ZABBIX.URL} et dans la valeur mettre https://zabbix2.bts.lan/zabbix
> Actualiser


Association du webhook Discord à l’utilisateur Admin (média Zabbix)


> Utilisateurs
> Utilisateurs
> Cliquer sur Admin
> Aller dans l'onglet média
> Cliquer sur Ajouter
  > Type : Discord
  > Envoyer à : coller l'url du webbhook de Discord
  > Laisser toutes les autres cases cochées
> Actualiser


Configuration des opérations d’alerte (déclencheur, récupération, mise à jour) dans Zabbix


> Alertes
> Actions 
> Actions de déclencheur7
> Dans la zone Opérations
  > Cliquer sur Report problems to Zabbix administrators
  > Allez dans l'onhglet Opérations et cliquer sur Edition
  > Envoyer au groupe d'utilisateurs : Zabbix Administrators
  > Envoyer au type de média : Tous disponible
> Dans la zone Opération de récupération
  > Cliquer sur REport problems to Zabbix administrators
  > Allez dans l'onglet Opérations et cliquer sur Edition
  > Envoyer au groupe d'utilisateurs : Zabbix Administrators
  > Envoyer au type de média : Tous disponible
> Dans la zone Opération de mise à jour
  > Cliquer sur REport problems to Zabbix administrators
  > Allez dans l'onglet Opérations et cliquer sur Edition
  > Envoyer au groupe d'utilisateurs : Zabbix Administrators
  > Envoyer au type de média : Tous disponible
> Actualiser


Test des notifications Zabbix (mail et Discord) par arrêt de l’agent


> Faire un test : stopper l'agent zabbix sur une vm (ex: vaultwarden)
> Une notification par mail et par Discord est envoyée
# créer l'alerte dans zabbix :



















1.9.4 Création d'un groupe d'utilisateurs et d'un utilisateur Technicien
Création d'un utilisateur
> Utilisateurs > Utilisateurs > Créer un utilisateur
    > Nom d'utilisateur : celina.mbakop
    > Prénom : celina
    > Nom de famille : Mbakop
    > Groupes : Guests, Internal
    > Mot de passe : (à définir)
    > Cocher "Connexion automatique"
    > Onglet Média : ajouter une adresse mail
    > Onglet Permissions :
        > Rôle : User role
Création d'un groupe d'utilisateurs
> Utilisateurs > Groupes d'utilisateurs > Créer un groupe d'utilisateurs
    > Nom du groupe : Techniciens
    > Utilisateurs : celina Mbakop
    > Onglet Autorisations de l'hôte :
        > Sélectionner tous les groupes d'hôtes
        > Cliquer sur Lecture
1.9.5 Création de l'action d'alerte avec escalade
L'escalade d'alerte se déroule en deux étapes :

Étape 1 : Dès que le problème est détecté, un mail est envoyé aux Techniciens.
Étape 2 : Si l'incident n'est pas pris en charge dans la minute (60s), l'alerte est relayée automatiquement aux Administrateurs.

> Alertes > Actions > Actions de déclencheur
> Cliquer sur "Report problems to Zabbix administrators"
> Nom : Alerte d'incident aux Techniciens puis Administrateurs

> Conditions > Ajouter :
    > Type : Sévérité du déclencheur
    > Opérateur : Est supérieur ou égal
    > Sévérité : Avertissement
    > Ajouter

> Onglet Opérations :
    > Durée de l'étape d'opération par défaut : 60s
    > Opérations > Ajouter :
        > Durée de l'étape : 0
        > Envoyer aux groupes d'utilisateurs : Techniciens
        > Envoyer au type de média : Email
        > Ajouter
    > Opérations > Ajouter :
        > Étapes : 2 - 2
        > Durée de l'étape : 0
        > Envoyer aux groupes d'utilisateurs : Zabbix Administrators
        > Envoyer au type de média : Email
        > Conditions > Ajouter > Ajouter
        > Ajouter

📸 Capture à ajouter : Configuration de l'action d'alerte avec escalade

1.9.6 Personnalisation du message d'alerte
Exemple de message d'alerte :
🚨 ALERTE ZABBIX 🚨
Hôte      : {HOST.NAME}
IP        : {HOST.IP}
Problème  : {TRIGGER.NAME}
Gravité   : {TRIGGER.SEVERITY}
Date      : {EVENT.DATE} {EVENT.TIME}

Détails :
{TRIGGER.DESCRIPTION}
1.9.7 Test de fonctionnement
Méthode 1 — Test SMTP dans Zabbix :
> Alertes > Types de média > Email
> Cliquer sur "Tester"
> Saisir une adresse de destination
> Vérifier la réception de l'email
Méthode 2 — Simulation de panne :
bash# Stopper l'agent pour simuler un incident
systemctl stop zabbix-agent
Résultat attendu : détection du problème + envoi automatique d'un email.
bash# Remettre en service
systemctl start zabbix-agent

📸 Captures à ajouter : Test SMTP réussi + email reçu dans la boîte mail

1.9.8 Problèmes rencontrés
ErreurCauseSolutionLogin deniedUtilisation du mot de passe Gmail classiqueCréer un mot de passe d'application

1.10 Mise en place du MFA TOTP (avec Vaultwarden)
Sur Zabbix
> Utilisateurs > Authentification
> Onglet Paramètres d'authentification multi-facteur :
    > Cocher "Activer l'authentification multi-facteur"
    > Ajouter :
        > Nom : TOTP-Bitwarden
        > Fonction de hachage : SHA-1
        > Longueur du code : 6
        > Ajouter
> Onglet Paramètres LDAP :
    > Cliquer sur AD_bts > ZABBIX_Administrateurs
    > Dans Groupes d'utilisateurs, ajouter Groupe TOTP > Actualiser
    > Cliquer sur ZABBIX_Techniciens
    > Dans Groupes d'utilisateurs, ajouter Groupe TOTP > Actualiser x2
Sur Vaultwarden (Bitwarden)
> Créer un utilisateur (ex : wmbakop@bts.lan)
> Mot de passe : P@ssword123*
> Dans le coffre-fort, cliquer sur "Nouvel identifiant"
    > Nom de l'élément : Zabbix – Admin
    > Nom d'utilisateur : wmbakop@bts.lan
    > Mot de passe : P@ssword123*
    > Clé d'authentification : (clé fournie par Zabbix)
    > Valider
> Copier le code de vérification TOTP et le coller dans Zabbix

2. VAULTWARDEN
2.1 Configuration réseau (IP statique)
bashsudo su
nano /etc/netplan/01-network-manager-all.yaml
yamlnetwork:
  version: 2
  renderer: NetworkManager
  ethernets:
    ens192:
      dhcp4: no
      addresses:
        - 192.168.40.41/24
      routes:
        - to: default
          via: 192.168.40.254
      nameservers:
        search:
          - bts.lan
        addresses:
          - 192.168.20.10
bashnetplan apply
ping 192.168.20.254
ping 192.168.40.10
nslookup bts.lan
apt update -y && apt upgrade -y

2.2 Intégration dans le domaine Active Directory
Installer les paquets nécessaires
bashsudo apt install -y \
  realmd sssd sssd-tools \
  libnss-sss libpam-sss \
  adcli samba-common-bin \
  oddjob oddjob-mkhomedir \
  packagekit
Découvrir le domaine AD
bashrealm discover bts.lan
Joindre la machine au domaine
bashsudo realm join bts.lan -U administrateur
Vérification
bashrealm list

2.3 Correspondance DNS
> Outils > DNS
> Déplier Zone de recherche directe
> Clic droit sur BTS.LAN > Nouvel hôte (A ou AAAA)
    > Nom : vaultwarden1
    > Adresse IP : 192.168.40.41
    > Cocher "Créer un pointeur d'enregistrement PTR associé"

2.4 Installation de Docker
bashsudo su
apt update -y && apt upgrade -y
apt install ca-certificates curl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

apt-get update
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
docker run hello-world
Créer la structure de répertoires
bashmkdir -p /opt/vaultwarden/{vw-data,nginx/conf.d,certs}
cd /opt/vaultwarden
Générer le token admin
bashdocker run --rm -it vaultwarden/server:latest /vaultwarden hash

Copier la chaîne de caractères entre guillemets générée (ADMIN_TOKEN).
Exemple :
ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$...'


2.5 Configuration du certificat HTTPS (CA locale)
Création du certificat d'autorité
bashsudo su
cd /opt
mkdir -p vaultwarden/ca vaultwarden/certs
cd vaultwarden/ca

openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.pem \
  -subj "/C=FR/O=CELIWILLI/CN=CELIWILLI Root CA"
ls -l
Création du certificat serveur
bashcd /opt/vaultwarden/certs
openssl genrsa -out privkey.pem 4096
vi vaultwarden.cnf
ini[req]
prompt = no
default_md = sha256
req_extensions = req_ext
distinguished_name = dn

[dn]
C = FR
O = CELIWILLI
CN = vaultwarden1.bts.lan

[req_ext]
subjectAltName = @alt_names

[alt_names]
DNS.1 = vaultwarden1.bts.lan
IP.1  = 192.168.40.41
bashopenssl req -new -key privkey.pem -out vaultwarden.csr -config vaultwarden.cnf
Signature du certificat
bashopenssl x509 -req \
  -in vaultwarden.csr \
  -CA /opt/vaultwarden/ca/ca.pem \
  -CAkey /opt/vaultwarden/ca/ca.key \
  -CAcreateserial \
  -out fullchain.pem \
  -days 825 \
  -sha256 \
  -extfile vaultwarden.cnf \
  -extensions req_ext

2.6 Configuration de NGINX
bashvi /opt/vaultwarden/nginx/conf.d/vaultwarden.conf
nginxupstream vaultwarden-default {
  zone vaultwarden-default 64k;
  server vaultwarden:80;
  keepalive 2;
}

map $http_upgrade $connection_upgrade {
  default upgrade;
  ''      "";
}

server {
  listen 80;
  listen [::]:80;
  server_name vaultwarden1.bts.lan;
  return 301 https://$host$request_uri;
}

server {
  listen 443 ssl;
  listen [::]:443 ssl;
  http2 on;
  server_name vaultwarden1.bts.lan;

  ssl_certificate     /etc/nginx/certs/fullchain.pem;
  ssl_certificate_key /etc/nginx/certs/privkey.pem;

  client_max_body_size 525M;

  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection $connection_upgrade;

  proxy_set_header Host $host;
  proxy_set_header X-Real-IP $remote_addr;
  proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  proxy_set_header X-Forwarded-Proto $scheme;

  location / {
    proxy_pass http://vaultwarden-default;
  }
}

2.7 Configuration du fichier .env
bashnano /opt/vaultwarden/.env
envDOMAIN=https://vaultwarden1.bts.lan
TZ=Europe/Paris
SIGNUPS_ALLOWED=true
ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$...'

2.8 Configuration de Docker Compose
bashnano /opt/vaultwarden/docker-compose.yml
yamlservices:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - ./vw-data:/data

  nginx:
    image: nginx:1.27-alpine
    container_name: vw-nginx
    restart: unless-stopped
    depends_on:
      - vaultwarden
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
Démarrage des conteneurs
bashcd /opt/vaultwarden
docker compose down
docker compose up -d
Test du serveur
bashopenssl s_client \
  -connect vaultwarden1.bts.lan:443 \
  -servername vaultwarden1.bts.lan \
  -CAfile /opt/vaultwarden/ca/ca.pem

2.9 Installation SSH (optionnel)
bashapt install ssh -y
systemctl status ssh

2.10 Importation du certificat CA dans Firefox (depuis Zabbix)
bashscp celina@vaultwarden1:/opt/vaultwarden/ca/ca.pem /home/celina/ca.pem
chown celina:celina /home/celina/ca.pem
Dans Firefox :
> Paramètres > Vie privée et sécurité > Certificats
> Afficher les certificats > Autorités > Importer
> Sélectionner /home/celina/ca.pem
> Cocher "Faire confiance pour identifier des sites web"
> Redémarrer Firefox
> Accéder à https://vaultwarden1.bts.lan

2.11 Création d'un compte Vaultwarden
> Créer un compte :
    > Email : admin@bts.lan
    > Nom : admin
    > Mot de passe : P@ssword123*
> Valider
> Ajouter l'extension navigateur

2.12 Désactiver les inscriptions publiques
bashvi /opt/vaultwarden/.env
Modifier :
SIGNUPS_ALLOWED=false
Redémarrer :
bashcd /opt/vaultwarden
docker compose down
docker compose up -d

2.13 Ajouter l'extension Vaultwarden au navigateur
> Se connecter sur le navigateur
> Ajouter l'extension du navigateur Bitwarden
> Attention : sélectionner "Autohébergé"
    > URL : https://vaultwarden1.bts.lan
    > Email : admin@bts.lan
    > Mot de passe : P@ssword123*

2.14 Gérer Vaultwarden comme un service systemd
bashvi /etc/systemd/system/vaultwarden.service
ini[Unit]
Description=Vaultwarden via Docker Compose
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
WorkingDirectory=/opt/vaultwarden
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
bashsystemctl daemon-reload
systemctl enable vaultwarden
systemctl start vaultwarden && systemctl status vaultwarden
Commandes disponibles :
bash# Démarrer
systemctl start vaultwarden

# Vérifier l'état
systemctl status vaultwarden

# Arrêter
systemctl stop vaultwarden

# Redémarrer
systemctl restart vaultwarden

Résultat final
Le système d'alerte Zabbix est fonctionnel :

Envoi automatique d'emails via SMTP sécurisé (STARTTLS / mot de passe d'application Gmail)
Notification en temps réel dès détection d'un incident
Supervision complète : CPU, RAM, services...
Escalade automatique vers les administrateurs si non traité dans les délais

Vaultwarden est déployé et sécurisé :

HTTPS avec CA locale
Intégré au domaine Active Directory
Géré comme un service systemd
MFA TOTP activé avec Zabbix


Améliorations possibles

Ajout d'alertes filtrées par gravité
Intégration SMS / Telegram
Escalade avancée multi-niveaux
Personnalisation poussée des messages d'alerte