+++
title = "Oneshot systemd tasks on NixOS"
date = 2020-08-16
[taxonomies]
tags = ["web"]
categories = ["devops"]
[extra]
discussion_id = "oneshot-systemd-tasks-on-nixos"
+++


A type of Systemd service are `oneshot` services. These are services that are expected to be short-lived (as opposed to background tasks). I have found it convenient to write global utility scripts as oneshot services, especially on NixOS, where I can deploy them to remote systems (e.g. with NixOps). For example, I wrote these nix expressions for backing up and restoring a database from/to S3.

```nix
{
  systemd.services.dbBackup = {
    environment = {
      AWS_ACCESS_KEY_ID = "XXXXXXXXXXXXXXXXXXX";
      AWS_SECRET_ACCESS_KEY = "XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX";
      AWS_BACKUP_BUCKET = "my-db-backups";
    };
    serviceConfig.Type = "oneshot";
    script = ''
      today=$(date +"%Y%m%d")
      ${pkgs.postgresql_11}/bin/pg_dump \
        --host localhost \
        --username postgres \
        --format=c \
        my-db | \
      ${pkgs.awscli}/bin/aws s3 cp \
        - \
        "s3://$AWS_BACKUP_BUCKET/backup.$today.dump"
    '';
  };
}
```

Then I can setup a timer to run this periodically:

```nix
{
  systemd.timers.dbBackup = {
    wantedBy = [ "timers.target" ];
    partOf = [ "dbBackup.service" ];
    timerConfig.OnCalendar = "*-*-* 2:30:00";
    timerConfig.Persistent = true;
  };
}
```

Now, for restoring backups, we want to be able to specify _which_ backup to restore. To do that, we need to make our service accept arguments, by exploiting the _instance name_ of the service.

From the [systemd manual](https://www.freedesktop.org/software/systemd/man/systemd.unit.html):

> Units names can be parameterized by a single argument called the "instance name". The unit is then constructed based on a "template file" which serves as the definition of multiple services or other units. A template unit must have a single "@" at the end of the name (right before the type suffix). The name of the full unit is formed by inserting the instance name between "@" and the unit type suffix. In the unit file itself, the instance parameter may be referred to using "%i"..

Here is my DB restore script. It takes the date of the backup to be restored as an argument.

```nix
{
  systemd.services."restoreDbBackup@" = {
    environment = {
      AWS_ACCESS_KEY_ID = "XXXXXXXXXXXXXXXXXXX";
      AWS_SECRET_ACCESS_KEY = "XXXXXXXXXXXXXXXXXXXXXX";
      AWS_BACKUP_BUCKET = "my-db-backups";
      BACKUP_DATE = "%i";
    };
    serviceConfig.Type = "oneshot";
    script = ''
      ${pkgs.awscli}/bin/aws s3 cp \
        "s3://$AWS_BACKUP_BUCKET/backup.$BACKUP_DATE.dump" \
        - | \
      ${pkgs.postgresql_11}/bin/pg_restore \
        --host localhost \
        --username postgres \
        --clean \
        --dbname my-db
    '';
  };
}
```

We can run this script like so:

```sh
$ systemctl start restoreDbBackup@20200822
```

### Future work

I would like to experiment with passing complex arguments to the service, e.g. flags, and the possibility of writing interactive services that accepts input from stdin.
