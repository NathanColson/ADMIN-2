# TP5 : Configuration du service web public

Noms des auteurs : Thomas Ciserane, Nathan Colson, WILLEMS Julien

Date de réalisation : 24/03/2025

### Mise en place de l'environnement de travail

Documentez ou mettez à jour votre documentation concernant votre infrastructure et votre environnement de travail : quels VPS sont utilisés pour quels services?

L'infastructure de travail n'a pas changé depuis le TP4.

Un unique VPS est utilisé pour tous les services.

Pour le service Web en particulier : quelle organisation utilisez-vous pour vos fichiers de config, sur Github et sur le VPS qui héberge le service Web?

La même logique mais légèrement différente.

```bash
tp5/
 └──web/
    ├── docker-compose.yml
    ├── Dockerfile
    ├── index.html
    └── config/
        └── nginx.conf
```

# 1. Configuration de base d'un serveur web

Documentez la première version de votre configuration Web avec :

- la configuration de base pour le premier site

On a directement effectué un Dockerfile et un docker-compose.yml avec les configurations suivantes :

**Dockerfile**:

```Dockerfile
FROM nginx:latest

RUN rm /etc/nginx/conf.d/default.conf # Pour éviter les potentiels conflits

COPY config/nginx.conf /etc/nginx/nginx.conf

RUN mkdir -p /var/www/html/www

COPY index.html /var/www/html/www/index.html

EXPOSE 80
```

**docker-compose.yml**:

```yaml
services:
  web:
    build: .
    image: web-l1-7
    container_name: web
    ports:
      - "80:80"
    volumes:
      - ./index.html:/var/www/html/www/index.html
```

Et voici le résultat :

