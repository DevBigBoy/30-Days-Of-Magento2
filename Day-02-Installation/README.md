# Day 02: Installation & Setup

## Overview

This guide covers multiple methods to install Magento 2, with a focus on Docker-based setup for development environments. We'll cover system requirements, installation methods, and post-installation configuration.

## Table of Contents

1. [System Requirements](#system-requirements)
2. [Installation Methods](#installation-methods)
3. [Docker Installation (Recommended for Development)](#docker-installation-recommended-for-development)
4. [Traditional Installation](#traditional-installation)
5. [Post-Installation Setup](#post-installation-setup)
6. [Essential CLI Commands](#essential-cli-commands)
7. [Troubleshooting](#troubleshooting)

## System Requirements

### Minimum Requirements

- **RAM**: 6GB minimum (8GB+ recommended)
- **Processor**: Dual-core processor minimum
- **Storage**: SSD hard drive recommended
- **Operating System**: Linux, macOS, or Windows (with WSL2)

### Software Requirements

#### For Docker Setup (Recommended)

- **Docker Desktop**: Latest version
- **Git**: For version control
- **Text Editor/IDE**: VS Code, PHPStorm, etc.

#### For Traditional Setup

- **Web Server**: Apache 2.4+ or Nginx 1.x
- **PHP**: 8.1, 8.2, or 8.3 (depending on Magento version)
- **Database**: MySQL 8.0+ or MariaDB 10.4+
- **PHP Extensions**:
  - ext-bcmath
  - ext-ctype
  - ext-curl
  - ext-dom
  - ext-gd
  - ext-hash
  - ext-iconv
  - ext-intl
  - ext-mbstring
  - ext-openssl
  - ext-pdo_mysql
  - ext-simplexml
  - ext-soap
  - ext-sodium
  - ext-xsl
  - ext-zip
  - ext-sockets
- **Composer**: 2.x
- **Elasticsearch** or **OpenSearch**: 7.x or 8.x
- **Redis**: Optional but recommended for cache and sessions

### Magento Version Compatibility

| Magento Version | PHP Version | MySQL/MariaDB | Elasticsearch/OpenSearch |
|----------------|-------------|---------------|-------------------------|
| 2.4.7-2.4.8    | 8.1, 8.2, 8.3 | MySQL 8.0 / MariaDB 10.6 | ES 8.x / OS 1.x-2.x |
| 2.4.6          | 8.1, 8.2    | MySQL 8.0 / MariaDB 10.6 | ES 7.17 / OS 1.x-2.x |
| 2.4.4-2.4.5    | 8.1         | MySQL 8.0 / MariaDB 10.4 | ES 7.16-7.17 |

## Installation Methods

### 1. Docker Installation (Recommended for Development)
Best for local development, easy setup, isolated environment

### 2. Composer Installation
Direct installation using Composer, requires manual server configuration

### 3. Git Clone
Clone Magento repository, useful for contributing to core

### 4. Cloud Installation
Adobe Commerce Cloud platform (for production)

## Docker Installation (Recommended for Development)

We'll use [markshust/docker-magento](https://github.com/markshust/docker-magento), a popular Docker configuration optimized for Magento 2 development.

### Prerequisites

1. **Install Docker Desktop**
   - [Mac](https://docs.docker.com/desktop/install/mac-install/)
   - [Windows](https://docs.docker.com/desktop/install/windows-install/)
   - [Linux](https://docs.docker.com/desktop/install/linux-install/)

2. **Configure Docker Resources**
   - Open Docker Desktop → Settings → Resources
   - Allocate at least 6GB RAM (8GB recommended)
   - Allocate at least 2 CPUs (4 recommended)

### Automated Setup (Quickest Method)

#### New Project Installation

```bash
# Create project directory
mkdir -p ~/Sites/magento
cd ~/Sites/magento

# One-line automated setup
curl -s https://raw.githubusercontent.com/markshust/docker-magento/master/lib/onelinesetup | bash -s -- magento.test community 2.4.8-p3
```

**What this does:**
- Downloads Docker configuration files
- Sets up container infrastructure
- Downloads Magento 2.4.8-p3 (Community Edition)
- Installs Magento with domain `magento.test`
- Configures `/etc/hosts` for local DNS

**You'll be prompted for:**
- System password (for `/etc/hosts` modification)
- Admin credentials during Magento installation

#### Access Your Site

After installation completes:
- **Frontend**: https://magento.test
- **Admin Panel**: https://magento.test/admin
  - Username: `admin`
  - Password: (set during installation, default: `admin123`)

#### Initialize Sample Data (Optional)

```bash
bin/init
```

This installs sample products, categories, and demo content for development.

### Manual Setup (Step-by-Step)

For more control over the installation process:

```bash
# 1. Create and navigate to project directory
mkdir -p ~/Sites/magento
cd ~/Sites/magento

# 2. Download Docker template files
curl -s https://raw.githubusercontent.com/markshust/docker-magento/master/lib/template | bash

# 3. Download Magento
# Options: community | enterprise
# Version: 2.4.8-p3, 2.4.7, etc.
bin/download community 2.4.8-p3

# 4. Setup Magento with your domain
bin/setup magento.test

# 5. Initialize development environment (sample data)
bin/init

# 6. Open in browser
open https://magento.test
```

### Docker Container Structure

The setup creates the following containers:

```
┌─────────────────────────────────────────────────┐
│              Docker Containers                  │
├─────────────────────────────────────────────────┤
│ nginx         - Web server (port 443)           │
│ phpfpm        - PHP 8.x-FPM                     │
│ db            - MySQL 8.0                       │
│ elasticsearch - Elasticsearch 8.x               │
│ redis         - Redis cache                     │
│ rabbitmq      - Message queue                   │
│ mailhog       - Email testing                   │
└─────────────────────────────────────────────────┘
```

### Essential Docker Commands

```bash
# Container Management
bin/start              # Start all containers
bin/stop               # Stop all containers
bin/restart            # Restart all containers
bin/status             # Check container status

# Development Commands
bin/bash               # Access PHP container shell
bin/cli                # Run commands in container
bin/composer           # Run Composer commands
bin/magento            # Run Magento CLI commands

# Database
bin/mysql              # Access MySQL CLI
bin/mysqldump > backup.sql  # Backup database

# Debugging & Logs
bin/log                # Tail Magento logs
bin/xdebug enable      # Enable Xdebug
bin/xdebug disable     # Disable Xdebug

# File Operations
bin/copyfromcontainer  # Copy files from container
bin/copytocontainer    # Copy files to container

# Cache & Cleanup
bin/cache-clean        # Clean Magento cache
bin/remove             # Remove containers
bin/removeall          # Remove containers and volumes
```

### Setting Up Existing Projects

If you have an existing Magento installation:

```bash
# 1. Create project directory
mkdir -p ~/Sites/existing-magento
cd ~/Sites/existing-magento

# 2. Download Docker template
curl -s https://raw.githubusercontent.com/markshust/docker-magento/master/lib/template | bash

# 3. Start containers (without dev mode)
bin/start --no-dev

# 4. Copy existing files to container
bin/copytocontainer --all

# 5. Install dependencies
bin/composer install

# 6. Import existing database
bin/mysql < ../path/to/backup.sql

# 7. Setup domain
bin/setup-domain yoursite.test

# 8. Restart containers
bin/restart

# 9. Update configuration
bin/magento setup:upgrade
bin/magento cache:flush
```

## Traditional Installation

### Using Composer (Direct Installation)

#### Prerequisites Check

```bash
# Check PHP version
php -v

# Check required PHP extensions
php -m | grep -E 'bcmath|ctype|curl|dom|gd|hash|iconv|intl|mbstring|openssl|pdo_mysql|simplexml|soap|sodium|xsl|zip|sockets'

# Check Composer version
composer -V
```

#### Installation Steps

```bash
# 1. Create project using Composer
composer create-project --repository-url=https://repo.magento.com/ magento/project-community-edition magento2

# 2. Navigate to project directory
cd magento2

# 3. Set file permissions
find var generated vendor pub/static pub/media app/etc -type f -exec chmod g+w {} +
find var generated vendor pub/static pub/media app/etc -type d -exec chmod g+ws {} +
chown -R :www-data .
chmod u+x bin/magento

# 4. Install Magento
php bin/magento setup:install \
  --base-url=http://magento.local \
  --db-host=localhost \
  --db-name=magento \
  --db-user=magento \
  --db-password=magento \
  --admin-firstname=Admin \
  --admin-lastname=User \
  --admin-email=admin@example.com \
  --admin-user=admin \
  --admin-password=admin123 \
  --language=en_US \
  --currency=USD \
  --timezone=America/Chicago \
  --use-rewrites=1 \
  --search-engine=elasticsearch7 \
  --elasticsearch-host=localhost \
  --elasticsearch-port=9200

# 5. Disable two-factor authentication (development only)
php bin/magento module:disable Magento_AdminAdobeImsTwoFactorAuth
php bin/magento module:disable Magento_TwoFactorAuth

# 6. Deploy static content
php bin/magento setup:static-content:deploy -f

# 7. Set developer mode
php bin/magento deploy:mode:set developer
```

### Get Magento Authentication Keys

To download Magento, you need authentication keys from Adobe:

1. Go to [https://marketplace.magento.com/](https://marketplace.magento.com/)
2. Sign in or create an account
3. Go to My Profile → Access Keys
4. Create new access key
5. Use Public Key as username, Private Key as password

```bash
# Configure Composer authentication
composer config --global http-basic.repo.magento.com <public-key> <private-key>
```

## Post-Installation Setup

### 1. Configure Web Server

#### Nginx Configuration

```nginx
upstream fastcgi_backend {
    server unix:/var/run/php/php8.2-fpm.sock;
}

server {
    listen 80;
    server_name magento.local;

    set $MAGE_ROOT /var/www/html/magento2;
    set $MAGE_MODE developer;

    include /var/www/html/magento2/nginx.conf.sample;
}
```

#### Apache Configuration

```apache
<VirtualHost *:80>
    ServerName magento.local
    DocumentRoot /var/www/html/magento2/pub

    <Directory /var/www/html/magento2/pub>
        Options Indexes FollowSymLinks MultiViews
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/magento_error.log
    CustomLog ${APACHE_LOG_DIR}/magento_access.log combined
</VirtualHost>
```

### 2. Configure Cron

```bash
# Add to crontab
crontab -e

# Add these lines
* * * * * /usr/bin/php /var/www/html/magento2/bin/magento cron:run 2>&1 | grep -v "Ran jobs by schedule" >> /var/www/html/magento2/var/log/magento.cron.log
* * * * * /usr/bin/php /var/www/html/magento2/update/cron.php >> /var/www/html/magento2/var/log/update.cron.log
* * * * * /usr/bin/php /var/www/html/magento2/bin/magento setup:cron:run >> /var/www/html/magento2/var/log/setup.cron.log
```

### 3. Configure Redis (Optional but Recommended)

```bash
bin/magento setup:config:set --cache-backend=redis --cache-backend-redis-server=127.0.0.1 --cache-backend-redis-db=0

bin/magento setup:config:set --page-cache=redis --page-cache-redis-server=127.0.0.1 --page-cache-redis-db=1

bin/magento setup:config:set --session-save=redis --session-save-redis-host=127.0.0.1 --session-save-redis-db=2
```

## Essential CLI Commands

### Installation & Setup

```bash
bin/magento setup:install          # Install Magento
bin/magento setup:upgrade           # Run database upgrades
bin/magento setup:di:compile        # Compile dependency injection
bin/magento setup:static-content:deploy  # Deploy static content
```

### Module Management

```bash
bin/magento module:status           # List module status
bin/magento module:enable Module_Name   # Enable module
bin/magento module:disable Module_Name  # Disable module
```

### Cache Management

```bash
bin/magento cache:status            # Check cache status
bin/magento cache:clean             # Clean cache
bin/magento cache:flush             # Flush cache storage
bin/magento cache:enable            # Enable all cache types
bin/magento cache:disable           # Disable all cache types
```

### Indexer Management

```bash
bin/magento indexer:status          # Check indexer status
bin/magento indexer:reindex         # Reindex all
bin/magento indexer:set-mode realtime  # Set realtime indexing
bin/magento indexer:set-mode schedule  # Set scheduled indexing
```

### Mode Management

```bash
bin/magento deploy:mode:show        # Show current mode
bin/magento deploy:mode:set developer   # Set developer mode
bin/magento deploy:mode:set production  # Set production mode
```

### Admin User Management

```bash
# Create admin user
bin/magento admin:user:create \
  --admin-user=admin \
  --admin-password=admin123 \
  --admin-email=admin@example.com \
  --admin-firstname=Admin \
  --admin-lastname=User

# Unlock admin user
bin/magento admin:user:unlock admin
```

## Troubleshooting

### Docker Installation Issues

#### "Project directory is not empty" Error

```bash
# Remove all Docker containers and volumes
bin/removeall

# Remove project directory
cd ..
rm -rf magento

# Start fresh installation
mkdir magento && cd magento
curl -s https://raw.githubusercontent.com/markshust/docker-magento/master/lib/onelinesetup | bash -s -- magento.test community 2.4.8-p3
```

#### Container Won't Start

```bash
# Check Docker is running
docker ps

# Check port conflicts
lsof -i :80
lsof -i :443

# Restart Docker Desktop
# Then restart containers
bin/restart
```

#### Permission Issues

```bash
# Fix file permissions in container
bin/bash
chown -R www-data:www-data /var/www/html
exit

# Or from host
bin/fixowns
bin/fixperms
```

### Traditional Installation Issues

#### PHP Extensions Missing

```bash
# Ubuntu/Debian
sudo apt-get install php8.2-{bcmath,ctype,curl,dom,gd,iconv,intl,mbstring,pdo-mysql,simplexml,soap,xsl,zip,sockets}

# CentOS/RHEL
sudo yum install php82-{bcmath,gd,intl,mbstring,mysqlnd,soap,xml,zip}
```

#### Memory Limit Errors

```bash
# Increase PHP memory limit
php -d memory_limit=2G bin/magento setup:install ...
```

#### Database Connection Issues

```bash
# Test MySQL connection
mysql -u magento -p -h localhost

# Check MySQL is running
sudo systemctl status mysql
```

#### Elasticsearch Connection Failed

```bash
# Check Elasticsearch is running
curl -X GET "localhost:9200"

# Restart Elasticsearch
sudo systemctl restart elasticsearch
```

### Common Post-Installation Issues

#### Admin 404 Error

```bash
# Rebuild URLs
bin/magento setup:upgrade
bin/magento cache:flush
```

#### Static Files Not Loading

```bash
# Deploy static content
bin/magento setup:static-content:deploy -f en_US

# Set proper permissions
chmod -R 777 pub/static
```

#### Blank Page After Installation

```bash
# Check error logs
tail -f var/log/system.log
tail -f var/log/exception.log

# Enable error display (developer mode only)
bin/magento deploy:mode:set developer
```

## Verification Checklist

After installation, verify:

- [ ] Frontend accessible (https://magento.test)
- [ ] Admin panel accessible (https://magento.test/admin)
- [ ] Can login to admin
- [ ] Cron jobs configured
- [ ] Cache working properly
- [ ] Static content deployed
- [ ] Sample data loaded (if desired)
- [ ] Email testing configured

## Next Steps

- [Day 03: Module Creation](../Day-03-Module-Creation/README.md)
- Configure your IDE for Magento development
- Set up Xdebug for debugging
- Explore the Magento admin panel

## Additional Resources

- [Official Magento Installation Guide](https://devdocs.magento.com/guides/v2.4/install-gde/bk-install-guide.html)
- [Docker Magento Documentation](https://github.com/markshust/docker-magento)
- [System Requirements](https://devdocs.magento.com/guides/v2.4/install-gde/system-requirements.html)
- [Magento DevDocs](https://devdocs.magento.com/)
