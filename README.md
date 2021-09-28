# otus-hw-ci-cd

## digital-ocean
login through ssh
``` 
ssh root@138.68.85.72
```

add user githubuser
```
adduser githubuser
usermod -aG sudo githubuser
```

переключимся на нового пользователя
``` 
su githubuser
```

отныне можно другим юзером коннектиться
``` 
ssh githubuser@138.68.85.72
```

```shell
# настроим базовй брэндмаузер
ufw app list
ufw allow OpenSSH
ufw enable -y
ufw status
#>> Firewall is active and enabled on system startup

# from SCRIPT.md for lesson 32
su githubuser
sudo apt update

sudo apt install curl git unzip nginx postgresql postgresql-contrib rabbitmq-server supervisor \
php-cli php-fpm php-json php-common php-mysql php-zip php-gd php-mbstring php-curl php-xml php-pear php-bcmath \
php-pgsql -y

exit # switch to root
curl -sS https://getcomposer.org/installer -o composer-setup.php

su githubuser

sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer

#exit # switch to root
# !!! везде ответ: could not change directory to "/root": Permission denied 
# solution: cd
sudo -u postgres bash -c "psql -c \"CREATE DATABASE twitter ENCODING 'UTF8' TEMPLATE = template0\""
sudo -u postgres bash -c "psql -c \"CREATE USER my_user WITH PASSWORD '1H8a61ceQW7htGRE6iVz'\""
sudo -u postgres bash -c "psql -c \"GRANT ALL PRIVILEGES ON DATABASE twitter TO my_user\""

sudo apt install mc
#mcedit /etc/postgresql/12/main/pg_hba.conf
sudo -u postgres bash -c "mcedit /etc/postgresql/12/main/pg_hba.conf"
#> host    all<--->        all             0.0.0.0/0               md5
#> host    all<--->        all             ::/0                    md5

#mcedit /etc/postgresql/12/main/postgresql.conf
sudo -u postgres bash -c "mcedit /etc/postgresql/12/main/postgresql.conf"
находим закомментированную строку с параметром
        `listen_addresses` и заменяем её на
            ```
            listen_addresses='*'
            ```
        1. Перезапускаем сервис `postgresql` командой `sudo service postgresql restart`
    1. Проверяем, что по порту 5432 можем попасть в БД twitter с реквизитами my_user / 1H8a61ceQW7htGRE6iVz
    1. Конфигурируем RabbitMQ командами (по одной команде за раз)
        ```shell script
        sudo rabbitmq-plugins enable rabbitmq_management 
        sudo rabbitmq-plugins enable rabbitmq_consistent_hash_exchange
        sudo rabbitmqctl add_user my_user T1y04lWk167MkyEK3YFk
        sudo rabbitmqctl set_user_tags my_user administrator
        sudo rabbitmqctl set_permissions -p / my_user ".*" ".*" ".*"
        ```
    1. Проверяем, что по порту 15672 можем залогиниться с реквизитами my_user / T1y04lWk167MkyEK3YFk
    1. В файле `/etc/nginx/nginx.conf` исправляем строку (актуально для AWS EC2)
        ```
        server_name_hash_bucket_size 128;
        ```
    1. В файл `/etc/sudoers` добавляем строку
        ```
        www-data ALL=(ALL) NOPASSWD:ALL
        ```
    1. Дадим права на работу с каталогом `var/www` всем командой `sudo chmod 777 /var/www`
1. В репозитории в GitLab
    1. заходим в раздел `Settings -> CI / CD` и добавляем переменные окружения
        1. `SERVER1` - адрес сервера
        1. `SSH_USER` - имя пользователя для входа по ssh (для AWS EC2 - `ubuntu`)
        1. `SSH_PRIVATE_KEY` - приватный ключ, закодированный в base64
        1. `DATABASE_HOST` - `localhost`
        1. `DATABASE_NAME` - `twitter`
        1. `DATABASE_USER` - `my_user`
        1. `DATABASE_PASSWORD` - `1H8a61ceQW7htGRE6iVz`
        1. `RABBITMQ_HOST` - `localhost`
        1. `RABBITMQ_USER` - `my_user`
        1. `RABBITMQ_PASSWORD` - `T1y04lWk167MkyEK3YFk`
    1. заходим в раздел `Settings -> Repository` и добавляем deploy token с правами `read_repository`, сохраняем пароль
1. В репозитории создаём файл `deploy/nginx.conf`
    ```
    server {
        listen 80;
    
        server_name %SERVER_NAME%;
        error_log  /var/log/nginx/error.log;
        access_log /var/log/nginx/access.log;
        root /var/www/demo/public;
    
        rewrite ^/index\.php/?(.*)$ /$1 permanent;
    
        try_files $uri @rewriteapp;
    
        location @rewriteapp {
            rewrite ^(.*)$ /index.php/$1 last;
        }
    
        # Deny all . files
        location ~ /\. {
            deny all;
        }
    
        location ~ ^/index\.php(/|$) {
            fastcgi_split_path_info ^(.+\.php)(/.*)$;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
            fastcgi_index index.php;
            send_timeout 1800;
            fastcgi_read_timeout 1800;
            fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        }
    }
    ```
1. Создаём файл `deploy/supervisor.conf`
    ```
    [program:add_follower]
    command=sudo -u www-data php /var/www/demo/bin/console rabbitmq:consumer -m 1000 add_follower --env=prod -vv
    process_name=add_follower_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/var/log/supervisor.add_follower.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/var/log/supervisor.add_follower.error.log
    stderr_capture_maxbytes=1MB
    
    [program:publish_tweet]
    command=sudo -u www-data php /var/www/demo/bin/console rabbitmq:consumer -m 1000 publish_tweet --env=prod -vv
    process_name=publish_tweet_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/var/log/supervisor.publish_tweet.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/var/log/supervisor.publish_tweet.error.log
    stderr_capture_maxbytes=1MB
    
    [program:send_notification_email]
    command=sudo -u www-data php /var/www/demo/bin/console rabbitmq:consumer -m 1000 send_notification.email --env=prod -vv
    process_name=send_notification_email_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/var/log/supervisor.send_notification_email.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/var/log/supervisor.send_notification_email.error.log
    stderr_capture_maxbytes=1MB
    
    [program:send_notification_sms]
    command=sudo -u www-data php /var/www/demo/bin/console rabbitmq:consumer -m 1000 send_notification.sms --env=prod -vv
    process_name=send_notification_sms_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/var/log/supervisor.send_notification_sms.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/var/log/supervisor.send_notification_sms.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_0]
    command=sudo -u www-data php /var/www/demo/bin/console rabbitmq:consumer -m 1000 update_feed_0 --env=prod -vv
    process_name=update_feed_0_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_1]
    command=sudo -u www-data php /var/www/demo/bin/console rabbitmq:consumer -m 1000 update_feed_1 --env=prod -vv
    process_name=update_feed_1_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_2]
    command=sudo -u www-data php /var/www/demo/bin/console rabbitmq:consumer -m 1000 update_feed_2 --env=prod -vv
    process_name=update_feed_2_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_3]
    command=sudo -u www-data php /var/www/demo/bin/console rabbitmq:consumer -m 1000 update_feed_3 --env=prod -vv
    process_name=update_feed_3_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_4]
    command=sudo -u www-data php /var/www/demo/bin/console rabbitmq:consumer -m 1000 update_feed_4 --env=prod -vv
    process_name=update_feed_4_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_5]
    command=sudo -u www-data php /var/www/demo/bin/console rabbitmq:consumer -m 1000 update_feed_5 --env=prod -vv
    process_name=update_feed_5_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_6]
    command=sudo -u www-data php /var/www/demo/bin/console rabbitmq:consumer -m 1000 update_feed_6 --env=prod -vv
    process_name=update_feed_6_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_7]
    command=sudo -u www-data php /var/www/demo/bin/console rabbitmq:consumer -m 1000 update_feed_7 --env=prod -vv
    process_name=update_feed_7_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_8]
    command=sudo -u www-data php /var/www/demo/bin/console rabbitmq:consumer -m 1000 update_feed_8 --env=prod -vv
    process_name=update_feed_8_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_9]
    command=sudo -u www-data php /var/www/demo/bin/console rabbitmq:consumer -m 1000 update_feed_9 --env=prod -vv
    process_name=update_feed_9_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    ```
1. Исправляем файл `.env`
    ```shell script
    # In all environments, the following files are loaded if they exist,
    # the latter taking precedence over the former:
    #
    #  * .env                contains default values for the environment variables needed by the app
    #  * .env.local          uncommitted file with local overrides
    #  * .env.$APP_ENV       committed environment-specific defaults
    #  * .env.$APP_ENV.local uncommitted environment-specific overrides
    #
    # Real environment variables win over .env files.
    #
    # DO NOT DEFINE PRODUCTION SECRETS IN THIS FILE NOR IN ANY OTHER COMMITTED FILES.
    #
    # Run "composer dump-env prod" to compile .env files for production use (requires symfony/flex >=1.2).
    # https://symfony.com/doc/current/best_practices.html#use-environment-variables-for-infrastructure-configuration
    
    ###> symfony/framework-bundle ###
    SHELL_VERBOSITY=-1
    APP_ENV=dev
    APP_SECRET=1f68e41015fad847398c75c74318f646
    #TRUSTED_PROXIES=127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
    #TRUSTED_HOSTS='^(localhost|example\.com)$'
    ###< symfony/framework-bundle ###
    
    ###> doctrine/doctrine-bundle ###
    # Format described at https://www.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/configuration.html#connecting-using-a-url
    # For an SQLite database, use: "sqlite:///%kernel.project_dir%/var/data.db"
    # For a PostgreSQL database, use: "postgresql://db_user:db_password@127.0.0.1:5432/db_name?serverVersion=11&charset=utf8"
    # IMPORTANT: You MUST configure your server version, either here or in config/packages/doctrine.yaml
    DATABASE_URL=postgresql://%DATABASE_USER%:%DATABASE_PASSWORD%@%DATABASE_HOST%:5432/%DATABASE_NAME%?serverVersion=11&charset=utf8
    ###< doctrine/doctrine-bundle ###
    
    ###> sentry/sentry-symfony ###
    SENTRY_DSN=http://974245dd19534d84b9bfac579914df26@sentry:9000/2
    ###< sentry/sentry-symfony ###
    
    RABBITMQ_URL=amqp://%RABBITMQ_USER%:%RABBITMQ_PASSWORD%@%RABBITMQ_HOST%:5672
    ```
1. Создаём файл `deploy.sh`
    ```shell script
    sudo cp deploy/nginx.conf /etc/nginx/conf.d/demo.conf -f
    sudo cp deploy/supervisor.conf /etc/supervisor/conf.d/demo.conf -f
    sudo sed -i -- "s|%SERVER_NAME%|$1|g" /etc/nginx/conf.d/demo.conf
    sudo service nginx restart
    sudo -u www-data composer install -q
    sudo service php7.4-fpm restart
    sudo -u www-data sed -i -- "s|%DATABASE_HOST%|$2|g" .env
    sudo -u www-data sed -i -- "s|%DATABASE_USER%|$3|g" .env
    sudo -u www-data sed -i -- "s|%DATABASE_PASSWORD%|$4|g" .env
    sudo -u www-data sed -i -- "s|%DATABASE_NAME%|$5|g" .env
    sudo -u www-data php bin/console doctrine:migrations:migrate --no-interaction
    sudo -u www-data sed -i -- "s|%RABBITMQ_HOST%|$6|g" .env
    sudo -u www-data sed -i -- "s|%RABBITMQ_USER%|$7|g" .env
    sudo -u www-data sed -i -- "s|%RABBITMQ_PASSWORD%|$8|g" .env
    sudo service supervisor restart
    ```
1. Создаём файл `.gitlab-ci.yml` (не забудьте указать корректный путь к репозиторию в git clone и реквизиты доступа)
    ```yml
    before_script:
      - apt-get update -qq
      - apt-get install -qq git
      - 'which ssh-agent || ( apt-get install -qq openssh-client )'
      - eval $(ssh-agent -s)
      - ssh-add <(echo "$SSH_PRIVATE_KEY" | base64 -d)
      - mkdir -p ~/.ssh
      - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
    
    deploy_server1:
      stage: deploy
      environment:
        name: server1
        url: $SERVER1
      script:
        - ssh $SSH_USER@$SERVER1 "sudo rm -rf /var/www/demo &&
              cd /var/www &&
              git clone https://deploy:ZsZAGdePfH2QCoBYQkh2@gitlab.com/raptor-mvk/deploy-test.git demo &&
              sudo chown www-data:www-data demo -R &&
              cd demo &&
              sh ./deploy.sh $SERVER1 $DATABASE_HOST $DATABASE_USER $DATABASE_PASSWORD $DATABASE_NAME $RABBITMQ_HOST $RABBITMQ_USER $RABBITMQ_PASSWORD"
      only:
        - master
    ```
1. Добавляем код в master-ветку и пушим в GitLab
1. В репозитории в GitLab в разделе `CI / CD -> Pipelines` можно следить за процессом
1. Проверяем в интерфейсе RabbitMQ, что консьюмеры запустились
1. Проверяем, что API отвечает POST-запросом на `/api/v1/user` с basic auth авторизацией admin / my_pass и телом
form-data вида
    ```
    login:admin
    email:admin@nomail.com
    phone:+1111111111
    preferEmail:1
    ```
1. Проверяем, что запись появилась в БД
1. Заходим на виртуалку и вносим изменения:
    ```shell script
    sudo rm -rf /var/www/demo/*
    sudo mkdir -p /var/www/demo/shared/log
    sudo chmod 777 /var/www/demo -R 
    ```
1. Исправляем файл `deploy/nginx.conf`
    ```
    server {
        listen 80;
    
        server_name %SERVER_NAME%;
        error_log  /var/log/nginx/error.log;
        access_log /var/log/nginx/access.log;
        root /var/www/demo/current/public;
    
        rewrite ^/index\.php/?(.*)$ /$1 permanent;
    
        try_files $uri @rewriteapp;
    
        location @rewriteapp {
            rewrite ^(.*)$ /index.php/$1 last;
        }
    
        # Deny all . files
        location ~ /\. {
            deny all;
        }
    
        location ~ ^/index\.php(/|$) {
            fastcgi_split_path_info ^(.+\.php)(/.*)$;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
            fastcgi_index index.php;
            send_timeout 1800;
            fastcgi_read_timeout 1800;
            fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
        }
    }
    ```
1. Исправлям файл `deploy/supervisor.conf`
    ```
    [program:add_follower]
    command=sudo -u www-data php /var/www/demo/current/bin/console rabbitmq:consumer -m 1000 add_follower --env=prod -vv
    process_name=add_follower_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/current/var/log/supervisor.add_follower.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/current/var/log/supervisor.add_follower.error.log
    stderr_capture_maxbytes=1MB
    
    [program:publish_tweet]
    command=sudo -u www-data php /var/www/demo/current/bin/console rabbitmq:consumer -m 1000 publish_tweet --env=prod -vv
    process_name=publish_tweet_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/current/var/log/supervisor.publish_tweet.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/current/var/log/supervisor.publish_tweet.error.log
    stderr_capture_maxbytes=1MB
    
    [program:send_notification_email]
    command=sudo -u www-data php /var/www/demo/current/bin/console rabbitmq:consumer -m 1000 send_notification.email --env=prod -vv
    process_name=send_notification_email_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/current/var/log/supervisor.send_notification_email.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/current/var/log/supervisor.send_notification_email.error.log
    stderr_capture_maxbytes=1MB
    
    [program:send_notification_sms]
    command=sudo -u www-data php /var/www/demo/current/bin/console rabbitmq:consumer -m 1000 send_notification.sms --env=prod -vv
    process_name=send_notification_sms_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/current/var/log/supervisor.send_notification_sms.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/current/var/log/supervisor.send_notification_sms.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_0]
    command=sudo -u www-data php /var/www/demo/current/bin/console rabbitmq:consumer -m 1000 update_feed_0 --env=prod -vv
    process_name=update_feed_0_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/current/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/current/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_1]
    command=sudo -u www-data php /var/www/demo/current/bin/console rabbitmq:consumer -m 1000 update_feed_1 --env=prod -vv
    process_name=update_feed_1_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/current/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/current/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_2]
    command=sudo -u www-data php /var/www/demo/current/bin/console rabbitmq:consumer -m 1000 update_feed_2 --env=prod -vv
    process_name=update_feed_2_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/current/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/current/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_3]
    command=sudo -u www-data php /var/www/demo/current/bin/console rabbitmq:consumer -m 1000 update_feed_3 --env=prod -vv
    process_name=update_feed_3_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/current/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/current/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_4]
    command=sudo -u www-data php /var/www/demo/current/bin/console rabbitmq:consumer -m 1000 update_feed_4 --env=prod -vv
    process_name=update_feed_4_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/current/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/current/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_5]
    command=sudo -u www-data php /var/www/demo/current/bin/console rabbitmq:consumer -m 1000 update_feed_5 --env=prod -vv
    process_name=update_feed_5_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/current/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/current/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_6]
    command=sudo -u www-data php /var/www/demo/current/bin/console rabbitmq:consumer -m 1000 update_feed_6 --env=prod -vv
    process_name=update_feed_6_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/current/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/current/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_7]
    command=sudo -u www-data php /var/www/demo/current/bin/console rabbitmq:consumer -m 1000 update_feed_7 --env=prod -vv
    process_name=update_feed_7_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/current/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/current/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_8]
    command=sudo -u www-data php /var/www/demo/current/bin/console rabbitmq:consumer -m 1000 update_feed_8 --env=prod -vv
    process_name=update_feed_8_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/current/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/current/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    
    [program:update_feed_9]
    command=sudo -u www-data php /var/www/demo/current/bin/console rabbitmq:consumer -m 1000 update_feed_9 --env=prod -vv
    process_name=update_feed_9_%(process_num)02d
    numprocs=1
    directory=/tmp
    autostart=true
    autorestart=true
    startsecs=3
    startretries=10
    user=www-data
    redirect_stderr=false
    stdout_logfile=/var/www/demo/current/var/log/supervisor.update_feed.out.log
    stdout_capture_maxbytes=1MB
    stderr_logfile=/var/www/demo/current/var/log/supervisor.update_feed.error.log
    stderr_capture_maxbytes=1MB
    ```
1. Исправляем `.gitlab-ci.yml` (не забудьте указать корректный путь к репозиторию в git clone и реквизиты доступа)
    ```yaml
    before_script:
      - apt-get update -qq
      - apt-get install -qq git
      - 'which ssh-agent || ( apt-get install -qq openssh-client )'
      - eval $(ssh-agent -s)
      - ssh-add <(echo "$SSH_PRIVATE_KEY" | base64 -d)
      - mkdir -p ~/.ssh
      - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
      - export DIR=$(date +%Y%m%d_%H%M%S)
    
    deploy_server1:
      stage: deploy
      environment:
        name: server1
        url: $SERVER1
      script:
        - ssh $SSH_USER@$SERVER1 "cd /var/www/demo &&
              git clone https://deploy:ZsZAGdePfH2QCoBYQkh2@gitlab.com/raptor-mvk/deploy-test.git $DIR &&
              sudo chown www-data:www-data $DIR -R &&
              cd $DIR &&
              sh ./deploy.sh $SERVER1 $DATABASE_HOST $DATABASE_USER $DATABASE_PASSWORD $DATABASE_NAME $RABBITMQ_HOST $RABBITMQ_USER $RABBITMQ_PASSWORD &&
              cd .. &&
              rm -rf /var/www/demo/$DIR/var/log &&
              ln -s /var/www/demo/shared/log /var/www/demo/$DIR/var/log &&
              ( [ ! -d /var/www/demo/current ] || mv -Tf /var/www/demo/current /var/www/demo/previous ) &&
              ln -s /var/www/demo/$DIR /var/www/demo/current"
      only:
        - master
    ```
1. Пушим код в репозиторий
1. Добавляем файл `rollback.sh`
    ```shell script
    sudo cp deploy/nginx.conf /etc/nginx/conf.d/demo.conf -f
    sudo cp deploy/supervisor.conf /etc/supervisor/conf.d/demo.conf -f
    sudo sed -i -- "s|%SERVER_NAME%|$1|g" /etc/nginx/conf.d/demo.conf
    sudo service nginx restart
    sudo service php7.4-fpm restart
    sudo -u www-data php bin/console cache:clear
    sudo service supervisor restart
    ```
1. Ещё раз исправляем `.gitlab-ci.yml` (не забудьте указать корректный путь к репозиторию в git clone и реквизиты доступа)
    ```yaml
    stages:
      - deploy
      - rollback

    before_script:
      - apt-get update -qq
      - apt-get install -qq git
      - 'which ssh-agent || ( apt-get install -qq openssh-client )'
      - eval $(ssh-agent -s)
      - ssh-add <(echo "$SSH_PRIVATE_KEY" | base64 -d)
      - mkdir -p ~/.ssh
      - echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config
      - export DIR=$(date +%Y%m%d_%H%M%S)
    
    deploy_server1:
      stage: deploy
      environment:
        name: server1
        url: $SERVER1
      script:
        - ssh $SSH_USER@$SERVER1 "cd /var/www/demo &&
              git clone https://deploy:ZsZAGdePfH2QCoBYQkh2@gitlab.com/raptor-mvk/deploy-test.git $DIR &&
              sudo chown www-data:www-data $DIR -R &&
              cd $DIR &&
              sh ./deploy.sh $SERVER1 $DATABASE_HOST $DATABASE_USER $DATABASE_PASSWORD $DATABASE_NAME $RABBITMQ_HOST $RABBITMQ_USER $RABBITMQ_PASSWORD
              cd .. &&
              rm -rf /var/www/demo/$DIR/var/log &&
              ln -s /var/www/demo/shared/log /var/www/demo/$DIR/var/log &&
              ( [ ! -d /var/www/demo/current ] || mv -Tf /var/www/demo/current /var/www/demo/previous ) &&
              ln -s /var/www/demo/$DIR /var/www/demo/current"
      only:
        - master
   
    rollback:
      stage: rollback
      script:
        - ssh $SSH_USER@$SERVER1 "unlink /var/www/demo/current &&
              mv -Tf /var/www/demo/previous /var/www/demo/current &&
              cd /var/www/demo/current &&
              sh ./rollback.sh $SERVER1"
      when: manual
    ```

```