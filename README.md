# Duplicacy autobackup

Docker image to easily benefit from automated backups. It uses [duplicacy](https://github.com/gilbertchen/duplicacy) under the hood, and therefore supports:

- Multiple storage backends: S3, Backblaze B2, Hubic, Dropbox, SFTP...
- Client-side encryption
- Deduplication
- Multi-versioning
- ... and more generally, all the features that duplicacy has.

## Usage

The following environment variables can be used to configure the backup strategy.

- `BACKUP_NAME`: The name of your backup (should be unique, e.g. `prod-db-backups`)
- `BACKUP_ENCRYPTION_KEY`: An optional passphrase to encrypt your backups with before they are stored remotely.
- `BACKUP_SCHEDULE`: Cron-like string to define the frequency at which backups should be made (e.g. `0 2 * * *` for `Every day at 2am`)
- `BACKUP_LOCATION`: [Duplicacy URI](https://github.com/gilbertchen/duplicacy/wiki/Storage-Backends) of where to store the backups.
    - S3: `s3://region@amazon.com/bucket/path/to/storage`
    - Backblaze B2: `b2://my-bucket/`
    - ...

Then, the directory you want to backup must be mounted to `/data` on the container.

Additionally, you will need to provide credentials for the storage provider your of your choice:
- AWS S3: `AWS_ACCESS_KEY_ID` and `AWS_SECRET_KEY`
- Backblaze B2: `B2_ID` and `B2_KEY`
- Dropbox: `DROPBOX_TOKEN`
- Azure: `AZURE_KEY`
- Google Cloud Datastore: `GCD_TOKEN`
- SSH/SFTP: `SSH_PASSWORD` or `SSH_KEY_FILE`*
- Hubic: `HUBIC_TOKEN_FILE`*
- Google Cloud Storage: `GCS_TOKEN_FILE`*
- Onedrive: `ONEDRIVE_TOKEN_FILE`*

*Environment variables marked with an asterix point to files. Those files must be mounted in the container so that they can be accessed from inside it*.

If you want to execute an out of schedule backup, you can do so by running the script `/app/backup.sh` inside the container :

``` 
$ docker exec your-container /app/backup.sh
```

## Example

Backup `/var/lib/mysql` to the S3 bucket `xtof-db-backups` in the AWS region `eu-west-1` every night at 2:00am, and encrypt them with the passphrase `correct horse battery staple`:

```bash
$ docker run \
    -v /var/lib/mysql:/data \
    -e BACKUP_NAME='prod-db-backups' \
    -e BACKUP_LOCATION='s3://eu-west-1@amazon.com/xtof-db-backups' \
    -e BACKUP_SCHEDULE='0 2 * * *' \
    -e BACKUP_ENCRYPTION_KEY='correct horse battery staple' \
    -e AWS_ACCESS_KEY_ID='AKIA...' \
    -e AWS_SECRET_KEY='...' \
    christophetd/duplicacy-autobackup
```

## Viewing and restoring backups

Backups are useless if you don't make sure they work. This shows the procedure to list files, versions, and restore a duplicacy backup made using duplicacy-autobackup.

- Install Duplicacy: download the latest Duplicacy binary from its [Github page](https://github.com/gilbertchen/duplicacy/releases), and put it in your path

- `cd` to a directory where you'll restore your files, e.g. `/tmp/restore`

- Run `duplicacy init backup_name backup_location`, where `backup_name` and `backup_location` correspond to the `BACKUP_NAME` and `BACKUP_LOCATION` environment variables of your setup.
    - If you used client-side encryption, add the `-encrypt` flag: `duplicacy init -encrypt backup_name backup_location`

  You will get a prompt asking for your storage provider's credentials, and, if applicable, your encryption key:

  ```
  Enter S3 Access Key ID: *****
  Enter S3 Secret Access Key: *************
  Enter storage password for s3://eu-west-1@amazon.com/xtof-db-backups:*******************
  The storage 's3://eu-west-1@amazon.com/xtof-db-backups' has already been initialized
  Compression level: 100
  Average chunk size: 4194304
  Maximum chunk size: 16777216
  Minimum chunk size: 1048576
  Chunk seed: fc7e56fb91f8f66b01ba033ec6f7b128bcb3420c66a31468a4f3541407d569bd
  /tmp/restore will be backed up to s3://eu-west-1@amazon.com/xtof-db-backups with id db-backups
  ```

- To list the versions of your backups, run:

  ```
  $ duplicacy list
  Storage set to s3://eu-west-1@amazon.com/xtof-db-backups
  Enter storage password:*******************
  Snapshot db-backups revision 1 created at 2018-04-19 09:47 -hash
  Snapshot db-backups revision 2 created at 2018-04-19 09:48 
  Snapshot db-backups revision 3 created at 2018-04-19 09:49 
  ```

- To view the files of a particular revision, run:

  ```bash
  $ duplicacy list -files -r 2  # 2 is the revision number
  ```

- To restore in the current directory all the files matching `*.txt` of the revision 2 of the backup, run:

  ```bash
  $ duplicacy restore -r 2 '*.txt'
  ```

- To restore in the current directory the whole revision 2 of your backup, run:

  ```
  $ duplicacy restore -r 2
  ```

More: see [Duplicacy's documentation](https://github.com/gilbertchen/duplicacy/wiki).

## Advanced options

Use the following environment variables if you want to customize duplicacy's behavior.

- `DUPLICACY_INIT_OPTIONS`: options passed to `duplicacy init` the first time a backup is made. By default, `-encrypt` if `BACKUP_ENCRYPTION_KEY` is not empty.
- `DUPLICACY_BACKUP_OPTIONS`: options passed to `duplicacy backup` when a backup is performed. By default: `-threads 4 -stats`