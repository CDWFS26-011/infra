# ARCHI_SERVEUR.md
**Note de synthèse — Déploiement Relais & Châteaux**  
Date : 14 avril 2026 | Serveur : Ubuntu 24.04 LTS | IP : 192.168.50.84

---

## 1. Stack technique installée

### Serveur Web — Nginx
Apache étant absent, Nginx a été installé directement comme serveur web principal.

```bash
sudo apt update && sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

**Choix justifié :** Nginx offre de meilleures performances qu'Apache pour les applications PHP modernes (moindre consommation mémoire, gestion asynchrone des connexions). Il est nativement adapté à Symfony via la directive `fastcgi_pass`.

### PHP 8.4 + Extensions

```bash
sudo apt install software-properties-common -y
sudo add-apt-repository ppa:ondrej/php -y
sudo apt update
sudo apt install php8.4 php8.4-fpm php8.4-mysql php8.4-xml php8.4-mbstring php8.4-zip php8.4-curl unzip -y
sudo phpenmod mbstring xml zip curl
```

**Version choisie :** PHP 8.4 (dernière version stable LTS), compatible avec Symfony 7.x. Utilisation de PHP-FPM pour la communication avec Nginx via socket Unix (`/var/run/php/php8.4-fpm.sock`), plus performant qu'un socket TCP.

### Base de données — MariaDB

```bash
sudo apt install mariadb-server -y
sudo mysql_secure_installation
```

Puis création de la base dédiée :

```sql
CREATE DATABASE relais_db;
CREATE USER 'symfony'@'localhost' IDENTIFIED BY 'Relais2026!';
GRANT ALL PRIVILEGES ON relais_db.* TO 'symfony'@'localhost';
FLUSH PRIVILEGES;
```

**Choix justifié :** MariaDB, fork communautaire de MySQL, offre une compatibilité totale avec Doctrine ORM (Symfony) et de meilleures performances sur les petites/moyennes charges.

### Composer

```bash
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
php -r "unlink('composer-setup.php');"
```

---

## 2. Déploiement du projet

### Arborescence

```bash
sudo mkdir -p /var/www/html/relais-chateaux
sudo chown -R user:www-data /var/www/html/relais-chateaux
sudo chmod -R 775 /var/www/html/relais-chateaux
```

### Clonage depuis GitHub

```bash
cd /var/www/html/relais-chateaux
git clone https://github.com/USER/relais-chateaux.git .
```

### Installation des dépendances

```bash
composer install
```

### Variables d'environnement (`.env.local`)

```env
APP_ENV=prod
APP_SECRET=d6ccbd5bb6dc1d43360318ce9d41a8a1
DATABASE_URL="mysql://symfony:Relais2026!@127.0.0.1:3306/relais_db"
```

### Mise en cache production

```bash
APP_ENV=prod php bin/console cache:clear
APP_ENV=prod php bin/console cache:warmup
```

---

## 3. Configuration Nginx — Virtual Host

Fichier : `/etc/nginx/sites-available/relais-chateaux`

```nginx
# Redirection HTTP → HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name relais-chateaux.com;
    return 301 https://$host$request_uri;
}

# Serveur HTTPS
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name relais-chateaux.com;

    root /var/www/html/relais-chateaux/public;
    index index.php;

    ssl_certificate /etc/ssl/certs/selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/selfsigned.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    access_log /var/log/nginx/relais.access.log;
    error_log  /var/log/nginx/relais.error.log;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    location / {
        try_files $uri /index.php$is_args$args;
    }

    location ~ ^/index\.php(/|$) {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        fastcgi_param DOCUMENT_ROOT $realpath_root;
        fastcgi_index index.php;
        fastcgi_pass unix:/var/run/php/php8.4-fpm.sock;
        internal;
    }

    location ~ \.php$ {
        return 404;
    }

    location ~* \.(css|js|jpg|jpeg|png|ico|svg|woff2?)$ {
        expires 30d;
        access_log off;
        add_header Cache-Control "public";
    }
}
```

Activation :

```bash
sudo ln -s /etc/nginx/sites-available/relais-chateaux /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

DNS local Windows (`C:\Windows\System32\drivers\etc\hosts`) :

```
192.168.50.84  relais-chateaux.com
```

---

## 4. Sécurisation et durcissement

### Certificat SSL auto-signé

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/selfsigned.key \
  -out /etc/ssl/certs/selfsigned.crt \
  -subj "/C=FR/ST=Paris/L=Paris/O=RelaisChateaux/CN=relais-chateaux.com"
```

Validité : 365 jours. Le navigateur affiche un avertissement (comportement normal pour un certificat auto-signé hors CA publique). TLS 1.2 et 1.3 uniquement — TLS 1.0 et 1.1 désactivés.

### Pare-feu UFW

```bash
sudo ufw allow 2222   # SSH (port personnalisé)
sudo ufw allow 80     # HTTP
sudo ufw allow 443    # HTTPS
sudo ufw enable
sudo ufw status
```

**Politique :** deny par défaut sur tout autre port entrant. Seuls les 3 ports strictement nécessaires sont ouverts.

### Renforcement SSH

Fichier : `/etc/ssh/sshd_config`

```bash
sudo nano /etc/ssh/sshd_config
```

Modifications appliquées :

```
Port 2222
PermitRootLogin no
PasswordAuthentication no
```

```bash
sudo systemctl restart ssh
```

**Justification :** La désactivation du login root réduit la surface d'attaque. Le changement de port (2222) limite les tentatives automatisées sur le port 22 standard. L'authentification par clé uniquement supprime les risques de brute-force par mot de passe.

---

## 5. Optimisation des performances

### PHP-FPM (`/etc/php/8.4/fpm/pool.d/www.conf`)

```ini
pm = dynamic
pm.max_children = 10
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 5
```

```bash
sudo systemctl restart php8.4-fpm
```

### Symfony — Mode production

```bash
APP_ENV=prod php bin/console cache:clear
APP_ENV=prod php bin/console cache:warmup
```

Le mode `prod` active l'optimisation de l'autoloader Composer et la compilation du container DI, réduisant significativement les temps de réponse par rapport au mode `dev`.

### Cache statique Nginx

Les assets CSS/JS/images sont servis avec un header `Cache-Control: public` et une expiration à 30 jours, évitant des requêtes répétées côté client.

---

## 6. Supervision et diagnostic

```bash
# Surveillance CPU / RAM en temps réel
htop

# Mémoire disponible
free -h

# Statistiques système
vmstat 1 5

# Logs Nginx en temps réel
sudo tail -f /var/log/nginx/relais.error.log

# Journal PHP-FPM
sudo journalctl -u php8.4-fpm
```

---

## 7. Récapitulatif des ports ouverts

| Port | Protocole | Service        |
|------|-----------|----------------|
| 2222 | TCP       | SSH            |
| 80   | TCP       | HTTP → HTTPS   |
| 443  | TCP       | HTTPS (Symfony)|

---

## 8. Arborescence finale du projet

```
/var/www/html/relais-chateaux/
├── public/             ← racine web (index.php)
├── src/
│   └── Controller/
│       └── HomeController.php
├── templates/
│   └── home/
│       └── index.html.twig
├── config/
│   ├── routes.yaml
│   └── routes/
│       └── attributes.yaml
├── .env
└── .env.local          ← variables sensibles (non versionné)
```
