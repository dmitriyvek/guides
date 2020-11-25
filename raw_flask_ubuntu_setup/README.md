# Ubuntu Server Set Up for Flask Instruction

Guide for seting up clean Ubuntu server for Flask project. Seting up SSH, Postgresql, Gunicorn, Nginx.

## Create user on server

Connect through SSH to remote Ubuntu server and update repositories and install some initial needed packages:

```
sudo apt-get update ; \
sudo apt-get install -y vim tmux htop git curl wget unzip zip gcc build-essential make ufw
```

Create new sudo user:

```
adduser www
usermod -aG sudo www
```

Or change password for already existed user:

```
sudo passwd www
```

## Setup SSH and SSH keys client

Generate new SSH keys:

```
ssh-keygen -t rsa -b 4096 -f /home/www/.ssh/KEY_NAME -C "COMMENT" -P PASS_PHRAZE
```

Thorw keys to server:

```
ssh-copy-id -i ~/.ssh/PUBLIC_KEY www@SERVER_ID
```

If SSH config does not exist:

```
touch ~/.ssh/config
sudo chmod 600 ~/.ssh/config
```

Change SSH config:

```
vim ~/.ssh/config
    Host HOST_NAME
        HostName IP_OR_DOMAIN
        User www
        Port PORT_NUMBER
        IdentityFile ~/.ssh/PRIVATE_KEY
        IdentitiesOnly yes
```

## Setup SSH on server

Configure SSH:

```
sudo vim /etc/ssh/sshd_config
    AllowUsers www
    AuthenticationMethods publickey
    ChallengeResponseAuthentication no
    PermitRootLogin no
    PasswordAuthentication no
    PermitEmptyPasswords no
    Port NEW_PORT
    PubkeyAuthentication yes
    UsePAM no
```

Restart SSH server:

```
sudo service ssh restart
```

Setup firewall:

```
sudo ufw allow SSH_PORT
sudo ufw enable
```

## Install must-have packages & ZSH

```
sudo apt-get install -y zsh tree nginx zlib1g-dev libbz2-dev libreadline-dev llvm libncurses5-dev libncursesw5-dev xz-utils liblzma-dev python3-dev python3-lxml python-libxml2  libffi-dev libssl-dev libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev libgdbm-dev libnss3-dev supervisor virtualen certbot python3-certbot-nginx
```

