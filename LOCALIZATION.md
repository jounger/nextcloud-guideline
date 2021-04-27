# User Guide to translate Nextcloud language

## Install nextcloud on WSL
### Step 1: Set proxy environment

    $ sudo su
    $ vi /etc/apt/apt.conf
        > Acquire::http::proxy "http://private_ip";
        > Acquire::https::proxy "http://private_ip";

### Step 2: Install required dependencies

    $ apt-get update -y
    $ apt install apache2 mariadb-server libapache2-mod-php7.4 -y
    $ apt install php7.4-gd php7.4-mysql php7.4-curl php7.4-mbstring php7.4-intl -y
    $ apt install php7.4-gmp php7.4-bcmath php-imagick php7.4-xml php7.4-zip -y
    $ apt-get install gettext

In case you want to use sqlite over mysql

    $ apt-get install php7.4-sqlite -y
    $ apt-get install sqlite -y

### Step 3: Clone repository into your directory

    $ git config --global http.proxy "http://private_ip"
    $ git clone https://github.com/nextcloud/server.git

### Step 4: Install submodule and grant permission for local user

    $ cd server
    $ git submodule update --init
    $ git checkout localization-vi
    $ mkdir data
    $ chown www-data:www-data .htaccess .user.ini * data

### Step 5: Change root document

    $ vi /etc/apache2/sites-available/000-default.conf

Change Root: DocumentRoot /var/www/html >  DocumentRoot /var/www/html/server

### Step 6: Run convertation

    $ php translationtool.phar convert-po-files
    $ service apache2 start

### Reference:
- https://docs.nextcloud.com/server/latest/admin_manual/installation/example_ubuntu.html
