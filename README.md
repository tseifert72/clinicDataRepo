
# Clinic Data Repo

How to create it on Ubuntu 22.04


## 1. Install Docker + Portainer

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

## 2. Create in Portainer Stack with Yml for Nginx Proxy Manager

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

## 3.  Yml for Prometheus

```bash
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  # external_labels:
  #  monitor: 'codelab-monitor'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9090']

  # Example job for node_exporter
  # - job_name: 'node_exporter'
  #   static_configs:
  #     - targets: ['node_exporter:9100']

  # Example job for cadvisor
  # - job_name: 'cadvisor'
  #   static_configs:
  #     - targets: ['cadvisor:8080']
```
## 4.  Yml for Grafana

```bash
---
version: '3'

volumes:
  prometheus-data:
    driver: local
  grafana-data:
    driver: local

services:
  prometheus:
    image: docker.io/prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    command: 
    volumes:
      - /etc/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    restart: unless-stopped

  grafana:
    image: docker.io/grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
    restart: unless-stopped

  node_exporter:
    image: quay.io/prometheus/node-exporter:latest
    container_name: node_exporter
    command:
      - '--path.rootfs=/host'
    network_mode: host
    pid: host
    restart: unless-stopped
    volumes:
      - '/:/host:ro,rslave'

  cadvisor:
    image: google/cadvisor/latest
    container_name: cadvisor
    #ports:
    #  - 8080:8080
    volumes:
      - /:/rootfs:ro
      - /run:/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    devices:
      - /dev/kmsg
    privileged: true
    restart: unless-stopped
```
## 5.   Yml for Keycloak

```bash
version: "3.7"

services:

  postgresql:
    image: postgres
    environment:
      - POSTGRES_USER=keycloak
      - POSTGRES_DB=keycloak
      - POSTGRES_PASSWORD=password
    volumes:
      - '/home/ubuntu/keycloak.postgresql.data:/var/lib/postgresql/data'
    healthcheck:
      test: "exit 0"
    ports:
      - "5436:5432"

  keycloak:
    container_name: keycloak
    image: quay.io/keycloak/keycloak:23.0.0

    restart: always
    command: start-dev
    depends_on:
      postgresql:
        condition: service_healthy
    environment:
      - KC_HOSTNAME=keycloak.tribal.place
      - KC_PROXY=edge
      - KC_DB=postgres
      - KC_DB_USERNAME=keycloak
      - KC_DB_PASSWORD=password
      - KC_DB_URL_HOST=postgresql
      - KC_DB_URL_PORT=5432
      - KC_DB_URL_DATABASE=keycloak
      - KEYCLOAK_ADMIN=AdminCreatedByYou
      - KEYCLOAK_ADMIN_PASSWORD=AdminPasswordCreatedByYou
    ports:
      - "8099:8080"

  oauth2proxy:
    container_name: oauth2-proxy
    image: quay.io/oauth2-proxy/oauth2-proxy:latest
    command:
      - --http-address
      - 0.0.0.0:4180
      - --code-challenge-method=S256
    environment:
        OAUTH2_PROXY_UPSTREAMS: http://localhost:8080/fhir
        OAUTH2_PROXY_COOKIE_SECRET: 'cookieSecretCreatedByYou'
        OAUTH2_PROXY_COOKIE_DOMAINS: '.tribal.place' # Required so cookie can be read on all subdomains.
        OAUTH2_PROXY_WHITELIST_DOMAINS: '.tribal.place' # Required to allow redirection back to original requested target.
        # Configure to use Keycloak
        OAUTH2_PROXY_PROVIDER: 'oidc'
        OAUTH2_PROXY_CLIENT_ID: 'oauth2-proxy'
        OAUTH2_PROXY_CLIENT_SECRET: 'keycloakClientSecretCreatedByYou'
        OAUTH2_PROXY_EMAIL_DOMAINS: '*'
        OAUTH2_PROXY_OIDC_ISSUER_URL: 'https://keycloak.tribal.place/realms/fhir'
        OAUTH2_PROXY_REDIRECT_URL: https://fhir.tribal.place/oauth2/callback 
        OAUTH2_PROXY_COOKIE_REFRESH: "1m"
    restart: unless-stopped
    depends_on:
      - keycloak
    ports:
      - '4180:4180'
```

# keycloak conf

```bash
events {}

http {
    server {
        listen 80;
        server_name keycloak.tribal.place;

        # Redirect all HTTP traffic to HTTPS
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        server_name keycloak.tribal.place;

        ssl_certificate /etc/letsencrypt/live/keycloak.tribal.place/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/keycloak.tribal.place/privkey.pem;

        location / {
            proxy_pass http://keycloak:8080; # Forward to Keycloak container
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto https;
        }
    }
}

```

## 6.  Yml for Hapi FHIR

