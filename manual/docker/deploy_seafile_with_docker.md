# Deploy Seafile with Docker

> The official Docker image uses multiple containers. If you need to run Seafile and related database in a single container for Synology platform, you can try the docker image provided by community: <https://github.com/codeoutin/seafile7/>

## Getting started

### Install docker

Seafile v7.x.x (or newer versions) image uses docker. You should first install docker.

[Install Docker Engine on CentOS](https://docs.docker.com/engine/install/centos/)

[Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)

### Install docker-compose

Seafile v7.x.x (or newer versions) image uses docker-compose. You should install the docker-compose command.

```bash
# for CentOS
yum install docker-compose -y

# for Ubuntu
apt-get install docker-compose -y

```

### Download and modify docker-compose.yml

Download [docker-compose.yml](https://download.seafile.com/d/320e8adf90fa43ad8fee/files/?p=/docker/docker-compose.yml) sample file to your host. Then modify the file according to your environment. The following fields are needed to be modified:

* The password of MySQL root (MYSQL_ROOT_PASSWORD and DB_ROOT_PASSWD)
* The volume directory of MySQL data (volumes)
* The volume directory of Seafile data (volumes).

### Start Seafile server

Start Seafile server with the following command

```bash
docker-compose up -d

```

Wait for a few minutes for the first time initialization, then visit `http://seafile.example.com` to open Seafile Web UI.

**NOTE: You should run the above command in a directory with the **`docker-compose.yml`**.**

## More configuration options

### Custom admin username and password

The default admin account is `me@example.com` and the password is `asecret`. You can use a different password  by setting the container's environment variables in the `docker-compose.yml`:
e.g.

```
seafile:
    ...

    environment:
        ...
        - SEAFILE_ADMIN_EMAIL=me@example.com
        - SEAFILE_ADMIN_PASSWORD=a_very_secret_password
        ...

```

### Let's encrypt SSL certificate

If you set `SEAFILE_SERVER_LETSENCRYPT` to `true`, the container would request a letsencrypt-signed SSL certificate for you automatically.

e.g.

```
seafile:
    ...
    ports:
        - "80:80"
        - "443:443"
    ...
    environment:
        ...
        - SEAFILE_SERVER_LETSENCRYPT=true
        - SEAFILE_SERVER_HOSTNAME=docs.seafile.com
        ...

```

If you want to use your own SSL certificate and the volume directory of Seafile data is `/opt/seafile-data`:

* create a folder `/opt/seafile-data/ssl`, and put your certificate and private key under the ssl directory.
* Assume your site name is `seafile.example.com`, then your certificate must have the name `seafile.example.com.crt`, and the private key must have the name `seafile.example.com.key`.

If you got the following error when SEAFILE_SERVER_LETSENCRYPT=true is set:

```log
subprocess.CalledProcessError: Command '/scripts/ssl.sh /shared/ssl cloud.seafile-demo.de' returned non-zero exit status 128.
```

In /scripts/ssl.sh (script in seafile container), `git clone git://` has to be replaced with `git clone https://`.

Then restart the container:

```shell
docker-compose restart
```

Since version 9.0.6, we use acme (not acme-tiny) to get certificate and fix this error.

### Modify Seafile server configurations

The config files are under `shared/seafile/conf`. You can modify the configurations according to [Seafile manual](https://manual.seafile.com/)

After modification, you need to restart the container:

```bash
docker-compose restart

```

### Find logs

To view Seafile docker logs, please use the following command

```shell
docker-compose logs -f
```

The Seafile logs are under `shared/logs/seafile` in the docker, or `/opt/seafile-data/logs/seafile` in the server that run the docker.

The system logs are under `shared/logs/var-log`, or `/opt/seafile-data/logs/var-log` in the server that run the docker.

### Add a new admin

Ensure the container is running, then enter this command:

```
docker exec -it seafile /opt/seafile/seafile-server-latest/reset-admin.sh

```

Enter the username and password according to the prompts. You now have a new admin account.

## Seafile directory structure

### `/shared`

Placeholder spot for shared volumes. You may elect to store certain persistent information outside of a container, in our case we keep various log files and upload directory outside. This allows you to rebuild containers easily without losing important information.

* /shared/seafile: This is the directory for seafile server configuration and data.
* /shared/logs: This is the directory for logs.
  * /shared/logs/var-log: This is the directory that would be mounted as `/var/log` inside the container. For example, you can find the nginx logs in `shared/logs/var-log/nginx/`.
  * /shared/logs/seafile: This is the directory that would contain the log files of seafile server processes. For example, you can find seaf-server logs in `shared/logs/seafile/seafile.log`.
* /shared/ssl: This is directory for certificate, which does not exist by default.
* /shared/bootstrap.conf: This file does not exist by default. You can create it by your self, and write the configuration of files similar to the `samples` folder.

## Upgrading Seafile server

To upgrade to latest version of seafile server:

```sh
docker pull seafileltd/seafile-mc:latest
docker-compose down
docker-compose up -d

```

## Backup and recovery

### Structure

We assume your seafile volumns path is in `/opt/seafile-data`. And you want to backup to `/opt/seafile-backup` directory.
You can create a layout similar to the following in /opt/seafile-backup directory:

```
/opt/seafile-backup
---- databases/  MySQL contains database backup files
---- data/  Seafile contains backups of the data directory

```

The data files to be backed up:

```
/opt/seafile-data/seafile/conf  # configuration files
/opt/seafile-data/seafile/seafile-data # data of seafile
/opt/seafile-data/seafile/seahub-data # data of seahub

```

### Backup

Steps:

1. Backup the databases;
2. Backup the seafile data directory;

Backup Order: Database First or Data Directory First

#### backing up Database:

```bash
# It's recommended to backup the database to a separate file each time. Don't overwrite older database backups for at least a week.
cd /opt/seafile-backup/databases
docker exec -it seafile-mysql mysqldump  -uroot --opt ccnet_db > ccnet_db.sql
docker exec -it seafile-mysql mysqldump  -uroot --opt seafile_db > seafile_db.sql
docker exec -it seafile-mysql mysqldump  -uroot --opt seahub_db > seahub_db.sql
```

#### Backing up Seafile library data:

##### To directly copy the whole data directory

```bash
cp -R /opt/seafile-data/seafile /opt/seafile-backup/data/
cd /opt/seafile-backup/data && rm -rf ccnet
```

##### Use rsync to do incremental backup

```bash
rsync -az /opt/seafile-data/seafile /opt/seafile-backup/data/
cd /opt/seafile-backup/data && rm -rf ccnet
```

### Recovery

#### Restore the databases:

```bash
docker cp /opt/seafile-backup/databases/ccnet_db.sql seafile-mysql:/tmp/ccnet_db.sql
docker cp /opt/seafile-backup/databases/seafile_db.sql seafile-mysql:/tmp/seafile_db.sql
docker cp /opt/seafile-backup/databases/seahub_db.sql seafile-mysql:/tmp/seahub_db.sql

docker exec -it seafile-mysql /bin/sh -c "mysql -uroot ccnet_db < /tmp/ccnet_db.sql"
docker exec -it seafile-mysql /bin/sh -c "mysql -uroot seafile_db < /tmp/seafile_db.sql"
docker exec -it seafile-mysql /bin/sh -c "mysql -uroot seahub_db < /tmp/seahub_db.sql"
```

### Restore the seafile data:

```bash
cp -R /opt/seafile-backup/data/* /opt/seafile-data/seafile/
```

## Garbage collection

When files are deleted, the blocks comprising those files are not immediately removed as there may be other files that reference those blocks (due to the magic of deduplication). To remove them, Seafile requires a ['garbage collection'](https://manual.seafile.com/maintain/seafile_gc.html) process to be run, which detects which blocks no longer used and purges them. (NOTE: for technical reasons, the GC process does not guarantee that _every single_ orphan block will be deleted.)

The required scripts can be found in the `/scripts` folder of the docker container. To perform garbage collection, simply run `docker exec seafile /scripts/gc.sh`. For the community edition, this process will stop the seafile server, but it is a relatively quick process and the seafile server will start automatically once the process has finished. The Professional supports an online garbage collection.

## FAQ

**You can run docker commands like "docker exec" to find errors.**

```sh
docker exec -it seafile /bin/bash
```

**LetsEncrypt SSL certificate is about to expire.**

If the certificate is not renewed automatically, you can execute the following command to manually renew the certificate.

```sh
/scripts/ssl.sh /shared/ssl/ <your-seafile-domain>
```

eg: ```/scripts/ssl.sh /shared/ssl/ example.seafile.com```

**SEAFILE_SERVER_LETSENCRYPT=false change to true.**

If you want to change to https after using http, first backup and move the seafile.nginx.conf.

```sh
mv /opt/seafile/shared/nginx/conf/seafile.nginx.conf /opt/seafile/shared/nginx/conf/seafile.nginx.conf.bak
```

Starting the new container will automatically apply a certificate.

```sh
docker-compose down
docker-compose up -d
```

You need to manually change http to https in other configuration files, SERVICE_URL and FILE_SERVER_ROOT in the system admin page also need to be modified.

If you have modified the old seafile.nginx.conf, now you can modify the new seafile.nginx.conf as you want. Then execute the following command to make the nginx configuration take effect.

```sh
docker exec seafile nginx -s reload
```
