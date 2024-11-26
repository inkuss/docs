---
title: "Single server: upgrade"
linkTitle: "Single server: upgrade"
weight: 10
description: 
tags: ["subtopic"]
---
A single server installation is being considered a non-production installation. For a production installation some kind of orchestration or failover should be applied. A single server installation of FOLIO is useful for demo and testing purposes.

![FOLIO Single Server components](/img/single_docker_compose.png)

A FOLIO instance is divided into two main components.  The first component is Okapi, the gateway.  The second component is the UI layer which is called Stripes.  The single server with containers installation method will install both.

## Upgrade-based installation

This is a documentation for an **upgrade** of your FOLIO system. 

* It assumes that you have already successfully installed a FOLIO system and now want to upgrade your system to Poppy. 

* If you are deploying FOLIO for the first time, or if you want to start with a fresh installation for whatever reasons, see [how to do a **fresh installation**]({{< ref "singleserverfreshinstall.md" >}}) of a single server deployment.

* This documentation shows how to upgrade to a **platform-complete** distribution of Quesnelia.

* Throughout this documentation, the sample tenant "diku" will be used. Replace with the name of your tenant, as appropriate.

## System requirements

**System versions**

| **System Type**                     | **Version referred to in this manual**     |
|-------------------------------------|--------------------------------------------|
| Operating system                    | Ubuntu 22.04.2 LTS 64-bits                 |
| FOLIO system to migrate from        | Poppy CSP#3 (R2-2023-CSP-3)               |


**Hardware requirements**

| **Requirement** | **FOLIO Platform Complete** |
|-----------------|-----------------------------|
| RAM             | 40GB                        |
| CPU             | 8                           |
| HD              | 350 GB SSD                  |

## I. Before the Upgrade

### I.i) Ubuntu Upgrade
First do Ubuntu Updates & Upgrades
  
```
sudo apt-get update
sudo apt-get upgrade
sudo reboot
```
Check if all Services have been restarted after reboot: <em>Okapi, postgres, docker, the docker containers</em> (do: docker ps --all | more ) <em>, Stripes and nginx.</em>

