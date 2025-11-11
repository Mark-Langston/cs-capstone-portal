# CS Capstone Programming Portal

An educational, student-built portal that demonstrates:

- Interactive code execution via a **hosted Judge0** instance
- A modern, responsive UI (dark theme) for teaching programming concepts
- Real-world DevOps: Docker, reverse proxying, SSL, VPS management

Live site:

- `https://cs-capstone.com`

Repo:

- `https://github.com/Mark-Langston/cs-capstone-portal`


---

## Features

- **Home Page (`index.html`)**
  - Overview of the Capstone project
  - Links to the C++ demo and GitHub repo
  - Styled with a shared `style.css` for consistent theming

- **C++ Hello World Demo (`cpp-hello-world.html`)**
  - In-browser editable C++ example
  - Sends submissions to a **hosted Judge0** API via `/judge0`
  - Displays stdout / errors with a clean, student-friendly interface

- **Judge0 Integration**
  - Judge0 runs in Docker on the same VPS
  - Apache proxies requests:
    - `https://cs-capstone.com/judge0` → Judge0 API (`http://127.0.0.1:2358`)


---

<details>
<summary><strong>Setup Guide: Deploying on an Ubuntu VPS (Full Stack)</strong></summary>

### 0. Prerequisites

- Ubuntu Server (e.g. 22.04 LTS)
- A domain name pointing to your VPS IP  
  - Example: `cs-capstone.com`
- Basic SSH access:
  ```bash
  ssh youruser@your-vps-ip
  ```

---

### 1. Update System

```bash
sudo apt update && sudo apt upgrade -y
```

---

### 2. Install Required Packages

```bash
sudo apt install -y   apache2   docker.io   docker-compose-plugin   git   curl
```

Enable and start Apache + Docker:

```bash
sudo systemctl enable apache2 docker
sudo systemctl start apache2 docker
```

---

### 3. Clone or Create the Web Root

```bash
sudo mkdir -p /var/www/cs-capstone
sudo chown -R $USER:$USER /var/www/cs-capstone
cd /var/www/cs-capstone

# Option A: clone your repo
git clone https://github.com/Mark-Langston/cs-capstone-portal.git .
```

Your key files here (example):

- `/var/www/cs-capstone/index.html`
- `/var/www/cs-capstone/cpp-hello-world.html`
- `/var/www/cs-capstone/style.css`

---

### 4. Create the Shared Stylesheet (`style.css`)

If not already present:

```bash
cd /var/www/cs-capstone
nano style.css
```

Paste the shared CSS used by `index.html` and `cpp-hello-world.html` (the same stylesheet currently referenced in those files).

Save and exit.

---

### 5. Configure Apache VirtualHost + Proxy for Judge0

Enable required Apache modules:

```bash
sudo a2enmod proxy proxy_http ssl rewrite headers
```

Create or edit the site config:

```bash
sudo nano /etc/apache2/sites-available/cs-capstone.conf
```

Example (HTTP → HTTPS redirect + HTTPS vhost):

```apacheconf
<VirtualHost *:80>
    ServerName cs-capstone.com
    ServerAlias www.cs-capstone.com

    RewriteEngine on
    RewriteCond %{HTTPS} off
    RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [L,R=301]
</VirtualHost>

<VirtualHost *:443>
    ServerName cs-capstone.com
    ServerAlias www.cs-capstone.com

    DocumentRoot /var/www/cs-capstone

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/cs-capstone.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/cs-capstone.com/privkey.pem
    Include /etc/letsencrypt/options-ssl-apache.conf

    <Directory /var/www/cs-capstone>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Proxy /judge0 to local Judge0 server
    ProxyPreserveHost On
    ProxyPass        /judge0/  http://127.0.0.1:2358/
    ProxyPassReverse /judge0/  http://127.0.0.1:2358/

    ErrorLog ${APACHE_LOG_DIR}/cs-capstone-error.log
    CustomLog ${APACHE_LOG_DIR}/cs-capstone-access.log combined
</VirtualHost>
```

Disable the default site and enable the new one:

