To install Magento 2.4 using **Mark Shust's Docker setup**, you can follow these steps. Mark Shust’s Docker configuration provides a pre-configured environment for running Magento 2.4, making it easier to get Magento up and running without dealing with complex configurations.

### Step-by-Step Installation of Magento 2.4 via Mark Shust's Docker

#### Prerequisites

Before starting, make sure the following are installed on your system:

1. **Docker** and **Docker Compose** (Ensure you have the latest versions):
    - Install Docker: [Docker Install](https://docs.docker.com/get-docker/)
    - Install Docker Compose: [Docker Compose Install](https://docs.docker.com/compose/install/)

You can verify your installation by running:

```bash
docker --version
docker-compose --version
```

#### Step 1: Create the Project Directory

Open your terminal and create the directory for your Magento project. You can customize the directory path if needed.

```bash
mkdir -p ~/Sites/magento
cd ~/Sites/magento
```

#### Step 2: Run the Installation Command

Once you're in the project directory (`~/Sites/magento`), run the following command:

```bash
curl -s https://raw.githubusercontent.com/markshust/docker-magento/master/lib/onelinesetup | bash -s -- magento.test 2.4.7 community
```

#### Explanation of the Command

- `magento.test`: This is the hostname for your Magento project. You can replace this with your preferred domain name (e.g., `my-magento.test`).
- `2.4.7`: This is the version of Magento you want to install (Magento 2.4.7 in this case).
- `community`: This indicates that you want the open-source **Community Edition**. If you have an enterprise version, you would replace this with `enterprise`.

This command will:

- Set up the necessary Docker containers (PHP, MySQL, Elasticsearch, Redis, etc.).
- Install Magento 2.4.7 in the directory you specified.
- Set up a `docker-compose.yml` file that defines your Magento environment.

#### Step 3: Set Up Your Hosts File

After the script finishes, you need to add the domain to your `/etc/hosts` file to access Magento from your browser.

Open the `/etc/hosts` file and add the following entry (replacing `magento.test` with your project’s domain if needed):

```bash
sudo nano /etc/hosts
```

Add this line:

```bash
127.0.0.1 magento.test
```

Save and exit.

#### Step 4: Access Magento

Once the installation is complete, you can access Magento from your browser by visiting:

```bash
http://magento.test
```

- **Admin URL**: The default admin URL is `http://magento.test/admin`.
- **Default credentials**:
  - Admin Username: `admin`
  - Admin Password: `admin123`

#### Step 5: Manage Your Magento Installation

Mark Shust’s setup includes several `bin/` helper scripts to manage your Magento installation.

Here are some useful commands:

- **Start Docker containers**:

  ```bash
  bin/start
  ```

- **Stop Docker containers**:

  ```bash
  bin/stop
  ```

- **Run Magento CLI commands**:

  ```bash
  bin/cli magento <command>
  ```

For example:

```bash
bin/cli magento setup:upgrade
```

- **Enable Xdebug**:

  ```bash
  bin/xdebug enable
  ```

- **Disable Xdebug**:

  ```bash
  bin/xdebug disable
  ```

- **Access the database**:

  ```bash
  bin/mysql
  ```

- **Check Elasticsearch status**:

  ```bash
  bin/cli curl -X GET "es:9200"
  ```

### Step 6: Optional (Switch to Developer Mode)

Magento by default runs in **production mode** after installation. To switch to **developer mode** (which is more suitable for local development), run:

```bash
bin/cli magento deploy:mode:set developer
```

### Disable Magento TwoFactorAuth

check the status of TwoFactorAuth

```bash
bin/magento module: status | grep Two
```

```bash
bin/magento module:enable MageMastery_Blog
```

### Conclusion

By following these steps, you’ll have Magento 2.4 installed and running using **Mark Shust's Docker** setup. This setup simplifies the process by providing pre-configured Docker containers for Magento, PHP, MySQL, Elasticsearch, and more.

The environment is ready for development, with support for debugging (Xdebug) and an easily configurable `.env` file for environment management.

You can find more detailed documentation in [Mark Shust's GitHub repository](https://github.com/markshust/docker-magento).