These are the official [Quesnelia Important Upgrade Considerations](https://folio-org.atlassian.net/wiki/spaces/REL/pages/105775773/Quesnelia+R1+2024+Important+upgrade+considerations)
Do these actions before the upgrade:


### I.ii) From Quesnelia Release Notes:
#### 1.) Postgres Upgrade
Migrate from PostgreSQL 12 to a more recent version by November 14, 2024.
PostgreSQL 12 will reach end of life (no security fixes!) on November 14, 2024.
FOLIO officially supports PostgreSQL 16 from Quesnelia on.
These are the high-level steps for moving your data to a new PostgreSQL database with a higher major version:

    - das System vom Netz nehmen 
    1. Disable or suspend any applications that write to your existing database. 
    *****************************************************************************
    - Alle Anwendungen stoppen, die Datenbankverbindung zur der postgres haben:
      sudo systemctl stop okapi
      (Stripes, Kafka und Elasticsearch, sowie die postgres selber, laufen noch)

    2. Take a backup of your existing database.
    ***************************************
    # Vielleicht hier nach vorgehen: https://codebeamer.com/cb/wiki/17817868
    # Dump all roles on the source database
    cd $HOME/dbupgrade
    pg_dumpall -g > roles.postgres12.psql
    # Dump databases
    pg_dump okapi > okapi.postgres12.psql
    pg_dump folio > folio.postgres12.psql  (1,5 GB)

    - nur die Tabellen eines bestimmten Schemas dumpen:
      # -c = "clean" ; das ist (nur) sinnvoll, wenn man bestehende Datenbanken überschreiben will (hier gibt es aber keine)
      # pg_dump -U folio -cv --schema=diku_mod_circulation_storage -f folio.diku_mod_circulation_storage-create.sql --dbname=folio
      pg_dump -U folio -v --schema=diku_mod_circulation_storage -f folio.diku_mod_circulation_storage.sql --dbname=folio

    3. Create a new database with the desired version.
    ***********************************************
    # Vielleicht hier nach vorgehen: https://www.linuxbuzz.com/how-to-install-postgresql-on-ubuntu/
      # Install prerequisites
      sudo apt update
      sudo apt install  gpgv gpgsm gnupg-l10n gnupg dirmngr wget vim -y
      # Import the PostgreSQL signing key, add the PostgreSQL apt repository and install PostgreSQL.
      sudo sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
      sudo wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
      sudo apt update
      sudo apt -y install postgresql-16 postgresql-client-16 postgresql-contrib-16
      oder einfach nur
      sudo apt -y install postgresql-16 postgresql-contrib-16
  Reading package lists... Done
  Building dependency tree... Done
  Reading state information... Done
  Note, selecting 'postgresql-16' instead of 'postgresql-contrib-16'
  Some packages could not be installed. This may mean that you have
  requested an impossible situation or if you are using the unstable
  distribution that some required packages have not yet been created
  or been moved out of Incoming.
  The following information may help to resolve the situation:

  The following packages have unmet dependencies:
   postgresql-16 : Depends: postgresql-common (>= 252~) but 238 is to be installed
                   Depends: libllvm15 but it is not installable
  E: Unable to correct problems, you have held broken packages.

    21.11.2024
    # Folge https://www.postgresql.org/download/linux/ubuntu/
    apt install postgresql
    sudo apt install -y postgresql-common
    sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
    sudo apt install curl ca-certificates
    sudo install -d /usr/share/postgresql-common/pgdg
    sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc
    sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
    sudo apt update
    sudo apt -y install postgresql-16
  Reading package lists... Done
  Building dependency tree... Done
  Reading state information... Done
  Some packages could not be installed. This may mean that you have
  requested an impossible situation or if you are using the unstable
  distribution that some required packages have not yet been created
  or been moved out of Incoming.
  The following information may help to resolve the situation:

  The following packages have unmet dependencies:
   postgresql-16 : Depends: postgresql-common (>= 252~) but 238 is to be installed
                   Depends: libllvm15 but it is not installable
  E: Unable to correct problems, you have held broken packages.

    22.11.2024
     Mariusz hat im SuSE-Manager auf Ubuntu 22 umgestellt.
     Jetzt geht
        sudo apt -y install postgresql-16 postgresql-client-16 postgresql-contrib-16

    Es muss auch alles gemacht werden, was man bei der Installation von Postgres 12 gemacht hat:

    # Configure PostgreSQL to listen on all interfaces and allow connections from all addresses (to allow Docker connections).

    cd  /etc/postgresql/16/main
    Edit (via sudo) the file /etc/postgresql/16/main/postgresql.conf 
      add line listen_addresses = '*' in the "Connection Settings" section.
      In the same file, increase max_connections = 1200
      In the same file, change log_timezone and timezone, otherwise they will be UTC.
    Edit (via sudo) the file /etc/postgresql/12/main/pg_hba.conf to add line host all all 0.0.0.0/0 md5
       cp ../../12/main/pg_hba.conf .  ## aber es war wirklich nur eine Zeile hinzugefügt ==> nur eine Zeile hinzufügen.
    Restart PostgreSQL with command sudo systemctl restart postgresql
    Die postgresql-16 läuft jetzt auf dem Port
           port = 5433 in postgresql.conf
    Es laufen jetzt beide postgresql.
    Nach erfolgter Datenmigration muss der Port in der postgresql.conf auf 5432 geändert werden + Restart !
    postgresql-12 kann dann deinstalliert werden.



    4. Restore the backup to your new database.
    *******************************************
      # -b = bindir (alt), -d = datadir (alt), -o : Optionen, die direkt an das alte postgres-Kommando übergeben werden
      # -B = bindir (neu), -D = datadir (neu), -O : Optionen, die direkt an das neue postgres-Kommando übergeben werden
    pg_upgrade -b /usr/lib/postgresql/12/bin -B /usr/lib/postgresql/16/bin -d /var/lib/postgresql/12/main -D /var/lib/postgresql/16/main [option...]
    -o '-c config_file=/etc/postgresql/12/main/postgresql.conf'
    -O '-c config_file=/etc/postgresql/16/main/postgresql.conf'
    -p 5432 -P 5433
    --check --verbose
    [ -U postgresql ]

    # Migration der Daten ohne Dumps (mit pg_upgrade):
    systemctl stop postgresql
    cd /usr/folio/dbupgrade (das Verzeichnis muss postgres:postgres gehören)
    su - postgres
    ./ks.upgrade.sh # erst mit --check laufen lassen, dann ohne
    # logfiles sind in /var/lib/postgresql/16/main/pg_upgrade_output.d
    Der Upgrade war erfolgreich:
    Your installation contains extensions that should be updated with the ALTER EXTENSION command.  The file update_extensions.sql when executed by psql by the database superuser will update these extensions.
"/usr/lib/postgresql/16/bin/pg_ctl" -w -D "/var/lib/postgresql/16/main" -o "-c config_file=/etc/postgresql/16/main/postgresql.conf" -m smart stop >> "/var/lib/postgresql/16/main/pg_upgrade_output.d/20241122T121638.416/log/pg_upgrade_server.log" 2>&1

Upgrade Complete
----------------
Optimizer statistics are not transferred by pg_upgrade.
Once you start the new server, consider running:
    /usr/lib/postgresql/16/bin/vacuumdb --all --analyze-in-stages
Running this script will delete the old cluster's data files:
    ./delete_old_cluster.sh

    Jetzt die postgresql-12 deinstallieren.
       sudo apt-get --purge remove postgresql-12
       # don't optionally remove postgresql directories after the purge
       # das hat auch postgresql-17 installiert
    Anschließend  den Port der postgresql-16 in der postgresql.conf auf 5432 ändern.
      cd /etc/postgresql/16/main ; vim postgresql.conf
      auch den Port der postgresql-12 (...die hat er nicht deinstalliert) nach 5433 ändern. Postgresql 17 ist auf 5434.
    Dann Start der postgresql-16.
      erst eine Datei ~/.postgresqlrc mit folgendem Inhalt angelegt (das funktioniert wahrscheinlich nicht)
# Version Clustername Database
16 main postgres
      das für alle 3 User gemacht: folio, root, postgres.
      systemctl stop postgresql
      systemctl start postgresql
      # das geht. Die 17 ist auch gestartet, aber sei's drum. Die 12 ist nicht gestartet.
      # Die 17 auch noch entfernen:
       systemctl stop postgresql
       sudo apt-get --purge remove postgresql-17
       # don't optionally remove postgresql directories after the purge
       systemctl start postgresql
       # Jetzt scheint das System in einem sauberen Zustand zu sein
       su - postgres ; psql ; \dg; zeigt die Rollen an.
    Jetzt noch die 3 o.g. Skripte ausführen:
       su - postgres
       /usr/lib/postgresql/16/bin/vacuumdb --all --analyze-in-stages
vacuumdb: processing database "folio": Generating minimal optimizer statistics (1 target)
vacuumdb: processing database "folio_modules": Generating minimal optimizer statistics (1 target)
vacuumdb: processing database "okapi": Generating minimal optimizer statistics (1 target)
vacuumdb: processing database "postgres": Generating minimal optimizer statistics (1 target)
vacuumdb: processing database "template1": Generating minimal optimizer statistics (1 target)
vacuumdb: processing database "folio": Generating medium optimizer statistics (10 targets)
vacuumdb: processing database "folio_modules": Generating medium optimizer statistics (10 targets)
vacuumdb: processing database "okapi": Generating medium optimizer statistics (10 targets)
vacuumdb: processing database "postgres": Generating medium optimizer statistics (10 targets)
vacuumdb: processing database "template1": Generating medium optimizer statistics (10 targets)
vacuumdb: processing database "folio": Generating default (full) optimizer statistics
vacuumdb: processing database "folio_modules": Generating default (full) optimizer statistics
vacuumdb: processing database "okapi": Generating default (full) optimizer statistics
vacuumdb: processing database "postgres": Generating default (full) optimizer statistics
vacuumdb: processing database "template1": Generating default (full) optimizer statistics
       cd /usr/folio/dbupgrade
       su - postgres ; psql -f update_extensions.sql
      ./delete_old_cluster.sh # das entfernt /var/lib/postgresql/12/main

    ODER (nicht gemacht)
    cd ~/dbupgrade
    # einspielen der Dumps (nicht gemacht)
    Run VACUUM ANALYZE; command to reorganize PostgreSQL indices. For better performance, it can be started parallel using the following command: VACUUM (PARALLEL 4, ANALYZE); where 4 is the number of allocated processors in server.

    - Okapi restart
      ssh folio@folio-hbz5 ; sudo su ; systemctl start okapi.service ; das okapi-log verfolgen
    - das System wieder ans Netz bringen
       testen der beiden Mandanten; sind User- und Katalogdaten verfügbar ? Ja

      hier weiter
#### 2.) mod-data-export-spring
This manual task is only needed if at least one other tenant stays on Poppy. This manual task is not needed if all tenants are migrated to Quesnelia at the same time.
Before migrating a tenant from Poppy to Quesnelia run
UPDATE mod_data_export_spring_quartz.databasechangelog SET md5sum = '8:cd7cacfe2480c5305d1eaff157a35e4f' WHERE id = 'quartz-init' .
After the migration run
UPDATE mod_data_export_spring_quartz.databasechangelog SET md5sum = '9:89aea1286f2c2901835da380fa17eff7' WHERE id = 'quartz-init' .

Without the manual task the Poppy version of mod-data-export-spring fails and shuts down on start and restart with 
"Change failed validation!
 Error creating bean with name 'quartzSchemaInitializer'
 initial-schema.xml::quartz-init::quartz was: 9:89aea1286f2c2901835da380fa17eff7 but is now: 8:cd7cacfe2480c5305d1eaff157a35e4f"

### More preparatory steps
There might be more preparatory steps that you need to take into account for your installation. If you are unsure what other steps you might need to take, study carefully the Release Notes.  Do all actions in the column "Action required", as appropriate for your installation.


## II. Upgrade Okapi and Prepare Backend Upgrade
### II.i) Fetch a new version of platform-complete
Fetch the new release version of platform-complete, change into that directory: 
```
cd platform-complete
git fetch
```
There is a branch R1-2024-csp-6 (released on 2. Nov 2024). We will deploy this version.
Check out this Branch.
Stash local changes. This should only pertain to stripes.config.js .
Discard any changes which you might have made on the install-jsons:

```
git restore install.json
git restore okapi-install.json
git restore stripes-install.json
git restore package.json
 
git stash save
git checkout master
git pull
git checkout R1-2024-csp-6
git stash pop
```

### II.ii) Upgrade Okapi
Upgrade the Okapi version and restart Okapi.
Read the Poppy Okapi version from install.json: **okapi-5.3.0**

Set JAVA_HOME in .profile if you haven't already done so
```
  echo export JAVA_HOME=`readlink -f /usr/bin/javac | sed "s:bin/javac::"` >> ~/.profile
  source ~/.profile
```

Build Okapi from source
```
  cd /usr/folio/okapi
  git fetch
  git checkout v5.3.0
  mvn clean install -DskipTests=true
    ...
    [INFO] BUILD SUCCESS
```

If you have built Okapi from source for the first time you may need to adjust LIB_DIR in /etc/default/okapi so that the Okapi service script (/lib/systemd/system/okapi.service) will find the newly built jar file:
```
  edit /etc/default/okapi
  LIB_DIR="/usr/folio/okapi/okapi-core/target"
```

If you have built Okapi from source for the first time you need to create a file okapi.json in /etc/folio/okapi. This file complements the file okapi.conf in the same directory. Here is a such file with default values: [okapi.json]({{< ref "okapi.json" >}}).

Restart Okapi
```
  sudo systemctl restart okapi
```

Follow /var/log/folio/okapi/okapi.log . 

Now Okapi will re-start your modules. Follow the okapi.log. It will run for 5 minutes or so until all modules are up again. Check if all modules are running:

```
docker ps | grep "mod-" | wc
  65
```
The above number applies if you had installed a complete platform of Orchid, previously.

Retrieve the list of modules which are now being enabled for your tenant (just for your information):

```
curl -w '\n' -XGET http://localhost:9130/_/proxy/tenants/diku/modules
...
}, {
  "id" : "okapi-5.3.0"
} ]
```

If you are starting with a complete platform of Orchid, you will see 9 Edge modules, 59 Frontend modules (folio_\*), 65 Backend modules (mod-\*) and the Quesnelia version of Okapi (5.3.0).

### II.iii) Pull module descriptors from the central registry

A module descriptor declares the basic module metadata (id, name, etc.), specifies the module's dependencies on other modules (interface identifiers to be precise), and reports all "provided" interfaces. As part of the continuous integration process, each module descriptor is published to the FOLIO Registry at https://folio-registry.dev.folio.org.

```
curl -w '\n' -D - -X POST -H "Content-type: application/json" \
  -d '{ "urls": [ "https://folio-registry.dev.folio.org" ]  }' http://localhost:9130/_/proxy/pull/modules
```
Okapi log should show something like

```
 INFO  ProxyContent         759951/proxy REQ 127.0.0.1:58954 supertenant POST /_/proxy/pull/modules okapi-5.3.0
 INFO  PullManager          Remote registry at https://folio-registry.dev.folio.org is version 5.3.0
 INFO  PullManager          pull smart
 INFO  PullManager          pull: 1440 MDs to insert
 INFO  ProxyContent         759951/proxy RES 200 32114437us okapi-5.3.0 /_/proxy/pull/modules
```


### II.iv) Configure Module Environment Variables
This part is the one in which a system operator needs to take the most care and will probably spend the most time on.

Check your Okapi environment:

```
 curl -X GET http://localhost:9130/_/env
```

At this point, (re-)configure the environment variables of your modules, as needed.
Study the release notes for any changes in module configurations.
Follow these instructions to change the environment variables for a module: [Change Environment Variables of a Module](https://wiki.folio.org/display/SYSOPS/Change+Environment+Variables+of+a+Module).

Make sure SYSTEM_USER_PASSWORD in /\_/env is the actual password for these system users:
  - system-user
  - pub-sub
  - mod-search
  - data-export-system-user
  - mod-entities-links
  - mod-innreach   (if you are utilizing this module)

Log in as these users with this password. If that doesn't work, change the password of these system users to the value of the system variable SYSTEM_USER_PASSWORD.

#### i. Set new global environment variables
Set the ENV var -- only required in a multi-tenant environment
If you are in a multi-tenant environment, set environment variable ENV to ENV = poppy . In a single tenant environment, you don't need to set it . It has the default value ENV = folio.
```
curl -w '\n' -D - -X POST -H "Content-Type: application/json" -d "{\"name\":\"ENV\",\"value\":\"poppy\"}" http://localhost:9130/_/env
```

Set INNREACH_TENANTS - otherwise mod-inn-reach will not start for your tenant:
```
curl -w '\n' -D - -X POST -H "Content-Type: application/json" -d "{\"name\":\"INNREACH_TENANTS\",\"value\":\"diku\"}" http://localhost:9130/_/env
```
If you have multiple tenant, separate their names by "|" in the INNREACH_TENANTS variable's value.

Make sure KAFKA_HOST and KAFKA_PORT are set in /env, otherwise mod-user will fail startup.

Set DB_CONNECTION_TIMEOUT to at least 250, otherwise mod-entities-links will fail startup.
  
#### ii. Incompatible Hazelcast version in mod-remote-storage
During the upgrade, the Orchid and Poppy versions of mod-remote-storage will run in parallel. This will only work if they are on different Hazelcast clusters. Set the Hazelcast cluster name for the Poppy version to "poppy":
```
  # Configure mod-remote-storage-3.0.1 (Poppy version)
  cd ~/folio-install
  # 1. Copy the module descriptor into a file
  curl -w '\n' -D -  http://localhost:9130/_/proxy/modules/mod-remote-storage-3.0.1 > mod-remote-storage-3.0.1-module-descriptor.json
  # 2. Add env var in launch descriptor manually :
      "name" : "HZ_CLUSTERNAME", "value" : "poppy"
  # 3. Delete the module descriptor
  curl -X DELETE -D - -w '\n' http://localhost:9130/_/proxy/modules/mod-remote-storage-3.0.1
  # 4. Send the new module descriptor to /_/proxy/modules
   curl -i -w '\n' -X POST -H 'Content-type: application/json' -d @mod-remote-storage-3.0.1-module-descriptor.json http://localhost:9130/_/proxy/modules
```

#### iii. OAI-PMH (mod-oai-pmh) configuration.
Of course you only need to do this if you want to _utilize_ the oai-pmh interface of your (test or demo FOLIO) instance.
For stable operation, mod-oai-pmh  requires the following memory configuration: 
```
Java: -XX:MetaspaceSize=384m -XX:MaxMetaspaceSize=512m -Xmx2160m
Amazon Container: cpu - 2048, memory - 3072, memoryReservation - 2765
```
edge-oai-pmh memory settings remain the same as in previous releases: 
```
Java: -XX:MetaspaceSize=384m -XX:MaxMetaspaceSize=512m -Xmx1440m
Amazon Container: cpu - 1024, memory - 1512, memoryReservation - 1360
```

#### iv. S3 storage compatible environment variables
Some modules make use of an S3 storage or compatible (AWS-S3 or minion server) persistent storage. 
mod-data-export-worker has been using it since some releases ago.
Modules that now also make use of it are:
  mod-bulk-operations,
  mod-data-import - if you want to use the splitting functionality for large MARC21 import files,
  mod-oai-pmh - if you want to store error logs,
  mod-lists - in order to support exporting list contents.
  mod-entities-links
  mod-data-export
Set at least those env vars in these modules (edit the launch descriptor):
       S3_URL, S3_REGION, S3_BUCKET, S3_ACCESS_KEY_ID, S3_SECRET_ACCESS_KEY.
Remove all AWS_* vars as they are replaced by S3_*. Set
  EXPORT_IDS_BATCH = 1000
  EXPORT_FILES_MAX_POOL_SIZE = 10

#### v. Set jwt.signing.key for mod-authtoken
In the launch descriptor of mod-authtoken-2.14.1, set jwt.signing.key in the JAVA_OPTION to the same value as you have set it in the Orchid version of mod-authtoken(2.13.0) :
```
      "name" : "JAVA_OPTIONS",
      "value" : "-XX:MaxRAMPercentage=66.0 -Dcache.permissions=true -Djwt.signing.key=folio-demo"
```

Poppy doesn't support expiring tokens in the frontend, yet. Declare your tenants as legacy tenants or don't set LEGACY_TOKEN_TENANTS and TOKEN_EXPIRATION_SECONDS in mod-authtoken-2.14.1 at all:
(for details cf. the README of [mod-authtoken](https://github.com/folio-org/mod-authtoken))
```
 }, {
      "name" : "LEGACY_TOKEN_TENANTS", 
      "value" : "*"
    }, {
      "name" : "TOKEN_EXPIRATION_SECONDS",
      "value" : "tenantId:diku,accessToken:600,refreshToken:604800;tenantId:mytenant,accessToken:3600,refreshToken:604800"
```

#### vi. Set env vars for mod-search
If you have set ENV = quesnelia, set KAFKA_EVENTS_CONSUMER_PATTERN for mod-search, using the value of ENV as a part of its value:
    KAFKA_EVENTS_CONSUMER_PATTERN = (${ENV}\.)(.*\.)inventory\.(instance|holdings-record|item|bound-with)
    
If you have set ENV = folio, set  KAFKA_EVENTS_CONSUMER_PATTERN = (folio\.)(.*\.)inventory\.(instance|holdings-record|item|bound-with)

Set SEARCH_BY_ALL_FIELDS_ENABLED to "true" if you want to activate the search option "all" (search in all fields). By default, SEARCH_BY_ALL_FIELDS_ENABLED is set to "false".

#### ix. mod-pubsub
Set SYSTEM_USER_NAME in mod-pubsub. Prevous default was "pub-sub", but that fallback has been removed. Set it explicitely in mod-pubsub. Also set SYSTEM_USER_PASSWORD or use the global value in /env (if a global value has been set, this overrides the values in the launch descriptor).

#### xi. mod-agreements
Increase "Memory" in the Launch Descriptor of mod-agreements to 8 GB, if you want to connect FOLIO to GOKB.
```
  "dockerArgs" : {
      "HostConfig" : {
        "Memory" : 8589934592,
```

#### viii. E-Mail-Konfiguration für Versand über hbz-SMTP-Server in mod-email anlegen
  in den Moduldeskriptor von mod-email-1.17.0  einfügeni (aus bisherigem Moduldeskriptor kopieren):
    }, {
      "name" : "FOLIO_HOST",
      "value" : "http://folio-hbz5.hbz-nrw.de"
    }, {
      "name" : "EMAIL_FROM",
      "value" : "noreply@folio-hbz5.hbz-nrw.de"
    }, {
      "name" : "EMAIL_USERNAME",
      "value" : ""
    }, {
      "name" : "EMAIL_PASSWORD",
      "value" : ""
    }, {
      "name" : "EMAIL_SMTP_HOST",
      "value" : "listen.hbz-nrw.de"
    }, {
      "name" : "EMAIL_SMTP_PORT",
      "value" : "25"
    }, {
      "name" : "EMAIL_SMTP_LOGIN_OPTION",
      "value" : "DISABLED"
    }, {
      "name" : "EMAIL_TRUST_ALL",
      "value" : "true"
    }, {
      "name" : "EMAIL_SMTP_SSL",
      "value" : "false"
    }, {
      "name" : "EMAIL_START_TLS_OPTIONS",
      "value" : "OPTIONAL"
    }, {
      "name" : "SERVICE_POINT",
      "value" : "UUID"
    }, {
      "name" : "SELF_CHECKOUT_CONFIG",
      "value" : "config"
    }, {
      "name" : "ACS_TENANT_CONFIG",
      "value" : "config"
    }, {
      "name" : "AUTH_METHODS",
      "value" : "plain"
    }, {

### II.v) Run Pre-Upgrade Scripts
from Quesnelia Release Notes:
  mod-users-bl-7.7.4
  Before or after the Upgrade:
 Token in the reset password link expires early. Ensure that mod-users-bl’s RESET_PASSWORD_LINK_EXPIRATION_TIME (default: 24 hours) does not exceed mod-authtoken’s token.expiration.seconds (default: 10 minutes, aber von mir auf 1 Stunde hoch gesetzt)
   - eine ENV-Variable im Moduldeskriptor hinzufügen:
       }, {
      "name" : "RESET_PASSWORD_LINK_EXPIRATION_TIME",
      "value" : "3600"
   

## III. Create new Frontend : Stripes
   Wir bauen zwei Stripes-Container für die beiden Mandanten (aber sie werden noch nicht deployed):
   Wir benutzen das Dockerfile in platform-complete/docker.
   Überprüfe platform-complete/docker: vim Dockerfile nginx.conf .
   Wenn letzesmal erfolgreich installiert wurde, sollte jetzt nichts zu ändern sein. Einfach "git diff" machen.
   cd ~/platform-complete
   git diff
   vim stripes.config.js
       - aboutInstallVersion anpassen => Quesnelia
       - aboutInstallDate auf aktuelles Datum ändern
   Benutze
     // use RTR instead of insecure legacy endpoints
     // since Q, default: FALSE
     // since R, default: TRUE, cannot be overridden
     useSecureTokens: true,
   in der "config"-Sektion, um Refresh Token Rotation zu aktivieren.
   sudo su
   # Build the docker container which will contain Stripes and nginx :
   docker build -f docker/Dockerfile --build-arg OKAPI_URL=https://hbz-test.folio.hbz-nrw.de/okapi --build-arg TENANT_ID=diku -t stripes .
Sending build context to Docker daemon  61.96MB
Step 1/21 : FROM node:18-alpine as stripes_build
...
Step 21/21 : ENTRYPOINT ["/usr/bin/entrypoint.sh"]
 ---> Running in 71e5acf59c42
Removing intermediate container 71e5acf59c42
 ---> 9c2c6b792eec
Successfully built 9c2c6b792eec
Successfully tagged stripes:latest
  Das läuft ca. 10 Minuten.

   cd /usr/folio/platform-complete-bthvn
   # erstmal die neue Plattform ausleihen!
   git fetch
   git stash save
   git checkout master
   git pull
   git checkout R1-2024-csp-6
   git stash pop
   vim docker/Dockerfile docker/nginx.conf (nichts zu tun)
   vim stripes.config.js
       - den Konflikt auflösen
       - aboutInstallVersion anpassen => Quesnelia
       - aboutInstallDate auf aktuelles Datum ändern
       - useSecureTokens: true
   git add stripes.config.js
   sudo su
   docker build -f docker/Dockerfile --build-arg OKAPI_URL=https://bthvn-test.folio.hbz-nrw.de/okapi --build-arg TENANT_ID=bthvn -t stripes_bthvn .

## IV. Deploy a new FOLIO backend and enable all modules of the new platform (backend & frontend)

Now do a snapshot of your system, so you will be able to replay the current status in case the upgrade fails.
Users should stop working with the system now because after the new backend has been deployed, the front end will be incompatible to the backend and will need to be redeployed, too.

Deploy all backend modules of the new release with a single post to okapi's install endpoint. 
This will deploy and enable all new modules. Start with a simulation run:
```
  curl -w '\n' -D - -X POST -H "Content-type: application/json" -d @$HOME/platform-complete/install.json http://localhost:9130/_/proxy/tenants/diku/install?simulate=true\&preRelease=false
```
Then try to run with "deploy=true" like this.
```
  curl -w '\n' -D - -X POST -H "Content-type: application/json" -d @$HOME/platform-complete/install.json http://localhost:9130/_/proxy/tenants/diku/install?deploy=true\&preRelease=false\&tenantParameters=loadReference%3Dfalse
```
This call fails because frontend modules can not be deployed (we do this call anyway). 
You will get a message "HTTP 400 - Module folio_developer-8.0.1 has no launchDescriptor".
But this call deploys all backend modules. It did not enable any backend modules.

You can follow the progress in deployment on the terminal screen and/or in /var/log/folio/okapi/okapi.log .
Don't continue before all new modules have been deployed. Check by executing
```
docker ps | grep mod- | wc
```
This number should go up to 135 before you continue.
This number comprises of 67 backend modules from Poppy and 71 backend modules from the Quesnelia release.
There are 3 modules which are present in both releases in the same module version and have not been deployed twice ??

We finish up by enabeling all modules (backend & frontend) with a single call without deploying any.

  We here choose to load reference data for all modules . If library has changed any reference data and migrates using loadReference = true: Make a backup before migration and restore afterwards. From Quesnelia Release Notes: "Migration from Poppy to Quesnelia resets reference data. 
  Other modules don’t touch existing reference data when migrating with parameter loadReference = true, but mod-circulation-storage does.
  Most notable reference data that gets reset to default: Circulation rules."

  curl -w '\n' -D - -X POST -H "Content-type: application/json" -d @$HOME/platform-complete/install.json http://localhost:9130/_/proxy/tenants/diku/install?deploy=false\&preRelease=false\&tenantParameters=loadReference%3Dtrue
HTTP/1.1 400 Bad Request
Content-Type: text/plain
content-length: 73

No running instances for module mod-lists-2.0.6. Can not invoke /_/tenant`
   es läuft nur mod-lists-1.0.5.  Da muss was nicht hoch gekommen sein.
      hier weiter

    Das dauert 90 Sekunden pro Mandant.

Repeat this step for any other tenants (who have enabled platform-complete) on your system.

If that fails, remedy the error cause and try again until the post succeeds. 
We will take care of old modules that are not needed anymore but are still running (deployed) in the "Clean up" section.

There should now be 131 modules deployed on your single server, try
```
  docker ps | grep "mod-" | wc
```
65 of those modules belong to the Orchid release, 66 belong to the Poppy release and one module belongs to both releases.

Modules enabled for your tenant are now those of the Poppy release:
```
curl -w '\n' -XGET http://localhost:9130/_/proxy/tenants/diku/modules | grep id | wc
  141
```

This number is the sum of the following:

 Poppy release:
 - 62 Frontend modules
 - 11 Edge modules
 - 67 Backend modules
 -  1 Okapi module (5.1.2)

These are all R2-2023 (Poppy) modules.


## V. Start the new Frontend
Stop the old Stripes container: 
```
docker stop stripes
docker rm stripes
```
Start the new stripes container.
Redirect port 80 from the outside to port 80 of the docker container:
```
  cd ~/platform-complete
  sudo su
  nohup docker run -d -p 80:80 --name stripes stripes &
```
  
Clear browser cache.
Log in to your frontend: E.g., go to http://<YOUR_HOST_NAME>/ in your browser. 

  - Can you see the Poppy modules in Settings - Installation details ?

  - Do you see the right Okapi version, 5.3.0 ? 

  - Does everything look good ?

Repeat these steps for the Stripes containers of any other tenants.

It is now possible to access the system via the UI, again.
However, changes in permission sets and long-running migration jobs still need to be carried out before the system can be used productively. You may not see the inventory data, yet.


## VI. Cleanup
  
Clean up. 
Clean up your docker environment: Remove all stopped containers, all networks not used by at least one container, all dangling images and all dangling build cash:
```
  docker system prune -a
```
This command might run for some minutes.

Undeploy all unused containers: Delete all modules from Okapi's discovery which are not part of the Poppy release.
These are 64 modules of the Orchid release -- all but mod-rtac-3.5.0.

Undeploy old module versions like this:
```
curl -w '\n' -D - -X DELETE http://localhost:9130/_/discovery/modules/mod-agreements-5.5.2
curl -w '\n' -D - -X DELETE http://localhost:9130/_/discovery/modules/mod-audit-2.7.0
...
curl -w '\n' -D - -X DELETE http://localhost:9130/_/discovery/modules/mod-z3950-3.3.2
```

### Result
  for Quesnelia CSP#6
  67 backend modules, "mod-\*" are contained in the list install.json. 
  Those 67 backend modules are now enabled for your tenant(s). 
  67 containers for those backend modules are running in docker on your system.

Now, finally once again get a list of the deployed backend modules:
  
  ```
  curl -w '\n' -D - http://localhost:9130/_/discovery/modules | grep srvcId | grep mod- | wc
 67
  ```
  
Compare this with the number of your running docker containers:
  
  ```
  docker ps | grep "mod-" | wc
    67
  ```

Perform a health check
  ```
  curl -w '\n' -XGET http://localhost:9130/_/discovery/health
  ```
Is everything OK ?
  
Congratulations, your Poppy system is complete and cleaned-up now !


## VII. Post Upgrade

### VII.i) From Quesnelia Release notes
#### 1.) Update MARC-Instance Mapping (Optional)
After upgrade follow the instructions to update the mapping rules. Change is optional.
https://folio-org.atlassian.net/wiki/spaces/FOLIJET/pages/1407190/Script+to+update+mapping+rules+with+required+condition+for+specified+fields

#### 2.) Data Import Job Profiles
Run script to identify Job Profiles that need to be reviewed and corrected.
https://folio-org.atlassian.net/wiki/spaces/FOLIJET/pages/168788217

#### 3.) mod-data-export-spring
This manual task is only needed if at least one other tenant stays on Poppy. This manual task is not needed if all tenants are migrated to Quesnelia at the same time.
Before migrating a tenant from Poppy to Quesnelia run
UPDATE mod_data_export_spring_quartz.databasechangelog SET md5sum = '8:cd7cacfe2480c5305d1eaff157a35e4f' WHERE id = 'quartz-init' .
After the migration run
UPDATE mod_data_export_spring_quartz.databasechangelog SET md5sum = '9:89aea1286f2c2901835da380fa17eff7' WHERE id = 'quartz-init' .

Without the manual task the Poppy version of mod-data-export-spring fails and shuts down on start and restart with 
"Change failed validation!
 Error creating bean with name 'quartzSchemaInitializer'
 initial-schema.xml::quartz-init::quartz was: 9:89aea1286f2c2901835da380fa17eff7 but is now: 8:cd7cacfe2480c5305d1eaff157a35e4f"


### VII.ii) Recreate OpenSearch or Elasticsearch index
Sometimes we need to recreate OpenSearch or Elasticsearch index, for example when a breaking change has been introduced to index structure (mapping). We must re-index after migrating to Poppy. It can be fixed by running reindex request:

  Assure the following permission has been assigned to user diku_admin:
    search.index.inventory.reindex.post (Search - starts inventory reindex operation)

  Get a new Token and re-index:
```
   export TOKEN=$( curl -s -S -D - -H "X-Okapi-Tenant: diku" -H "Content-type: application/json" -H "Accept: application/json" -d '{ "tenant" : "diku", "username" : "diku_admin", "password" : "admin" }' http://localhost:9130/authn/login | grep -i "^x-okapi-token: " )
  curl -w '\n' -D - -X POST -H "$TOKEN" -H "X-Okapi-Tenant: diku" -H "Content-type: application/json" -d '{ "recreateIndex": true, "resourceName": "instance" }' http://localhost:9130/search/index/inventory/reindex
```
 ### Monitoring reindex process ( https://github.com/folio-org/mod-search#monitoring-reindex-process )

There is no end-to-end monitoring implemented yet, however it is possible to monitor it partially. In order to check how many records published to Kafka topic use inventory API:
```
    curl -w '\n' -D - -X GET -H "$TOKEN" -H "X-Okapi-Tenant: diku" -H "Content-type: application/json" http://localhost:9130/instance-storage/reindex/{id}
```
Repeat the re-indexing process for other tenants that you might host on your server and have also migrated to Poppy.

### VII.iii) Update permissions
Update permissions as described in the [Permissions Updates](https://folio-org.atlassian.net/wiki/spaces/REL/pages/105775925/Quesnelia+R1+2024+Permissions+Updates).

### VII.iv) Manual Tests
- Login to the frontend, user=diku_admin, passwd=admin
- Delete browser cache (this is important, otherwise you will see the old frontend modules)
- Go to Settings page 
    check against "incompatible interface versions". There should be none.
    check if "Quesnelia CSP-6" is being displayed on the settings page
- check if circulation log can be downloaded (this checks if mod-data-export-worker works)
- check if emails are being sent (do a checkout for a test user)


Congratulation, your system is ready!
