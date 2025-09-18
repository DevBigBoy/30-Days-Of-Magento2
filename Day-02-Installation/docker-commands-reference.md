# Docker Magento Commands Reference

Quick reference for markshust/docker-magento commands.

## Container Management

### Start/Stop Containers

```bash
bin/start              # Start all containers
bin/start --no-dev     # Start without development tools
bin/stop               # Stop all containers
bin/restart            # Restart all containers
bin/status             # Check container status
```

### Remove Containers

```bash
bin/remove             # Remove containers (keeps volumes/data)
bin/removeall          # Remove containers AND volumes (fresh start)
bin/removevolumes      # Remove volumes only
```

## Development Commands

### Access Container Shell

```bash
bin/bash               # Open bash in PHP container
bin/bash root          # Open bash as root user
bin/cli                # Execute command in container
bin/clinotty           # Execute without TTY (for scripts)
```

### Composer Commands

```bash
bin/composer install           # Install dependencies
bin/composer update            # Update dependencies
bin/composer require vendor/package  # Add new package
bin/composer dump-autoload     # Regenerate autoloader
```

### Magento CLI Commands

```bash
bin/magento                    # Run any Magento CLI command
bin/magento cache:flush        # Flush cache
bin/magento setup:upgrade      # Run upgrades
bin/magento indexer:reindex    # Reindex
bin/magento module:status      # Check module status
```

## File Operations

### Copy Files Between Host & Container

```bash
bin/copytocontainer --all      # Copy all files to container
bin/copytocontainer app/       # Copy specific directory
bin/copyfromcontainer app/     # Copy from container to host
```

### Fix Permissions

```bash
bin/fixowns                    # Fix file ownership
bin/fixperms                   # Fix file permissions
bin/fixownsperms               # Fix both ownership and permissions
```

## Database Management

### MySQL Access

```bash
bin/mysql                      # Open MySQL CLI
bin/mysql -e "SHOW TABLES"     # Execute SQL query
bin/mysqldump > backup.sql     # Create database backup
bin/mysql < backup.sql         # Import database backup
```

### Database Operations

```bash
# Export database
bin/mysqldump > ~/backups/magento_$(date +%Y%m%d).sql

# Import database
bin/mysql < ~/backups/magento_20250115.sql

# Clear database
bin/mysql -e "DROP DATABASE magento; CREATE DATABASE magento;"
```

## Debugging & Logs

### View Logs

```bash
bin/log                        # Tail Magento logs (var/log)
bin/log --follow               # Follow log output
docker-compose logs -f         # Follow all container logs
docker-compose logs phpfpm     # View specific container logs
```

### Xdebug

```bash
bin/xdebug enable              # Enable Xdebug
bin/xdebug disable             # Disable Xdebug
bin/xdebug status              # Check Xdebug status
```

### Debug Commands

```bash
# Check which containers are running
docker ps

# View container resource usage
docker stats

# Inspect container
docker inspect dockermagento_phpfpm_1
```

## Cache Management

```bash
bin/cache-clean                # Clean Magento cache
bin/cache-flush                # Flush Magento cache
bin/magento cache:disable      # Disable all cache
bin/magento cache:enable       # Enable all cache
```

## Setup & Configuration

### Domain Setup

```bash
bin/setup magento.test         # Setup new installation
bin/setup-domain magento.test  # Configure domain for existing install
bin/setup-ssl magento.test     # Setup SSL certificate
bin/setup-ssl-ca               # Setup SSL certificate authority
```

### Download & Install Magento

```bash
bin/download community 2.4.8-p3    # Download Community Edition
bin/download enterprise 2.4.8-p3   # Download Enterprise Edition
bin/init                            # Initialize with sample data
```

## Node.js & Frontend Tools

```bash
bin/node                       # Run Node.js command
bin/npm                        # Run npm command
bin/grunt                      # Run Grunt tasks
```

## Additional Services

### Redis

```bash
bin/redis                      # Access Redis CLI
bin/redis FLUSHALL             # Clear all Redis data
```

### RabbitMQ

```bash
# Access RabbitMQ Management UI
open http://localhost:15672
# Username: guest
# Password: guest
```

### MailHog (Email Testing)

```bash
# Access MailHog UI
open http://localhost:8025
```

## Advanced Commands

### Root Access

```bash
bin/root                       # Execute command as root
bin/rootnotty                  # Execute as root without TTY
```

### Composer Auth

```bash
# Add Magento authentication keys
bin/composer config --auth http-basic.repo.magento.com <public-key> <private-key>
```

### Performance Optimization