Install [oh-my-zsh](https://github.com/robbyrussell/oh-my-zsh):

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

Install [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting):

```
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git $ZSH_CUSTOM/plugins/zsh-syntax-highlighting
```

Change default shell (if not yet):

```
chsh -s /bin/zsh
```

Configure zsh and some needed aliases:

```
vim ~/.zshrc
    ZSH_THEME="agnoster"
    plugins=(zsh-syntax-highlighting)

    alias ll='ls -alF'
    alias la='ls -A'
    alias l='ls -CF'
    alias ls="ls --color -l"

source ~/.zshrc
```

## Install and configure PostgreSQL

Install PostgreSQL 12 and configure locales:

```
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - ; \
sudo apt update ; \
sudo apt -y install postgresql-12

# if locales are needed
sudo localedef ru_RU.UTF-8 -i ru_RU -fUTF-8 ; \
export LANGUAGE=ru_RU.UTF-8 ; \
export LANG=ru_RU.UTF-8 ; \
export LC_ALL=ru_RU.UTF-8 ; \
sudo locale-gen ru_RU.UTF-8 ; \
sudo dpkg-reconfigure locale
```

Add locales (if needed) to `/etc/profile`:

```
sudo vim /etc/profile
    export LANGUAGE=ru_RU.UTF-8
    export LANG=ru_RU.UTF-8
    export LC_ALL=ru_RU.UTF-8
```

Change `postges` password, create clear database named `DB_NAME` (test db if needed):

```
sudo passwd postgres
su - postgres
export PATH=$PATH:/usr/lib/postgresql/12/bin
createdb --encoding UNICODE DB_NAME --username postgres
exit
```

Create `DB_USER` db user and grand privileges to him:

```
sudo -u postgres psql
create user DB_USER with password 'some_password';
alter user DB_USER CREATEDB;
grant all privileges on database DB_NAME to DB_USER;
\c DB_NAME
grant all on all tables in schema public to DB_USER;
grant all on all sequences in schema public to DB_USER;
grant all on all functions in schema public to DB_USER;

# if trigram is needed
CREATE EXTENSION pg_trgm;
ALTER EXTENSION pg_trgm SET SCHEMA public;
UPDATE pg_opclass SET opcdefault = true WHERE opcname='gin_trgm_ops';

\q
```

Now can test connection. Create `~/.pgpass` with login and password to db for fast connect:

```
vim ~/.pgpass
	localhost:5432:DB_NAME:DB_USER:some_password

sudo chmod 600 ~/.pgpass
psql -h localhost -U DB_USER DB_NAME
```

Create and run SQL dump if needed:

```
pg_dump -U USER_NAME DB_NAME > dump.sql
# vim dump.sql (may want to change username or something in dump)
scp dump.sql test_server:~/

psql -h localhost DB_NAME DB_USER  < ~/dump.sql
```

## Install python 3.8

Build from source python 3.8.6, install with prefix to ~/.python folder:

```
wget https://www.python.org/ftp/python/3.8.6/Python-3.8.6.tgz ; \
tar -xzvf Python-3.8.6.tgz ; \
cd Python-3.8.6 ; \
mkdir ~/.python ; \
./configure --enable-optimizations --prefix=/home/www/.python ; \
make -j8 ; \
sudo make altinstall

cd ../ ; \
sudo rm -rf Python-3.8.6 Python-3.8.6.tgz
```

Change PATH:

```
vim ~/.zshrc
    PATH=/home/www/.python/bin:$PATH

source ~/.zshrc
```

Now python3.8 in `/home/www/.python/bin/python3.8`. Update pip:

```
sudo /home/www/.python/bin/python3.8 -m pip install -U pip
```

## Setup flask application (from github with ssh auth)

Generate new ssh key and add it to github:

```
ssh-keygen -b 3072

# if custom key name
vim ~/.ssh/config
    Host github.com
        IdentityFile ~/.ssh/github_rsa
        IdentitiesOnly yes

# if want to save passphrase
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/PRIVATE_KEY
```

Get project and setup virtual enviroment:

```
mkdir code && cd code
git clone git@github.com:dmitriyvek/PROJECT_NAME.git PROJECT_FOLDER
cd PROJECT_FOLDER
virtualenv -p /home/www/.python/bin/python3.8 env ((python3.8 -m venv env))
. ./env/bin/activate
pip install -U pip
pip install -r requirements.txt

mkdir -p log/{app,gunicorn}
```

If project is package:

```
pip install -e .
```

Setup .env file:

```
mv required_env.txt .env && vim .env
source .env
```

Setup db with migrations and init data (if no dump):

```
flask db upgrade
flask init_db_data
```

## Setup gunicorn and add it to supervisor

Create gunicorn config if needed:

```
vim config/gunicorn_config.py
    command = '/home/www/code/PROJECT_NAME/env/bin/gunicorn'
    pythonpath = '/home/www/code/PROJECT_NAME'
    bind = '127.0.0.1:8001'
    workers = 2 * CORE_NUM + 1
    user = 'www'
    limit_request_fields = 32000
    limit_request_field_size = 0
    raw_env = 'FLASK_APP=wsgi.py'
```

Create gunicorn starter script if needed:

```
mkdir bin
vim bin/start_gunicorn.sh
    #!/bin/bash
    source /home/www/code/PROJECT_NAME/env/bin/activate
    source /home/www/code/PROJECT_NAME/.env
    exec gunicorn -c "/home/www/code/PROJECT_NAME/config/gunicorn_config.py" wsgi:app

chmod +x bin/start_gunicorn.sh
```

Setup supervisor:

```
sudo vim /etc/supervisor/conf.d/PROJECT_NAME.conf
	[program:PROJECT_NAME]
	command=/home/www/code/PROJECT_NAME/bin/start_gunicorn.sh
	user=www
	process_name=%(program_name)s
	numprocs=1
	autostart=true
	autorestart=true
    stdout_logfile=/var/logs/supervisor.log
	redirect_stderr=true
    stopasgroup=true
    killasgroup=true

sudo service supervisor restart
```

## Nginx setup

Delete default config and create custom:

```
sudo rm /etc/nginx/sites-available/default
sudo vim /etc/nginx/sites-available/PROJECT_NAME
    server {
        listen [::]:443 ssl ipv6only=on; # managed by Certbot
        listen 443 ssl; # managed by Certbot
        server_name dmitriyvek.com www.dmitriyvek.com;

        ssl_certificate /etc/letsencrypt/live/dmitriyvek.com/fullchain.pem; # managed by Certbot
        ssl_certificate_key /etc/letsencrypt/live/dmitriyvek.com/privkey.pem; # managed by Certbot

        include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

        access_log /var/log/myblog_access.log;
        error_log /var/log/myblog_error.log;

        location / {
            proxy_pass http://127.0.0.1:8001;
            proxy_set_header X-Forwarded-Host $server_name;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            #proxy_set_header Referer $http_referer;
            proxy_set_header X-Forwarded-Proto $scheme;
            add_header Access-Control-Allow-Origin *;
        }
    }

    server {
        if ($host = www.dmitriyvek.com) {
            return 301 https://$host$request_uri;
        } # managed by Certbot


        if ($host = dmitriyvek.com) {
            return 301 https://$host$request_uri;
        } # managed by Certbot


        listen 80 default_server;
        listen [::]:80 default_server;

        server_name dmitriyvek.com www.dmitriyvek.com;
        return 404; # managed by Certbot
    }

sudo ln -s /etc/nginx/sites-available/PROJECT_NAME /etc/nginx/sites-enabled/
sudo certbot --nginx -d dmitriyvek.com -d www.dmitriyvek.com
sudo nginx -t
sudo nginx -s reload
sudo certbot renew --dry-run
```

Make letsencrypt keys backup:

```
cd /etc/letsencrypt && sudo tar -czvf ~/backup.tar.gz archive/ live/ renewal/ options-ssl-nginx.conf

# from remote server to local (run from local)
scp test_server:~/backup.tar.gz ~/place_on_local_machine

# from remote to another remote
scp backup.tar.gz [your-user]@[your-srever-ip]:/my/new/letsenrcypt/path

rm ~/backpup.tar.gz

# on new remote server
sudo tar -xzvf backup.tar.gz
sudo mv archive/ live/ renewal/ options-ssl-nginx.conf /etc/letsencrypt/
```

## New project version

```
git pull
sudo supervisorctl stop PROJECT_NAME
flask db upgrade
sudo supervisorctl start PROJECT_NAME
```
