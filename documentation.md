1. ZABBIX
2. VAULTWARDEN

# 1. ZABBIX
## 1.1 Configuration réseau (IP statique)

```bash
sudo su
```

```bash
vi /etc/netplan/01-network-manager-all.yaml
```


```bash
network:
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
```

```bash
netplan apply
```

```bash
ping 192.168.20.254
```

```bash
ping 192.168.40.10
```

```bash
nslookup bts.lan
```



```bash
apt update -y && apt upgrade -y 
```



## 1.2 integration dans le domaine 
## Installer les paquets nécessaires

```bash
sudo apt update
sudo apt install -y \
realmd sssd sssd-tools \
libnss-sss libpam-sss \
adcli samba-common-bin \
oddjob oddjob-mkhomedir \
packagekit
```

## Découvrir le domaine AD

```bash
realm discover bts.lan
```

## Joindre la machine au domaine

```bash
sudo realm join bts.lan -U administrateur
```

## Effectuer la vérification

```bash
realm list
```
> Aller dans le controleur de domaine
> Outils et ordinateurs Active Directory
> Computers 
> On voit la VM Zabbix


## 1.3 Correspondance DNS 

```
> Outils
> DNS
> Déplier Zone de recherche directe
> Cliquer droit sur BTS.LAN
> Nouvel hôte (A ou AAAA)
> Nom : zabbix
> Adresse IP : 192.168.80.40
> Cocher Créer un pointeur d'enregistrement PTR associé

```

## 1.4 Installer et configurer Zabbix

```bash
sudo -su
```

```bash
apt update -y && apt upgrade -y
```

Installer le dépôt Zabbix

```bash
wget https://repo.zabbix.com/zabbix/7.4/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.4+ubuntu22.04_all.deb
```

```bash
dpkg -i zabbix-release_latest_7.4+ubuntu22.04_all.deb
```

```bash
apt update
```

Installer le serveur Zabbix, l’interface web et l’agent

```bash
apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent
```

Créer la base de données initiale

```bash
apt install mariadb-server mariadb-client
```

```bash
mysql -uroot -p
```

```bash
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
```

```bash
CREATE USER zabbix@localhost IDENTIFIED BY 'P@ssword';
```

```bash
GRANT ALL PRIVILEGES ON zabbix.* TO zabbix@localhost;
```

```bash
SET GLOBAL log_bin_trust_function_creators = 1;
```

```bash
QUIT;
```

Sur le serveur Zabbix, importez le schéma initial et les données (le mot de passe de l’utilisateur zabbix sera demandé) :

```bash
zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

Désactiver l’option après l’import du schéma :

```bash
mysql -uroot -p
```

```bash
SET GLOBAL log_bin_trust_function_creators = 0;
```

```bash
QUIT;
```
## correspondance DNS:
```
> outils
> DNS
> Déplier zone de recherche directe
> clic droit sur BTS.LAN
> nouvel hote (A ou AAAA)
> Nom : 
```

### Configurer la base de données pour le serveur Zabbix


```bash
nano /etc/zabbix/zabbix_server.conf
```

Configurer le mot de passe de la base de données :

```
DBPassword=P@ssword
```

### Démarrer les services Zabbix


```bash
systemctl restart zabbix-server zabbix-agent apache2
systemctl enable zabbix-server zabbix-agent apache2
systemctl status zabbix-server zabbix-agent apache2 mariadb
```

### Accéder à l’interface web Zabbix

```bash
http://zabbix/zabbix
```

## 1.5 Mise en place du HTTPS (CA locale)

Création des répertoires

```bash
mkdir -p /etc/ssl/ca /etc/ssl/apache
```


### 1.5.1 Création de la CA

```bash
cd /etc/ssl/ca
```

```bash
openssl genrsa -out ca.key 4096
```

```bash
openssl req -x509 -new -nodes \
  -key ca.key \
  -sha256 \
  -days 3650 \
  -out ca.crt \
  -subj "/C=FR/O=bts/CN=bts-CA"
