# **Installation openstack rocky service par service**

_**1)environnement :**_
  **a)sécurité :**
  
  Pour accepter les caractères spéciaux  dans les mots de passe :
 `$ openssl rand -hex 10`
 

> 5254d6acef68517f5a33

 
**b) Configurer les interfaces réseau**

nous avons configurer deux interfaces réseau :
le premier sous réseaux est dans un plage 192.168.2.0/24 pour avoir une communication entre noeud controller et compute ,et l'autre sous réseaux 192.168.1.0/24 pour avoir acceder au internet.
Nous avons modifier le fichier /etc/hosts et le fichier /etc/hostname :

` # nano /etc/hosts `
192.168.2.22 controller
192.168.2.3 compute
` # nano /etc/hostname `
controller ou compute selon le noeud 

**NB**:cette manipulation sera fait dans chaque noeud 
 **c)Network Time Protocol (NTP)**
 Effectuer ces opérations sur le nœud contrôleur. 
 `# apt-get install chrony`
 
 Editer le fichier chrony.conf 
`# nano /etc/chrony/chrony.conf`

 server NTP_SERVER iburst
 Remplacer `NTP_SERVER` par le hostname ou l’adresse IP d’un serveur NTP approprié
-> server server 0.africa.pool.ntp.org iburst 
Pour permettre aux autres  de se connecter au démon chrony présent sur le nœud controlleur, ajouter cette clef au fichier `chrony.conf`:
-> allow 192.168.2.0/24

Redémarrer le service NTP :
`# service chrony restart`
 Effectuer ces opérations sur le nœud compute. 
        ◦ Installer et configurer NTP
`# apt-get install chrony`
éditez le fichier /etc/chrony/chrony.conf
 server controller iburst
Redémarrez le service NTP
 `# service chrony restart`
**d) Activer le référentiel pour Ubuntu Cloud Archive**
Executez ces commandes sur les deux noeuds 
`# add-apt-repository cloud-archive:rocky`
`# apt install software-properties-common`

 * Mettre à niveau les packages sur tous les nœuds:
`# apt-get update && apt-get dist-upgrade`
`# reboot`

 * Installer le client OpenStack:
`# apt-get install python-openstackclient`

**e) Base de données SQL sur Ubuntu**
Effectuer ces opérations sur le nœud contrôleur.
`# apt-get install mariadb-server python-pymysql`

Créer et éditer le fichier `` /etc/mysql/mariadb.conf.d/99-openstack.cnf`` et effectuer les modifications suivantes :

    [mysqld]
    bind-address = 192.168.2.22
    
    default-storage-engine = innodb
    innodb_file_per_table = on
    max_connections = 4096
    collation-server = utf8_general_ci
    character-set-server = utf8
    
Redémarrer le service de base de données :
`# service mysql restart`

Sécuriser le service de base de données en exécutant:
`# mysql_secure_installation`

**f) File de messages**
Effectuer ces opérations sur le nœud contrôleur.

Installer le package :
`# apt-get install rabbitmq-server`

Ajouter l’utilisateur `openstack` :
`# rabbitmqctl add_user openstack RABBIT_PASS `
Remplacer `RABBIT_PASS` par un mot de passe approprié.
> Creating user "openstack" ...

Permet la configuration, les accès en lecture et écriture pour l’utilisateur `openstack`.
`# rabbitmqctl set_permissions openstack ".*" ".*" ".*"`

> Setting permissions for user "openstack" in vhost "/" ...

**g)Memcached sur Ubuntu**
Effectuer ces opérations sur le nœud contrôleur.
Installez les paquets :
`# apt-get install memcached python-memcache`
Editez le fichier /etc/memcached.conf et configurez le service pour qu'il utilise l'adresse IP de gestion du nœud du contrôleur. 
-l 192.168.2.x

Redémarrer le service Memcached :
`#service memcached restart`

**h) Etcd sur Ubuntu**
Installer le package etcd :
`# apt-get install etcd`
Editer le fichier `/etc/default/etcd` :

    ETCD_NAME="controller"
    ETCD_DATA_DIR="/var/lib/etcd"
    ETCD_INITIAL_CLUSTER_STATE="new"
    ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
    ETCD_INITIAL_CLUSTER="controller=http://192.168.2.22:2380"
    ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.2.22:2380"
    ETCD_ADVERTISE_CLIENT_URLS="http://192.168.2.22:2379"
    ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
    ETCD_LISTEN_CLIENT_URLS="http://192.168.2.22:2379"
    
Activer et démarrer le service etcd:
`# systemctl enable etcd`
`# systemctl start etcd`

## Didacticiel d'installation de Keystone
installer ce service sur le noeud controller.
Keystone est le service d'identité utilisé par OpenStack pour l'authentification  et l'autorisation de haut niveau .
## Prérequis

 1. Avant d'installer et de configurer le service Identity, vous devez
    créer une base de données.

`# mysql`

 2. Créez la base de données `keystone` :

