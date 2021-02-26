# User Guide 
 
### Step 1: Clone nextcloud/docker 

Run bellow command line 

    $ mkdir nextcloud
    $ cd nextcloud
    $ git clone http://10.61.173.178/nextcloud/docker.git 

### Step 2: Clone nextcloud/server 

Enter same folder which you cloned nextcloud/docker, then clone nextcloud/server. 

    $ git clone http://10.61.173.178/nextcloud/server.git 
    $ cd server 
    $ git checkout d50e8d85c712991702894e80811c861df4262bbe 
    $ git submodule update --init

Note: d50e8d85c712991702894e80811c861df4262bbe is the commit of nextcloud version 20.0.4 (tag release) 

We will use nextcloud/docker 20.0/apache to build Dockerfile, so you need go into it and copy some files and folders bellow to nextcloud/server:

- config
- cron.sh
- Dockerfile
- entrypoint.sh
- upgrade.exclude

### Step 3: Edit Dockerfile and create .dockerignore

You can edit Dockerfile or plit it out become 2 files: Dockerfile_prebuild and Dockerfile_1 (it the same content if you put it into 1 file Dockerfile, use only in case you want quick build another time) 

After *FROM php:7.4-apache-buster* line of **Dockerfile_prebuild** add the following to setup proxy for pecl install: 

    ...
    RUN pear config-set http_proxy http://10.61.11.42:3128 
    ...


All **Dockerfile_1** file look like this: 

    FROM nextcloud:prebuild

    ENV NEXTCLOUD_VERSION 20.0.4

    RUN mkdir -p /usr/src/nextcloud;
    
    COPY . /usr/src/nextcloud
    
    RUN set -ex; \
        fetchDeps=" \
            gnupg \
            dirmngr \
        "; \
        apt-get update; \
        apt-get install -y --no-install-recommends $fetchDeps; \
        \
        export GNUPGHOME="$(mktemp -d)"; \
        gpgconf --kill all; \
        rm -rf "$GNUPGHOME" /usr/src/nextcloud/updater; \
        mkdir -p /usr/src/nextcloud/data; \
        mkdir -p /usr/src/nextcloud/custom_apps; \
        chmod +x /usr/src/nextcloud/occ; \
        \
        apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false $fetchDeps; \
        rm -rf /var/lib/apt/lists/*
    
    COPY cron.sh entrypoint.sh upgrade.exclude /
    
    ENTRYPOINT ["/entrypoint.sh"]
    CMD ["apache2-foreground"]

Create **.dockerignore** file in nextcloud/server:

    .*
    !.htaccess
    !.user.ini
    .git
    .github
    .gitignore
    .idea
    .tx
    build
    contribute
    test
    *.md
    *.js
    *.json
    *.xml
    *.sh
    !cron.sh
    !entrypoint.sh
    !upgrade.exclude
    LICENSE
    Makefile
    Dockerfile*
    COPYING-README

### Step 4: Build Dockerfile 

After all of step above run following cmd to build prebuild from **Dockerfile_prebuild**: 

    $ docker build \
    --build-arg http_proxy=http://10.61.11.42:3128 \
    --build-arg https_proxy=http://10.61.11.42:3128 \
    -t nextcloud:prebuild -f Dockerfile_prebuild . 

After success prebuild run cmd to build nextcloud image from **Dockerfile_1**: 

    $ docker build \
    --build-arg http_proxy=http://10.61.11.42:3128 \
    --build-arg https_proxy=http://10.61.11.42:3128 \
    -t nextcloud:20.0.4-apache -f Dockerfile_1 . 

### Step 5: Create docker compose file 

Create **docker-compose.yml**: 

    version: '3'

    services:

      db:
        image: postgres:alpine
        container_name: nextcloud-db
        restart: unless-stopped
        networks: 
          - nextcloud_network
        ports: 
          - 5432:5432
        volumes:
          - postgresql:/var/lib/postgresql/data
          - /etc/localtime:/etc/localtime:ro
        env_file:
          - db.env

      app:
        image: nextcloud:20.0.4-apache
        container_name: nextcloud-app
        restart: unless-stopped
        networks: 
          - nextcloud_network
          - default
        volumes:
          - nextcloud:/var/www/html
          - /etc/localtime:/etc/localtime:ro
        ports: 
          - 80:80
          - 443:443
        environment:
          - VIRTUAL_HOST=
          - VIRTUAL_PORT=
          - LETSENCRYPT_HOST=
          - LETSENCRYPT_EMAIL=
          - POSTGRES_HOST=db
        env_file:
          - app.env
          - db.env
        depends_on:
          - db


    volumes:
      postgresql:
      nextcloud:

    networks: 
      nextcloud_network:

And create **db.env** file: 

    POSTGRES_PASSWORD=nextcloud
    POSTGRES_DB=nextcloud
    POSTGRES_USER=nextcloud
 
And create **app.env** file: 

    HTTP_PROXY=10.61.11.42:3128
    HTTPS_PROXY=10.61.11.42:3128
    NO_PROXY=localhost,127.0.0.*

You can delete all volumes before setup new app by:

    $ docker volume rm $(docker volume ls -q)

### Step 6: Deploy Nextcloud 

Finally, same dir of docker-compose.yml file run bellow cmd to run nextcloud: 

    $ docker-compose up -d 

Go to http://localhost and start setup admin user. 

#### Note*:
 - The most popular issue when you try to build and deploy Nextcloud on Docker is **proxy**, alway take care of it

### File:
