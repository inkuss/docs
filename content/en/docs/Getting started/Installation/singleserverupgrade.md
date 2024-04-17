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

* This documentation shows how to upgrade to a **platform-complete** distribution of Orchid.

* Throughout this documentation, the sample tenant "diku" will be used. Replace with the name of your tenant, as appropriate.

## System requirements

**System versions**

| **System Type**                     | **Version referred to in this manual**     |
|-------------------------------------|--------------------------------------------|
| Operating system                    | Ubuntu 22.04.2 LTS 64-bits                 |
| FOLIO system to migrate from        | Orchid CSP#5 (R1-2023-CSP-5)               |


**Hardware requirements**

| **Requirement** | **FOLIO Platform Complete** |
|-----------------|-----------------------------|
| RAM             | 40GB                        |
| CPU             | 8                           |
| HD              | 350 GB SSD                  |

## I. Before the Upgrade

First do a snapshot of your system, so you will be able to replay the current status in case the upgrade fails.

First do Ubuntu Updates & Upgrades
  
```
sudo apt-get update
sudo apt-get upgrade
sudo reboot
```
Check if all Services have been restarted after reboot: <em>Okapi, postgres, docker, the docker containers</em> (do: docker ps --all | more ) <em>, Stripes and nginx.</em>

These are the [Poppy Release Notes](https://folio-org.atlassian.net/wiki/spaces/REL/pages/5210730/Poppy+R2+2023+Release+Notes)
Do these actions before the upgrade:

From Poppy Release Notes:

### More preparatory steps
There might be more preparatory steps that you need to take into account for your installation. If you are unsure what other steps you might need to take, study carefully the Release Notes.  Do all actions in the column "Action required", as appropriate for your installation.


## II. Reinstall the Backend, Migrate from Orchid to Poppy
### II.i) Fetch a new version of platform-complete
Fetch the new release version of platform-complete, change into that directory: 
```
cd platform-complete
git fetch
```
There is a branch R2-2023-csp-3 (released on April 11, 2024). We will deploy this version.
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
git checkout R2-2023-csp-3
git stash pop
```

### II.ii) Upgrade Okapi
Upgrade the Okapi version and restart Okapi.
Read the Poppy Okapi version from install.json: **okapi-5.1.2**

Update the Okapi Debian package:
```
sudo add-apt-repository "deb https://repository.folio.org/packages/ubuntu focal/"
sudo apt-get update
sudo apt-get -y --allow-change-held-packages install okapi=5.1.2-1
```
   E: Version '5.1.2-1' for 'okapi' was not found

  Okapi im Container bauen 
  # Okapi ausleihen: https://github.com/folio-org/okapi/tree/master/okapi-core ==> braucht man nicht, ich nehme ein Image
  # git clone https://github.com/folio-org/okapi.git
  # cd okapi
  # git checkout v5.1.2
  # cd okapi-core
  Die okapi.conf ins JSON-Format umwandeln => okapi.json
  sudo su
  cd /etc/folio/okapi
Hier eine Vorlage für okapi.json:
{
  "role": "dev",
  "enable_metrics": 0,
  "carbon_host": "localhost",
  "carbon_port": "2003",
  "port": "9130",
  "port_start": "9400",
  "port_end": "10400",
  "host": "<YOUR_IP_ADDRESS>",
  "storage": "postgres",
  "postgres_host": "<YOUR_IP_ADDRESS>",
  "postgres_port": "5432",
  "postgres_username": "<DB_OKAPI_USER>",
  "postgres_password": "<DB_OKAPI_PW>",
  "postgres_database": "<DB_OKAPI_DB>",
  "dockerurl": "http://localhost:4243",
  "okapiurl": "http://<YOUR_IP_ADDRESS>:9130",
  "token_cache_max_size": "10000",
  "token_cache_ttl_ms": "180000",
  "deploy_waitIterations": "60",
  "loglevel": "INFO",
  "log4j_config": "okapi/log4j2.properties",
  "vertx_cache_dir_base": "/tmp/vertx-cache-okapi"
}

```
  sudo su
  nohup docker run -p 9130:9130 --network=host -e JAVA_OPTIONS="-Dloglevel=DEBUG" -v /etc/folio/okapi:/usr/verticles/okapi --name okapi folioorg/okapi:5.1.2 dev -conf /usr/verticles/okapi/okapi.json & 
```
    # Braucht man auch die env vars in /usr/folio/okapi/dist/okapi.env ?
    # => okapi.sh, okapi.env wird nicht mehr benötigt (in https://github.com/folio-org/okapi/tree/master/dist )

 
    ALTERNATIV: Alle Env-Variablen einzeln mit -e an docker übergeben (klingt eher komplizierter)
    ODER die okapi.conf in dem Dockerfile kopieren (dann muss ich aber den Container aber selber bauen)
    "Frage der Philosophie" Florian Kreft: "Besser das bereitgestellte Image benutzen und conf-Datei mitschicken".
    Das Log auf stdout umleiten; dann das log mit docker logs <okapi> angucken. // das scheint schon standardmäßig auf stdout geleitet zu werden.

docker logs -f okapi
WARN  ?                    Deployment of mod-<irgendwas> failed (ignored): null
11:08:31 [] [] [] [] ERROR ?                    No running instances for module mod-permissions-6.3.2. Can not invoke /_/tenantpermissions

Alternatives Deployen mit docker-compose:
cd /usr/folio/okapi-docker-compose
cat docker-compose.yml 
version: '3'

services:
  okapi:
    image: "folioorg/okapi:5.1.2"
    container_name: okapi
    volumes:
      - /etc/folio/okapi:/usr/verticles/okapi:ro
    network_mode: host
    restart: always
    command: "dev -conf okapi/okapi.json"
    environment:
      JAVA_OPTIONS: "-Djava.awt.headless=true"
    labels:
      service: okapi
sudo su
docker-compose up -d
Die Fehlermeldungen bleiben dieselben wie beim Start mit docker.


Make sure Okapi's port range doesn't collide with Elasticsearch's standard port 9200, if you are using that port for ES.
Change Okapi's port range, edit okapi.conf:
```
cd /etc/folio/okapi
edit okapi.conf
   port_start="19000"
   port_end="20000"
```

Check Okapi logging properties in /etc/folio/okapi. Check that this line is in okapi.conf:
```
log4j_config="/etc/folio/okapi/log4j2.properties"
```
Edit log4j2.properties. Make sure Okapi logs into a file and define a RollingFileAppender.

Restart Okapi:

```
sudo systemctl daemon-reload
sudo systemctl restart okapi.service
```

Follow /var/log/folio/okapi/okapi.log . 

Now Okapi will re-start your modules. Follow the okapi.log. It will run for 5 minutes or so until all modules are up again. Check if all modules are running:

```
docker ps | grep "mod-" | wc
  62
```
The above number applies if you had installed a complete platform of Nolana.

Retrieve the list of modules which are now being enabled for your tenant (just for your information):

```
curl -w '\n' -XGET http://localhost:9130/_/proxy/tenants/diku/modules
...
}, {
  "id" : "okapi-5.0.1"
} ]
```

If you are starting with a complete platform of Nolana, you will see 9 Edge modules, 59 Frontend modules (folio_\*), 62 Backend modules (mod-\*) and the Orchid version of Okapi (5.0.1).

### II.iii) Pull module descriptors from the central registry

A module descriptor declares the basic module metadata (id, name, etc.), specifies the module's dependencies on other modules (interface identifiers to be precise), and reports all "provided" interfaces. As part of the continuous integration process, each module descriptor  is published to the FOLIO Registry at https://folio-registry.dev.folio.org.

```
curl -w '\n' -D - -X POST -H "Content-type: application/json" \
  -d '{ "urls": [ "https://folio-registry.dev.folio.org" ]  }' http://localhost:9130/_/proxy/pull/modules