MariaDB [(none)]> CREATE DATABASE keystone;

 3. Accordez un accès approprié à la base de données `keystone` :

       MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO    'keystone'@'localhost' \ IDENTIFIED BY 'KEYSTONE_DBPASS'; MariaDB    [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \    IDENTIFIED BY 'KEYSTONE_DBPASS';

Remplacez `KEYSTONE_DBPASS` par un mot de passe approprié.

 4. Quittez le client d'accès à la base de données.
 
 ## Installer et configurer les composants 
 **NB:** le serveur HTTP Apache agit avec `mod_wsgi` pour répondre aux demandes du service Identity sur le port 5000.
 1. Exécutez la commande suivante pour installer les packages:
`# apt-get install keystone  apache2 libapache2-mod-wsgi`

 2. Editez le fichier `/etc/keystone/keystone.conf` et effectuez les
    actions suivantes:
   * Dans la section `[database]` , configurez l'accès à la base de données:

>  [database]
>     connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@controller/keystone

Remplacez `KEYSTONE_DBPASS` par le mot de passe que vous avez choisi pour la base de données.

**NB:** Mettez en commentaire ou supprimez toute autre option de `connection` dans la section `[database]` .

* Dans la section `[token]` , configurez le fournisseur de jetons Fernet:

>  [token] 
>  provider = fernet

3. Remplir la base de données du service d'identité:
 `# su -s /bin/sh -c "keystone-manage db_sync" keystone`
 
4. Initialiser les référentiels de clés Fernet:
 
`# keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone`
`# keystone-manage credential_setup --keystone-user keystone --keystone-group keystone`

 5. Bootstrap le service d'identité:
`# keystone-manage bootstrap --bootstrap-password ADMIN_PASS \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne`
  
  Remplacez `ADMIN_PASS` par un mot de passe approprié pour un utilisateur administratif.
  ## Configurer le serveur HTTP Apache
  Editez le fichier `/etc/apache2/apache2.conf` et configurez l'option `ServerName` pour référencer le nœud du contrôleur: 
  

>  ServerName controller

6. Redémarrez le service Apache:
`# service apache2 restart`

7. Configurer le compte administratif
`$ export OS_USERNAME = admin `
`$ export OS_PASSWORD = ADMIN_PASS`
`$ export OS_PROJECT_NAME = admin`
`$ export OS_USER_DOMAIN_NAME = Default`
`$ export OS_PROJECT_DOMAIN_NAME = Default`
`$ export OS_AUTH_URL = http://controller:5000/v3`
`$ export OS_IDENTITY_API_VERSION = 3`

Remplacez `ADMIN_PASS` par le mot de passe utilisé dans la commande `keystone-manage bootstrap`
## Créer un domaine, des projets, des utilisateurs et des rôles
Le service d'identité fournit des services d'authentification pour chaque service OpenStack. Le service d'authentification utilise une combinaison de domaines, projets, utilisateurs et rôles.

1. Ce guide utilise un projet de service contenant un utilisateur unique pour chaque service ajouté à votre environnement. Créez le projet de `service` :

`$ openstack project create --domain default \`
 ` --description "Service Project" service`

`+-------------+----------------------------------+`
`| Field       | Value                            |`
`+-------------+----------------------------------+`
`| description | Service Project                  |`
`| domain_id   | default                          |`
`| enabled     | True                             |`
`| id          | 24ac7f19cd944f4cba1d77469b2a73ed |`
`| is_domain   | False                            |`
`| name        | service                          |`
`| parent_id   | default                          |`
`| tags        | []                               |`
`+-------------+----------------------------------+`

2. Les tâches régulières (non administratives) doivent utiliser un projet et un utilisateur non privilégiés. À titre d'exemple, ce guide crée le projet `myuser` et l'utilisateur `myuser` .
* Créez le projet `myproject` :
`$ openstack project create --domain default \`
 ` --description "Demo Project" myproject`

`+-------------+----------------------------------+`
`| Field       | Value                            |`
`+-------------+----------------------------------+`
`| description | Demo Project                     |`
`| domain_id   | default                          |`
`| enabled     | True                             |`
`| id          | 231ad6e7ebba47d6a1e57e1cc07ae446 |`
`| is_domain   | False                            |`
`| name        | myproject                        |`
`| parent_id   | default                          |`
`| tags        | []                               |`
`+-------------+----------------------------------+`
* Créez l'utilisateur `myuser` :
 `$ openstack user create --domain default \`
 ` --password-prompt myuser`

`User Password:`
`Repeat User Password:`
`+---------------------+----------------------------------+`
`| Field               | Value                            |`
`+---------------------+----------------------------------+`
`| domain_id           | default                          |`
`| enabled             | True                             |`
`| id                  | aeda23aa78f44e859900e22c24817832 |`
`| name                | myuser                           |`
`| options             | {}                               |`
`| password_expires_at | None                             |`
`+---------------------+----------------------------------+`
* Créez le rôle `myrole` :
` $ openstack role create myrole`

`+-----------+----------------------------------+`
`| Field     | Value                            |`
`+-----------+----------------------------------+`
`| domain_id | None                             |`
`| id        | 997ce8d05fc143ac97d83fdfb5998552 |`
`| name      | myrole                           |`
`+-----------+----------------------------------+`

* Ajoutez le rôle `myrole` au projet `myuser` et à l'utilisateur `myuser` :
`$ openstack role add --project myproject --user myuser myrole`

## Vérifier le fonctionnement
Vérifiez le fonctionnement du service d'identité avant d'installer d'autres services.
1. Désactivez les variables d'environnement temporaires OS_AUTH_URL et OS_PASSWORD:
`$ unset OS_AUTH_URL OS_PASSWORD`
2. En tant qu'utilisateur `admin` , demandez un jeton d'authentification:
`$ openstack --os-auth-url http://controller:5000/v3 \`
 ` --os-project-domain-name Default --os-user-domain-name Default \`
 ` --os-project-name admin --os-username admin token issue`

`Password:`
`+------------+-----------------------------------------------------------------+`
`| Field      | Value                                                           |`
`+------------+-----------------------------------------------------------------+`
`| expires    | 2016-02-12T20:14:07.056119Z                                     |`
`| id         | gAAAAABWvi7_B8kKQD9wdXac8MoZiQldmjEO643d-e_j-XXq9AmIegIbA7UHGPv |`
`|            | atnN21qtOMjCFWX7BReJEQnVOAj3nclRQgAYRsfSU_MrsuWb4EDtnjU7HEpoBb4 |`
`|            | o6ozsA_NmFWEpLeKy0uNn_WeKbAhYygrsmQGA49dclHVnz-OMVLiyM9ws       |`
`| project_id | 343d245e850143a096806dfaefa9afdc                                |`
`| user_id    | ac3377633149401296f6c0d92d79dc16                                |`
`+------------+-----------------------------------------------------------------+`

3. En `myuser` qu'utilisateur `myuser` créé précédemment, demandez un jeton d'authentification:
`$ openstack --os-auth-url http://controller:5000/v3 \`
`--os-project-domain-name Default --os-user-domain-name Default \`
`--os-project-name myproject --os-username myuser token issue`

`Password:`
`+------------+-----------------------------------------------------------------+`
`| Field      | Value                                                           |`
`+------------+-----------------------------------------------------------------+`
`| expires    | 2016-02-12T20:15:39.014479Z                                     |`
`| id         | gAAAAABWvi9bsh7vkiby5BpCCnc-JkbGhm9wH3fabS_cY7uabOubesi-Me6IGWW |`
`|            | yQqNegDDZ5jw7grI26vvgy1J5nCVwZ_zFRqPiz_qhbq29mgbQLglbkq6FQvzBRQ |`
`|            | JcOzq3uwhzNxszJWmzGC7rJE_H0A_a3UFhqv8M4zMRYSbS2YF0MyFmp_U       |`
`| project_id | ed0b60bf607743088218b0a533d5943f                                |`
`| user_id    | 58126687cbcc4888bfa9ab73a2256f27                                |`
`+------------+-----------------------------------------------------------------+`
## Créer des scripts d'environnement client OpenStack
Créez des scripts d’environnement client pour les projets et les utilisateurs `admin` et de `demo` .
1. Créez et éditez le fichier `admin-openrc` et ajoutez le contenu suivant:
`# nano admin-openrc`

> export OS_PROJECT_DOMAIN_NAME=Default 
> export OS_USER_DOMAIN_NAME=Default 
> export OS_PROJECT_NAME=admin 
> export OS_USERNAME=admin 
> export OS_PASSWORD=ADMIN_PASS 
> export OS_AUTH_URL=http://controller:5000/v3 
> export OS_IDENTITY_API_VERSION=3
> export OS_IMAGE_API_VERSION=2

 Remplacez `ADMIN_PASS` par le mot de passe choisi pour l'utilisateur `admin` dans le service d'identité.
    
2. Créez et éditez le fichier `demo-openrc` et ajoutez le contenu suivant:
`# nano demo-openrc`

> export OS_PROJECT_DOMAIN_NAME=Default 
> export OS_USER_DOMAIN_NAME=Default 
> export OS_PROJECT_NAME=myproject 
> export OS_USERNAME=myuser 
> export OS_PASSWORD=MYUSER_PASS 
> export OS_AUTH_URL=http://controller:5000/v3 
> export OS_IDENTITY_API_VERSION=3
> export OS_IMAGE_API_VERSION=2

Remplacez `DEMO_PASS` par le mot de passe que vous avez choisi pour l'utilisateur de `demo` dans le service d'identité.
## Utiliser les scripts
`$ . admin-openrc`
1. Demander un jeton d'authentification:
`$ openstack token issue`

`+------------+-----------------------------------------------------------------+`
`| Field      | Value                                                           |`
`+------------+-----------------------------------------------------------------+`
`| expires    | 2016-02-12T20:44:35.659723Z                                     |`
`| id         | gAAAAABWvjYj-Zjfg8WXFaQnUd1DMYTBVrKw4h3fIagi5NoEmh21U72SrRv2trl |`
`|            | JWFYhLi2_uPR31Igf6A8mH2Rw9kv_bxNo1jbLNPLGzW_u5FC7InFqx0yYtTwa1e |`
`|            | eq2b0f6-18KZyQhs7F3teAta143kJEWuNEYET-y7u29y0be1_64KYkM7E       |`
`| project_id | 343d245e850143a096806dfaefa9afdc                                |`
`| user_id    | ac3377633149401296f6c0d92d79dc16                                |`
`+------------+-----------------------------------------------------------------+`

## Installer et configurer service d'image 
## Prérequis
Avant d'installer et de configurer le service Image, vous devez créer une base de données, des informations d'identification de service.
1. Pour créer la base de données, procédez comme suit:
`# mysql`
* Créez la base de données de `glance` :
`MariaDB [(none)]> CREATE DATABASE glance;`
* Accordez un accès approprié à la base de données de `glance` :
 `MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \`
` IDENTIFIED BY 'GLANCE_DBPASS';`
`MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \`
`IDENTIFIED BY 'GLANCE_DBPASS';`

Remplacez `GLANCE_DBPASS` par un mot de passe approprié.
* Quittez le client d'accès à la base de données.
2. Indiquez les informations d'identification de l' `admin` pour accéder aux commandes CLI destinées uniquement à l'administrateur:
 `$ . admin-openrc`
 `$ openstack user create --domain default --password-prompt glance`

`User Password:`
`Repeat User Password:`
`+---------------------+----------------------------------+`
`| Field               | Value                            |`
`+---------------------+----------------------------------+`
`| domain_id           | default                          |`
`| enabled             | True                             |`
`| id                  | 3f4e777c4062483ab8d9edd7dff829df |`
`| name                | glance                           |`
`| options             | {}                               |`
`| password_expires_at | None                             |`
`+---------------------+----------------------------------+`

* Ajoutez le rôle d' `admin` au projet d'utilisateur et de projet `service`:
` $ openstack role add --project service --user glance admin`

* Créez l'entité de service de `glance` :
`$ openstack service create --name glance \`
 ` --description "OpenStack Image" image`

`+-------------+----------------------------------+`
`| Field       | Value                            |`
`+-------------+----------------------------------+`
`| description | OpenStack Image                  |`
`| enabled     | True                             |`
`| id          | 8c2c7f1b9b5049ea9e63757b5533e6d2 |`
`| name        | glance                           |`
`| type        | image                            |`
`+-------------+----------------------------------+`

3. Créez les points de terminaison de l'API du service image:
 `$ openstack endpoint create --region RegionOne \`
 ` image public http://controller:9292`

`+--------------+----------------------------------+`
`| Field        | Value                            |`
`+--------------+----------------------------------+`
`| enabled      | True                             |`
`| id           | 340be3625e9b4239a6415d034e98aace |`
`| interface    | public                           |`
`| region       | RegionOne                        |`
`| region_id    | RegionOne                        |`
`| service_id   | 8c2c7f1b9b5049ea9e63757b5533e6d2 |`
`| service_name | glance                           |`
`| service_type | image                            |`
`| url          | http://controller:9292           |`
`+--------------+----------------------------------+`

`$ openstack endpoint create --region RegionOne \`
 ` image internal http://controller:9292`

`+--------------+----------------------------------+`
`| Field        | Value                            |`
`+--------------+----------------------------------+`
`| enabled      | True                             |`
`| id           | a6e4b153c2ae4c919eccfdbb7dceb5d2 |`
`| interface    | internal                         |`
`| region       | RegionOne                        |`
`| region_id    | RegionOne                        |`
`| service_id   | 8c2c7f1b9b5049ea9e63757b5533e6d2 |`
`| service_name | glance                           |`
`| service_type | image                            |`
`| url          | http://controller:9292           |`
`+--------------+----------------------------------+`

`$ openstack endpoint create --region RegionOne \`
 ` image admin http://controller:9292`

`+--------------+----------------------------------+`
`| Field        | Value                            |`
`+--------------+----------------------------------+`
`| enabled      | True                             |`
`| id           | 0c37ed58103f4300a84ff125a539032d |`
`| interface    | admin                            |`
`| region       | RegionOne                        |`
`| region_id    | RegionOne                        |`
`| service_id   | 8c2c7f1b9b5049ea9e63757b5533e6d2 |`
`| service_name | glance                           |`
`| service_type | image                            |`
`| url          | http://controller:9292           |`
`+--------------+----------------------------------+`

## Installer et configurer les composants 
1. Installez les paquets:
`# apt-get install glance`
2. Editez le fichier `/etc/glance/glance-api.conf` et `/etc/glance/glance-api.conf` comme suit:

* Dans la section `[database]` , configurez l'accès à la base de données:

>  [database] connection =
> mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

Remplacez `GLANCE_DBPASS` par le mot de passe que vous avez choisi pour la base de données du service d’image.

* Dans les sections `[keystone_authtoken]` et `[paste_deploy]` , configurez l'accès au service d'identité:

>  [keystone_authtoken] 
>  www_authenticate_uri = http://controller:5000
> auth_url = http://controller:5000 
> memcached_servers = controller:11211
> auth_type = password 
> project_domain_name = Default 
> user_domain_name = Default 
> project_name = service 
> username = glance 
> password = GLANCE_PASS

> [paste_deploy] 
> flavor = keystone

Remplacez `GLANCE_PASS` par le mot de passe que vous avez choisi pour l'utilisateur `glance` d' `glance` dans le service Identité.

**NB:** Mettez en commentaire ou supprimez toute autre option de la section `[keystone_authtoken]` .
* Dans la section `[glance_store]` , configurez le magasin du système de fichiers local et l'emplacement des fichiers image:

>  [glance_store] stores = file,http 
>  default_store = file
> filesystem_store_datadir = /var/lib/glance/images/

3. Editez le fichier `/etc/glance/glance-registry.conf` et effectuez les actions suivantes:
* Dans la section `[database]` , configurez l'accès à la base de données:

> [database] 
>connection = mysql+pymysql://glance:GLANCE_DBPASS@controller/glance

Remplacez `GLANCE_DBPASS` par le mot de passe que vous avez choisi pour la base de données du service d’image.
    
-   Dans les sections `[keystone_authtoken]` et `[paste_deploy]` , configurez l'accès au service d'identité:

> [keystone_authtoken] 
> www_authenticate_uri = http://controller:5000
> auth_url = http://controller:5000 
> memcached_servers = controller:11211
> auth_type = password
>  project_domain_name = Default 
>  user_domain_name = Default 
>  project_name = service 
>  username = glance 
>  password = GLANCE_PASS

> [paste_deploy] 
> flavor = keystone

Remplacez `GLANCE_PASS` par le mot de passe que vous avez choisi pour l'utilisateur `glance` d' `glance` dans le service Identité.

**NB:** Mettez en commentaire ou supprimez toute autre option de la section `[keystone_authtoken]` .

4. Remplir la base de données du service d’image:
`# su -s /bin/sh -c "glance-manage db_sync" glance`
5. Redémarrez les services d'image:
`# service glance-registry restart`
`# service glance-api restart`
## Vérifier le fonctionnement
Vérifiez le fonctionnement du service Image à l'aide de CirrOS , une petite image Linux qui vous permet de tester votre déploiement OpenStack.
1. ndiquez les informations d'identification de l' `admin` pour accéder aux commandes CLI destinées uniquement à l'administrateur:
    ` $ . admin-openrc`
    
2. Téléchargez l'image source:
      ` $ wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img`
3. Téléchargez l'image sur le service Image à l'aide du format de disque QCOW2 , du format de conteneur nu et de la visibilité publique afin que tous les projets puissent y accéder:
`$ openstack image create "cirros" \`
`--file cirros-0.4.0-x86_64-disk.img \`
`--disk-format qcow2 --container-format bare \`
`--public`

`+------------------+------------------------------------------------------+`
`| Field            | Value                                                |`
`+------------------+------------------------------------------------------+`
`| checksum         | 133eae9fb1c98f45894a4e60d8736619                     |`
`| container_format | bare                                                 |`
`| created_at       | 2015-03-26T16:52:10Z                                 |`
`| disk_format      | qcow2                                                |`
`| file             | /v2/images/cc5c6982-4910-471e-b864-1098015901b5/file |`
`| id               | cc5c6982-4910-471e-b864-1098015901b5                 |`
`| min_disk         | 0                                                    |`
`| min_ram          | 0                                                    |`
`| name             | cirros                                               |`
`| owner            | ae7a98326b9c455588edd2656d723b9d                     |`
`| protected        | False                                                |`
`| schema           | /v2/schemas/image                                    |`
`| size             | 13200896                                             |`
`| status           | active                                               |`
`| tags             |                                                      |`
`| updated_at       | 2015-03-26T16:52:10Z                                 |`
`| virtual_size     | None                                                 |`
`| visibility       | public                                               |`
`+------------------+------------------------------------------------------+`

4. Confirmez le téléchargement de l'image et validez les attributs:
` $ openstack image list`

`+--------------------------------------+--------+--------+`
`| ID                                   | Name   | Status |`
`+--------------------------------------+--------+--------+`
`| 38047887-61a7-41ea-9b49-27987d5e8bb9 | cirros | active |`
`+--------------------------------------+--------+--------+`

## Installer et configurer le service Compute sur le nœud du contrôleur 

## Prérequis 
Avant d'installer et de configurer le service Compute, vous devez créer des bases de données, des informations d'identification de service.
1. Pour créer les bases de données, procédez comme suit:
* accèder à la base de données:
`# mysql`
* Créez les `nova_api` , `nova` , `nova_cell0` et `placement` :
`MariaDB [(none)]> CREATE DATABASE nova_api;`
`MariaDB [(none)]> CREATE DATABASE nova;`
`MariaDB [(none)]> CREATE DATABASE nova_cell0;`
`MariaDB [(none)]> CREATE DATABASE placement;`
* Accordez un accès approprié aux bases de données:
`MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \`
 `IDENTIFIED BY 'NOVA_DBPASS';`
`MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \`
 `IDENTIFIED BY 'NOVA_DBPASS';`

`MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \`
 `IDENTIFIED BY 'NOVA_DBPASS';`
`MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \`
 `IDENTIFIED BY 'NOVA_DBPASS';`

`MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \`
 `IDENTIFIED BY 'NOVA_DBPASS';`
`MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \`
 `IDENTIFIED BY 'NOVA_DBPASS';`

`MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' \`
 `IDENTIFIED BY 'PLACEMENT_DBPASS';`
`MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' \`
 `IDENTIFIED BY 'PLACEMENT_DBPASS';`
 
Remplacez `NOVA_DBPASS` par un mot de passe approprié.

   * Quittez le client d'accès à la base de données.
  2. Indiquez les informations d'identification de l' `admin` pour accéder aux commandes CLI destinées uniquement à l'administrateur:

        `$ . admin-openrc`
  3. Créez les informations d'identification du service de traitement:
-   Créez l'utilisateur `nova` :
`$ openstack user create --domain default --password-prompt nova`

`User Password:`
`Repeat User Password:`
`+---------------------+----------------------------------+`
`| Field               | Value                            |`
`+---------------------+----------------------------------+`
`| domain_id           | default                          |`
`| enabled             | True                             |`
`| id                  | 8a7dbf5279404537b1c7b86c033620fe |`
`| name                | nova                             |`
`| options             | {}                               |`
`| password_expires_at | None                             |`
`+---------------------+----------------------------------+`

* Ajoutez le rôle `admin` à l'utilisateur `nova` :
` $ openstack role add --project service --user nova admin`
* Créez l'entité de service `nova` :
`$ openstack service create --name nova \`
 `--description "OpenStack Compute" compute`

`+-------------+----------------------------------+`
`| Field       | Value                            |`
`+-------------+----------------------------------+`
`| description | OpenStack Compute                |`
`| enabled     | True                             |`
`| id          | 060d59eac51b4594815603d75a00aba2 |`
`| name        | nova                             |`
`| type        | compute                          |`
`+-------------+----------------------------------+`

4. Créez les points de terminaison du service API Compute:
`$ openstack endpoint create --region RegionOne \`
  `compute public http://controller:8774/v2.1`

`+--------------+-------------------------------------------+`
`| Field        | Value                                     |`
`+--------------+-------------------------------------------+`
`| enabled      | True                                      |`
`| id           | 3c1caa473bfe4390a11e7177894bcc7b          |`
`| interface    | public                                    |`
`| region       | RegionOne                                 |`
`| region_id    | RegionOne                                 |`
`| service_id   | 060d59eac51b4594815603d75a00aba2          |`
`| service_name | nova                                      |`
`| service_type | compute                                   |`
`| url          | http://controller:8774/v2.1               |`
`+--------------+-------------------------------------------+`

`$ openstack endpoint create --region RegionOne \`
  `compute internal http://controller:8774/v2.1`

`+--------------+-------------------------------------------+`
`| Field        | Value                                     |`
`+--------------+-------------------------------------------+`
`| enabled      | True                                      |`
`| id           | e3c918de680746a586eac1f2d9bc10ab          |`
`| interface    | internal                                  |`
`| region       | RegionOne                                 |`
`| region_id    | RegionOne                                 |`
`| service_id   | 060d59eac51b4594815603d75a00aba2          |`
`| service_name | nova                                      |`
`| service_type | compute                                   |`
`| url          | http://controller:8774/v2.1               |`
`+--------------+-------------------------------------------+`

`$ openstack endpoint create --region RegionOne \`
  `compute admin http://controller:8774/v2.1`

`+--------------+-------------------------------------------+`
`| Field        | Value                                     |`
`+--------------+-------------------------------------------+`
`| enabled      | True                                      |`
`| id           | 38f7af91666a47cfb97b4dc790b94424          |`
`| interface    | admin                                     |`
`| region       | RegionOne                                 |`
`| region_id    | RegionOne                                 |`
`| service_id   | 060d59eac51b4594815603d75a00aba2          |`
`| service_name | nova                                      |`
`| service_type | compute                                   |`
`| url          | http://controller:8774/v2.1               |`
`+--------------+-------------------------------------------+`

5. Créez un utilisateur de service de placement à l'aide de votre `PLACEMENT_PASS` choisi:
`$ openstack user create --domain default --password-prompt placement`

`User Password:`
`Repeat User Password:`
`+---------------------+----------------------------------+`
`| Field               | Value                            |`
`+---------------------+----------------------------------+`
`| domain_id           | default                          |`
`| enabled             | True                             |`
`| id                  | fa742015a6494a949f67629884fc7ec8 |`
`| name                | placement                        |`
`| options             | {}                               |`
`| password_expires_at | None                             |`
`+---------------------+----------------------------------+`

6. Ajoutez l'utilisateur de placement au projet de service avec le rôle admin:
`$ openstack role add --project service --user placement admin`
7. Créez l'entrée API de placement dans le catalogue de services:
`$ openstack service create --name placement \`
 `--description "Placement API" placement`

`+-------------+----------------------------------+`
`| Field       | Value                            |`
`+-------------+----------------------------------+`
`| description | Placement API                    |`
`| enabled     | True                             |`
`| id          | 2d1a27022e6e4185b86adac4444c495f |`
`| name        | placement                        |`
`| type        | placement                        |`
`+-------------+----------------------------------+`

8. Créez les points de terminaison du service API de placement:
`$ openstack endpoint create --region RegionOne \`
  `placement public http://controller:8778`

`+--------------+----------------------------------+`
`| Field        | Value                            |`
`+--------------+----------------------------------+`
`| enabled      | True                             |`
`| id           | 2b1b2637908b4137a9c2e0470487cbc0 |`
`| interface    | public                           |`
`| region       | RegionOne                        |`
`| region_id    | RegionOne                        |`
`| service_id   | 2d1a27022e6e4185b86adac4444c495f |`
`| service_name | placement                        |`
`| service_type | placement                        |`
`| url          | http://controller:8778           |`
`+--------------+----------------------------------+`

`$ openstack endpoint create --region RegionOne \`
`  placement internal http://controller:8778`

`+--------------+----------------------------------+`
`| Field        | Value                            |`
`+--------------+----------------------------------+`
`| enabled      | True                             |`
`| id           | 02bcda9a150a4bd7993ff4879df971ab |`
`| interface    | internal                         |`
`| region       | RegionOne                        |`
`| region_id    | RegionOne                        |`
`| service_id   | 2d1a27022e6e4185b86adac4444c495f |`
`| service_name | placement                        |`
`| service_type | placement                        |`
`| url          | http://controller:8778           |`
`+--------------+----------------------------------+`

`$ openstack endpoint create --region RegionOne \`
 ` placement admin http://controller:8778`

`+--------------+----------------------------------+`
`| Field        | Value                            |`
`+--------------+----------------------------------+`
`| enabled      | True                             |`
`| id           | 3d71177b9e0f406f98cbff198d74b182 |`
`| interface    | admin                            |`
`| region       | RegionOne                        |`
`| region_id    | RegionOne                        |`
`| service_id   | 2d1a27022e6e4185b86adac4444c495f |`
`| service_name | placement                        |`
`| service_type | placement                        |`
`| url          | http://controller:8778           |`
`+--------------+----------------------------------+`

## Installer et configurer les composants 
1. Installez les paquets:
` # apt-get install nova-api nova-conductor nova-consoleauth \`
  `nova-novncproxy nova-scheduler nova-placement-api`
2. Editez le fichier `/etc/nova/nova.conf` et `/etc/nova/nova.conf` comme suit:
-   Dans les `[api_database]` , `[database]` et `[placement_database]` , configurez l'accès à la base de données:

> [api_database] 
> connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova_api

> [database] 
> connection = mysql+pymysql://nova:NOVA_DBPASS@controller/nova
 
> [placement_database] 
> connection = mysql+pymysql://placement:PLACEMENT_DBPASS@controller/placement

Remplacez `NOVA_DBPASS` par le mot de passe que vous avez choisi pour les bases de données Compute et la base de données `PLACEMENT_DBPASS` for Placement.
    
-   Dans la section `[DEFAULT]` , configurez l'accès à la file d'attente de messages `RabbitMQ` :

> [DEFAULT] 
> transport_url = rabbit://openstack:RABBIT_PASS@controller

Remplacez `RABBIT_PASS` par le mot de passe que vous avez choisi pour le compte `openstack` dans `RabbitMQ` .
    
-   Dans les sections `[api]` et `[keystone_authtoken]` , configurez l'accès au service d'identité:

>  [api] 
>  auth_strategy = keystone

> [keystone_authtoken] 
> auth_url = http://controller:5000/v3
> memcached_servers = controller:11211 
> auth_type = password
> project_domain_name = Default 
> user_domain_name = Default 
> project_name= service 
> username = nova 
> password = NOVA_PASS

Remplacez `NOVA_PASS` par le mot de passe que vous avez choisi pour l'utilisateur `nova` dans le service d'identité.

**NB:** Mettez en commentaire ou supprimez toute autre option de la section `[keystone_authtoken]` .

Dans la section `[DEFAULT]` , configurez l'option `my_ip` pour utiliser l'adresse IP de l'interface de gestion du nœud du contrôleur:

>  [DEFAULT] 
>  my_ip = 192.168.2.22

Dans la section `[DEFAULT]` , activez la prise en charge du service de mise en réseau:

>  [DEFAULT] 
>  use_neutron = true 
>  firewall_driver = nova.virt.firewall.NoopFirewallDriver

* Dans la section `[vnc]` , configurez le proxy VNC pour utiliser l'adresse IP de l'interface de gestion du nœud du contrôleur:

>  [vnc] 
>  enabled = true 
>  server_listen = $my_ip
> server_proxyclient_address = $my_ip

* Dans la section `[glance]` , configurez l'emplacement de l'API du service d'image:

> [glance] 
> api_servers = http://controller:9292

-   Dans la section `[oslo_concurrency]` , configurez le chemin de verrouillage:

> [oslo_concurrency]
> lock_path = /var/lib/nova/tmp
-   En raison d'un bogue d'emballage, supprimez l'option `log_dir` de la section `[DEFAULT]` .
    
-   Dans la section `[placement]` , configurez l'API de placement:
> [placement] 
> region_name = RegionOne 
> project_domain_name = Default
> project_name = service 
> auth_type = password 
> user_domain_name = Default
> auth_url = http://controller:5000/v3 
> username = placement password = PLACEMENT_PASS

-   Remplacez `PLACEMENT_PASS` par le mot de passe choisi pour l'utilisateur de l' `placement` dans le service d'identité.
-  Mettez en commentaire toutes les autres options de la section `[placement]` .
        
3. Remplissez les bases de données `nova-api` et `placement` :
`# su -s /bin/sh -c "nova-manage api_db sync" nova`
4. Enregistrez la base de données `cell0` :
`# su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova`
5. Créez la cellule 1:
`# su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova`
6. Remplir la base de données nova:
`# su -s /bin/sh -c "nova-manage db sync" nova`
7. Vérifiez que nova cell0 et cell1 sont correctement enregistrés:
`# su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova`

`+-------+--------------------------------------+`
`| Name  | UUID                                 |`
`+-------+--------------------------------------+`
`| cell1 | 109e1d4b-536a-40d0-83c6-5f121b82b650 |`
`| cell0 | 00000000-0000-0000-0000-000000000000 |`
`+-------+--------------------------------------+`

8. Redémarrez les services de calcul:

`# service nova-api restart`
`# service nova-consoleauth restart`
`# service nova-scheduler restart`
`# service nova-conductor restart`
`# service nova-novncproxy restart`

## Installer et configurer le service compute sur le nœud de compute
Le service prend en charge plusieurs hyperviseurs pour déployer des instances ou des machines virtuelles (VM). Pour des raisons de simplicité, cette configuration utilise l'hyperviseur Quick EMUlator (QEMU) avec l'extension KVM basée sur le noyau sur des nœuds de calcul prenant en charge l'accélération matérielle des machines virtuelles.

1. Installez les paquets:
`# apt install nova-compute`
2. Editez le fichier `/etc/nova/nova.conf` et `/etc/nova/nova.conf` comme suit:

-   Dans la section `[DEFAULT]` , configurez l'accès à la file d'attente de messages `RabbitMQ` :

>  [DEFAULT] 
>  transport_url = rabbit://openstack:RABBIT_PASS@controller

Remplacez `RABBIT_PASS` par le mot de passe que vous avez choisi pour le compte `openstack` dans `RabbitMQ` .
* Dans les sections `[api]` et `[keystone_authtoken]` , configurez l'accès au service d'identité:

>  [api] 
>  auth_strategy = keystone

> [keystone_authtoken] 
> auth_url = http://controller:5000/v3
> memcached_servers = controller:11211 
> auth_type = password
> project_domain_name = Default 
> user_domain_name = Default 
> project_name = service 
> username = nova 
> password = NOVA_PASS

Remplacez `NOVA_PASS` par le mot de passe que vous avez choisi pour l'utilisateur `nova` dans le service d'identité.
**NB:** Mettez en commentaire ou supprimez toute autre option de la section `[keystone_authtoken]` .
* Dans la section `[DEFAULT]` , configurez l'option `my_ip` :

> [DEFAULT] 
> my_ip = MANAGEMENT_INTERFACE_IP_ADDRESS

Remplacez `MANAGEMENT_INTERFACE_IP_ADDRESS` par l'adresse IP de l'interface réseau 192.168.2.x 
* Dans la section `[DEFAULT]` , activez la prise en charge du service de mise en réseau:
 

> [DEFAULT] 
> use_neutron = true 
> firewall_driver = nova.virt.firewall.NoopFirewallDriver

* Dans la section `[vnc]` , activez et configurez l'accès à la console distante:

>  [vnc] 
>  enabled = true 
>  server_listen = 0.0.0.0
> server_proxyclient_address = $my_ip 
> novncproxy_base_url = http://controller:6080/vnc_auto.html

* Dans la section `[glance]` , configurez l'emplacement de l'API du service d'image:

> [glance] 
> api_servers = http://controller:9292

* Dans la section `[oslo_concurrency]` , configurez le chemin de verrouillage:

>  [oslo_concurrency] 
>  lock_path = /var/lib/nova/tmp

-   En raison d'un bogue d'emballage, supprimez l'option `log_dir` de la section `[DEFAULT]` .
    
-   Dans la section `[placement]` , configurez l'API de placement:

>  [placement] 
>  region_name = RegionOne 
>  project_domain_name = Default
> project_name = service 
> auth_type = password 
> user_domain_name = Default
> auth_url = http://controller:5000/v3 
> username = placement 
> password = PLACEMENT_PASS

Remplacez `PLACEMENT_PASS` par le mot de passe choisi pour l'utilisateur de l' `placement` dans le service d'identité.
Mettez en commentaire toutes les autres options de la section `[placement]` .

3. Déterminez si votre noeud de traitement prend en charge l'accélération matérielle pour les machines virtuelles:
`$ egrep -c '(vmx|svm)' /proc/cpuinfo`

Si cette commande renvoie une valeur `un ou plus` , votre nœud de calcul prend en charge l'accélération matérielle qui, en règle générale, ne nécessite aucune configuration supplémentaire.

Si cette commande renvoie la valeur `zero` , votre noeud de traitement ne prend pas en charge l'accélération matérielle et vous devez configurer `libvirt` pour qu'il utilise QEMU au lieu de KVM.

-   Modifiez la section `[libvirt]` du fichier `/etc/nova/nova-compute.conf` comme suit:   

> [libvirt] 
> virt_type = qemu

4. Redémarrez le service Compute:
`# service nova-compute restart`

## Ajoutez le noeud de traitement à la base de données de cellules
* Exécutez les commandes suivantes sur le nœud du **contrôleur** .
1. Indiquez les informations d'identification de l'administrateur pour activer les commandes CLI destinées uniquement à l'administrateur, puis vérifiez que la base de données contient des hôtes de calcul:
 `$ . admin-openrc`
`$ openstack compute service list --service nova-compute`
`+----+-------+--------------+------+-------+---------+----------------------------+`
`| ID | Host  | Binary       | Zone | State | Status  | Updated At                 |`
`+----+-------+--------------+------+-------+---------+----------------------------+`
`| 1  | node1 | nova-compute | nova | up    | enabled | 2017-04-14T15:30:44.000000 |`
`+----+-------+--------------+------+-------+---------+----------------------------+`

2. Découvrez les hôtes de calcul:
 `# su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova`
 
**NB:** Lorsque vous ajoutez de nouveaux nœuds de calcul, vous devez exécuter `nova-manage cell_v2 discover_hosts` sur le nœud du contrôleur pour enregistrer ces nouveaux nœuds de calcul. Vous pouvez également définir un intervalle approprié dans `/etc/nova/nova.conf` :

> [scheduler] 
> discover_hosts_in_cells_interval = 300

## Vérifier le fonctionnement
Vérifiez le fonctionnement du service de calcul.
Effectuez ces commandes sur le nœud du contrôleur.

 `$ . admin-openrc`
 
 Répertoriez les composants de service pour vérifier le lancement et l'enregistrement de chaque processus:
` $ openstack compute service list`

`+----+--------------------+------------+----------+---------+-------+----------------------------+`
`| Id | Binary             | Host       | Zone     | Status  | State | Updated At                 |`
`+----+--------------------+------------+----------+---------+-------+----------------------------+`
`|  1 | nova-consoleauth   | controller | internal | enabled | up    | 2016-02-09T23:11:15.000000 |`
`|  2 | nova-scheduler     | controller | internal | enabled | up    | 2016-02-09T23:11:15.000000 |`
`|  3 | nova-conductor     | controller | internal | enabled | up    | 2016-02-09T23:11:16.000000 |`
`|  4 | nova-compute       | compute   | nova     | enabled | up    | 2016-02-09T23:11:20.000000 |`
`+----+--------------------+------------+----------+---------+-------+----------------------------+`

**NB:** Cette sortie doit indiquer trois composants de service activés sur le nœud du contrôleur et un composant de service activé sur le nœud de calcul.

1. Répertoriez les points de terminaison d'API dans le service d'identité pour vérifier la connectivité avec le service d'identité:
 `$ openstack catalog list`

`+-----------+-----------+-----------------------------------------+`
`| Name      | Type      | Endpoints                               |`
`+-----------+-----------+-----------------------------------------+`
`| keystone  | identity  | RegionOne                               |`
`|           |           |   public: http://controller:5000/v3/    |`
`|           |           | RegionOne                               |`
`|           |           |   internal: http://controller:5000/v3/  |`
`|           |           | RegionOne                               |`
`|           |           |   admin: http://controller:5000/v3/     |`
`|           |           |                                         |`
`| glance    | image     | RegionOne                               |`
`|           |           |   admin: http://controller:9292         |`
`|           |           | RegionOne                               |`
`|           |           |   public: http://controller:9292        |`
`|           |           | RegionOne                               |`
`|           |           |   internal: http://controller:9292      |`
`|           |           |                                         |`
`| nova      | compute   | RegionOne                               |`
`|           |           |   admin: http://controller:8774/v2.1    |`
`|           |           | RegionOne                               |`
`|           |           |   internal: http://controller:8774/v2.1 |`
`|           |           | RegionOne                               |`
`|           |           |   public: http://controller:8774/v2.1   |`
`|           |           |                                         |`
`| placement | placement | RegionOne                               |`
`|           |           |   public: http://controller:8778        |`
`|           |           | RegionOne                               |`
`|           |           |   admin: http://controller:8778         |`
`|           |           | RegionOne                               |`
`|           |           |   internal: http://controller:8778      |`
`|           |           |                                         |`
`+-----------+-----------+-----------------------------------------+`
 
 3. Vérifiez que les cellules et l'API de positionnement fonctionnent correctement:
  `# nova-status upgrade check`

`+---------------------------+`
`| Upgrade Check Results     |`
`+---------------------------+`
`| Check: Cells v2           |`
`| Result: Success           |`
`| Details: None             |`
`+---------------------------+`
`| Check: Placement API      |`
`| Result: Success           |`
`| Details: None             |`
`+---------------------------+`
`| Check: Resource Providers |`
`| Result: Success           |`
`| Details: None             |`
`+---------------------------+`

## Installer et configurer le service neutron sur le nœud du contrôleur
## Prérequis 
Avant de configurer le service OpenStack Networking (neutron), vous devez créer une base de données, des informations d'identification de service et des points de terminaison d'API.

1.  Pour créer la base de données, procédez comme suit:
    
    -   Utilisez le client d'accès à la base de données pour vous connecter au serveur de base de données en tant qu'utilisateur `root` :
    `# mysql `
    Créez la base de données `neutron` :
    `MariaDB [(none)] CREATE DATABASE neutron;`
* Accordez un accès approprié à la base de données sur les `neutron` remplaçant `NEUTRON_DBPASS` par un mot de passe approprié:
 `MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \`
` IDENTIFIED BY 'NEUTRON_DBPASS';`
`MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \`
` IDENTIFIED BY 'NEUTRON_DBPASS';`

-  Quittez le client d'accès à la base de données.
        
2.   Indiquez les informations d'identification de l' `admin` pour accéder aux commandes CLI destinées uniquement à l'administrateur:
`$ . admin-openrc`
3. Pour créer les informations d'identification du service, procédez comme suit:

-   Créez l'utilisateur `neutron` :
`$ openstack user create --domain default --password-prompt neutron`

`User Password:`
`Repeat User Password:`
`+---------------------+----------------------------------+`
`| Field               | Value                            |`
`+---------------------+----------------------------------+`
`| domain_id           | default                          |`
`| enabled             | True                             |`
`| id                  | fdb0f541e28141719b6a43c8944bf1fb |`
`| name                | neutron                          |`
`| options             | {}                               |`
`| password_expires_at | None                             |`
`+---------------------+----------------------------------+`

* Ajoutez le rôle `admin` à l'utilisateur `neutron` :
`$ openstack role add --project service --user neutron admin`
* Créez l'entité de service `neutron` :
 `$ openstack service create --name neutron \`
 ` --description "OpenStack Networking" network`

`+-------------+----------------------------------+`
`| Field       | Value                            |`
`+-------------+----------------------------------+`
`| description | OpenStack Networking             |`
`| enabled     | True                             |`
`| id          | f71529314dab4a4d8eca427e701d209e |`
`| name        | neutron                          |`
`| type        | network                          |`
`+-------------+----------------------------------+`

4. Créez les points de terminaison de l'API de service de mise en réseau:
`$ openstack endpoint create --region RegionOne \`
 ` network public http://controller:9696`

`+--------------+----------------------------------+`
`| Field        | Value                            |`
`+--------------+----------------------------------+`
`| enabled      | True                             |`
`| id           | 85d80a6d02fc4b7683f611d7fc1493a3 |`
`| interface    | public                           |`
`| region       | RegionOne                        |`
`| region_id    | RegionOne                        |`
`| service_id   | f71529314dab4a4d8eca427e701d209e |`
`| service_name | neutron                          |`
`| service_type | network                          |`
`| url          | http://controller:9696           |`
`+--------------+----------------------------------+`

`$ openstack endpoint create --region RegionOne \`
  `network internal http://controller:9696`

`+--------------+----------------------------------+`
`| Field        | Value                            |`
`+--------------+----------------------------------+`
`| enabled      | True                             |`
`| id           | 09753b537ac74422a68d2d791cf3714f |`
`| interface    | internal                         |`
`| region       | RegionOne                        |`
`| region_id    | RegionOne                        |`
`| service_id   | f71529314dab4a4d8eca427e701d209e |`
`| service_name | neutron                          |`
`| service_type | network                          |`
`| url          | http://controller:9696           |`
`+--------------+----------------------------------+`

`$ openstack endpoint create --region RegionOne \`
 ` network admin http://controller:9696`

`+--------------+----------------------------------+`
`| Field        | Value                            |`
`+--------------+----------------------------------+`
`| enabled      | True                             |`
`| id           | 1ee14289c9374dffb5db92a5c112fc4e |`
`| interface    | admin                            |`
`| region       | RegionOne                        |`
`| region_id    | RegionOne                        |`
`| service_id   | f71529314dab4a4d8eca427e701d209e |`
`| service_name | neutron                          |`
`| service_type | network                          |`
`| url          | http://controller:9696           |`
`+--------------+----------------------------------+`

## Configurer les options de mise en réseau

Installez et configurez les composants de mise en réseau sur le nœud du contrôleur
## Installer les composants 

 `# apt-get install neutron-server neutron-plugin-ml2 \`
 `neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent \`
 ` neutron-metadata-agent`
 
 ## Configurez le composant serveur 
-   Editez le fichier `/etc/neutron/neutron.conf` et effectuez les actions suivantes:
    
    -   Dans la section `[database]` , configurez l'accès à la base de données:

>  [database] 
>  connection = mysql+pymysql://neutron:NEUTRON_DBPASS@controller/neutron

Remplacez `NEUTRON_DBPASS` par le mot de passe que vous avez choisi pour la base de données.

**NB:** Mettez en commentaire ou supprimez toute autre option de `connection` dans la section `[database]` .

* Dans la section `[DEFAULT]` , activez le plug-in, le service de routeur et les adresses IP superposées de Modular Layer 2 (ML2):
 

> [DEFAULT]
>  core_plugin = ml2 
>  service_plugins = router
> allow_overlapping_ips = true

Dans la section `[DEFAULT]` , configurez l'accès à la file d'attente de messages `RabbitMQ` :

> [DEFAULT] 
> transport_url = rabbit://openstack:RABBIT_PASS@controller

-   Remplacez `RABBIT_PASS` par le mot de passe que vous avez choisi pour le compte `openstack` dans RabbitMQ.
    
-   Dans les sections `[DEFAULT]` et `[keystone_authtoken]` , configurez l'accès au service d'identité:

> [DEFAULT] 
> auth_strategy = keystone

> [keystone_authtoken] 
> www_authenticate_uri = http://controller:5000
> auth_url = http://controller:5000 
> memcached_servers = controller:11211
> auth_type = password 
> project_domain_name = default 
> user_domain_name = default 
> project_name = service 
> username = neutron 
> password = NEUTRON_PASS

Remplacez `NEUTRON_PASS` par le mot de passe que vous avez choisi pour l'utilisateur de `neutron` dans le service d'identité.
**NB:** Mettez en commentaire ou supprimez toute autre option de la section `[keystone_authtoken]` .

* Dans les sections `[DEFAULT]` et `[nova]` , configurez Networking pour informer Compute des modifications de la topologie du réseau:

> [DEFAULT] 
> notify_nova_on_port_status_changes = true
> notify_nova_on_port_data_changes = true

> [nova] 
> auth_url = http://controller:5000 
> auth_type = password
> project_domain_name = default 
> user_domain_name = default 
> region_name = RegionOne 
> project_name = service 
> username = nova 
> password = NOVA_PASS

-    Remplacez `NOVA_PASS` par le mot de passe que vous avez choisi pour l'utilisateur `nova` dans le service d'identité.
        
-   Dans la section `[oslo_concurrency]` , configurez le chemin de verrouillage:

> [oslo_concurrency] 
> lock_path = /var/lib/neutron/tmp

## Configurez le plug-in Modular Layer 2 (ML2) 

Le plug-in ML2 utilise le mécanisme de pont Linux pour créer une infrastructure de réseau virtuel de couche 2 (pontage et commutation) pour les instances.

-   Editez le fichier `/etc/neutron/plugins/ml2/ml2_conf.ini` et effectuez les actions suivantes:
    
    -   Dans la section `[ml2]` , activez les réseaux plats, VLAN et VXLAN:
  
>    [ml2] 
>    type_drivers = flat,vlan,vxlan

Dans la section `[ml2]` , activez les réseaux libre-service VXLAN:

>  [ml2] 
>  tenant_network_types = vxlan

Dans la section `[ml2]` , activez les mécanismes de pontage et de population de couche 2 de Linux:

>  [ml2] 
>  mechanism_drivers = linuxbridge,l2population

Dans la section `[ml2]` , activez le pilote de l'extension de sécurité du port:

>  [ml2] 
>  extension_drivers = port_security

Dans la section `[ml2_type_flat]` , configurez le réseau virtuel du fournisseur en tant que réseau plat:

>  [ml2_type_flat] 
>  flat_networks = provider

Dans la section `[ml2_type_vxlan]` , configurez la plage d'identificateurs de réseau VXLAN pour les réseaux en libre service:

>  [ml2_type_vxlan] 
>  vni_ranges = 1:1000

Dans la section `[securitygroup]` , activez ipset pour améliorer l'efficacité des règles du groupe de sécurité:

>  [securitygroup] 
>  enable_ipset = true

## Configurez l'agent de pont Linux 

L'agent pont Linux crée une infrastructure de réseau virtuel pour les instances de couche 2 (pontage et commutation) et gère les groupes de sécurité.

-   Editez le fichier `/etc/neutron/plugins/ml2/linuxbridge_agent.ini` et `/etc/neutron/plugins/ml2/linuxbridge_agent.ini` comme suit:
    
    -   Dans la section `[linux_bridge]` , `[linux_bridge]` le réseau virtuel du fournisseur sur l'interface réseau physique du fournisseur:

> [linux_bridge] 
> physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

Remplacez `PROVIDER_INTERFACE_NAME` par le nom de l'interface réseau physique du fournisseur

Dans la section `[vxlan]` , activez les réseaux superposés VXLAN, configurez l'adresse IP de l'interface réseau physique qui gère les réseaux superposés et activez le remplissage de la couche 2:

> [vxlan] 
> enable_vxlan = true 
> local_ip = OVERLAY_INTERFACE_IP_ADDRESS
> l2_population = true

remplacez `OVERLAY_INTERFACE_IP_ADDRESS` par l'adresse IP de gestion du nœud du contrôleur.

Dans la section `[securitygroup]` , activez les groupes de sécurité et configurez le pilote du pare-feu iptables du pont Linux:

> [securitygroup] 
> enable_security_group = true 
> firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

Assurez-vous que le noyau de votre système d'exploitation Linux prend en charge les filtres de pont réseau en vérifiant que toutes les valeurs `sysctl` suivantes sont définies sur `1` :
`sysctl net.bridge.bridge-nf-call-iptables`
`sysctl net.bridge.bridge-nf-call-ip6tables`

## Configurez l'agent de couche 3 

L'agent de couche 3 (L3) fournit des services de routage et NAT pour les réseaux virtuels en libre-service.

-   Editez le fichier `/etc/neutron/l3_agent.ini` et effectuez les actions suivantes:
    
    -   Dans la section `[DEFAULT]` , configurez le pilote d'interface de pont Linux et le pont de réseau externe:
 

>    [DEFAULT] 
>    interface_driver = linuxbridge

## Configurez l'agent DHCP 

L'agent DHCP fournit des services DHCP pour les réseaux virtuels.

-   Editez le fichier `/etc/neutron/dhcp_agent.ini` et effectuez les actions suivantes:
    
    -   Dans la section `[DEFAULT]` , configurez le pilote d'interface de pont Linux, le pilote DHCP Dnsmasq et activez les métadonnées isolées afin que les instances sur les réseaux du fournisseur puissent accéder aux métadonnées sur le réseau:
   

>  [DEFAULT]
>   interface_driver = linuxbridge 
>   dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
>   enable_isolated_metadata = true

## Configurez l'agent de métadonnées 

L'agent de métadonnées fournit des informations de configuration, telles que les informations d'identification aux instances.

-   Editez le fichier `/etc/neutron/metadata_agent.ini` et effectuez les actions suivantes:
    
    -   Dans la section `[DEFAULT]` , configurez l'hôte de métadonnées et le secret partagé:
  

>   [DEFAULT] 
>   nova_metadata_host = controller
> metadata_proxy_shared_secret = METADATA_SECRET

Remplacez `METADATA_SECRET` par un secret approprié pour le proxy de métadonnées.
## Configurez le service Compute pour utiliser le service de mise en réseau
Editez le fichier `/etc/nova/nova.conf` et effectuez les opérations suivantes:

-   Dans la section `[neutron]` , configurez les paramètres d'accès, activez le proxy de métadonnées et configurez le secret:

> [neutron] 
> url = http://controller:9696 
> auth_url = http://controller:5000 
> auth_type = password 
> project_domain_name = default 
> user_domain_name = default 
> region_name = RegionOne
> project_name = service 
> username = neutron 
> password = NEUTRON_PASS
> service_metadata_proxy = true 
> metadata_proxy_shared_secret = METADATA_SECRET

Remplacez `NEUTRON_PASS` par le mot de passe que vous avez choisi pour l'utilisateur de `neutron` dans le service d'identité.

Remplacez `METADATA_SECRET` par le secret que vous avez choisi pour le proxy de métadonnées.

1. Remplir la base de données:
`# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \`
 `--config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron`
 2. Redémarrez le service API Compute:
 `# service nova-api restart`
 3. Redémarrez les services de mise en réseau.
` # service neutron-server restart`
`# service neutron-linuxbridge-agent restart`
`# service neutron-dhcp-agent restart`
`# service neutron-metadata-agent restart`
`# service neutron-l3-agent restart`


   ## Installer et configurer le service neutron sur le noeud compute
   Ce noeud  gère les groupes de connectivité et de sécurité pour les instances.
   ## Installer les composants

 `# apt-get install neutron-linuxbridge-agent`
 ## Configurez le composant
 Editez le fichier `/etc/neutron/neutron.conf` et effectuez les actions suivantes:

-   Dans la section `[database]` , commentez toutes `connection` options de `connection` , car les nœuds de calcul n'accèdent pas directement à la base de données.
    
-   Dans la section `[DEFAULT]` , configurez l'accès à la file d'attente de messages `RabbitMQ` :

> [DEFAULT] 
> transport_url = rabbit://openstack:RABBIT_PASS@controller

Remplacez `RABBIT_PASS` par le mot de passe que vous avez choisi pour le compte `openstack` dans RabbitMQ.

* Dans les sections `[DEFAULT]` et `[keystone_authtoken]` , configurez l'accès au service d'identité:

> [DEFAULT] 
> auth_strategy = keystone

> [keystone_authtoken] 
> www_authenticate_uri = http://controller:5000
> auth_url = http://controller:5000 
> memcached_servers = controller:11211
> auth_type = password 
> project_domain_name = default 
> user_domain_name = default 
> project_name = service 
> username = neutron 
> password = NEUTRON_PASS

Remplacez `NEUTRON_PASS` par le mot de passe que vous avez choisi pour l'utilisateur de `neutron` dans le service d'identité.

**NB:** Mettez en commentaire ou supprimez toute autre option de la section `[keystone_authtoken]` .

* Dans la section `[oslo_concurrency]` , configurez le chemin de verrouillage:

> [oslo_concurrency] 
> lock_path = /var/lib/neutron/tmp

## Configurez l'agent de pont Linux 

L'agent pont Linux crée une infrastructure de réseau virtuel pour les instances de couche 2 (pontage et commutation) et gère les groupes de sécurité.

-   Editez le fichier `/etc/neutron/plugins/ml2/linuxbridge_agent.ini` et `/etc/neutron/plugins/ml2/linuxbridge_agent.ini` comme suit:
    
    -   Dans la section `[linux_bridge]` , `[linux_bridge]` le réseau virtuel du fournisseur sur l'interface réseau physique du fournisseur:
    

> [linux_bridge] 
> physical_interface_mappings = provider:PROVIDER_INTERFACE_NAME

Remplacez `PROVIDER_INTERFACE_NAME` par le nom de l'interface réseau physique du fournisseur

* Dans la section `[vxlan]` , activez les réseaux superposés VXLAN, configurez l'adresse IP de l'interface réseau physique qui gère les réseaux superposés et activez le remplissage de la couche 2:

> [vxlan] 
> enable_vxlan = true 
> local_ip = OVERLAY_INTERFACE_IP_ADDRESS
> l2_population = true

remplacez `OVERLAY_INTERFACE_IP_ADDRESS` par l'adresse IP de gestion 192.168.2.x 
* Dans la section `[securitygroup]` , activez les groupes de sécurité et configurez le pilote du pare-feu iptables du pont Linux:

> [securitygroup] 
> enable_security_group = true 
> firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

Assurez-vous que le noyau de votre système d'exploitation Linux prend en charge les filtres de pont réseau en vérifiant que toutes les valeurs `sysctl` suivantes sont définies sur `1` :
`sysctl net.bridge.bridge-nf-call-iptables`
`sysctl net.bridge.bridge-nf-call-ip6tables`

## Configurez le service Compute pour utiliser le service de mise en réseau 

-   Editez le fichier `/etc/nova/nova.conf` et `/etc/nova/nova.conf` comme suit:
    
    -   Dans la section `[neutron]` , configurez les paramètres d'accès:
 

>    [neutron] 
>    url = http://controller:9696 
>    auth_url = http://controller:5000 
>    auth_type = password 
>    project_domain_name = default 
>    user_domain_name = default 
>    region_name = RegionOne
> project_name = service 
> username = neutron 
> password = NEUTRON_PASS

Remplacez `NEUTRON_PASS` par le mot de passe que vous avez choisi pour l'utilisateur de `neutron` dans le service d'identité.
        

## Finaliser l'installation 
1.  Redémarrez le service Compute:
    
     `# service nova-compute restart`
    
2.  Redémarrez l'agent de pont Linux:
    
     `# service neutron-linuxbridge-agent restart`

## Vérifier le fonctionnement

Effectuez ces commandes sur le nœud du contrôleur.
1.  Indiquez les informations d'identification de l' `admin` pour accéder aux commandes CLI destinées uniquement à l'administrateur:
    
     `$ . admin-openrc`
    
2.   Répertoriez les extensions chargées pour vérifier le bon lancement du processus du `neutron-server` :
    
     `$ openstack extension list --network`

`+---------------------------+---------------------------+----------------------------+`
`| Name                      | Alias                     | Description                |`
`+---------------------------+---------------------------+----------------------------+`
`| Default Subnetpools       | default-subnetpools       | Provides ability to mark   |`
`|                           |                           | and use a subnetpool as    |`
`|                           |                           | the default                |`
`| Availability Zone         | availability_zone         | The availability zone      |`
`|                           |                           | extension.                 |`
`| Network Availability Zone | network_availability_zone | Availability zone support  |`
`|                           |                           | for network.               |`
`| Port Binding              | binding                   | Expose port bindings of a  |`
`|                           |                           | virtual port to external   |`
`|                           |                           | application                |`
`| agent                     | agent                     | The agent management       |`
`|                           |                           | extension.                 |`
`| Subnet Allocation         | subnet_allocation         | Enables allocation of      |`
`|                           |                           | subnets from a subnet pool |`
`| DHCP Agent Scheduler      | dhcp_agent_scheduler      | Schedule networks among    |`
`|                           |                           | dhcp agents                |`
`| Neutron external network  | external-net              | Adds external network      |`
`|                           |                           | attribute to network       |`
`|                           |                           | resource.                  |`
`| Neutron Service Flavors   | flavors                   | Flavor specification for   |`
`|                           |                           | Neutron advanced services  |`
`| Network MTU               | net-mtu                   | Provides MTU attribute for |`
`|                           |                           | a network resource.        |`
`| Network IP Availability   | network-ip-availability   | Provides IP availability   |`
`|                           |                           | data for each network and  |`
`|                           |                           | subnet.                    |`
`| Quota management support  | quotas                    | Expose functions for       |`
`|                           |                           | quotas management per      |`
`|                           |                           | tenant                     |`
`| Provider Network          | provider                  | Expose mapping of virtual  |`
`|                           |                           | networks to physical       |`
`|                           |                           | networks                   |`
`| Multi Provider Network    | multi-provider            | Expose mapping of virtual  |`
`|                           |                           | networks to multiple       |`
`|                           |                           | physical networks          |`
`| Address scope             | address-scope             | Address scopes extension.  |`
`| Subnet service types      | subnet-service-types      | Provides ability to set    |`
`|                           |                           | the subnet service_types   |`
`|                           |                           | field                      |`
`| Resource timestamps       | standard-attr-timestamp   | Adds created_at and        |`
`|                           |                           | updated_at fields to all   |`
`|                           |                           | Neutron resources that     |`
`|                           |                           | have Neutron standard      |`
`|                           |                           | attributes.                |`
`| Neutron Service Type      | service-type              | API for retrieving service |`
`| Management                |                           | providers for Neutron      |`
`|                           |                           | advanced services          |`
`| resources: subnet,        |                           | more L2 and L3 resources.  |`
`| subnetpool, port, router  |                           |                            |`
`| Neutron Extra DHCP opts   | extra_dhcp_opt            | Extra options              |`
`|                           |                           | configuration for DHCP.    |`
`|                           |                           | For example PXE boot       |`
`|                           |                           | options to DHCP clients    |`
`|                           |                           | can be specified (e.g.     |`
`|                           |                           | tftp-server, server-ip-    |`
`|                           |                           | address, bootfile-name)    |`
`| Resource revision numbers | standard-attr-revisions   | This extension will        |`
`|                           |                           | display the revision       |`
`|                           |                           | number of neutron          |`
`|                           |                           | resources.                 |`
`| Pagination support        | pagination                | Extension that indicates   |`
`|                           |                           | that pagination is         |`
`|                           |                           | enabled.                   |`
`| Sorting support           | sorting                   | Extension that indicates   |`
`|                           |                           | that sorting is enabled.   |`
`| security-group            | security-group            | The security groups        |`
`|                           |                           | extension.                 |`
`| RBAC Policies             | rbac-policies             | Allows creation and        |`
`|                           |                           | modification of policies   |`
`|                           |                           | that control tenant access |`
`|                           |                           | to resources.              |`
`| standard-attr-description | standard-attr-description | Extension to add           |`
`|                           |                           | descriptions to standard   |`
`|                           |                           | attributes                 |`
`| Port Security             | port-security             | Provides port security     |`
`| Allowed Address Pairs     | allowed-address-pairs     | Provides allowed address   |`
`|                           |                           | pairs                      |`
`| project_id field enabled  | project-id                | Extension that indicates   |`
`|                           |                           | that project_id field is   |`
`|                           |                           | enabled.                   |`
`+---------------------------+---------------------------+----------------------------+`


* Liste des agents pour vérifier le lancement réussi des agents neutron:

`$ openstack network agent list`

`+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+`
`| ID                                   | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |`
`+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+`
`| f49a4b81-afd6-4b3d-b923-66c8f0517099 | Metadata agent     | controller | None              | True  | UP    | neutron-metadata-agent    |`
`| 27eee952-a748-467b-bf71-941e89846a92 | Linux bridge agent | controller | None              | True  | UP    | neutron-linuxbridge-agent |`
`| 08905043-5010-4b87-bba5-aedb1956e27a | Linux bridge agent | compute   | None              | True  | UP    | neutron-linuxbridge-agent |`
`| 830344ff-dc36-4956-84f4-067af667a0dc | L3 agent           | controller | nova              | True  | UP    | neutron-l3-agent          |`
`| dd3644c9-1a3a-435a-9282-eb306b4b0391 | DHCP agent         | controller | nova              | True  | UP    | neutron-dhcp-agent        |`
`+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+`

## Installer et configurer Horizon sur le noeud controller

* Cette section explique comment installer et configurer le tableau de bord sur le nœud du contrôleur.
1.    Installez les paquets:
     ` # apt-get install openstack-dashboard`
     
 2. Editez le fichier `/etc/openstack-dashboard/local_settings.py` et effectuez les actions suivantes:

-   Configurez le tableau de bord pour utiliser les services OpenStack sur le nœud du `controller` :

>  OPENSTACK_HOST = "controller"

    
-   Dans la section Configuration du tableau de bord, autorisez vos hôtes à accéder à Dashboard:
    
     

> ALLOWED_HOSTS = [ 'one.example.com' , 'two.example.com' ]

**NB:** `ALLOWED_HOSTS` peut également être `['*']` pour accepter tous les hôtes. Cela peut être utile pour les travaux de développement, mais est potentiellement peu sûr et ne devrait pas être utilisé en production.

Configurez le service de stockage de session `memcached` :
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}

