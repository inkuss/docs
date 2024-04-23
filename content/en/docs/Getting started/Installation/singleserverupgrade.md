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


## II. Upgrade Okapi and Prepare Backend Upgrade
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

Set JAVA_HOME in .profile if you haven't already done so
```
  echo export JAVA_HOME=`readlink -f /usr/bin/javac | sed "s:bin/javac::"` >> ~/.profile
  source ~/.profile
```

Build Okapi from source
```
  cd /usr/folio/okapi
  git fetch
  git checkout v5.2.1
  mvn clean install -DskipTests=true
    ...
    [INFO] BUILD SUCCESS
```

If you have built Okapi from source for the first time you may need to adjust LIB_DIR in /etc/default/okapi so that the Okapi service script (/lib/systemd/system/okapi.service) will find the newly built jar file:
```
  edit /etc/default/okapi
  LIB_DIR="/usr/folio/okapi/okapi-core/target"
```

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
  "id" : "okapi-5.2.1"
} ]
```

If you are starting with a complete platform of Orchid, you will see 9 Edge modules, 59 Frontend modules (folio_\*), 65 Backend modules (mod-\*) and the Poppy version of Okapi (5.2.1).

### II.iii) Pull module descriptors from the central registry

A module descriptor declares the basic module metadata (id, name, etc.), specifies the module's dependencies on other modules (interface identifiers to be precise), and reports all "provided" interfaces. As part of the continuous integration process, each module descriptor is published to the FOLIO Registry at https://folio-registry.dev.folio.org.

```
curl -w '\n' -D - -X POST -H "Content-type: application/json" \
  -d '{ "urls": [ "https://folio-registry.dev.folio.org" ]  }' http://localhost:9130/_/proxy/pull/modules
```
Okapi log should show something like

```
 INFO  ProxyContent         759951/proxy REQ 127.0.0.1:58954 supertenant POST /_/proxy/pull/modules okapi-5.2.1
 INFO  PullManager          Remote registry at https://folio-registry.dev.folio.org is version 5.0.1
 INFO  PullManager          pull smart
 INFO  PullManager          pull: 1440 MDs to insert
 INFO  ProxyContent         759951/proxy RES 200 32114437us okapi-5.2.1 /_/proxy/pull/modules
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

#### i. Set environment varianle env (only required in a multi-tenant environment)
If you are in a multi-tenant environment, set environment variable ENV to ENV = poppy . In a single tenant environment, you don't need to set it . It has the default value ENV = folio.
```
curl -w '\n' -D - -X POST -H "Content-Type: application/json" -d "{\"name\":\"ENV\",\"value\":\"poppy\"}" http://localhost:9130/_/env
```
  
From Orchid release notes, do these steps:
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

#### iv. Set jwt.signing.key for mod-authtoken
In the launch descriptor of mod-authtoken-2.14.1, set jwt.signing.key in the JAVA_OPTION to the same value as you have set it in the Orchid version of mod-authtoken(2.13.0) :
```
      "name" : "JAVA_OPTIONS",
      "value" : "-XX:MaxRAMPercentage=66.0 -Dcache.permissions=true -Djwt.signing.key=folio-demo"
```

Also activate and configure expiring tokens for your tenants in the same launch descriptor (see the README of [mod-authtoken](https://github.com/folio-org/mod-authtoken))
```
 }, {
      "name" : "LEGACY_TOKEN_TENANTS", 
      "value" : ""
    }, {
      "name" : "TOKEN_EXPIRATION_SECONDS",
      "value" : "tenantId:diku,accessToken:600,refreshToken:604800;tenantId:mytenant,accessToken:3600,refreshToken:604800"
```

#### v. Set env vars for mod-search-3.0.8
If you have set ENV = poppy, set KAFKA_EVENTS_CONSUMER_PATTERN for mod-search, using the value of ENV as a part of its value:
    KAFKA_EVENTS_CONSUMER_PATTERN = (poppy\.)(.*\.)inventory\.(instance|holdings-record|item|bound-with)
    
If you have set ENV = folio, set  KAFKA_EVENTS_CONSUMER_PATTERN = (folio\.)(.*\.)inventory\.(instance|holdings-record|item|bound-with)

Set SEARCH_BY_ALL_FIELDS_ENABLED to "true" if you want to activate the search option "all" (search in all fields). By default, SEARCH_BY_ALL_FIELDS_ENABLED is set to "false".

  HIER WEITER
  weiter bei Poppy Release Notes, "Changes & Required Actions", "Inventory, SRS, Data import"

## III. Create new Frontend : Stripes

Create a new frontend (but don't deploy it, yet).
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

## IV. Deploy a new FOLIO backend and enable all modules of the new platform (backend & frontend)
Now do a snapshot of your system, so you will be able to replay the current status in case the upgrade fails.
Users should stop working with the system now because after the new backend has been deployed, the front end will be incompatible to the backend and will need to be redeployed, too.

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


## IV. Start the new Frontend
Stop the old Stripes container: 
systemctl stop stripes
;docker stop <container id of your old stripes container which is still running>

Start the new stripes container:
 
;Redirect port 80 from the outside to port 80 of the docker container:
  
```
  #cd ~/platform-complete
  sudo su
  #nohup docker run -d -p 80:80 stripes &
  systemctl start stripes
```
  
Log in to your frontend: E.g., go to http://<YOUR_HOST_NAME>/ in your browser. 

  - Can you see the Orchid modules in Settings - Installation details ?

  - Do you see the right okapi version, 5.0.1-1 ? 

  - Does everything look good ?

If so, remove the old stripes container: docker rm \<container id of your old stripes container\> .

It is now possible to access the system via the UI, again.
However, changes in permission sets and long-running migration jobs still need to be carried out before the system can be used productively.


## V. Cleanup
  
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
  Those 65 backend modules are now enabled for your tenant(s). 
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


## VI. Post Upgrade
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