```
Okapi log should show something like

```
 INFO  ProxyContext         510602/proxy REQ 127.0.0.1:49950 supertenant POST /_/proxy/pull/modules  okapi-5.0.1
 INFO  PullManager          Remote registry at https://folio-registry.dev.folio.org is version 5.0.1
 INFO  PullManager          pull smart
  ...
 INFO  PullManager          pull: 3653 MDs to insert
 INFO  ProxyContent         510602/proxy RES 200 38836856us okapi-5.0.1 /_/proxy/pull/modules
```


### II.iv) Pre-Upgrade

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

Log in as these users with this password. If that doesn't work, change the password of these system users to the value of the system variable SYSTEM_USER_PASSWORD.

If you are in a multi-tenant environment, set environment variable ENV to ENV = orchid . In a single tenant environment, you don't need to set it . It has the default value ENV = folio.
```
curl -w '\n' -D - -X POST -H "Content-Type: application/json" -d "{\"name\":\"ENV\",\"value\":\"orchid\"}" http://localhost:9130/_/env
```
  
From Orchid release notes, do these steps:
### i. Incompatible Hazelcast version in mod-remote-storage
When running Nolana and Orchid version of mod-remote-storage in parallel they need distinct Hazelcast configurations. One possibility is to use different Hazelcast cluster name environment variables:
```
HZ_CLUSTERNAME=nolana (mod-remote-storage-1.7.2)
HZ_CLUSTERNAME=orchid (mod-remote-storage-2.0.3)
```

### ii. Data Import jobs for update of >5000 MARC records
If you expect to have these kind of data import jobs, increase database connection pool size for mod-source-record-manager-3.6.4 and mod-source-record-storage to 30 (default value is 15). Set the environment variable:
```
{
          "name": "DB_MAXPOOLSIZE",
          "value": "30"
}
```
Set the newly provided "DB_CONNECTION_TIMEOUT" environment variable to 40 for mod-source-record-storage-5.6.10:
```
{
           "name":"DB_CONNECTION_TIMEOUT",
           "value": "40"
}
```
### iii. OAI-PMH (mod-oai-pmh) configuration.
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

### iv. Set jwt.signing.key for mod-authtoken
In the Launch Descriptor of mod-authtoken-2.13.0, set jwt.signing.key in the JAVA_OPTION to the same value as you have set it in the Nolana version of mod-authtoken(-2.12.0) :
```
      "name" : "JAVA_OPTIONS",
      "value" : "-XX:MaxRAMPercentage=66.0 -Dcache.permissions=true -Djwt.signing.key=folio-demo"
```

### v. Set env vars for mod-search
If you have set ENV = orchid, set KAFKA_EVENTS_CONSUMER_PATTERN for mod-search, using the value of ENV as a part of its value:
    KAFKA_EVENTS_CONSUMER_PATTERN = (orchid\.)(.*\.)inventory\.(instance|holdings-record|item|bound-with)
    
If you have set ENV = folio, set  KAFKA_EVENTS_CONSUMER_PATTERN = (folio\.)(.*\.)inventory\.(instance|holdings-record|item|bound-with)

Set SEARCH_BY_ALL_FIELDS_ENABLED to "true" if you want to activate the search option "all" (search in all fields). By default, SEARCH_BY_ALL_FIELDS_ENABLED is set to "false".

### II.v) Deploy a new FOLIO backend and enable all modules of the new platform (backend & frontend)

Deploy all backend modules of the new release with a single post to okapi's install endpoint. 
This will deploy and enable all new modules. Start with a simulation run:
```
  curl -w '\n' -D - -X POST -H "Content-type: application/json" -d @/usr/folio/platform-complete/install.json http://localhost:9130/_/proxy/tenants/diku/install?simulate=true\&preRelease=false
```
Then try to run with "deploy=true" like this.
Use loadReference%3Dfalse if you have changed reference data to local values in your installation.
Use loadReference%3Dtrue if your reference data is in the initial state.
If you do loadReference%3Dfalse, new reference data will not be loaded and you will need to load them manually after the upgrade process.
If you do loadReference%3Dtrue, your local changes to reference data might become overwritten and you will need to correct them later.
```
  curl -w '\n' -D - -X POST -H "Content-type: application/json" -d @/usr/folio/platform-complete/install.json http://localhost:9130/_/proxy/tenants/diku/install?deploy=true\&preRelease=false\&tenantParameters=loadReference%3Dfalse
```
This call fails because frontend modules can not be deployed (we do this call anyway). 
You will get a message "HTTP 400 - Module folio_developer-7.0.0 has no launchDescriptor".
But this call deploys all backend modules. 

You can follow the progress in deployment on the terminal screen and/or in /var/log/folio/okapi/okapi.log .
Don't continue before all new modules have been deployed. Check by executing
```
docker ps | grep mod- | wc
```
This number should go up to 126 before you continue.

We finish up by enabeling all modules (backend & frontend) with a single call without deploying any. We don't load reference data because we are doing a system upgrade (reference data have been loaded before):
```
  curl -w '\n' -D - -X POST -H "Content-type: application/json" -d @/usr/folio/platform-complete/install.json http://localhost:9130/_/proxy/tenants/diku/install?deploy=false\&preRelease=false\&tenantParameters=loadReference%3Dfalse