-   Activer l’API d’identité version 3:
    
> OPENSTACK_KEYSTONE_URL = "http:// %s :5000/v3" % OPENSTACK_HOST

   -   Activer le support pour les domaines:

    
     

> OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True
   
-   Configurer les versions de l'API:
> 
>      OPENSTACK_API_VERSIONS = {
>         "identity" : 3 ,
>         "image" : 2 ,
>         "volume" : 2 ,
>     }

    
-   Configurez `Default` comme domaine par défaut pour les utilisateurs que vous créez via le tableau de bord:
    
     

> OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

    
-   Configurez l' `admin` tant que rôle par défaut pour les utilisateurs que vous créez via le tableau de bord:
    
     OPENSTACK_KEYSTONE_DEFAULT_ROLE = "admin"

Eventuellement, configurez le fuseau horaire:

 

> TIME_ZONE = "TIME_ZONE"

Remplacez `TIME_ZONE` par un identifiant de fuseau horaire approprié.
* Africa/Tunis
*  Ajoutez la ligne suivante à `/etc/apache2/conf-available/openstack-dashboard.conf` si elle n’est pas incluse.
    
 

>    WSGIApplicationGroup %{GLOBAL}

    

## Finaliser l'installation 

-   Rechargez la configuration du serveur Web:
    
     `# service apache2 reload`

