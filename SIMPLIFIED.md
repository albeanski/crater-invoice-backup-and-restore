# crater-invoice-backup-and-restore (simplified)
**Simplified** instructions for backing up and restoring Crater Invoice using docker. Same as [README.md](README.md) minus some of the extra explanations.

## Quick Links
- [Creating Backup](#backup)
- [Restoring Backup](#restore)

---

## Backup
First `cd` into your current docker compose Crater project
```
cd crater
```

Create the backup:
```
docker compose exec db /usr/bin/mysqldump -u crater -pcrater crater > /mnt/my-nfs-share/crater/crater_backup.sql
```

Copy that file to our backup location:
```
docker cp crater-db-1:/tmp/crater_backup.sql /mnt/my-nfs-share/crater/crater_backup.sql
```

Copy the `.env` file:
```
cp .env /mnt/my-nfs-share/crater/.env
```

Backup complete.

---

## Restore
In a fresh install, after cloning the Crater repo, let's `cd` into it:
```
cd crater
```
Stop the app container:
```
docker compose stop app
```

Copy the backup .env file to the cloned directory.
```
cp /mnt/my-nfs-share/crater/.env .env
```

Then we need to copy the backup file into the db container:
```
docker cp /mnt/my-nfs-share/crater/backup.sql crater-db-1:/tmp/backup.sql
```

Exec a shell into the db container:
```
docker compose exec db /bin/bash
```

Now we need to import the mysql dump into our new database:
```
mysql -u crater -pcrater crater < /tmp/backup.sql
```

Exit out of the container:
```
exit
```

Create an empty `database_created` file in the `storage/app` folder:
```
touch storage/app/database_created
```

Spin up the app container again:
```
docker compose start app
```

The Crater app should now be restored from backup!
