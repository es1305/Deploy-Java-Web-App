# Деплой приложений. Java

## 1. Установка Oracle JDK 8

Скачиваем [нужную версию JDK](https://www.oracle.com/java/technologies/downloads/#java8) и копируем на удалённый хост:

```bash
scp jdk-8u311-linux-x64.tar.gz root@es1305-www-1.devops.rebrain.srwx.net:~
```

На целевой ВМ:

```bash
mkdir /usr/lib/jvm 
cd /usr/lib/jvm 
tar -xvzf ~/jdk-8u311-linux-x64.tar.gz
nano /etc/environment
```

```bash
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/usr/lib/jvm/jdk1.8.0_311/bin:/usr/lib/jvm/jdk1.8.0_311/db/bin:/usr/lib/jvm/jdk1.8.0_311/jre/bin"
J2SDKDIR="/usr/lib/jvm/jdk1.8.0_311"
J2REDIR="/usr/lib/jvm/jdk1.8.0_311/jre"
JAVA_HOME="/usr/lib/jvm/jdk1.8.0_311"
DERBY_HOME="/usr/lib/jvm/jdk1.8.0_311/db"
```

```bash
update-alternatives --install "/usr/bin/java" "java" "/usr/lib/jvm/jdk1.8.0_311/bin/java" 0
update-alternatives --install "/usr/bin/javac" "javac" "/usr/lib/jvm/jdk1.8.0_311/bin/javac" 0
update-alternatives --set java /usr/lib/jvm/jdk1.8.0_311/bin/java
update-alternatives --set javac /usr/lib/jvm/jdk1.8.0_311/bin/javac
```

Перелогинимся для применения переменных окружения.

## 2. Установка Apache Maven

```bash
cd /opt
wget https://downloads.apache.org/maven/maven-3/3.8.4/binaries/apache-maven-3.8.4-bin.tar.gz
tar -xzvf apache-maven-3.8.4-bin.tar.gz
rm apache-maven-3.8.4-bin.tar.gz
nano ~/.bashrc
```

```bash
export PATH=/opt/apache-maven-3.8.4/bin:$PATH
```

```bash
source ~/.bashrc
```

## 3. Сборка пакета

```bash
cd ~
git clone https://github.com/otale/tale
cd tale/
mvn package
mvn install
```

## 4. Запуск пакета

```bash
apt install sqlite3 nginx
mkdir /var/www/tale
cd /var/www/tale
tar -xzvf ~/tale/target/dist/tale.tar.gz
chmod +x tool
./tool start
```

```bash
Starting tale ...
(pid=5045) [OK]
```

```bash
./tool stop
```

## 5. Настройка проксирования

### 5.1. HTTP

```bash
nano /etc/nginx/sites-available/es1305-www-1.devops.rebrain.srwx.net
```

```bash
server {
  listen 80;
  server_name es1305-www-1.devops.rebrain.srwx.net;
  access_log off;

  location / {
    proxy_pass http://127.0.0.1:9000;
    proxy_read_timeout 300;
    proxy_connect_timeout 300;
    proxy_redirect     off;

    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_set_header   Host              $http_host;
    proxy_set_header   X-Real-IP         $remote_addr;
  }

}
```

```bash
rm /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/es1305-www-1.devops.rebrain.srwx.net /etc/nginx/sites-enabled/
nginx -t
nginx -s reload
```

### 5.2. HTTPS

```bash
apt install certbot python3-certbot-nginx
certbot --nginx -d es1305-www-1.devops.rebrain.srwx.net

nano /etc/nginx/sites-available/es1305-www-1.devops.rebrain.srwx.net
```

```bash
server {
  listen 443 ssl http2;
  server_name es1305-www-1.devops.rebrain.srwx.net;
  access_log off;

  location / {
    proxy_pass http://127.0.0.1:9000;
    proxy_read_timeout 300;
    proxy_connect_timeout 300;
    proxy_redirect     off;

    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_set_header   Host              $http_host;
    proxy_set_header   X-Real-IP         $remote_addr;
  }

    ssl_certificate /etc/letsencrypt/live/es1305-www-1.devops.rebrain.srwx.net/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/es1305-www-1.devops.rebrain.srwx.net/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
  listen 80;
  server_name es1305-www-1.devops.rebrain.srwx.net;
    if ($host = es1305-www-1.devops.rebrain.srwx.net) {
        return 301 https://$host$request_uri;
    }
}
```

```bash
nginx -t
nginx -s reload
cd /var/www/tale
./tool start
```