## Vérifier le fonctionnement 

Vérifier le fonctionnement du tableau de bord.

Accédez au tableau de bord à l'aide d'un navigateur Web à l' `http://controller/horizon` .

Authentifiez-vous à l'aide des informations d'identification d' `admin` ou d'utilisateur et du domaine `default` .
**NB:** si le tableau de bord n'apparait , il doit que vous essayez de modifier le fichier `/etc/memcached.conf` 
remplacez `-l 192.168.2.x ` par `-l 0.0.0.0`

## Guide d'installation de Cinder

Le service de stockage en bloc (cinder) fournit des périphériques de stockage en bloc aux instances d'invité.
L'API de stockage en bloc s'exécute  sur les nœuds de contrôleur.
## Prérequis 

Avant d'installer et de configurer le service Block Storage, vous devez créer une base de données, des informations d'identification de service et des points de terminaison d'API.

1.  Pour créer la base de données, procédez comme suit:
    
    *  Utilisez le client d'accès à la base de données pour vous connecter au serveur de base de données en tant qu'utilisateur `root` :
     `# mysql`
    
-   Créez la base de données `cinder` :
    
   

>   MariaDB [(none)]> CREATE DATABASE cinder;

    
-   Accordez un accès approprié à la base de données `cinder` :
    
    ` MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'localhost' \`
   `  IDENTIFIED BY 'CINDER_DBPASS';`
   ` MariaDB [(none)]> GRANT ALL PRIVILEGES ON cinder.* TO 'cinder'@'%' \`
   `  IDENTIFIED BY 'CINDER_DBPASS';`
    
    Remplacez `CINDER_DBPASS` par un mot de passe approprié.
    
-   Quittez le client d'accès à la base de données.
2. Indiquez les informations d'identification de l' `admin` pour accéder aux commandes CLI destinées uniquement à l'administrateur:

      `$ . admin-openrc`
3. Pour créer les informations d'identification du service, procédez comme suit:

* Créer un utilisateur de `cinder` :
`$ openstack user create --domain default --password-prompt cinder`

`User Password:`
`Repeat User Password:`
`+---------------------+----------------------------------+`
`| Field               | Value                            |`
`+---------------------+----------------------------------+`
`| domain_id           | default                          |`
`| enabled             | True                             |`
`| id                  | 9d7e33de3e1a498390353819bc7d245d |`
`| name                | cinder                           |`
`| options             | {}                               |`
`| password_expires_at | None                             |`
`+---------------------+----------------------------------+`

* Ajoutez le rôle `admin` à l'utilisateur `cinder` :
` $ openstack role add --project service --user cinder admin`

Créez les entités de service `cinderv2` et `cinderv3` :
` $ openstack service create --name cinderv2 \`
`  --description "OpenStack Block Storage" volumev2`

`+-------------+----------------------------------+`
`| Field       | Value                            |`
`+-------------+----------------------------------+`
`| description | OpenStack Block Storage          |`
`| enabled     | True                             |`
`| id          | eb9fd245bdbc414695952e93f29fe3ac |`
`| name        | cinderv2                         |`
`| type        | volumev2                         |`
`+-------------+----------------------------------+`

` $ openstack service create --name cinderv3 \`
 ` --description "OpenStack Block Storage" volumev3`

`+-------------+----------------------------------+`
`| Field       | Value                            |`
`+-------------+----------------------------------+`
`| description | OpenStack Block Storage          |`
`| enabled     | True                             |`
`| id          | ab3bbbef780845a1a283490d281e7fda |`
`| name        | cinderv3                         |`
`| type        | volumev3                         |`
`+-------------+----------------------------------+`

**NB:** Les services de stockage en mode bloc nécessitent deux entités de service.

4. Créez les points de terminaison de l'API du service Block Storage:
` $ openstack endpoint create --region RegionOne \`
  `volumev2 public http://controller:8776/v2/% \( project_id \) s`