```bash
# Compile & deploy for production
bin/magento setup:di:compile
bin/magento setup:static-content:deploy -f
bin/magento deploy:mode:set production
```

## Complete Workflow Examples

### Starting Fresh

```bash
# 1. Create new Magento installation
curl -s https://raw.githubusercontent.com/markshust/docker-magento/master/lib/onelinesetup | bash -s -- magento.test community 2.4.8-p3

# 2. Install sample data
bin/init

# 3. Access your site
open https://magento.test
```

### Daily Development Workflow

```bash
# 1. Start containers
bin/start

# 2. Watch logs (in separate terminal)
bin/log --follow

# 3. Make code changes
# Edit files in your IDE

# 4. Run Magento commands as needed
bin/magento cache:flush
bin/magento setup:upgrade

# 5. Stop containers when done
bin/stop
```

### Installing a New Module

```bash
# 1. Require module via Composer
bin/composer require vendor/module-name

# 2. Enable module
bin/magento module:enable Vendor_ModuleName

# 3. Run setup upgrade
bin/magento setup:upgrade

# 4. Compile if needed
bin/magento setup:di:compile

# 5. Deploy static content
bin/magento setup:static-content:deploy -f

# 6. Clear cache
bin/magento cache:flush
```

### Importing Existing Site

```bash
# 1. Setup Docker template
curl -s https://raw.githubusercontent.com/markshust/docker-magento/master/lib/template | bash

# 2. Start containers
bin/start --no-dev

# 3. Copy files
bin/copytocontainer --all

# 4. Install dependencies
bin/composer install

# 5. Import database
bin/mysql < ~/backups/production.sql

# 6. Update base URLs
bin/magento setup:store-config:set --base-url="https://magento.test/"
bin/magento setup:store-config:set --base-url-secure="https://magento.test/"

# 7. Setup domain
bin/setup-domain magento.test

# 8. Restart
bin/restart

# 9. Clear cache
bin/magento cache:flush
```

### Troubleshooting

```bash
# 1. Check container status
bin/status

# 2. View container logs
docker-compose logs -f phpfpm

# 3. Fix permissions
bin/fixownsperms

# 4. Restart containers
bin/restart

# 5. If all else fails, start fresh
bin/removeall
# Then reinstall
```

## Tips & Best Practices

1. **Always use `bin/` commands** instead of direct docker commands
2. **Run `bin/fixownsperms`** if you encounter permission errors
3. **Use `bin/xdebug enable`** only when debugging (impacts performance)
4. **Monitor logs** with `bin/log` to catch errors early
5. **Backup database** regularly with `bin/mysqldump`
6. **Use `--no-dev` flag** when testing production-like environment

## Container Architecture

```
┌─────────────────────────────────────────────┐
│              nginx (Port 443)               │
│         Web Server / SSL Termination        │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│          phpfpm (PHP 8.2/8.3)               │
│         PHP-FPM / Application Layer         │
└──────────────────┬──────────────────────────┘
                   │
     ┌─────────────┼─────────────┬─────────────┐
     ▼             ▼             ▼             ▼
┌─────────┐  ┌──────────┐  ┌─────────┐  ┌──────────┐
│   db    │  │elasticsearch│ │  redis  │  │ rabbitmq │
│MySQL 8.0│  │    8.x    │  │  Cache  │  │  Queue   │
└─────────┘  └──────────┘  └─────────┘  └──────────┘

                   ┌──────────┐
                   │ mailhog  │
                   │Email Test│
                   └──────────┘
```

## Port Mappings

| Service       | Container Port | Host Port | Access URL                    |
|---------------|----------------|-----------|-------------------------------|
| Nginx         | 443            | 443       | https://magento.test          |
| Nginx         | 80             | 80        | http://magento.test           |
| MySQL         | 3306           | 3306      | localhost:3306                |
| Elasticsearch | 9200           | 9200      | http://localhost:9200         |
| Redis         | 6379           | 6379      | localhost:6379                |
| RabbitMQ      | 15672          | 15672     | http://localhost:15672        |
| MailHog       | 8025           | 8025      | http://localhost:8025         |

## Environment Variables

Edit `env/db.env` to customize database settings:

```env
MYSQL_DATABASE=magento
MYSQL_USER=magento
MYSQL_PASSWORD=magento
MYSQL_ROOT_PASSWORD=magento
```

## Additional Resources

- [Official Documentation](https://github.com/markshust/docker-magento)
- [Video Tutorials](https://m.academy)
- [Troubleshooting Guide](https://github.com/markshust/docker-magento/blob/master/docs/troubleshooting.md)