```bash
sudo a2dissite 000-default.conf
sudo a2ensite cs-capstone.conf
sudo systemctl reload apache2
```

🔐 Get SSL certs (if not done already) with Certbot (example using Apache plugin).

---

### 6. Setup Judge0 in `/opt/judge0`

Create directory:

```bash
sudo mkdir -p /opt/judge0/box
sudo chown -R root:root /opt/judge0
sudo chmod 777 /opt/judge0/box

cd /opt/judge0
```

Create `judge0.conf`:

```bash
sudo nano judge0.conf
```

Example:

```bash
# Database
POSTGRES_HOST=db
POSTGRES_PORT=5432
POSTGRES_DB=judge0
POSTGRES_USER=judge0
POSTGRES_PASSWORD=CHANGEME_db_password

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=CHANGEME_redis_password

# Rails / JWT
SECRET_KEY_BASE=CHANGEME_secret_key_base
JWT_SECRET=CHANGEME_jwt_secret
RAILS_ENV=production
QUEUE=1.13.1
PORT=2358
```

Create `docker-compose.yml`:

```bash
sudo nano docker-compose.yml
```

Example:

```yaml
x-logging: &default-logging
  logging:
    driver: json-file
    options:
      max-size: 100M

services:
  server:
    image: judge0/judge0:latest
    env_file: judge0.conf
    depends_on:
      - db
      - redis
    ports:
      - "2358:2358"
    restart: always
    volumes:
      - /opt/judge0/box:/box
      - ./judge0.conf:/judge0.conf:ro
    <<: *default-logging

  worker:
    image: judge0/judge0:latest
    env_file: judge0.conf
    depends_on:
      - db
      - redis
    command: ["./scripts/workers"]
    privileged: true
    restart: always
    volumes:
      - /opt/judge0/box:/box
      - /var/run/docker.sock:/var/run/docker.sock
      - ./judge0.conf:/judge0.conf:ro
    <<: *default-logging

  db:
    image: postgres:16.2
    environment:
      POSTGRES_USER: judge0
      POSTGRES_PASSWORD: CHANGEME_db_password
      POSTGRES_DB: judge0
    volumes:
      - data:/var/lib/postgresql/data
    restart: always
    <<: *default-logging

  redis:
    image: redis:7.2.4
    command: >
      bash -lc 'redis-server --appendonly no --requirepass "$REDIS_PASSWORD"'
    env_file: judge0.conf
    restart: always
    <<: *default-logging

volumes:
  data:
```

Start Judge0:

```bash
cd /opt/judge0
sudo docker compose up -d
sudo docker ps
```

You should see `judge0-server-1`, `judge0-worker-1`, `judge0-db-1`, `judge0-redis-1`.

---

### 7. Enable User Namespaces (If Needed)

If submissions fail with `clone failed: Operation not permitted` or cgroup issues, ensure:

```bash
echo "kernel.unprivileged_userns_clone=1" | sudo tee /etc/sysctl.d/99-unprivileged_userns.conf
echo "user.max_user_namespaces=15000" | sudo tee /etc/sysctl.d/99-max_user_namespaces.conf
sudo sysctl --system
```

Then restart Judge0:

```bash
cd /opt/judge0
sudo docker compose down
sudo docker compose up -d
```

---

### 8. Verify End-to-End

From the VPS:

```bash
curl -s http://127.0.0.1:2358/languages | head
```

From anywhere (over HTTPS, via Apache proxy):

```bash
curl -k -s https://cs-capstone.com/judge0/languages | head
```

Test a C++ submission:

```bash
curl -k -s -X POST   "https://cs-capstone.com/judge0/submissions?base64_encoded=false&wait=true&fields=stdout,stderr,compile_output,message,status"   -H "Content-Type: application/json"   -d '{
    "source_code": "#include <iostream>\nint main(){ std::cout << \"Hello from Judge0\"; return 0; }",
    "language_id": 54
  }'
```

Expected:

```json
{"stdout":"Hello from Judge0","stderr":null,"compile_output":null,"message":null,"status":{"id":3,"description":"Accepted"}}
```

If that works, your browser C++ demo page will work too 🎉

</details>