`+--------------+------------------------------------------+`
`| Field        | Value                                    |`
`+--------------+------------------------------------------+`
`| enabled      | True                                     |`
`| id           | 513e73819e14460fb904163f41ef3759         |`
`| interface    | public                                   |`
`| region       | RegionOne                                |`
`| region_id    | RegionOne                                |`
`| service_id   | eb9fd245bdbc414695952e93f29fe3ac         |`
`| service_name | cinderv2                                 |`
`| service_type | volumev2                                 |`
`| url          | http://controller:8776/v2/%(project_id)s |`
`+--------------+------------------------------------------+`

`$ openstack endpoint create --region RegionOne \`
`  volumev2 internal http://controller:8776/v2/%\(project_id\)s`

`+--------------+------------------------------------------+`
`| Field        | Value                                    |`
`+--------------+------------------------------------------+`
`| enabled      | True                                     |`
`| id           | 6436a8a23d014cfdb69c586eff146a32         |`
`| interface    | internal                                 |`
`| region       | RegionOne                                |`
`| region_id    | RegionOne                                |`
`| service_id   | eb9fd245bdbc414695952e93f29fe3ac         |`
`| service_name | cinderv2                                 |`
`| service_type | volumev2                                 |`
`| url          | http://controller:8776/v2/%(project_id)s |`
`+--------------+------------------------------------------+`