```

### 1.5.2 Creation du certificat serveur Zabbix


```bash
cd /etc/ssl/apache
```

```bash
vi openssl-zabbix.cnf
```



```bash
[ req ]
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
```

```bash
openssl genrsa -out zabbix.key 2048
```


```bash
openssl req -new -key zabbix.key -out zabbix.csr -config openssl-zabbix.cnf
```


```bash
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
```

Vérification :


```bash
openssl x509 -in zabbix.crt -noout -subject -issuer
```

### 1.5.3 Configuration Apache HTTPS


VirtualHost HTTPS

```bash
vi /etc/apache2/sites-available/secure.conf
```


```bash
<VirtualHost *:443>
    ServerName zabbix1.bts.lan
    DocumentRoot /usr/share/zabbix/ui

    SSLEngine on
    SSLCertificateFile /etc/ssl/apache/zabbix.crt
    SSLCertificateKeyFile /etc/ssl/apache/zabbix.key

    <Directory /usr/share/zabbix/ui>
        Require all granted
    </Directory>
</VirtualHost>
```


Redirection HTTP → HTTPS

```bash
vi /etc/apache2/sites-available/zabbix-http.conf
```


```bash
<VirtualHost *:80>
    ServerName zabbix1.bts.lan
    Redirect permanent / https://zabbix1.bts.lan/
</VirtualHost>
```

### 1.5.4 Activation Apache


```bash
a2enmod ssl
```

```bash
a2ensite secure.conf
```

```bash
a2ensite zabbix-http.conf
```

```bash
a2dissite 000-default.conf
```

ServerName global :

```bash
vi /etc/apache2/conf-available/servername.conf
```

```bash
ServerName zabbix.bts.lan
```


```bash
a2enconf servername
```

```bash
apachectl configtest
```

```bash
systemctl reload apache2 && systemctl status apache2
```

Vérifications finales

```bash
apt install curl -y
```


```bash
curl -I http://zabbix1.bts.lan
```

Résultat attendu :

HTTP/1.1 301 Moved Permanently
Location: https://zabbix1.bts.lan/


### 1.5.5 Importer le certificat dans Firefox

```
> Paramètres
> Vie privée et sécurité
> Dans la zone Sécurité, cliquer sur Afficher les certificats
> Importer
> Aller dans /etc/ssl/ca, sélectionner le certificat et cliquer sur importer
> Cocher la case
> Valider
```

## 1.6 Accès à l’interface Zabbix
https://zabbix1.bts.lan

```
Identifiants par défaut :

> Utilisateur : Admin

> Mot de passe : zabbix
```


## 1.7 Configuration de Zabbix
### 1.7.1 - Modification des identifiants de connexion par défaut

```
> Utilisateurs > Utilisateurs 
> Cliquer sur Admin 
> Changer le mot de passe
> Mot de passe actuel : zabbix
> Mot de passe : P@ssword*
> Mot de passe (une autre fois) : P@ssword*
> Actualiser 
> Se connecter avec les nouveaux identifiants
```


## connexion au https://zabbix1.bts.lan
```
> identifiant zabbix : Admin
> mot de passe : P@ssword*
```


## 1.7.2 Configuration LDAP et JIT
### Creation de l'OU zabbixAdmin
```
> outil
> utilisateurs et ordinateur active diractory
> clic sur bts 
  > clic sur l'icone : creer une nouvelle unité d'organisation dans le conteneur actuel
     > nom : ZabbixAdmin
     > clic sur ok
  
```
#### creation groupe zabbix administrateur dans l'OU 
```
  > clic droit sur ZabbixAdmin
     > nouveau groupe 
        > nom du groupe : zabbix_Administrateurs 
        > etendu du groupe : globale 
        > type du groupe : sécurité 
        > ok
    > double clic sur zabbix_Administrateur 
    > aller dans l'anglet membre :
       > clic sur ajouter 
         > Dans entrée les noms :
            > saisir le nom des personne qu'on souhaite mettre dans le groupe 
             > clic sur verifier les noms 
               > ajouter 
