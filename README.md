
# Clinic Data Repo

How to create it on Ubuntu 22.04


## Installation Docker / Portainer

Install with

```bash
  apt update
  apt install docker.io -y
  systemctl start docker
  docker pull portainer/portainer-ce:latest
  docker run -d -p 9000:9000 --restart always -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer-ce:latest
```
    
## Portainer User Interface

`http://server.ip:9000`

## Create Stack with Yml for Nginx Proxy Manager

```bash
version: '3'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    environment:
      DB_MYSQL_HOST: "db"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "npm"
      DB_MYSQL_PASSWORD: "npm"
      DB_MYSQL_NAME: "npm"
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
  db:
    image: 'jc21/mariadb-aria:latest'
    environment:
      MYSQL_ROOT_PASSWORD: 'npm'
      MYSQL_DATABASE: 'npm'
      MYSQL_USER: 'npm'
      MYSQL_PASSWORD: 'npm'
    volumes:
      - ./mysql:/var/lib/mysql
```

Nginx Proxy Manager User Interface on http://server.ip:81/
- first password Nginx Proxy Manager :: User: admin@example.com pswd: changeme