```
Repeat this step for any other tenants (who have enabled platform-complete) on your system.

If that fails, remedy the error cause and try again until the post succeeds. 
We will take care of old modules that are not needed anymore but are still running (deployed) in the "Clean up" section.

There should now be 126 modules deployed on your single server, try
```
  docker ps | grep "mod-" | wc
```
62 of those modules belong to the Nolana release, 65 belong to the Orchid release.
1 module appears with the same version number in both releases, it has not been deployed twice ( mod-template-engine:1.18.0 ).

Modules enabled for your tenant are now those of the Orchid release:
```
curl -w '\n' -XGET http://localhost:9130/_/proxy/tenants/diku/modules | grep id | wc
  134
```

This number is the sum of the following:

 Orchid release:
 - 59 Frontend modules
 -  9 Edge modules
 - 65 Backend modules
 -  1 Okapi module (5.0.1)

These are all R1-2023 (Orchid) modules.


### II.vi) Cleanup
  
Clean up. 
Clean up your docker environment: Remove all stopped containers, all networks not used by at least one container, all dangling images and all dangling build cash:
```
  docker system prune -a
```
This command might run for some minutes.

Undeploy all unused containers: Delete all modules from Okapi's discovery which are not part of the Orchid release.
These are 61 modules of the Nolana release -- all but mod-template-engine:1.18.0 (this one is also part of Orchid).

Undeploy old module versions like this:
```
curl -w '\n' -D - -X DELETE http://localhost:9130/_/discovery/modules/mod-agreements-5.4.4
curl -w '\n' -D - -X DELETE http://localhost:9130/_/discovery/modules/mod-audit-2.6.2
...
curl -w '\n' -D - -X DELETE http://localhost:9130/_/discovery/modules/mod-z3950-3.1.0
```

### Result
  for Orchid CSP#5
  65 backend modules, "mod-\*" are contained in the list install.json. 
  Those 65 backend modules are now being enabled for your tenant(s). 
  65 containers for those backend modules are running in docker on your system.

Now, finally once again get a list of the deployed backend modules:
  
  ```
  curl -w '\n' -D - http://localhost:9130/_/discovery/modules | grep srvcId | grep mod- | wc
 65
  ```
  
Compare this with the number of your running docker containers:
  
  ```
  docker ps | grep "mod-" | wc
    65
  ```

Perform a health check
  ```
  curl -w '\n' -XGET http://localhost:9130/_/discovery/health
  ```
Is everything OK ?
  
Congratulations, your Orchid backend is complete and cleaned-up now ! Now, you have to set up a new Stripes instance for the frontend of the tenant.


## III. Frontend installation : Stripes

Install Stripes and nginx in a Docker container. Use the docker file in platform-complete/docker.
Check if everything looks o.k. in platform-complete/docker. 
If you have successfully installed last time, you should not need to change anything. Just do a "git diff".

```
cd ~/platform-complete
git diff
```

Check if docker/Dockerfile, docker/nginx.conf and stripes.conifg.js look o.k.

Build the docker container which will contain Stripes and nginx :
  
```
  docker build -f docker/Dockerfile --build-arg OKAPI_URL=http(s)://<YOUR_DOMAIN_NAME>/okapi --build-arg TENANT_ID=diku -t stripes .
Sending build context to Docker daemon  61.96MB
Step 1/21 : FROM node:18-alpine as stripes_build
...
Step 21/21 : ENTRYPOINT ["/usr/bin/entrypoint.sh"]
 ---> Running in 71e5acf59c42
Removing intermediate container 71e5acf59c42
 ---> 9c2c6b792eec
Successfully built 9c2c6b792eec
Successfully tagged stripes:latest
```

This will run for approximately 10 minutes. 

Stop the old Stripes container: docker stop <container id of your old stripes container which is still running>

Start the new stripes container:
 
  Redirect port 80 from the outside to port 80 of the docker container:
  
```
  nohup docker run -d -p 80:80 stripes