```

#### Création d'un utilisateur Zabbix Bind dans l'Active Directory

```
> Ouvrir le gestionnaire de serveur
> Outils
> Utilisateurs et ordinateurs Active Directory*
> Action
> Nouveau
> Utilisateur
> Prénom : Zabbix
> Nom : Bind
> Nom d'ouverture de session de l'utilisateur : zbind
> Suivant
> Saisir le mot de passe x2
> decocher l'utilisateur doit changer le mot de passe lors de la nouvelle session 
> Suivant
> Terminer
> 
```

#### Création de groupe d'utilisateurs dans zabbix

```
    > Utilisateurs
    > Groupe d'utilisateus
        > On utilise Zabbix_administrators par défaut. il servira au groupe des administrateurs Zabbix
```
_______________________________________________________
# Sur Zabbix 

#### Configuration de l'authentification :

```
    > Utilisateurs
    > Authentification
        > Onglet Authentification :
            > Authentification par défaut : LDAP 
            > Groupe d'utilisateurs déprovisionnés : Disabled
        > Onglet Parametres LDAP :
            > Cocher Activer l'authentification LDAP
            > Cocher Activer le provisionnement JIT
            > Dans onglet Serveurs, cliquer sur Ajouter :
                > Nom : AD_bts
                > Hôte : 192.168.20.10
                > Port : 389
                > DN de base : OU=ZabbixAdmin,DC=bts,DC=lan
                > Attribut recherché : sAMAccountName
                > DN de lien : zbind@bts.lan
                > Mot de passe de lien : P@ssword*
                > Cocher Configurer le provisionnement JIT
                > Sélectionner Configuration du groupe
                > Attribut du nom du groupe : cn
                > Attribut d'appartenance au groupe d'utilisateurs : memberOf
                > Attribut du nom d'utilisateur : sAMAccountName
                > Attribut du nom de famille de l'utilisateur : sn
                > Correspondance des groupes d'utilisateurs : 
                    > Cliquer sur Ajouter
                        > Modèle de groupe LDAP (le nom du groupe doit correspondre au nom du groupe LDAP) : ZABBIX_Administrateurs
                        > Groupes d'utilisateurs :  Zabbix administrators
                        > Role utilisateur : Super admin role
                        > Actualiser
                     > Cliquer sur Ajouter X2
                > Actualiser
```
___________________________________________________________________________________

### 5.3.2 - Le Serveur Zabbix et les clients Linux

#### Enregistrement automatique des agents LINUX
  
```
    > Cliquer sur Alertes 
    > Actions 
    > Actions d'enregistrement automatique 
    > Cliquer sur Créer une action 
    > Nom : Saisir  : Linux - Enregistrement automatique des agents
    > Conditions
        > Cliquer sur Ajouter 
        > Type : Métadonnées de l'hôte
        > Valeur : Saisir Linux
        > Cliquer sur Ajouter
    > Dans l'onglet opérations,
        > cliquer sur Ajouter 
        > Dans le champs Opération :
            > Sélectionner Ajouter hôte et Cliquer sur Ajouter
            > Sélectionner Ajouter au groupe d'hôtes et dans le champs Groupes d'hôtes, sélectionner Discovered hosts et Cliquer sur Ajouter
            > Sélectionner Lier le modèle et dans Modèles Sélectionner Linux By Zabbix Agent et Cliquer sur Ajouter
        > Cliquer sur Ajouter x2
```

#### Installation de l'agent Zabbix sur les clients Linux (Ex. clientlinux1)

```bash
apt update
```

```bash
apt install zabbix-agent -y
```

```bash
vi /etc/zabbix/zabbix_agentd.conf
```

Modifier les données suivantes:

```
Server=192.168.40.40
ServerActive=192.168.40.40
Hostname=clientlinux1
HostMetadata=Linux
```

```bash
systemctl restart zabbix-agent && systemctl status zabbix-agent && systemctl enable zabbix-agent
```
_____________________________________________________________________________________

Aller sur http://http://192.168.20.4/zabbix/ 


![](../images/LAB-SRV-ZABBIX/img-2.png)

![](../images/LAB-SRV-ZABBIX/img-3.png)


### 5.3.3 - Le Serveur Zabbix et les clients Windows

#### Enregistrement automatique des agents Windows

```
> Cliquer sur Alertes > Actions > Actions d'enregistrement automatique 
> Cliquer sur Créer une action 
> Nom : Saisir Linux - Enregistrement automatique des agents
> Conditions
    > Cliquer sur Ajouter 
    > Type : Métadonnées de l'hôte
    > Valeur : Saisir Windows
    > Cliquer sur Ajouter