`$ openstack endpoint create --region RegionOne \`
 ` volumev2 admin http://controller:8776/v2/%\(project_id\)s`

`+--------------+------------------------------------------+`
`| Field        | Value                                    |`
`+--------------+------------------------------------------+`
`| enabled      | True                                     |`
`| id           | e652cf84dd334f359ae9b045a2c91d96         |`
`| interface    | admin                                    |`
`| region       | RegionOne                                |`
`| region_id    | RegionOne                                |`
`| service_id   | eb9fd245bdbc414695952e93f29fe3ac         |`
`| service_name | cinderv2                                 |`
`| service_type | volumev2                                 |`
`| url          | http://controller:8776/v2/%(project_id)s |`
`+--------------+------------------------------------------+`

`$ openstack endpoint create --region RegionOne \`
 ` volumev3 public http://controller:8776/v3/%\(project_id\)s`

`+--------------+------------------------------------------+`
`| Field        | Value                                    |`
`+--------------+------------------------------------------+`
`| enabled      | True                                     |`
`| id           | 03fa2c90153546c295bf30ca86b1344b         |`
`| interface    | public                                   |`
`| region       | RegionOne                                |`
`| region_id    | RegionOne                                |`
`| service_id   | ab3bbbef780845a1a283490d281e7fda         |`
`| service_name | cinderv3                                 |`
`| service_type | volumev3                                 |`
`| url          | http://controller:8776/v3/%(project_id)s |`
`+--------------+------------------------------------------+`