```bash
version: "3.7"
services:
  hapi-fhir:
    container_name: hapi-fhir
    image: "hapiproject/hapi:v7.6.0"
    ports:
      - "8080:8080"
    configs:
      - source: hapi
        target: /app/config/application.yaml
    depends_on:
      - db
      
  db:
    container_name: db
    image: postgres:14.15-alpine3.21

    restart: always
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: "hapi"
      POSTGRES_USER: "userCreatedByYou"
      POSTGRES_PASSWORD: "userPasswordCreatedByYou"
    volumes:
      - /home/ubuntu/hapi.postgress.data:/var/lib/postgresql/data

configs:
  hapi:
     file: /home/ubuntu/hapi-fhir-config/application.yaml

```

#  /home/ubuntu/hapi-fhir-config/application.yaml

```bash
hapi:
  fhir:
    # Just in case we want to allow dynamic IG changes
    # Ideally they're loaded once on first startup from simplifier
    ig_runtime_upload_enabled: true
    # This should allow the $validate operator to function
    # enable_repository_validating_interceptor: true
    subscription:
      resthook_enabled: true
    implementationguides:
      hl7_fhir_r4_core:
        name: hl7.fhir.r4.core
        version: 4.0.1
        url: https://packages.simplifier.net/hl7.fhir.r4.core/4.0.1
        installMode: STORE_AND_INSTALL
        reloadExisting: false
      hl7_fhir_uv_ips:
        name: hl7.fhir.uv.ips
        version: 1.0.0
        url: https://packages.simplifier.net/hl7.fhir.uv.ips/1.0.0
        # installMode: STORE_AND_INSTALL
        reloadExisting: false
      hl7_terminology_r4:
        name: hl7.terminology.r4
        version: 5.5.0
        url: https://packages.simplifier.net/hl7.terminology.r4/5.5.0
        # installMode: STORE_AND_INSTALL
        reloadExisting: false
      ihe_mhd_fhir:
        name: ihe.mhd.fhir
        version: 4.0.1
        url: https://packages.simplifier.net/ihe.mhd.fhir/4.0.1
        # installMode: STORE_AND_INSTALL
        reloadExisting: false
      de_basisprofil_r4:
        name: de.basisprofil.r4
        version: 1.4.0
        url: https://packages.simplifier.net/de.basisprofil.r4/1.4.0
        installMode: STORE_AND_INSTALL
        reloadExisting: false
      kbv_basis:
        name: kbv.basis
        version: 1.6.0
        url: https://packages.simplifier.net/kbv.basis/1.6.0
        # installMode: STORE_AND_INSTALL
        reloadExisting: false
      de_gematik_isik-basismodul:
        name: de.gematik.isik-basismodul
        version: 2.0.6
        url: https://packages.simplifier.net/de.gematik.isik-basismodul/2.0.6
        installMode: STORE_AND_INSTALL
        reloadExisting: false
      de_medizininformatikinitiative_kerndatensatz_medikation:
        name: de.medizininformatikinitiative.kerndatensatz.medikation
        version: 1.0.10
        url: https://packages.simplifier.net/de.medizininformatikinitiative.kerndatensatz.medikation/1.0.10
        installMode: STORE_AND_INSTALL
        reloadExisting: false
      dvmd_kdl_r4_2021:
        name: dvmd.kdl.r4.2021
        version: 2021.1.1
        url: https://packages.simplifier.net/dvmd.kdl.r4.2021/2021.1.1
        installMode: STORE_AND_INSTALL
        reloadExisting: false
      de_gematik_isik-dokumentenaustausch:
        name: de.gematik.isik-dokumentenaustausch
        version: 2.0.1
        url: https://packages.simplifier.net/de.gematik.isik-dokumentenaustausch/2.0.1
        installMode: STORE_AND_INSTALL
        reloadExisting: false
      de_gematik_isik-medikation:
        name: de.gematik.isik-medikation
        version: 2.0.2
        url: https://packages.simplifier.net/de.gematik.isik-medikation/2.0.2
        installMode: STORE_AND_INSTALL
        reloadExisting: false
      de_gematik_isik-terminplanung:
        name: de.gematik.isik-terminplanung
        version: 2.0.4
        url: https://packages.simplifier.net/de.gematik.isik-terminplanung/2.0.4
        installMode: STORE_AND_INSTALL
        reloadExisting: false
      de_gematik_isik-vitalparameter:
        name: de.gematik.isik-vitalparameter
        version: 2.0.3
        url: https://packages.simplifier.net/de.gematik.isik-vitalparameter/2.0.3
        installMode: STORE_AND_INSTALL
        reloadExisting: false
      rki_demis_common:
        name: rki.demis.common
        version: 1.0.1
        url: https://packages.simplifier.net/rki.demis.common/1.0.1
        installMode: STORE_AND_INSTALL
        reloadExisting: false
      rki_demis_disease:
        name: rki.demis.disease
        version: 1.3.0
        url: https://packages.simplifier.net/rki.demis.disease/1.3.0
        installMode: STORE_AND_INSTALL
        reloadExisting: false

spring:
  datasource:
    url: 'jdbc:postgresql://db:5432/hapi'
    username: userCreatedByYou
    password: userPasswordCreatedByYou
    driverClassName: org.postgresql.Driver
  jpa:
    properties:
      hibernate.dialect: ca.uhn.fhir.jpa.model.dialect.HapiFhirPostgresDialect
      hibernate.search.enabled: true 
    hibernate:
      ddl-auto: update
```