> Dans l'onglet opérations,
    > cliquer sur Ajouter 
    > Dans le champs Opération :
        > Sélectionner Ajouter hôte et Cliquer sur Actualiser
        > Sélectionner Ajouter au groupe d'hôtes et dans le champs Groupes d'hôtes, sélectionner Discovered hosts et Cliquer sur Actualiser
        > Sélectionner Lier le modèle et dans Modèles Sélectionner Windows By Zabbix Agent et Cliquer sur Actualiser
    > Cliquer sur Ajouter x2



> Aller sur https://www.zabbix.com/download_agents?version=7.4&release=7.4.6&os=Windows&os_version=Server+2016+%2B&hardware=amd64&encryption=OpenSSL&packaging=MSI&show_legacy=0
> Télécharger l'agent Zabbix et l'exécuter 
> Next 
> Cocher I accept the term in the Licence Agreement 
> Next x2 
> Zabbix server IP/DNS : 192.168.40.40
> Server or Proxy for active checks : 192.168.40.40
> Cocher Add agent location to the PATH 
> Next 
> Install 
> Finish
> Télécharger NotePad++
> L'ouvrir en tant qu'administrateur
> Ouvrir le fichier C:\Program Files\Zabbix Agent\zabbix_agentd.conf 
> Décommenter et Modifier HostMetadata=Windows 
> Enregistrer et fermer le fichier
> Ouvrir Powershell et saisir Stop-Service "Zabbix Agent" et Start-Service "Zabbix Agent" pour redémarrer le service Zabbix Agent
```

### 5.3.4 - Création des actions d’alerte avec escalade

Le but est de d'alerter automatiquement en cas de problème le technicien et de remonter l'information à un niveau hierarchique élevé si le dysfonctionnement n'a pas été traitée dans les temps impartis.


#### Configuration de l'adresse email Administrateur

Il s'agit de l'adresse mail sur laquelle les alertes seront envoyées.

```
Utilisateurs > Utilisateurs > Cliquer sur Admin > Cliquer sur l'onglet Média > Cliquer sur ajouter > Envoyer à : renseigner l'adresse mail > personnaliser le reste des champs selon les besoins > Cliquer sur Ajouter > Cliquer sur Actualiser
```
![](../images/LAB-SRV-ZABBIX/img-6.png)


#### Configuration de l'envoi d'email

Si la boite mail d'envoi est appartient au domain gmail.com, il faut créer un mot de passe d'application Gmail :

    > Aller sur https://myaccount.google.fr/security

    > Activer la validation en deux étapes si ce n’est pas déjà fait

    > Aller sur https://myaccount.google.com/apppasswords
    
    > Sélectionner Autre et le nommer "Zabbix"

    > Cliquer sur Générer

    > Google va générer un mot de passe à 16 caractères

    > Le copier


Alertes > Types de média > Cliquer sur Email > 

![](../images/LAB-SRV-ZABBIX/img-7.png)

Concernant le mot de passe, il faudra renseigner le mot de passe copié (plus haut)


#### Création d'un groupe d'utilisateur et d'un utilisateur

Nous allons créer un utilisateur "celina MBAKOP" qui sera affecté au groupe utilisateur "Techniciens" que nous créerons également.

```
    CREATION D'UN UTILISATEUR :

        Utilisateurs > Utilisateurs > Créer un utilisateur :
            > Nom d'utilisateur : celina Mbakop
            > Prénom : celina
            > Nom de famille :  Mbakop
            > groupe : Guests, Internal
            > reseigner un mot de passe
            > Cocher connexion automatique
            > Dans l'onglet Média, Ajouter une adresse mail
            > Dans l'onglet permissions :
                > Rôle : User role


    CREATION D'UN GROUPE D'UTILISATEURS 

        Utilisateurs > Groupes d'utilisateurs > Créer un groupe d'utilisateur :
            > Nom du groupe : Techniciens
            > Utilisateurs : Sélectionner l'utilisateur celina Mbakop
            > Dans l'onglet Autorisations de l'hôte :
                > Sélectionner tous les groupes d'hotes groupe d'hotes 
                > Cliquer sur Lecture
