# Guacamole-docker-container

## SCHEMA
<img width="625" height="571" alt="image" src="https://github.com/user-attachments/assets/941b5fd2-3b8f-4cee-8b9d-7c91505b7c62" />


| Machine | Guacamole |
| --------------- | --------------- |
| CPU | 4vCPU | 
| RAM | 16GB | 
| DISK | 150GB | 
| System | Oracle Linux 8.10 | 

## 1 STEP. Install system Oracle Linux 8, update server and install docker

## 2 STEP. Download container images on Docker Desktop

```bash
docker pull mysql:8.0
docker pull guacamole/guacd:1.5.5
docker pull guacamole/guacamole:1.5.5
docker pull nginx:stable-alpine
```

## 3 STEP. Save container as tar file and copy to machine guacamole
```bash
docker save -o nginx_stable-alpine.tar nginx:stable-alpine
docker save -o guacamole_1.5.5.tar guacamole/guacamole:1.5.5
docker save -o guacd_1.5.5.tar guacamole/guacd:1.5.5
docker save -o mysql_8.0.tar mysql:8.0
```

## 4 STEP. Create directory on machine
```bash
mkdir -p /opt/guacamole/{images,mysql,ca}
```

## 5 STEP. Start doker and add to automaticaly start on reboot. Load docker images.
```bash
systemct enable --now docker
systemctl start docker
docker load -i nginx_stable-alpine.tar
docker load -i guacamole_1.5.5.tar
docker load -i guacd_1.5.5.tar
docker load -i mysql_8.0.tar
```

## 6 STEP. Create CA and server certificate
```bash
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt   -subj "/C=PL/ST=Mazowieckie/L=Warszawa/O=LocalLab/OU=IT/CN=LocalRootCA"
openssl genrsa -out guac.key 4096
openssl req -new -key guac.key -out guac.csr   -subj "/C=PL/ST=Mazowieckie/L=Warszawa/O=Lab/OU=IT/CN=guacamole.test.pl"
cat > guac.ext <<'EOF'
subjectAltName = DNS:guacamole.test.pl,IP:192.168.100.65
extendedKeyUsage = serverAuth
EOF
openssl x509 -req -in guac.csr -CA ca.crt -CAkey ca.key -CAcreateserial   -out guac.crt -days 825 -sha256 -extfile guac.ext
```

## 7 STEP. Initialize database mysql
```bash
docker run --rm guacamole/guacamole:1.5.5 /opt/guacamole/bin/initdb.sh --mysql > initdb.sql
docker exec -i guacamole-db mysql -uguacuser -pPa552025! guacadb < initdb.sql
```

## 8 STEP. Create docker-compose.yml and change password and ldap attribute
```bash
version: '3.9'

services:
  guacamole-db:
    image: mysql:8.0
    container_name: guacamole-db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: Passw0rd
      MYSQL_DATABASE: guacadb
      MYSQL_USER: guacuser
      MYSQL_PASSWORD: Passw0rd
    volumes:
      - ./mysql:/var/lib/mysql

  guacd:
    image: guacamole/guacd:1.5.5
    container_name: guacd
    restart: always

  guacamole:
    image: guacamole/guacamole:1.5.5
    container_name: guacamole
    restart: always
    environment:
      GUACD_HOSTNAME: guacd
      MYSQL_HOSTNAME: guacamole-db
      MYSQL_DATABASE: guacadb
      MYSQL_USER: guacuser
      MYSQL_PASSWORD: Passw0rd

      # ===== LDAP / Active Directory =====
      LDAP_HOSTNAME: 192.168.100.10
      LDAP_PORT: 389
      LDAP_ENCRYPTION_METHOD: none        # options: none | starttls | ssl
      LDAP_USER_BASE_DN: "OU=USERS,DC=TEST,DC=PL"
      LDAP_GROUP_BASE_DN: "OU=06_Security_Groups,OU=USERS,DC=TEST,DC=PL"
      LDAP_MEMBER_ATTRIBUTE: "member"
      LDAP_GROUP_NAME_ATTRIBUTE: "cn"
      LDAP_USERNAME_ATTRIBUTE: sAMAccountName
      LDAP_SEARCH_BIND_DN: "CN=guacamole,OU=07_Service_Accounts,OU=USERS,DC=TEST,DC=PL"
      LDAP_SEARCH_BIND_PASSWORD: "Passw0rd
    depends_on:
      - guacd
      - guacamole-db

  nginx:
    image: nginx:stable-alpine
    container_name: guac-nginx
    restart: always
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./ca/guac.crt:/etc/ssl/certs/guac.crt:ro
      - ./ca/guac.key:/etc/ssl/private/guac.key:ro
    depends_on:
      - guacamole
```

## 9 STEP. Create nginx.conf
```bash
server {
    listen 443 ssl http2;
    server_name guacamole.test.pl;

    ssl_certificate     /etc/ssl/certs/guac.crt;
    ssl_certificate_key /etc/ssl/private/guac.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    proxy_read_timeout 3600;
    proxy_send_timeout 3600;

    location / {
        proxy_pass http://guacamole:8080/guacamole/;
        proxy_http_version 1.1;
        proxy_buffering off;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $http_connection;
    }
}
````
## 10 STEP. Start all docker image
```bash
cd /opt/guacamole
docker compose up -d
```

## STEP 11. Check docker status
```bash
docker ps
```

## 12 STEP. Login on website Guacamole

<img width="1020" height="730" alt="image" src="https://github.com/user-attachments/assets/04da7f54-d538-4553-afbd-0aec4b07bd59" />



