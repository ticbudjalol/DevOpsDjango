# 🚀 DevOps Django Setup – Full Guide

Kompleten vodič za postavitev Django aplikacije z Docker, PostgreSQL, Nginx Proxy Manager, pgAdmin, Uptime Kuma in več.

---

## 🧰 Pogoji

- VPS (npr. Hetzner CX22)
- Domena (npr. `ticbudja.com`)
- OS: Debian ali Ubuntu
- SSH dostop
- GitHub račun (za verzije ali CI/CD)
- Lokalen Git + Python + Django (za razvoj)

---

## 📦 1. Osnovna priprava strežnika

### 1.1 SSH ključ

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

Dodaj javni ključ v strežnik in se poveži:

```bash
ssh root@YOUR_SERVER_IP
```

---

### 1.2 Namesti Docker in Docker Compose

```bash
apt update && apt install -y curl git
curl -fsSL https://get.docker.com | sh
apt install -y docker-compose-plugin
usermod -aG docker $USER
```

---

## ⚙️ 2. DevOps orodja: Portainer, Nginx Proxy Manager, pgAdmin, Uptime Kuma

```bash
mkdir -p /opt/devops-stack && cd /opt/devops-stack
```

### `docker-compose.yml`

```yaml
version: "3.8"
services:
  portainer:
    image: portainer/portainer-ce
    container_name: portainer
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    restart: unless-stopped

  nginx-proxy-manager:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    ports:
      - "80:80"
      - "81:81"
      - "443:443"
    volumes:
      - npm_data:/data
      - npm_letsencrypt:/etc/letsencrypt
    restart: unless-stopped

  pgadmin:
    image: dpage/pgadmin4
    container_name: pgadmin
    ports:
      - "8001:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@ticbudja.com
      PGADMIN_DEFAULT_PASSWORD: supersecret
    volumes:
      - pgadmin_data:/var/lib/pgadmin
    restart: unless-stopped

  uptime-kuma:
    image: louislam/uptime-kuma
    container_name: uptime-kuma
    ports:
      - "3001:3001"
    volumes:
      - uptime_kuma_data:/app/data
    restart: unless-stopped

volumes:
  portainer_data:
  npm_data:
  npm_letsencrypt:
  pgadmin_data:
  uptime_kuma_data:
```

Zagon:

```bash
docker compose up -d
```

---

## 🌍 3. Nastavi domene prek Nginx Proxy Managerja

1. Obišči `http://your-ip:81`
2. Prijava: `admin@example.com` / `changeme`
3. Dodaj Proxy Host:
   - Domain: `app.ticbudja.com`
   - Forward: `127.0.0.1:8000`
   - SSL certifikat (Let's Encrypt)

---

## 🐘 4. Django + PostgreSQL aplikacija

Struktura:

```
/opt/DevOpsDjango/
├── config/           # Django projekt
├── manage.py
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

---

### requirements.txt

```txt
Django>=4.2,<5.0
gunicorn
psycopg2-binary
```

---

### Dockerfile

```Dockerfile
FROM python:3.11-slim

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["gunicorn", "config.wsgi:application", "--bind", "0.0.0.0:8000"]
```

---

### docker-compose.yml

```yaml
version: "3.8"
services:
  django:
    build: .
    container_name: django-app
    restart: unless-stopped
    networks:
      - nginx-network
    environment:
      - DEBUG=True
      - DJANGO_ALLOWED_HOSTS=app.ticbudja.com
      - DATABASE_NAME=devopsdb
      - DATABASE_USER=devuser
      - DATABASE_PASSWORD=devpass123
      - DATABASE_HOST=db
      - DATABASE_PORT=5432
    depends_on:
      - db

  db:
    image: postgres:15
    container_name: postgres-db
    restart: unless-stopped
    networks:
      - nginx-network
    environment:
      POSTGRES_DB: devopsdb
      POSTGRES_USER: devuser
      POSTGRES_PASSWORD: devpass123
    volumes:
      - postgres_data:/var/lib/postgresql/data

networks:
  nginx-network:
    external: true

volumes:
  postgres_data:
```

---

### settings.py → DATABASES

```python
import os

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': os.environ.get("DATABASE_NAME"),
        'USER': os.environ.get("DATABASE_USER"),
        'PASSWORD': os.environ.get("DATABASE_PASSWORD"),
        'HOST': os.environ.get("DATABASE_HOST"),
        'PORT': os.environ.get("DATABASE_PORT"),
    }
}
```

---

## 🚀 5. Deploy in migracija

```bash
docker compose down -v
docker compose up -d --build
docker exec -it django-app python manage.py migrate
docker exec -it django-app python manage.py createsuperuser
```

---

## 📊 6. Uptime Kuma – spremljanje aplikacije

1. Odpri `http://your-ip:3001`
2. Prijava
3. Add New Monitor:
   - Type: `HTTPS`
   - URL: `https://app.ticbudja.com`
   - Interval: 30s ali po želji

📬 Po želji nastavi obvestila (Email, Discord, Telegram...)

---

## ✅ Zaključek

Imamo:
- Django aplikacijo na domeni z HTTPS
- PostgreSQL bazo + pgAdmin
- Proxy + certifikate preko Nginx Proxy Manager
- Spremljanje aplikacije z Uptime Kuma
- Portainer za Docker GUI

---

## 🔜 Naslednji koraki

- 🔐 Uporaba `.env` datotek
- ⚙️ GitHub CI/CD
- 📦 Več aplikacij (FastAPI, admin portal)
- 🛡️ Varnost: firewall, SSH hardening, Docker secrets

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->
