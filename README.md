# crater-invoice-backup-and-restore
Instructions for backing up and restoring Crater Invoice using docker

This method is specifically for the docker installation of Crater: 
https://docs.craterapp.com/installation.html#docker-installation

Instructions on backing up and restoring Crater are slim to none. The aim of this repo is to simplify and clarify backing up and restoring, with an emphasis on restoring.

## Quick Links
- [Creating Backup](#backup)
- [Restoring Backup](#restore)


---

## Backup
There already exists a way to backup Crater using the web interface. However, we will be doing this manually.
First `cd` into your current docker compose Crater project
```
cd crater
```

Now we will create the backup using the `docker compose` plugin (if you're using the legacy docker-compose binary, simply replace `docker compose` with `docker-compose`:
```
docker compose exec db /usr/bin/mysqldump -u <user> -p<pass> <database> > <output_file>
```
| param | description |
| --- | --- |
| user | set the database username (default: `crater`) |
| pass | set the database password (default: `crater`) |
| database | set the database name (default: `crater`) |
| output_file | set the path for the backup to be saved to |

So for example:
```
docker compose exec db /usr/bin/mysqldump -u crater -pcrater crater > /mnt/my-nfs-share/crater/crater_backup.sql
```

> **Note:** There is no space after the -p argument. That is intentional.

Next we need to copy that file to our backup location, in this example we will be saving it to an nfs mounted directory.

```
docker cp <db_container_name>:<output_file> <dest_file>
```

| param | description |
| --- | --- |
| db_container_name | The name of the crater database container. This is most likely: `crater-db-1` |
| output_file | The output file we set in the previous command (`/tmp/crater_backup.sql`) |
| dest_file | The destination path for the backup to be copied to. We will just use the local tmp dir as well: `/mnt/my-nfs-share/crater/crater_backup.sql`) |

Here as an example of the full command
```
docker cp crater-db-1:/tmp/crater_backup.sql /mnt/my-nfs-share/crater/crater_backup.sql
```

If you want to double check the name of the crater db container run `docker ps --filter name=crater-db`
```
$ docker ps --filter name=crater-db
CONTAINER ID   IMAGE     COMMAND                  CREATED       STATUS       PORTS                                         NAMES
9ed6a552fa19   mariadb   "docker-entrypoint.s…"   2 hours ago   Up 2 hours   0.0.0.0:33006->3306/tcp, :::33006->3306/tcp   crater-db-1
```

The name of the container is this case is `crater-db-1`

We're not done yet, we also should copy the `.env` file. The .env file looks something like this with the less important variables left out:
```bash
APP_ENV=production
APP_KEY=base64:CUf0Orr81PRzRTT7Ba6IAxeZyqFYkrLYPY0llz8OTHM=
...
APP_URL=http://workstation.home

DB_CONNECTION=mysql
DB_HOST=crater-db-1
DB_PORT=3306
DB_DATABASE=crater
DB_USERNAME=crater
DB_PASSWORD="crater"

...

MAIL_FROM_ADDRESS=your.name@email.com
MAIL_FROM_NAME="Your Name"

...

SANCTUM_STATEFUL_DOMAINS=workstation.home
SESSION_DOMAIN=workstation.home
```

| param | description |
| --- | --- |
| APP_ENV | Ignore this variable, it will stay as production |
| APP_KEY | This variable can be left out as well, it gets populated dynamically by `php artisan` in the `setup.sh` script |
| APP_URL | The url which leads to the Cater app. Must have an `http://` or `https://` protocol prefix |
| DB_CONNECTION | Can be ignored, defaults to mysql |
| DB_HOST | This can be: `127.0.0.1` (localhost) or can be the name of the db container: `cater-db-1` from earlier |
| DB_PORT | The database port (Default: `3306`) |
| DB_DATABASE | Database name (Default: `crater`) |
| DB_USERNAME | Database username (Default: `crater`) |
| DB_PASSWORD | Database password (Default: `crater`) |
| MAIL_FROM_ADDRESS | Your email address |
| MAIL_FROM_NAME | The name you want to have associated with your email |
| SANCTUM_STATEFUL_DOMAINS | Same as your APP_URL without the protocol `http://` or `https://` |
| SESSION_DOMAIN | Same as your APP_URL without the protocol `http://` or `https://` |

None of these need to be filled out manually. These are automatically set when you complete the Crater installation wizard. Which is why we need to 
back this up for later.

```
cp .env /mnt/my-nfs-share/crater/.env
```

That's all we need for the backup. Next we'll be restoring from that backup. 

---

## Restore
In a fresh install, after cloning the Crater repo, let's `cd` into it:
```
cd crater
```

Follow the instructions for initial setup (keep in mind we will be copying over our backed up .env file so the .env step will be redundant):
https://docs.craterapp.com/installation.html#docker-installation

> This will not work until you've completed all the steps up to the `./docker-compose/setup.sh` step. Make sure you complete the initial setup.

Before we begin the restore, we should stop the app container just in case (once again replace `docker compose` with `docker-compose` if using the legacy docker-compose binary):
```
docker compose stop app
```

Next, assuming we mounted that same nfs share we had for the backup, we need to copy over the necessary files.
First we'll copy the .env file.

```
cp /mnt/my-nfs-share/crater/.env .env
```

Then we need to copy the backup file into the db container:
```
docker cp <backup_file> <db_container>:<dest_file>
```
| params | description |
| --- | --- |
| backup_file | The path of the backup file we created |
| db_container | The container name of the database. (Default: `crater-db-1`) |
| dest_file | The path inside the container to copy to |

```
docker cp /mnt/my-nfs-share/crater/backup.sql crater-db-1:/tmp/backup.sql
```

Exec a shell into the db container
```
docker compose exec db /bin/bash
```

Now we need to import the mysql dump into our new database:
```
mysql -u <user> -p<pass> <db> < <backup_file>
```
| params | description |
| --- | --- |
| user | The username of the database. (Default: `crater`) | 
| pass | The password of the database. (Default: `crater`) |
| db | The name of the database. (Default: `crater`) |
| backup_file | The path of the backup we copied to the container, in our example we used `/tmp/backup.sql` |

The full command looks like this:
```
mysql -u crater -pcrater crater < /tmp/backup.sql
```

Exit out of the container.
```
exit
```

Then we need to let the app know to skip the install process by creating an empty `database_created` file in the `storage/app` folder:
```
touch storage/app/database_created
```

Finally we can spin up the app container again
```
docker compose start app
```

The Crater app should now be restored from backup!