`$ openstack endpoint create --region RegionOne \`
 ` volumev3 internal http://controller:8776/v3/%\(project_id\)s`

`+--------------+------------------------------------------+`
`| Field        | Value                                    |`
`+--------------+------------------------------------------+`
`| enabled      | True                                     |`
`| id           | 94f684395d1b41068c70e4ecb11364b2         |`
`| interface    | internal                                 |`
`| region       | RegionOne                                |`
`| region_id    | RegionOne                                |`
`| service_id   | ab3bbbef780845a1a283490d281e7fda         |`
`| service_name | cinderv3                                 |`
`| service_type | volumev3                                 |`
`| url          | http://controller:8776/v3/%(project_id)s |`
`+--------------+------------------------------------------+`

`$ openstack endpoint create --region RegionOne \`
 ` volumev3 admin http://controller:8776/v3/%\(project_id\)s`

`+--------------+------------------------------------------+`
`| Field        | Value                                    |`
`+--------------+------------------------------------------+`
`| enabled      | True                                     |`
`| id           | 4511c28a0f9840c78bacb25f10f62c98         |`
`| interface    | admin                                    |`
`| region       | RegionOne                                |`
`| region_id    | RegionOne                                |`
`| service_id   | ab3bbbef780845a1a283490d281e7fda         |`
`| service_name | cinderv3                                 |`
`| service_type | volumev3                                 |`
`| url          | http://controller:8776/v3/%(project_id)s |`
`+--------------+------------------------------------------+`

