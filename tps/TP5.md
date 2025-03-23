# TP5 : Configuration du service web public

Noms des auteurs : Thomas Ciserane, Nathan Colson, WILLEMS Julien

Date de réalisation : xx/03/2025

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
![Logs formaté](https://github.com/ShiSui97x/EphecCourses/blob/main/docs/images/tp5_screenshots/formatted_logs.png?raw=true "Logs formaté")

# 2. Site web dynamique

- Documentez la mise en oeuvre de votre site web dynamique.  
- Rédigez une procédure de validation et les scenarii qu'elle comporte.  
- Appliquez votre procédure de validation à votre configuration et prouvez, via quelques screenshots bien choisis et soigneusement expliqués, que chaque élément fonctionne comme attendu.
- Documentez votre déploiement Docker Compose, et vérifiez, via la même procédure de validation que plus haut, que tout fonctionne de la même manière qu'à l'étape précédente.  
