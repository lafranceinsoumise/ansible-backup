ansible-backup
=============

[![GitHub release](https://img.shields.io/github/tag/lafranceinsoumise/ansible-backup.svg)]() 
[![Travis status](https://api.travis-ci.org/lafranceinsoumise/ansible-backup.svg?branch=master)]()

Ansible role which manage backups. Support file backups, PostgreSQL, MySQL, MongoDB and Redis backups.

Redis backup is experimental and only works with [AOF](https://redis.io/topics/persistence) disabled.

Supports export of backup status to Prometheus. Metrics are written in a directory
in order to be exposed by the [Textfile collector](https://github.com/prometheus/node_exporter#textfile-collector).

Initially forked from [Stouts.backup](https://github.com/Stouts/Stouts.backup).


## Variables

The role variables and default values.

```yaml
backup_enabled: yes             # Enable the role
backup_remove: no               # Set yes for uninstall the role from target system
backup_cron: yes                # Setup cron tasks for backup

backup_user: root               # Run backups as user
backup_group: "{{ backup_user }}"

backup_home: /etc/duply         # Backup configuration directory
backup_work: /var/duply         # Working directory

backup_duplicity_ppa: ppa:duplicity-team/ppa  # Set empty for skipping PPA addition
backup_duplicity_pkg: duplicity
backup_duplicity_version:       # Set duplicity version

# Logging
backup_logdir: /var/log/duply   # Place where logs will be keepped
backup_logrotate: yes           # Setup logs rotation
backup_prometheus: no
backup_prometheus_dir: /var/lib/node_exporter
backup_node_exporter_group: "{{ node_exporter_system_group | default('node-exp') }}"
                            # Default is compatible with cloudalchemy.node-exporter ansible role.

# Posgresql
backup_postgres_user: postgres
backup_postgres_host: ""

# Mysql
backup_mysql_user: mysql
backup_mysql_pass: ""

# Redis
backup_redis_user: redis
backup_redis_group: "{{ backup_redis_user }}"

backup_profiles: []           # Setup backup profiles
                              # Ex. backup_profiles:
                              #       - name: www               # required param
                              #         schedule: 0 0 * * 0     # if defined enabled cronjob
                              #         source: /var/www
                              #         max_age: 10D
                              #         target: s3://my.bucket/www
                              #         params:
                              #           - "BEST_PASSWORD={{ best_password }}"
                              #         exclude:
                              #           - *.pyc
                              #       - name: postgresql
                              #         schedule: 0 4 * * *
                              #         action: backup_purge         # If you want to run the purge command along with the backup one to delete obsolete backups
                              #         source: postgresql://db_name
                              #         target: s3://my.bucket/postgresql

# Default values (overide them in backup profiles bellow)
# =======================================================
# (every value can be replaced in jobs individually)

# GPG
backup_gpg_key: disabled
backup_gpg_pw: ""
backup_gpg_opts: ''

# TARGET
# syntax is
#   scheme://[user:password@]host[:port]/[/]path
# probably one out of
#   file://[/absolute_]path
#   ftp[s]://user[:password]@other.host[:port]/some_dir
#   hsi://user[:password]@other.host/some_dir
#   cf+http://container_name
#   imap[s]://user[:password]@host.com[/from_address_prefix]
#   rsync://user[:password]@other.host[:port]::/module/some_dir
#   rsync://user@other.host[:port]/relative_path
#   rsync://user@other.host[:port]//absolute_path
#   s3://host/bucket_name[/prefix]
#   ssh://user[:password]@other.host[:port]/some_dir
#   tahoe://alias/directory
#   webdav[s]://user[:password]@other.host/some_dir
backup_target: 'file:///var/backup'
# optionally the username/password can be defined as extra variables
backup_target_user:
backup_target_pass:

# Time frame for old backups to keep, Used for the "purge" command.  
# see duplicity man page, chapter TIME_FORMATS)
backup_max_age: 1M

# Number of full backups to keep. Used for the "purge-full" command.
# See duplicity man page, action "remove-all-but-n-full".
backup_max_full_backups: 1

# forces a full backup if last full backup reaches a specified age
backup_full_max_age: 1M

# set the size of backup chunks to VOLSIZE MB instead of the default 25MB.
backup_volsize: 50

# verbosity of output (error 0, warning 1-2, notice 3-4, info 5-8, debug 9)
backup_verbosity: 3

backup_exclude: [] # List of filemasks to exlude
```

## Usage

Add `lafranceinsoumise.backup` to your roles and set variables in your playbook file.

Example:

```yaml

- hosts: all

  roles:
    - lafranceinsoumise.backup

  vars:
    backup_env:
      AWS_ACCESS_KEY_ID: aws_access_key
      AWS_SECRET_ACCESS_KEY: aws_secret
    backup_profiles:

    # Backup file path
    - name: uploads                               # Required params
        schedule: 0 3 * * *                       # At 3am every day
        source: /usr/lib/project/uploads
        target: s3://s3-eu-west-1.amazonaws.com/backup.bucket/{{ inventory_hostname }}/uploads
  

    # Backup postgresql database
    - name: postgresql
        schedule: 0 4 * * *                       # At 4am every day
        source: postgresql://project              # Backup prefixes: postgresql://, mysql://, mongo://, redis://
        target: s3://s3-eu-west-1.amazonaws.com/backup.bucket/{{ inventory_hostname }}/postgresql
        user: postgres

```

### S3, Azure, Cloudfiles...

Some backends do not support the user/pass auth scheme. In this case,
you should provide the necessary environment variables through
`backup_env` or profile `env`. The following is an incomplete list from duply
documentation.

* Azure: AZURE_ACCOUNT_NAME, AZURE_ACCOUNT_KEY
* Cloudfiles: CLOUDFILES_USERNAME, CLOUDFILES_APIKEY, CLOUDFILES_AUTHURL
* Google Cloud Storage: GS_ACCESS_KEY_ID, GS_SECRET_ACCESS_KEY
* Pydrive: GOOGLE_DRIVE_ACCOUNT_KEY, GOOGLE_DRIVE_SETTINGS
* S3: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
* Swift: SWIFT_USERNAME, SWIFT_PASSWORD, SWIFT_AUTHURL, SWIFT_TENANTNAME OR SWIFT_PREAUTHURL, SWIFT_PREAUTHTOKEN

### Manage backups manually

Run backup for profile `uploads` manually:

    $ duply uploads backup

Load backup for profile `postgresql` from cloud and restore database (logged as postgres user)

    $ duply postgresql restore

Also see `duply usage`


## License

Licensed under the MIT License. See the LICENSE file for details.