![Site web www.l1-7.ephec-ti.be](https://github.com/ShiSui97x/EphecCourses/blob/main/docs/images/tp5_screenshots/website.png?raw=true "Site web www.l1-7.ephec-ti.be")

- le Virtual Hosting

Pour réaliser le virtual hosting, nous avons modifié le Dockerfile, le docker-compose.yml, le nginx.conf et surtout ajouter la ligne suivante `blog    IN  CNAME   www.l1-7.ephec-ti.be.` dans le fichier l1-7.zone afin de pouvoir acceder au site via `blog.l1-7.ephec-ti.be`.

Architecture du réperroire après les modifications:

```bash
tp5/
└── web/
    ├── config/
    │   └── nginx.conf
    ├── html/
    │   ├── www/
    │   │   └── index.html
    │   └── blog/
    │       └── index.html
    ├── docker-compose.yml
    └── Dockerfile
```

**Dockerfile**:

```Dockerfile
FROM nginx:latest

RUN rm /etc/nginx/conf.d/default.conf

COPY config/nginx.conf /etc/nginx/nginx.conf

RUN mkdir -p /var/www/html/www
RUN mkdir -p /var/www/html/blog

COPY html/www/index.html /var/www/html/www/index.html
COPY html/blog/index.html /var/www/html/blog/index.html

EXPOSE 80
```

**docker-compose.yml**:

```yaml
services:
  web:
    build: .
    image: shi-web
    container_name: web
    ports:
      - "80:80"
    volumes:
      - ./html/www:/var/www/html/www
      - ./html/blog:/var/www/html/blog
      - ./config/nginx.conf:/etc/nginx/nginx.conf
```

**nginx.conf**:

```nginx
events {
}
http {
 server {
     listen          80;
     server_name     www.l1-7.ephec-ti.be;
     index           index.html;
     root            /var/www/html/www/;
 }

 server {
     listen          80;
     server_name     blog.l1-7.ephec-ti.be;
     index           index.html;
     root            /var/www/html/blog/;
 }
}
```

Les 2 sites sont accessibles :

![Site blog blog.l1-7.ephec-ti.be](https://github.com/ShiSui97x/EphecCourses/blob/main/docs/images/tp5_screenshots/blog.png?raw=true "Site blog.l1-7.ephec-ti.be")

- la configuration des logs.

Dans notre container `web`, les fichiers de logs `/var/log/nginx/access.log` et `error.log` sont des liens symboliques vers `/dev/stdout` et `/dev/stderr`, ce qui permet à Docker de centraliser automatiquement les logs et les rendre accessibles via la commande `docker logs`.

![Logs](https://github.com/ShiSui97x/EphecCourses/blob/main/docs/images/tp5_screenshots/logs.png?raw=true "Logs")

Puis nous avons formaté les logs afin de savoir à quel hôte est destiné chaque requête. Pour ce faire, le fichier `nginx.conf` a évolué de la forme suivante :

```nginx
events {
}
http {
    log_format log_per_virtualhost '[$host] $remote_addr [$time_local] $status '
                                   '"$request" $body_bytes_sent';

 server {
     listen          80;
     server_name     www.l1-7.ephec-ti.be;
     index           index.html;
     root            /var/www/html/www/;
        
  access_log /dev/stdout log_per_virtualhost;
 }

 server {
     listen          80;
     server_name     blog.l1-7.ephec-ti.be;
     index           index.html;
     root            /var/www/html/blog/;

  access_log /dev/stdout log_per_virtualhost;
 }
}
```

Et voici le resultat avec les requêtes surlignées:

<img src="https://github.com/ShiSui97x/EphecCourses/blob/main/docs/images/tp5_screenshots/formatted_logs.png?raw=true" alt="Logs formaté" width="70%">

# 2. Site web dynamique

- Documentez la mise en oeuvre de votre site web dynamique.

## 2.1.1. Premier test de MariaDB

 Pour commencer, nous avons réalisé un test simple de connexion à une base de données **MariaDB** en exécutant directement un container via la commande suivante :

 ```bash
 docker run --name mariadbtest -e MYSQL_ROOT_PASSWORD=mypass --rm -d mariadb
 ```

 Cette commande permet de lancer un container MariaDB avec un mot de passe root défini (`mypass`). Le flag `--rm` supprime automatiquement le container à l’arrêt, ce qui est pratique pour un test temporaire.

 Nous avons ensuite récupéré l'adresse IP du container grâce à la commande :

 ```bash
 docker inspect mariadbtest | grep "IPAddress"
 ```

 À l’aide de cette IP, nous avons pu nous connecter à la base de données depuis notre VPS via le client `mysql` :

 ```bash
 mysql -h <IP_du_container> -u root -p
 ```

 La connexion a été établie avec succès. Une fois connectés, nous avons utilisé la commande `SHOW DATABASES;` pour vérifier que l'accès fonctionnait correctement.

<img src="https://github.com/ShiSui97x/EphecCourses/blob/main/docs/images/tp5_screenshots/test_db.png?raw=true" alt="Test de la BDD" width="60%">

## 2.1.2. Ajouter du contenu à la base de données

 Après avoir vérifié la connectivité à MariaDB, nous avons ajouté du contenu de test dans une nouvelle base de données nommée **woodytoys**.  

 1. Nous avons d’abord relancé une connexion au container MariaDB via :

 ```bash
 mysql -h <IP_du_container> -u root -p
 ```

 2. Depuis l’interface MariaDB, nous avons créé la base de données :

 ```sql
 CREATE DATABASE woodytoys;
 ```

 3. Ensuite, nous avons créé un fichier `woodytoys.sql` sur notre VPS contenant le script suivant :

 ```sql
 USE woodytoys;

 CREATE TABLE products (
     id mediumint(8) unsigned NOT NULL auto_increment,
     product_name varchar(255) default NULL,
     product_price varchar(255) default NULL,
     PRIMARY KEY (id)
 ) AUTO_INCREMENT=1;

 INSERT INTO products (product_name, product_price) VALUES 
 ("Set de 100 cubes multicolores", "50"),
 ("Yoyo", "10"),
 ("Circuit de billes", "75"),
 ("Arc à flèches", "20"),
 ("Maison de poupées", "150");
 ```

 4. Ce fichier a ensuite été injecté dans MariaDB avec la commande :

 ```bash
 mysql -h <IP_du_container> -u root -pmypass < woodytoys.sql
 ```

 5. Enfin, nous avons vérifié l’insertion correcte des données en consultant la base :

 ```sql
 USE woodytoys;
 SHOW TABLES;
 SELECT * FROM products;
 ```

<img src="https://github.com/ShiSui97x/EphecCourses/blob/main/docs/images/tp5_screenshots/woodytoys.png?raw=true" alt="Woodytoys" width="50%">

Tous les produits sont bien présents, confirmant que notre script fonctionne correctement.

- Rédigez une procédure de validation et les scenarii qu'elle comporte.  
- Appliquez votre procédure de validation à votre configuration et prouvez, via quelques screenshots bien choisis et soigneusement expliqués, que chaque élément fonctionne comme attendu.
- Documentez votre déploiement Docker Compose, et vérifiez, via la même procédure de validation que plus haut, que tout fonctionne de la même manière qu'à l'étape précédente.  

## 2.2. Premier script PHP

Pour pouvoir interpréter du code PHP dans notre site web, nous avons mis en place un container dédié basé sur PHP-FPM.
Celui-ci s'occupera de traiter les fichiers `.php` appelés depuis le virtualhost `www`.

1. Création de l’image PHP personnalisée

- Un sous-répertoire `php` a été créé à la racine du projet.
- À l’intérieur, un `Dockerfile` a été ajouté avec ce contenu :

```Dockerfile
FROM php:8.3-fpm
RUN docker-php-ext-install mysqli
```

- L’image a ensuite été buildée via :

```bash
docker build -t php php/
```

2. Mise en place du fichier PHP de test

- Dans le répertoire `html/www/` (déjà utilisé par nginx), nous avons ajouté un fichier `products.php` :

```php
<?php
phpinfo();
?>
```

3. Lancement du container PHP

- On a monté le même répertoire HTML que celui utilisé par nginx, pour que les deux containers puissent accéder aux mêmes fichiers :

```bash
docker run --name php --rm -d \
  --mount type=bind,source=$(pwd)/html/www,target=/var/www/html/www \
  php
```

4. Connexion entre nginx et PHP-FPM

- On a récupéré l’**IP du container PHP** avec :

```bash
docker inspect -f php | grep IPAddress
```

- Puis, on a modifié le `nginx.conf` comme suit :

```nginx
events {
}
http {
    log_format log_per_virtualhost '[$host] $remote_addr [$time_local]  $status '
                                   '"$request" $body_bytes_sent';

    server {
        listen          80;
        server_name     www.l1-7.ephec-ti.be;
        index           index.html;
        root            /var/www/html/www/;
        access_log      /dev/stdout log_per_virtualhost;

 location ~* \.php$ {
         fastcgi_pass php:9000;  # IP du conteneur PHP-FPM
         include fastcgi_params;
         fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }
    }

    server {
        listen          80;
        server_name     blog.l1-7.ephec-ti.be;
        index           index.html;
        root            /var/www/html/blog/;
        access_log      /dev/stdout log_per_virtualhost;
        
    }
}
```

5. Redémarrage de nginx

```bash
docker run -p80:80 --name web --rm -d \
  --mount type=bind,source=$(pwd)/html,target=/var/www/html/ \
  web
```

6. Résultat

En accédant à `http://www.l1-7.ephec-ti.be/products.php`, la page PHP affiche bien toutes les infos système.  
La communication entre nginx et PHP-FPM via FastCGI fonctionne parfaitement.

<img src="https://github.com/ShiSui97x/EphecCourses/blob/main/docs/images/tp5_screenshots/products.png?raw=true" alt="Products" width="75%">

## 2.3. Connexion entre l’application web et la base de données

Dans cette partie, nous avons établi une connexion entre le script PHP de notre serveur web et le serveur MariaDB hébergeantla base de données `woodytoys`.

Pour ce faire, nous avons créer un fichier `catalogue.php` placé dans le répertoire `/var/www/html/www` partagé entre le conteneur PHP et Nginx.

**Fichier `catalogue.php`**:

```php
<html>

<head>
<title>Catalogue WoodyToys</title>
</head>

<body>
<h1>Catalogue WoodyToys</h1>

<?php
$dbname = 'woodytoys';
$dbuser = 'user';
$dbpass = 'mypass';
$dbhost = 'db';
$connect = mysqli_connect($dbhost, $dbuser, $dbpass) or die("Unable to connect to '$dbhost'");
mysqli_select_db($connect,$dbname) or die("Could not open the database '$dbname'");
$result = mysqli_query($connect,"SELECT id, product_name, product_price FROM products");
?>

<table>
<tr>
 <th>Numéro de produit</th>
 <th>Descriptif</th>
 <th>Prix</th>
</tr>

<?
while ($row = mysqli_fetch_array($result)) {
  printf("<tr><th>%s</th> <th>%s</th> <th>%s</th></tr>", $row[0], $row[1],$row[2]);
}
?>

</table>
</body>
</html>
```

Le scrpit est accessible depuis `http://www.l1-7.ephec-ti.be/catalogue.php`.

<img src="https://github.com/ShiSui97x/EphecCourses/blob/main/docs/images/tp5_screenshots/catalogue.png?raw=true" alt="Catalogue" width="55%">

Ce résulat montre que  l’environnement est correctement configuré, que les **bind mounts fonctionnent**, que **les services communiquent via le réseau Docker**, et que **la base de données est bien initialisée**.

## 2.4. Docker Compose

Une fois tous les services configurés, notre architecture évolue vers la suivante :

```bash
tp5/
 ├── compose.yml
 ├── php
 │   └── Dockerfile
 ├── web
 │   ├── Dockerfile
 │   ├── config
 │   │   └── nginx.conf
 │   └── html
 │       ├── blog
 │       │   └── index.html
 │       └── www
 │           ├── catalogue.php
 │           ├── index.html
 │           └── products.php
 └── woodytoys.sql
```

Puis nous avons créé un fichier `compose.yml` afin d'utiliser un Docker Compose pour faciliter la configuration de notre environnement.

**Fichier `compose.yml`**:

```yaml
services:
  db:
    image: mariadb
    container_name: mariadb
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
      - ./woodytoys.sql:/docker-entrypoint-initdb.d/woodytoys.sql # Fichier SQL de création de la base de données

  php:
    build: ./php
    image: php-l1-7
    container_name: php
    volumes:
      - ./web/html/www:/var/www/html/www
    depends_on:
      - db

  web:
    build: ./web
    image: nginx-l1-7
    container_name: web
    ports:
      - "${NGINX_PORT}:80"
    volumes:
      - ./web/html:/var/www/html  # Fichiers du site catalogue & blog
      - ./web/config/nginx.conf:/etc/nginx/nginx.conf  # Configuration Nginx
    depends_on:
      - php

volumes:
  db_data:
```
