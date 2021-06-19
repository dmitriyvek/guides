# Heroku Deployment Set Up for Flask Instruction

Deploying my [flask_blog](https://github.com/dmitriyvek/flask-blog) app on Heroku service.

## Heroku cli set up

Install and login:

```
sudo snap install heroku --classic
heroku login
```

## Flask project set up

Creating heroku git repository:

```
# git init
heroku apps:create app-name --ssh-git --region eu
# git remote -v
```

Seting up ssh:

```
cd ~/.ssh/
ssh-keygen -b 3072 -f heroku_rsa

vim ~/.ssh/config
  Host herouku.com
    IdentityFile ~/.ssh/heroku_rsa
    IdentitiesOnly yes

heroku keys:add ~/.ssh/heroku_rsa.pub
# heroku keys
# heroku keys:remove dmitriy@laptop

ssh -v git@heroku.com
```

Adding postgres database:

```
heroku addons:add heroku-postgresql:hobby-dev --name unique-accross-all-apps-name
```

App must logging to stdout:

```
heroku config:set LOG_TO_STDOUT=1
```

Creating runtime.txt in root of the project:

```
vim runtime.txt
  python-3.8.10
```

Creating Procfile in root of the project:

```
vim Procfile
  web: flask db upgrade; gunicorn -b :$PORT wsgi:app
```

Seting up required envs (assuming you have a heroku.env file in a given format):

```
vim heroku.env
  key1=value1
  key2=value2
  ...

heroku config:set $(cat heroku.env | sed '/^$/d; /#[[:print:]]*$/d')
```

## Setting up nginx

Adding nginx buildpack:

```
heroku buildpacks:add heroku-community/nginx
```

Changing gunicorn config:

```
vim ./config/gunicorn_heroku_config.py
  def when_ready(server):
      # touch app-initialized when ready
      open('/tmp/app-initialized', 'w').close()

  bind = 'unix:///tmp/nginx.socket'
  workers = 3
```

Edditing Procfile:

```
web: flask db upgrade; bin/start-nginx gunicorn -c ./config/gunicorn_heroku_config.py wsgi:app
```

## Deploying app

Pushing to heroku repo and seting dyno scale:

```
git push heroku main
heroku ps:scale web=1
heroku open
```

Initializing database:

```
heroku run flask init_db_data
```

## View logs

```
heroku logs --tail
```
