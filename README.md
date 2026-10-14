# WordPress with Docker Compose

A Docker Compose setup for running WordPress with a MariaDB database.

This repository contains everything needed to run a WordPress website with Docker.

It uses two official Docker images:

- **`wordpress:7.1.3-apache`** – WordPress with the Apache web server and PHP 8.3
- **`mariadb:11.8`** – the MariaDB database (LTS release)

Both images use fixed versions instead of `latest`, so the setup always starts with the same software and stays reproducible.

Docker Compose starts both services in a shared network, makes WordPress available on port **8080**, stores the database in a persistent volume and restarts the containers automatically if they crash. All credentials are loaded from a local `.env` file and are never stored in the repository.

The purpose of this project is to describe a complete WordPress setup as code, so it can be started on any machine with Docker in a reproducible way.

## Table of Contents

- [Quickstart](#quickstart)
  - [Prerequisites](#prerequisites)
  - [Start WordPress](#start-wordpress)
  - [Finish the Installation](#finish-the-installation)
- [Repository Contents](#repository-contents)
- [Usage](#usage)
  - [How It Works](#how-it-works)
  - [Configuration with the .env File](#configuration-with-the-env-file)
  - [Non-Critical Configuration](#non-critical-configuration)
  - [Changing the Port](#changing-the-port)
  - [Changing the Passwords](#changing-the-passwords)
  - [Updating the Image Versions](#updating-the-image-versions)
  - [Persistence](#persistence)
  - [Network](#network)
  - [Restart Behaviour](#restart-behaviour)
  - [Useful Commands](#useful-commands)
- [Testing](#testing)
- [Security Notes](#security-notes)

## Quickstart

### Prerequisites

- A Linux machine or cloud VM with at least **1 GB of free RAM**
- **Docker Engine** and the **Docker Compose plugin v2** (`docker compose version`)
- Port **8080/TCP** open in the firewall of the VM / cloud provider

> [!IMPORTANT]
> Use `docker compose` (with a space, v2). The old `docker-compose` (v1.29) is no longer maintained and fails with newer Docker versions (`KeyError: 'ContainerConfig'`).

### Start WordPress

```bash
# 1. Clone the repository
git clone <repository-url>
cd wordpress

# 2. Create your configuration from the template
cp .env.example .env

# 3. Set your own passwords in .env
nano .env

# 4. Start both services in the background
docker compose up -d

# 5. Check that both containers are running
docker compose ps
```

> [!IMPORTANT]
> Change `DB_PASSWORD` and `DB_ROOT_PASSWORD` in `.env` **before** the first start. MariaDB only sets the passwords when the database is created for the first time.

### Finish the Installation

1. Open `http://<VM-IP>:8080` in the browser (on your own computer: `http://localhost:8080`)
2. Choose the language
3. Enter the site title and create the **admin user** (username, password, email)
4. Click **Install WordPress** and log in with the admin user

> [!NOTE]
> The admin user is created in the browser and stored in the database. It is not configured in any file of this repository.

## Repository Contents

| File&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Purpose |
|---|---|
| `docker-compose.yaml` | Defines the services `wordpress` and `db` with image versions, port, environment variables, volume, network and restart policy. |
| `.env.example` | Template for the configuration. Copy it to `.env` and set your own values. `.env` itself is not committed. |
| `.gitignore` | Keeps secrets (`.env`, keys) and editor files out of the repository. |
| `README.md` | This documentation. |

## Usage

### How It Works

```
.env ──► docker compose up -d
                │
                ├──► db (mariadb:11.8)
                │       creates the database and the user from .env
                │       Volume db-data ──► /var/lib/mysql
                │
                └──► wordpress (wordpress:7.1.3-apache)
                        connects to db:3306
                        HOST_PORT (host) ──► 80 (container)

        both services run in the network wp-net
```

On the first start, MariaDB creates the database `DB_NAME` and the user `DB_USER` with the password `DB_PASSWORD`. WordPress uses the same values to connect to the database, so they always match.

### Configuration with the .env File

All variables that can change are configured in the `.env` file (created from `.env.example`). Docker Compose reads it automatically and inserts the values with the `${VARIABLE}` notation.

> [!NOTE]
> `.env` is your local configuration and is never committed. Only `.env.example` is part of the repository.

| Variable&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Default | Secret | Description |
|---|---|---|---|
| `HOST_PORT` | `8080` | no | Port on the host where WordPress is reachable |
| `DB_NAME` | `wordpress` | no | Name of the database |
| `DB_USER` | – | yes | Database user for WordPress |
| `DB_PASSWORD` | – | yes | Password of the database user |
| `DB_ROOT_PASSWORD` | – | yes | Password of the MariaDB `root` user |

`HOST_PORT` and `DB_NAME` have default values in `docker-compose.yaml` (`${HOST_PORT:-8080}`, `${DB_NAME:-wordpress}`), so they also work if they are missing in `.env`. The credentials have no default values on purpose: they must be set by the user.

### Non-Critical Configuration

Values that are not secret are written directly in `docker-compose.yaml`, for example the database host:

```yaml
environment:
  WORDPRESS_DB_HOST: db:3306
```

`db` is the name of the database service. Docker resolves it to the IP address of the database container inside the network.

**Example – adding another non-critical setting** (a custom table prefix):

1. Add the variable to the `wordpress` service in `docker-compose.yaml`:

   ```yaml
   WORDPRESS_TABLE_PREFIX: wp_
   ```

2. Apply the change:

   ```bash
   docker compose up -d
   ```

> [!WARNING]
> Change the table prefix only before the installation. With an existing installation, WordPress no longer finds its tables.

Other variables supported by the WordPress image are listed on [Docker Hub](https://hub.docker.com/_/wordpress).

### Changing the Port

WordPress is reachable on `HOST_PORT` (default `8080`). Inside the container, Apache always listens on port `80`. To use another port, change it in `.env`:

```dotenv
HOST_PORT=8081
```

Then run `docker compose up -d` and open the new port in the firewall.

> [!NOTE]
> Only `HOST_PORT` changes. The port inside the container stays `80`.

### Changing the Passwords

Generate a strong password:

```bash
openssl rand -base64 24
```

The passwords from `.env` are only used when the database volume is empty (first start). To start again with new passwords, remove the volume:

```bash
docker compose down -v
# change the passwords in .env
docker compose up -d
```

> [!CAUTION]
> `docker compose down -v` deletes the volume and therefore all WordPress content (posts, pages, users, settings).

### Updating the Image Versions

The image versions are set in `docker-compose.yaml`:

```yaml
image: wordpress:7.1.3-apache
image: mariadb:11.8
```

1. Look up the new tag on Docker Hub ([wordpress](https://hub.docker.com/_/wordpress), [mariadb](https://hub.docker.com/_/mariadb))
2. Replace the tag in `docker-compose.yaml`
3. Download the new image and restart:

```bash
docker compose pull
docker compose up -d
```

Other WordPress variants are available, for example with another PHP version (`7.1.3-php8.4-apache`). The `fpm` variants contain no web server and would need an additional web server container.

> [!WARNING]
> Make a backup of the database before updating. Do not switch MariaDB to an older major version with existing data.
# WordPress with Docker Compose

A Docker Compose setup for running WordPress with a MariaDB database.

## Description

This repository contains everything needed to run a WordPress website with Docker.

It uses two official Docker images:

- **`wordpress:7.1.3-apache`** – WordPress with the Apache web server and PHP 8.3
- **`mariadb:11.8`** – the MariaDB database (LTS release)

Both images use fixed versions instead of `latest`, so the setup always starts with the same software and stays reproducible.

Docker Compose starts both services in a shared network, makes WordPress available on port **8080**, stores the database in a persistent volume and restarts the containers automatically if they crash. All credentials are loaded from a local `.env` file and are never stored in the repository.

The purpose of this project is to describe a complete WordPress setup as code, so it can be started on any machine with Docker in a reproducible way.

## Table of Contents

- [Description](#description)
- [Quickstart](#quickstart)
  - [Prerequisites](#prerequisites)
  - [Start WordPress](#start-wordpress)
  - [Finish the Installation](#finish-the-installation)
- [Repository Contents](#repository-contents)
- [Usage](#usage)
  - [How It Works](#how-it-works)
  - [Configuration with the .env File](#configuration-with-the-env-file)
  - [Non-Critical Configuration](#non-critical-configuration)
  - [Changing the Port](#changing-the-port)
  - [Changing the Passwords](#changing-the-passwords)
  - [Updating the Image Versions](#updating-the-image-versions)
  - [Persistence](#persistence)
  - [Network](#network)
  - [Restart Behaviour](#restart-behaviour)
  - [Useful Commands](#useful-commands)
- [Testing](#testing)
- [Security Notes](#security-notes)



## Quickstart

### Prerequisites

- A Linux machine or cloud VM with at least **1 GB of free RAM**
- **Docker Engine** and the **Docker Compose plugin v2** (`docker compose version`)
- Port **8080/TCP** open in the firewall of the VM / cloud provider

> [!IMPORTANT]
> Use `docker compose` (with a space, v2). The old `docker-compose` (v1.29) is no longer maintained and fails with newer Docker versions (`KeyError: 'ContainerConfig'`).

### Start WordPress

```bash
# 1. Clone the repository
git clone <repository-url>
cd wordpress

# 2. Create your configuration from the template
cp .env.example .env

# 3. Set your own passwords in .env
nano .env

# 4. Start both services in the background
docker compose up -d

# 5. Check that both containers are running
docker compose ps
```

> [!IMPORTANT]
> Change `DB_PASSWORD` and `DB_ROOT_PASSWORD` in `.env` **before** the first start. MariaDB only sets the passwords when the database is created for the first time.

### Finish the Installation

1. Open `http://<VM-IP>:8080` in the browser (on your own computer: `http://localhost:8080`)
2. Choose the language
3. Enter the site title and create the **admin user** (username, password, email)
4. Click **Install WordPress** and log in with the admin user

> [!NOTE]
> The admin user is created in the browser and stored in the database. It is not configured in any file of this repository.

## Repository Contents

| File | Purpose |
|----------|---|
| `docker-compose.yaml` | Defines the services `wordpress` and `db` with image versions, port, environment variables, volume, network and restart policy. |
| `.env.example` | Template for the configuration. Copy it to `.env` and set your own values. `.env` itself is not committed. |
| `.gitignore` | Keeps secrets (`.env`, keys) and editor files out of the repository. |
| `README.md` | This documentation. |

## Usage

### How It Works

```
.env ──► docker compose up -d
                │
                ├──► db (mariadb:11.8)
                │       creates the database and the user from .env
                │       Volume db-data ──► /var/lib/mysql
                │
                └──► wordpress (wordpress:7.1.3-apache)
                        connects to db:3306
                        HOST_PORT (host) ──► 80 (container)

        both services run in the network wp-net
```

On the first start, MariaDB creates the database `DB_NAME` and the user `DB_USER` with the password `DB_PASSWORD`. WordPress uses the same values to connect to the database, so they always match.

### Configuration with the .env File

All variables that can change are configured in the `.env` file (created from `.env.example`). Docker Compose reads it automatically and inserts the values with the `${VARIABLE}` notation.

> [!NOTE]
> `.env` is your local configuration and is never committed. Only `.env.example` is part of the repository.

| Variable | Default | Secret | Description |
|---|---|---|---|
| `HOST_PORT` | `8080` | no | Port on the host where WordPress is reachable |
| `DB_NAME` | `wordpress` | no | Name of the database |
| `DB_USER` | – | yes | Database user for WordPress |
| `DB_PASSWORD` | – | yes | Password of the database user |
| `DB_ROOT_PASSWORD` | – | yes | Password of the MariaDB `root` user |

`HOST_PORT` and `DB_NAME` have default values in `docker-compose.yaml` (`${HOST_PORT:-8080}`, `${DB_NAME:-wordpress}`), so they also work if they are missing in `.env`. The credentials have no default values on purpose: they must be set by the user.

### Non-Critical Configuration

Values that are not secret are written directly in `docker-compose.yaml`, for example the database host:

```yaml
environment:
  WORDPRESS_DB_HOST: db:3306
```

`db` is the name of the database service. Docker resolves it to the IP address of the database container inside the network.

**Example – adding another non-critical setting** (a custom table prefix):

1. Add the variable to the `wordpress` service in `docker-compose.yaml`:

   ```yaml
   WORDPRESS_TABLE_PREFIX: wp_
   ```

2. Apply the change:

   ```bash
   docker compose up -d
   ```

> [!WARNING]
> Change the table prefix only before the installation. With an existing installation, WordPress no longer finds its tables.

Other variables supported by the WordPress image are listed on [Docker Hub](https://hub.docker.com/_/wordpress).

### Changing the Port

WordPress is reachable on `HOST_PORT` (default `8080`). Inside the container, Apache always listens on port `80`. To use another port, change it in `.env`:

```dotenv
HOST_PORT=8081
```

Then run `docker compose up -d` and open the new port in the firewall.

> [!NOTE]
> Only `HOST_PORT` changes. The port inside the container stays `80`.

### Changing the Passwords

Generate a strong password:

```bash
openssl rand -base64 24
```

The passwords from `.env` are only used when the database volume is empty (first start). To start again with new passwords, remove the volume:

```bash
docker compose down -v
# change the passwords in .env
docker compose up -d
```

> [!CAUTION]
> `docker compose down -v` deletes the volume and therefore all WordPress content (posts, pages, users, settings).

### Updating the Image Versions

The image versions are set in `docker-compose.yaml`:

```yaml
image: wordpress:7.1.3-apache
image: mariadb:11.8
```

1. Look up the new tag on Docker Hub ([wordpress](https://hub.docker.com/_/wordpress), [mariadb](https://hub.docker.com/_/mariadb))
2. Replace the tag in `docker-compose.yaml`
3. Download the new image and restart:

```bash
docker compose pull
docker compose up -d
```

Other WordPress variants are available, for example with another PHP version (`7.1.3-php8.4-apache`). The `fpm` variants contain no web server and would need an additional web server container.

> [!WARNING]
> Make a backup of the database before updating. Do not switch MariaDB to an older major version with existing data.

### Persistence

The named volume `db-data` is mounted to `/var/lib/mysql` in the database container. All WordPress content (posts, pages, users, settings) is stored in the database, so it survives container restarts, `docker compose down` and image updates.

Docker stores the volume in its own folder (`/var/lib/docker/volumes/`), not in the project folder. Show it with:

```bash
docker volume ls
docker volume inspect wordpress_db-data
```

Docker Compose puts the project folder name in front of the volume name (here `wordpress_`).

Create a database backup:

```bash
docker compose exec db sh -c 'mariadb-dump -u root -p"${MARIADB_ROOT_PASSWORD}" --all-databases' > backup.sql
```

> [!NOTE]
> Only the database is persisted. Files uploaded to the media library and installed plugins or themes are stored in the WordPress container (`/var/www/html`) and are lost when the container is removed. To keep them as well, add a second volume for `/var/www/html`.

### Network

Both services are connected to the network `wp-net`, defined at the end of `docker-compose.yaml`:

```yaml
networks:
  wp-net:
```

In this network, the containers find each other by service name (`db`, `wordpress`). The database port `3306` is not published to the host, so the database is only reachable from the WordPress container.

### Restart Behaviour

Both services use `restart: unless-stopped`. Docker restarts a container automatically when it stops because of an error or after a reboot of the machine. It is only not restarted when it was stopped manually (`docker compose stop`).

### Useful Commands

| Command | Description |
|---|---|
| `docker compose up -d` | Start the services or apply changes |
| `docker compose ps` | Show status and ports |
| `docker compose logs -f wordpress` | Follow the WordPress logs |
| `docker compose logs -f db` | Follow the database logs |
| `docker compose restart` | Restart both services |
| `docker compose down` | Stop and remove the containers (the data stays in the volume) |
| `docker compose exec db mariadb -u root -p` | Open the MariaDB console |

## Testing

**1. Both services are running:**

```bash
docker compose ps
```

Both services must have the status `Up`, and `wordpress` must show `0.0.0.0:8080->80/tcp`.

**2. WordPress is reachable:**

Open `http://<VM-IP>:8080` in the browser. The WordPress site or the installation page appears.

**3. Login works:**

Open `http://<VM-IP>:8080/wp-admin`, log in with the admin user and open some pages of the dashboard (Posts, Pages, Settings).

**4. The data is kept after a restart:**

1. Create a test post in the dashboard and publish it
2. Restart the setup:

   ```bash
   docker compose down
   docker compose up -d
   ```

3. Log in again: the test post and the admin user still exist

**5. The containers restart automatically after an error:**

```bash
docker inspect -f '{{.RestartCount}}' $(docker compose ps -q db)
docker compose exec db kill 1
sleep 15
docker compose ps
docker inspect -f '{{.RestartCount}}' $(docker compose ps -q db)
```

The database process is stopped without `docker stop`. Docker starts the container again and the restart counter increases by one. The same test works for WordPress (replace `db` with `wordpress`).

## Security Notes

- No passwords, tokens, usernames, IP addresses or SSH keys are stored in this repository.
- All credentials are loaded from `.env`, which is excluded by `.gitignore`.
- The credentials have no default values, so the setup never runs with a known password.
- The database port `3306` is not published to the host.
- The WordPress admin user is created in the browser and only stored in the database.

> [!TIP]
> Use a strong admin password and keep WordPress, plugins and themes up to date.

### Persistence

The named volume `db-data` is mounted to `/var/lib/mysql` in the database container. All WordPress content (posts, pages, users, settings) is stored in the database, so it survives container restarts, `docker compose down` and image updates.

Docker stores the volume in its own folder (`/var/lib/docker/volumes/`), not in the project folder. Show it with:

```bash
docker volume ls
docker volume inspect wordpress_db-data
```

Docker Compose puts the project folder name in front of the volume name (here `wordpress_`).

Create a database backup:

```bash
docker compose exec db sh -c 'mariadb-dump -u root -p"${MARIADB_ROOT_PASSWORD}" --all-databases' > backup.sql
```

> [!NOTE]
> Only the database is persisted. Files uploaded to the media library and installed plugins or themes are stored in the WordPress container (`/var/www/html`) and are lost when the container is removed. To keep them as well, add a second volume for `/var/www/html`.

### Network

Both services are connected to the network `wp-net`, defined at the end of `docker-compose.yaml`:

```yaml
networks:
  wp-net:
```

In this network, the containers find each other by service name (`db`, `wordpress`). The database port `3306` is not published to the host, so the database is only reachable from the WordPress container.

### Restart Behaviour

Both services use `restart: unless-stopped`. Docker restarts a container automatically when it stops because of an error or after a reboot of the machine. It is only not restarted when it was stopped manually (`docker compose stop`).

### Useful Commands

| Command&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Description |
|---|---|
| `docker compose up -d` | Start the services or apply changes |
| `docker compose ps` | Show status and ports |
| `docker compose logs -f wordpress` | Follow the WordPress logs |
| `docker compose logs -f db` | Follow the database logs |
| `docker compose restart` | Restart both services |
| `docker compose down` | Stop and remove the containers (the data stays in the volume) |
| `docker compose exec db mariadb -u root -p` | Open the MariaDB console |

## Testing

**1. Both services are running:**

```bash
docker compose ps
```

Both services must have the status `Up`, and `wordpress` must show `0.0.0.0:8080->80/tcp`.

**2. WordPress is reachable:**

Open `http://<VM-IP>:8080` in the browser. The WordPress site or the installation page appears.

**3. Login works:**

Open `http://<VM-IP>:8080/wp-admin`, log in with the admin user and open some pages of the dashboard (Posts, Pages, Settings).

**4. The data is kept after a restart:**

1. Create a test post in the dashboard and publish it
2. Restart the setup:

   ```bash
   docker compose down
   docker compose up -d
   ```

3. Log in again: the test post and the admin user still exist

**5. The containers restart automatically after an error:**

```bash
docker inspect -f '{{.RestartCount}}' $(docker compose ps -q db)
docker compose exec db kill 1
sleep 15
docker compose ps
docker inspect -f '{{.RestartCount}}' $(docker compose ps -q db)
```

The database process is stopped without `docker stop`. Docker starts the container again and the restart counter increases by one. The same test works for WordPress (replace `db` with `wordpress`).

## Security Notes

- No passwords, tokens, usernames, IP addresses or SSH keys are stored in this repository.
- All credentials are loaded from `.env`, which is excluded by `.gitignore`.
- The credentials have no default values, so the setup never runs with a known password.
- The database port `3306` is not published to the host.
- The WordPress admin user is created in the browser and only stored in the database.

> [!TIP]
> Use a strong admin password and keep WordPress, plugins and themes up to date.