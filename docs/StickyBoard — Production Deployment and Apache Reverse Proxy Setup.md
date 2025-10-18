- # StickyBoard — Unified Production Deployment and Reverse Proxy Setup

  *Last updated: 2025-10-18*

  ------

  ## 1. Overview

  This guide describes how to deploy **StickyBoard**, a `.NET 9` + `PostgreSQL` web application with an optional worker service, on a self-hosted **Ubuntu Server** using **Apache 2** as the reverse proxy.

  All code (frontend + backend + Docker setup) is deployed as a single project folder:

  ```
  /var/www/html/stickyboard.aedev.pro/
  ```

  Apache serves your static frontend at `https://stickyboard.aedev.pro`,
   while `/api/**` requests are proxied to the Dockerized backend API (`http://127.0.0.1:8080`).

  ------

  ## 2. System Prerequisites

  ### Install Required Packages

  ```
  sudo apt update
  sudo apt install -y docker docker-compose apache2 git
  ```

  ### Enable Apache Modules

  ```
  sudo a2enmod proxy
  sudo a2enmod proxy_http
  sudo a2enmod headers
  sudo a2enmod ssl
  sudo a2enmod rewrite
  sudo systemctl restart apache2
  ```

  ------

  ## 3. Project Directory Layout

  Everything for StickyBoard lives in one place:

  ```
  /var/www/html/stickyboard.aedev.pro/
  ├── frontend/                 # Static frontend (served by Apache)
  │   ├── index.html
  │   ├── assets/
  │   └── js/
  │
  ├── api/                      # .NET 9 backend (Dockerized)
  │   ├── Dockerfile
  │   ├── Program.cs
  │   └── ...
  │
  ├── worker/                   # Optional background service
  │
  ├── logs/
  │   └── api/
  │
  ├── docker-compose.yml        # Defines backend stack
  ├── .env                      # Environment variables
  └── .github/workflows/deploy.yml  # CI/CD automation
  ```

  Ownership:

  ```
  sudo chown -R www-data:www-data /var/www/html/stickyboard.aedev.pro
  ```

  ------

  ## 4. Environment Configuration

  ```
  /var/www/html/stickyboard.aedev.pro/.env
  POSTGRES_USER=stickyadmin
  POSTGRES_PASSWORD=strongpassword
  POSTGRES_DB=stickyboard
  ASPNETCORE_ENVIRONMENT=Production
  ASPNETCORE_URLS=http://+:8080
  ```

  ------

  ## 5. Docker Compose Setup

  ```
  /var/www/html/stickyboard.aedev.pro/docker-compose.yml
  version: '3.9'
  
  services:
    postgres:
      image: postgres:17
      container_name: stickyboard_db
      restart: unless-stopped
      environment:
        POSTGRES_USER: ${POSTGRES_USER}
        POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
        POSTGRES_DB: ${POSTGRES_DB}
      volumes:
        - sticky_pgdata:/var/lib/postgresql/data
      networks:
        - sticky_net
      healthcheck:
        test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
        interval: 10s
        timeout: 5s
        retries: 5
  
    api:
      build: ./api
      container_name: stickyboard_api
      depends_on:
        - postgres
      environment:
        ASPNETCORE_ENVIRONMENT: ${ASPNETCORE_ENVIRONMENT}
        ASPNETCORE_URLS: ${ASPNETCORE_URLS}
        DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      expose:
        - "8080"
      volumes:
        - ./logs/api:/app/logs
      restart: unless-stopped
      networks:
        - sticky_net
  
    worker:
      build: ./worker
      container_name: stickyboard_worker
      depends_on:
        - postgres
      environment:
        DATABASE_URL: postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      restart: unless-stopped
      networks:
        - sticky_net
  
  networks:
    sticky_net:
      driver: bridge
  
  volumes:
    sticky_pgdata:
  ```

  ------

  ## 6. Apache Configuration

  ### 6.1 HTTP (port 80)

  ```
  /etc/apache2/sites-available/stickyboard.aedev.pro.conf
  <VirtualHost *:80>
      ServerName stickyboard.aedev.pro
      ServerAdmin webmaster@aedev.pro
      DocumentRoot /var/www/html/stickyboard.aedev.pro/frontend
  
      RewriteEngine On
      RewriteCond %{HTTPS} off
      RewriteRule ^/?(.*) https://%{SERVER_NAME}/$1 [R=301,L]
  
      ErrorLog ${APACHE_LOG_DIR}/stickyboard_error.log
      CustomLog ${APACHE_LOG_DIR}/stickyboard_access.log combined
  </VirtualHost>
  ```

  Enable & reload:

  ```
  sudo a2ensite stickyboard.aedev.pro.conf
  sudo systemctl reload apache2
  ```

  ------

  ### 6.2 HTTPS (port 443) with API Proxy

  ```
  /etc/apache2/sites-available/stickyboard.aedev.pro-le-ssl.conf
  <IfModule mod_ssl.c>
  <VirtualHost *:443>
      ServerName stickyboard.aedev.pro
      ServerAdmin webmaster@aedev.pro
      DocumentRoot /var/www/html/stickyboard.aedev.pro/frontend
  
      SSLEngine on
      SSLCertificateFile /etc/letsencrypt/live/stickyboard.aedev.pro/fullchain.pem
      SSLCertificateKeyFile /etc/letsencrypt/live/stickyboard.aedev.pro/privkey.pem
      Include /etc/letsencrypt/options-ssl-apache.conf
  
      # Proxy /api requests to backend
      ProxyPreserveHost On
      ProxyRequests Off
      ProxyPass /api/ http://127.0.0.1:8080/
      ProxyPassReverse /api/ http://127.0.0.1:8080/
  
      <Directory /var/www/html/stickyboard.aedev.pro/frontend>
          Options FollowSymLinks
          AllowOverride All
          Require all granted
      </Directory>
  
      <IfModule mod_headers.c>
          Header always set X-Content-Type-Options "nosniff"
          Header always set X-Frame-Options "DENY"
          Header always set X-XSS-Protection "1; mode=block"
      </IfModule>
  
      ErrorLog ${APACHE_LOG_DIR}/stickyboard_error.log
      CustomLog ${APACHE_LOG_DIR}/stickyboard_access.log combined
  </VirtualHost>
  </IfModule>
  ```

  Reload Apache:

  ```
  sudo systemctl reload apache2
  ```

  ------

  ## 7. SSL Management (Let’s Encrypt)

  If you used the legacy client:

  ```
  sudo letsencrypt renew
  ```

  If using Certbot:

  ```
  sudo apt install certbot python3-certbot-apache
  sudo certbot --apache -d stickyboard.aedev.pro
  ```

  Automatic renewal (recommended):

  ```
  sudo crontab -e
  ```

  Add:

  ```
  0 3 * * * /usr/bin/letsencrypt renew --quiet && systemctl reload apache2
  ```

  ------

  ## 8. Deployment & CI/CD

  You can now deploy your entire repository directly to the server — Apache, Docker, and the frontend are all self-contained.

  ### 8.1 GitHub Actions Workflow

  Add `.github/workflows/deploy.yml` to your repo:

  ```
  name: Deploy StickyBoard to Production
  
  on:
    push:
      branches: [ main ]
  
  jobs:
    deploy:
      runs-on: ubuntu-latest
  
      steps:
        - name: Checkout repository
          uses: actions/checkout@v4
  
        - name: Deploy via SSH
          uses: appleboy/ssh-action@v1.1.0
          with:
            host: ${{ secrets.SERVER_HOST }}
            username: ${{ secrets.SERVER_USER }}
            key: ${{ secrets.SERVER_SSH_KEY }}
            script: |
              set -e
              echo "Deploying StickyBoard..."
              cd /var/www/html/stickyboard.aedev.pro
              sudo docker compose down || true
              sudo rm -rf *
              git clone https://github.com/${{ github.repository }} .
              sudo docker compose up -d --build
              sudo systemctl reload apache2
              echo "Deployment complete."
  ```

  ### Required repository secrets

  | Secret           | Description                           |
  | ---------------- | ------------------------------------- |
  | `SERVER_HOST`    | Your server’s IP or domain            |
  | `SERVER_USER`    | SSH user (e.g., `a3emond`)            |
  | `SERVER_SSH_KEY` | Private key with access to the server |

  ------

  ## 9. Verifying Deployment

  Check the frontend:

  ```
  curl -I https://stickyboard.aedev.pro/
  ```

  Check the API:

  ```
  curl -I https://stickyboard.aedev.pro/api/health
  ```

  You should see:

  ```
  HTTP/2 200 OK
  ```

  If not:

  ```
  sudo tail -f /var/log/apache2/stickyboard_error.log
  sudo docker logs stickyboard_api -f
  ```

  ------

  ## 10. Backup & Maintenance

  ### PostgreSQL Backups

  ```
  sudo docker exec stickyboard_db pg_dump -U stickyadmin stickyboard > /var/www/html/stickyboard.aedev.pro/db_backup_$(date +%F).sql
  ```

  ### Logs

  - Apache → `/var/log/apache2/`
  - API → `/var/www/html/stickyboard.aedev.pro/logs/api/`

  ### Restart Services

  ```
  sudo docker compose restart
  ```

  ------

  ## 11. Security Best Practices

  - Use strong database credentials.

  - Keep your OS and containers updated.

  - Allow only ports 80 and 443 in UFW:

    ```
    sudo ufw allow 80,443/tcp
    sudo ufw enable
    ```

  - Schedule regular SSL renewals and backups.

  - Use `.env` to isolate secrets from version control.

  ------

  ## 12. Result

  | Component   | Location                                       | Served by                     |
  | ----------- | ---------------------------------------------- | ----------------------------- |
  | Frontend    | `/frontend`                                    | Apache                        |
  | Backend API | `/api/` → `127.0.0.1:8080`                     | Docker (.NET 9)               |
  | Database    | `postgres:17`                                  | Docker volume `sticky_pgdata` |
  | Logs        | `/logs/api/`                                   | Host                          |
  | SSL         | `/etc/letsencrypt/live/stickyboard.aedev.pro/` | Let’s Encrypt                 |

  ------

  ### ✅ **Deployment Summary**

  1. **Clone project** → `/var/www/html/stickyboard.aedev.pro`
  2. **Run** `sudo docker compose up -d --build`
  3. **Apache** serves `/frontend` and proxies `/api/`
  4. **Let’s Encrypt** handles HTTPS automatically
  5. **GitHub Actions** deploys updates in one step