```
  
Log in to your frontend: E.g., go to http://<YOUR_HOST_NAME>/ in your browser. 

  - Can you see the Orchid modules in Settings - Installation details ?

  - Do you see the right okapi version, 5.0.1-1 ? 

  - Does everything look good ?

If so, remove the old stripes container: docker rm \<container id of your old stripes container\> .

## IV. Post Upgrade
From Orchid release notes:
### i. Additional cluster-level permissions are required in Opensearch 
Before initializing mod-search 2.0.x for a tenant,  add the following cluster-level permissions to the Opensearch role used by mod-search:   
```
cluster:admin/script/get
cluster:admin/script/put
cluster:admin/script/delete
```
This change may also impact Elasticsearch as well (this is unveryfied, however).
For an unsecured Elasticsearch installation, no action is required.

### ii. **Breaking Change** : Instance data in mod-inventory-storage have to be migrated. 

To initialize migration use the endpoint POST /inventory-storage/migrations/jobs with body.
First get a new Token:
```
  export TOKEN=$( curl -s -S -D - -H "X-Okapi-Tenant: diku" -H "Content-type: application/json" -H "Accept: application/json" -d '{ "tenant" : "diku", "username" : "diku_admin", "password" : "admin" }' http://localhost:9130/authn/login | grep -i "^x-okapi-token: " )
  curl -w '\n' -D - -X POST -H "$TOKEN" -H "X-Okapi-Tenant: diku" -H "Content-type: application/json" -d '{ "migrations" : [ "subjectSeriesMigration" ], "affectedEntities": [ "INSTANCE" ] }' http://localhost:9130/inventory-storage/migrations/jobs
```
Migrate any other tenants in the same way.
To check the status of migration use the endpoint GET /inventory-storage/migrations/jobs/<id> where id - is the id from the POST response.
Migration could be done after the upgrade.
Migration could be sped up with scaling up mod-inventory-storage's replicas.

On a single server with a single instance of mod-inventory-storage, migration takes approx. 15 minutes for 100,000 instances.

### iii. Recreate OpenSearch or Elasticsearch index
Sometimes we need to recreate OpenSearch or Elasticsearch index, for example when a breaking change has been introduced to index structure (mapping). We must re-index after migrating to Orchid. It can be fixed by running reindex request:

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
Repeat the re-indexing process for other tenants that you might host on your server and have also migrated to Orchid.

### iv. New database indexes for MARC fields
New indexes to the DB were added for the "010" and "035" MARC fields, to improve stability and decrease timeouts.
Indexes are added automatically during the upgrade process. Default DB configuration implies automatic analyzing of tables. In case you should have disabled automatic analyzing, execute the ANALYZE command on mod-source-record-storage schemas.

### v. Default MARC-Instance mapping updated to change how the Relator term is populated on an instance record
See [Update of mapping to change how Relator term is populated on instance record R1 2023 Orchid release](https://wiki.folio.org/display/FOLIJET/Update+of+mapping+to+change+how+Relator+term+is+populated+on+instance+record+R1+2023+Orchid+release) for additional details.
Update: 21 April: an [update script](https://wiki.folio.org/display/FOLIJET/Orchid+MARC-to-Instance+mapping+rules+update+instructions) has been provided. 
**Mandatory change.**
Note that any revised mappings will only apply to Instances created or updated via MARC Bibs **after** the map is updated. To refresh existing Instances against the current SRS MARC Bibs and current map, the library may consider running [Script 3 described here: Scripts for Inventory, Source Record Storage, and Data Import Cleanup](https://wiki.folio.org/display/FOLIOtips/Scripts+for+Inventory%2C+Source+Record+Storage%2C+and+Data+Import+Cleanup).

### vi. Default MARC-Instance mapping rule added for MARC 720 field
Follow the description in the [Release Notes](https://wiki.folio.org/display/REL/Orchid+%28R1+2023%29+Release+Notes).

### vii. "marc_indexers" table data in mod-source-record-storage have to be migrated.
Follow the description in the [Release Notes](https://wiki.folio.org/display/REL/Orchid+%28R1+2023%29+Release+Notes).

Add new permissions as described in the [Release Notes](https://wiki.folio.org/display/REL/Orchid+%28R1+2023%29+Release+Notes) **Permission Updates**.



Congratulation, your system is ready!