**NB:** Les services de stockage en mode bloc nécessitent des points de terminaison pour chaque entité de service.

## Installer et configurer les composants 

1.  Installez les paquets:
    
    ` # apt install cinder-api cinder-scheduler`
    
2.  Editez le fichier `/etc/cinder/cinder.conf` et effectuez les actions suivantes:
    
    *  Dans la section `[database]` , configurez l'accès à la base de données:
   

>  [database] 
>  connection = mysql+pymysql://cinder:CINDER_DBPASS@controller/cinder

-   Remplacez `CINDER_DBPASS` par le mot de passe choisi pour la base de données Block Storage.
    
-   Dans la section `[DEFAULT]` , configurez l'accès à la file d'attente de messages `RabbitMQ` :

> [DEFAULT] 
> transport_url = rabbit://openstack:RABBIT_PASS@controller

-   Remplacez `RABBIT_PASS` par le mot de passe que vous avez choisi pour le compte `openstack` dans `RabbitMQ` .
    
-   Dans les sections `[DEFAULT]` et `[keystone_authtoken]` , configurez l'accès au service d'identité:

> [DEFAULT] 
> auth_strategy = keystone

> [keystone_authtoken] 
> www_authenticate_uri = http://controller:5000
> auth_url = http://controller:5000 
> memcached_servers = controller:11211
> auth_type = password 
> project_domain_id = default 
> user_domain_id = default 
> project_name = service 
> username = cinder 
> password = CINDER_PASS

Remplacez `CINDER_PASS` par le mot de passe que vous avez choisi pour l'utilisateur `cinder` dans le service d'identité.
**NB:** Mettez en commentaire ou supprimez toute autre option de la section `[keystone_authtoken]` .
* Dans la section `[DEFAULT]` , configurez l'option `my_ip` pour utiliser l'adresse IP de l'interface de gestion du nœud du contrôleur:

> [DEFAULT] 
> my_ip = 192.168.2.x

3. Dans la section `[oslo_concurrency]` , configurez le chemin de verrouillage:

> [oslo_concurrency] 
> lock_path = /var/lib/cinder/tmp

4. Remplir la base de données Block Storage:
`# su -s /bin/sh -c "cinder-manage db sync" cinder`

## Configurer Compute pour utiliser Block Storage 

1.  Editez le fichier `/etc/nova/nova.conf`  :

> [cinder] 
> os_region_name = RegionOne

## Finaliser l'installation 

1.  Redémarrez le service API Compute:
    
     `# service nova-api restart`
    
2.  Redémarrez les services de stockage en bloc:
    
   `# service cinder-scheduler restart`
   `# service apache2 restart`
## Vérifier le fonctionnement de Cinder 
Effectuez ces commandes sur le nœud du contrôleur.

1. Indiquez les informations d'identification de l' admin pour accéder aux commandes CLI destinées uniquement à l'administrateur:

     `$ . admin-openrc`

2. Répertoriez les composants de service pour vérifier le lancement réussi de chaque processus:

     `$ openstack volume service list`

    +------------------+------------+------+---------+-------+----------------------------+
    | Binary           | Host       | Zone | Status  | State | Updated_at                 |
    +------------------+------------+------+---------+-------+----------------------------+
    | cinder-scheduler | controller | nova | enabled | up    | 2019-04-30T02:27:41.000000 |
    +------------------+------------+------+---------+-------+----------------------------+