```

#### Configuration des actions de déclencheurs et Paramètre des escalades d’alerte

L'escalade d'alerte consitera en deux étapes :

- Étape 1 : Dès que le problème est détecté, un mail est envoyé aux Techniciens.

- Étape 2 : Si l’incident n’est pas pris en charge dans la minute (60s choisies dans notre exemple) (pas d’accusé de résolution ou d’acknowledgement), l’alerte est relayée automatiquement aux Administrateurs.

```
    CREATION DE L'ESCALADE D'ALERTE

        Alertes > Actions > Actions de déclencheur > Cliquer sur Report problems to Zabbix administrators :
            > Nom : Alerte d'incident aux Techniciens puis Administrateurs
            
            > Conditions : cliquer sur Ajouter :
                > Type : Sévérité du déclencheur
                > Opérateur : Est supérieur ou égal
                > Sévérité : Avertissement
                > Cliquer sur Ajouter

            > Dans l'onglet Opérations :
                > Durée de l'étape d'opération par défaut : 60s (adapter la durée selon les besoins - 60s ici c'est un exemple)
                > Dans opérations, cliquer sur Ajouter :
                    > Durée de l'étape : 0
                    > Envoyer aux groupes d'utilisateurs : Sélectionner Techniciens
                    > Envoyer au type de média : Sélectionner Email
                    > Cliquer sur Ajouter
                > Dans opérations, cliquer sur Ajouter :
                    > Etapes : 2 - 2
                    > Durée de l'étape : 0
                    > Envoyer aux groupes d'utilisateurs : Sélectionner Zabbix Administrators
                    > Envoyer au type de média : Sélectionner Email
                    > Conditions : cliquer sur Ajouter et dans la fenetre qui s'affiche cliquer de nouveau sur Ajouter
                    > Cliquer sur Ajouter
```

# 2. VAULTWARDEN

## Configuration réseau (IP statique)

```bash
sudo su
```


```bash
nano /etc/netplan/01-network-manager-all.yaml
```


```bash
network:
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
```

```bash
netplan apply
```

```bash
ping 192.168.20.254
```

```bash
ping 192.168.40.10
```

```bash
nslookup bts.lan
```


```bash
apt update -y && apt upgrade -y 
```


## integration dans le domaine 
## Installer les paquets nécessaires

```bash
sudo apt update
sudo apt install -y \
realmd sssd sssd-tools \
libnss-sss libpam-sss \
adcli samba-common-bin \
oddjob oddjob-mkhomedir \
packagekit
```

## Découvrir le domaine AD

```bash
realm discover bts.lan
```

## Joindre la machine au domaine

```bash
sudo realm join bts.lan -U administrateur
```

## Effectuer la vérification

```bash
realm list
```




## Configuration réseau (IP statique)

```bash
sudo su
```


```bash
vi /etc/netplan/01-network-manager-all.yaml
```


```bash
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    ens192:
      dhcp4: no
      addresses:
        - 192.168.80.41/24
      routes:
        - to: default
          via: 192.168.80.254
      nameservers:
        search:
          - bts.lan
        addresses:
          - 192.168.60.10
```

```bash
netplan apply
```

```bash
ping 192.168.80.254
```

```bash
ping 192.168.60.10
```

```bash
nslookup bts.lan
```

```bash
apt update -y && apt upgrade -y
```

## Intégration dans le domaine

### Installer les paquets nécessaires

```bash
sudo apt install -y \
realmd sssd sssd-tools \
libnss-sss libpam-sss \
adcli samba-common-bin \
oddjob oddjob-mkhomedir \
packagekit
```


### Découvrir le domaine AD

```bash
realm discover bts.lan
```

### Joindre la machine au domaine

```bash
sudo realm join bts.lan -U administrateur
```

## Effectuer la vérification

```bash
realm list
```

```
> Aller dans le controleur de domaine
> Outils et ordinateurs Active Directory
> Computers 
> On voit la VM Zabbix
```

## Correspondance DNS 

```
> Outils
> DNS
> Déplier Zone de recherche directe
> Cliquer droit sur BTS.LAN
> Nouvel hôte (A ou AAAA)
> Nom : zabbix
> Adresse IP : 192.168.80.40
> Cocher Créer un pointeur d'enregistrement PTR associé
```


## Installation de Docker

```bash
sudo su
```

```bash
apt update -y && apt upgrade -y
```

```bash
apt install ca-certificates curl
```

```bash
install -m 0755 -d /etc/apt/keyrings
```

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
```

```bash
chmod a+r /etc/apt/keyrings/docker.asc
```

```bash
tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

```bash
apt-get update
```

```bash
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

```bash
docker run hello-world
```

```bash
mkdir -p /opt/vaultwarden/{vw-data,nginx/conf.d,certs}
```

```bash
cd /opt/vaultwarden
```

```bash
docker run --rm -it vaultwarden/server:latest /vaultwarden hash
```

Copier le token ADMIN_TOKEN (la chaîne de caractères uniquement qui est dans les guillemets). On en aura besoin plus tard.

Exemple :

ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$p/BLipIEhcpvfzIMMHO1XT2bZDv48K+2W6sH5Rgzo3g$jo/QXrh8A+K7FnKUD/6kL0zlE3yq6WwlbV3l5oLo9gw'

le token à copier dans cet exemple est $argon2id$v=19$m=65540,t=3,p=4$p/BLipIEhcpvfzIMMHO1XT2bZDv48K+2W6sH5Rgzo3g$jo/QXrh8A+K7FnKUD/6kL0zlE3yq6WwlbV3l5oLo9gw



## Configuration du certificat d'autorité locale et de l'HTTPS

### Sur le serveur Vaultwarden

#### Création du certificat d'autorité

```bash
sudo su
```

```bash
cd /opt
```

```bash
mkdir -p vaultwarden/ca vaultwarden/certs
```

```bash
cd vaultwarden/ca
```

```bash
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.pem \
  -subj "/C=FR/O=CELIWILLI/CN=CELIWILLI Root CA"
```

```bash
ls -l
```

#### Création du certificat serveur

```bash
cd /opt/vaultwarden/certs
```

```bash
openssl genrsa -out privkey.pem 4096
```

```bash
vi vaultwarden.cnf
```

```bash
[req]
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
```

```bash
openssl req -new -key privkey.pem -out vaultwarden.csr -config vaultwarden.cnf
```

#### SIGNATURE

```bash
openssl x509 -req \
  -in vaultwarden.csr \
  -CA /opt/vaultwarden/ca/ca.pem \
  -CAkey /opt/vaultwarden/ca/ca.key \
  -CAcreateserial \
  -out fullchain.pem \
  -days 825 \
  -sha256 \
  -extfile vaultwarden.cnf \
  -extensions req_ext
```


#### Configuration de NGINX

```bash
vi /opt/vaultwarden/nginx/conf.d/vaultwarden.conf
```

```bash
upstream vaultwarden-default {
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
```

#### Configuration du fichier .env

```bash
nano /opt/vaultwarden/.env
```

```bash
DOMAIN=https://vaultwarden1.bts.lan
TZ=Europe/Paris
SIGNUPS_ALLOWED=true
ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$p/BLipIEhcpvfzIMMHO1XT2bZDv48K+2W6sH5Rgzo3g$jo/QXrh8A+K7FnKUD/6kL0zlE3yq6WwlbV3l5oLo9gw'
```

#### Configuration de Docker Compose

```bash
nano /opt/vaultwarden/docker-compose.yml
```

```bash
services:
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

```

```bash
cd /opt/vaultwarden
```

```bash
docker compose down
```

```bash
docker compose up -d
```

#### Test Serveur

```bash
openssl s_client \
  -connect vaultwarden.celiwilli.lan:443 \
  -servername vaultwarden.celiwilli.lan \
  -CAfile /opt/vaultwarden/ca/ca.pem
```


## installation ssh
``` bash
apt install ssh -y
``` 

``` bash
systemctl status ssh
``` 




### Sur le serveur Zabbix1



#### Importation du certificat d'autorité 


```bash
scp celina@vaultwarden1:/opt/vaultwarden/ca/ca.pem /home/celina/ca.pem
```

```bash
chown celina:celina /home/celina/ca.pem
```

Dans le navigateur firefox

```
> Paramètres
> Vie privée et sécurité
> Certificats
> Afficher les certificats
> Autorités
> Importer
> sélectionner /home/celina/ca.pem
> Faire confiance pour identifier des sites web
> Redémarrer Firefox
> saisir https://vaultwarden.celiwilli.lan
```

#### VCréation d'un compte

```
> Créer un compte :
> Email : admin@celiwilli.lan
> Name : admin
> Mot de passe : P@ssword123*
> Valider
> Ajouter l'extension
```

## Si on souhaite empêcher n’importe qui de créer un compte Vaultwarden

```bash
vi /opt/vaultwarden/.env
```

Modifier

```
SIGNUPS_ALLOWED=false  
```

Redémarrer Docker

```bash
cd /opt/vaultwarden
```

```bash
docker compose down
```

```bash
docker compose up -d
```

## Ajouter l'extension Vaultwarden au navigateur


```
> Se connecter sur le navigateur
> Ajouter  l'extension du navigateur
> Attention : sélectionner Autohébergé
> Url : https://vaultwarden.celiwilli.lan
> Email : vaultwarden1.bts@gmail.com
> Mot de passe : P@ssword123*
```


## Procédure pour que le stack Vaultwarden (Vaultwarden + Nginx via Docker Compose) soit géré comme un service systemd.

```bash
vi /etc/systemd/system/vaultwarden.service
```


```bash
[Unit]
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
```

```bash
systemctl daemon-reload
```

```bash
systemctl enable vaultwarden
```

```bash
systemctl start vaultwarden && systemctl status vaultwarden
```

Les commandes suivantes sont dorénavant possibles :

```
# Démarrer Vaultwarden
systemctl start vaultwarden

# Vérifier l’état
systemctl status vaultwarden

# Arrêter Vaultwarden
systemctl stop vaultwarden

# Redémarrer Vaultwarden
sudo systemctl restart vaultwarden
```

Il ne faudra plus effectuer les commandes ci-dessous pour démarrer 

```bash
cd /opt/vaultwarden
```

```bash
docker compose down
```

```bash
docker compose up -d
```


## Mise en place du MFA TOTP

### Sur Zabbix
> Utilisateurs
> Authentification
> Onglet Paramètres d'authentification multi-facteur
    > Cocher Activer l'authentification multi-facteur
    > Ajouter
    > Nom :  TOTP-Bitwarden
    > Fonction de hachage : SHA-1
    > Longueur du code : 6
    > Ajouter
> Onglet Parametres LDAP
    > Cliquer sur AD_celiwilli
    > Cliquer sur ZABBIX_Administrateurs
    > Dans Groupes d'utilisateurs, Ajouter Groupe TOTP
    > Actualiser
    > Cliquer sur ZABBIX_Techniciens
    > Dans Groupes d'utilisateurs, Ajouter Groupe TOTP
    > Actualiser x2


### Sur Bitwarden

> Créer un utilisateur (ex : wmbakop@celiwilli.lan)
> Mot de passe : P@ssword123*
> Dans le coffe-fort, cliquer sur Nouvel identifiant
    > Nom de l'élément : Zabbix – Admin
    > Nom d'utilisateur : wmbakop@celiwilli.lan
    > Mot de passe : Password123*
    > Clé d'authentification : La clé donnée par Zabbix (cf. capture écran)
    > Valider
    > Copier le code de vérification TOTP et le coller dans Zabbix


### Sur Zabbix

> Coller le code de vérification TOTP et se connecter
    https://vaultwarden.celiwilli.lan/admin
    Mot de passe : Password
  > Il sera demandé le code de vérification. Ce code se trouve dans Vaultwarden et est réinitilialiser toutes les 30 secondes