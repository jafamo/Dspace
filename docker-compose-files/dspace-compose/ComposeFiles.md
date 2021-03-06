## Proposed Changes to Docker Compose Startup

| Compose File | Purpose |
| -- | -- |
| docker-compose.yml | Base image for DSpace 4, 5, 6, 7.  Ingests content (if needed) and starts tomcat. |
| docker-compose-8081.yml | Variant base image that runs tomcat on port 8081 to allow 2 DSpace instances to run simultaneously |
| d4.override.yml | Sets base image for DSpace4x.  <br/>Since Flyway is not supported in DSpace4x, a special 4x postgres image. |
| d5.override.yml | Sets base image for DSpace5x.  Activates Mirage2. |
| d6.override.yml | Sets base image for DSpace6x.  Activates Mirage2. |
| d7.override.yml | Sets base image for DSpace7x.  <br/>Adds Angular UI to replace XMLUI and JSPUI. <br/> REST: http://localhost:8080/spring-rest <br/> Angular UI: http://localhost:3000 |
| d7solr.override.yml | Adds externalized solr to d7.override.yml <br/> Solr http://localhost:8983 |
| src.override.yml | Optional add-on to trigger and redeploy and tomcat start. |
| rdf.override.yml | Optional RDF Add-on for DSpace6x and DSpace7x. <br/>http://localhost:3030 |

## Basic Startup Commands
_Note: only one compose file should be running at a given time.  
Because each version of DSpace reuses the same ports and container names, you must run `docker-compose -p dX down` to stop one instance before starting another.  Each instance will persist data in a separate volume._

```
docker-compose -p d7 -f docker-compose.yml -f d7.override.yml up -d
docker-compose -p d6 -f docker-compose.yml -f d6.override.yml up -d
docker-compose -p d5 -f docker-compose.yml -f d5.override.yml up -d
docker-compose -p d4 -f docker-compose.yml -f d4.override.yml up -d
```

---

### Override the default branch/pr to use.

Each compose file points to a specific [tagged DSpace Docker image](https://hub.docker.com/r/dspace/dspace/tags).
- Set the DOCKER_OWNER environment variable to change the dockerhub repository
- Set the DSPACE_VER environment variable to change the DSpace tagged version
- Set the AIPZIP environment variable to point to a zip file of AIP files that will be ingested on startup.
  - After ingest, `/dspace/assetstore/ingest.hasrun.flag` will be created.  This will prevent additional ingest on subsequent startup.
- Set the SKIPAIP=Y environment variable to skip AIP processing on startup.

## Build DSpace Image Locally
Adding the `-f src.override.yml` to your startup command will allow you to buid a local image `dspace/dspace:local-build`.

Set DSPACE_SRC to your source directory for DSpace.

Run the build
```
docker-compose -p d6 -f docker-compose.yml -f d6.override.yml -f src.override.yml build
```

10 GB of temporary images will be retained from the build process.  Run the following to free space.
```
docker image prune
```
Start the application with the following command.
```
docker-compose -p d6 -f docker-compose.yml -f d6.override.yml -f src.override.yml up -d
```

---

## Add the RDF Service (DSpace6 and DSpace7)

Add `-f rdf.override.yml` to add an RDF Triplestore to your DSpace distribution.

#### Create Dataset in the RDF Triplestore (fuseki)
- http://localhost:3030
- user: admin
- password: dspace

Create a dataset named **dspace**

#### Build the RDF dataset from the DSpace repository

Bash
```
docker exec -it dspace /dspace/bin/dspace rdfizer -c -v
```

Git-Bash Windows
```
winpty docker exec -it dspace //dspace/bin/dspace rdfizer -c -v
```

#### View Metadata via the RDF service

- http://localhost:8080/rdf/handle/123456789/1/rdf?text=true

---

## Miscellaneous Notes

### Passing Variables and Properties to DSpace

DSpace uses Apache Commons Config to access runtime properties.  Values can be passed to Apache commons through the dspace.cfg file, the local.cfg file, as system properties (-Ddspace.name=Foo), and as environment variables.

Note that enviroment variables containing dots in their names are not supported by some Linux shells.  While it is possible to pass an ENV variable (`-e dspace.name=Foo`) through docker run or Docker Compose, that value will not propagate to the DSpace runtime becuase the underlying shell will reject the variable name.

If you need to set a property name that contains a period, the recommended approach is to pass the override as a system property set in JAVA_OPTS.  `-e JAVA_OPTS=-Ddspace.name=Foo`.

### Verify DSpace Version

#### Bash
```
docker exec -it dspace /dspace/bin/dspace version
```

#### Git-Bash Windows
```
winpty docker exec -it dspace //dspace/bin/dspace version
```

### Verify Database Schema

#### Bash
```
docker exec -it dspacedb psql -U dspace -c "select * from schema_version order by installed_rank desc limit 1"
```

#### Git-Bash Windows
```
winpty docker exec -it dspacedb psql -U dspace -c "select * from schema_version order by installed_rank desc limit 1"
```

## Accessing the Command Line

### Tomcat

#### Bash
```
docker exec -it --detach-keys "ctrl-p" dspace /bin/bash
```

#### Git-Bash Windows
```
winpty docker exec -it --detach-keys "ctrl-p" dspace //bin/bash
```

### Database

#### Bash
```
docker exec -it --detach-keys "ctrl-p" dspacedb psql -U dspace
```

#### Git-Bash Windows
```
winpty docker exec -it --detach-keys "ctrl-p" dspacedb psql -U dspace
```

## Destroying Docker Resources
If you no longer need to retain your Docker volumes, run the following command.  Alter the volume prefix to match your version of DSpace

```
docker volume rm d6_assetstore d6_pgdata d6_solr_authority d6_solr_oai d6_solr_search d6_solr_statistics
```
A helper script exists in this repository to remove volumes.  You will need to save or copy this script in order to run it.

[removeVols script](../../tools/removeVols.md)
