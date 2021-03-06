# Migration Utilities [![Build Status](https://travis-ci.com/fcrepo4-exts/migration-utils.svg?branch=master)](https://travis-ci.com/fcrepo4-exts/migration-utils)

A framework to support migration of data from Fedora 3 to Fedora 4, 5, or 6 repositories

## Overview

This utility iterates the foxml files of a fedora 2 or 3 repository, and populates a fedora 4, 5, or 6 repository.

For migrations to Fedora 4 and 5, the utility populates the repository via its APIs.  You will need a running fedora 4 or fedora 5 repository to perform the migration.
The utility will perform various mapping operations in order to fit the fedora 2/3 model onto LDP as supported in Fedora 4 and 5.  In particular:

* All RDF URIs will be re-mapped.  Fedora 2 and 3 use `info:fedora/` URIs in `RELS-EXT` and `RELS-INT`.  The migration utility will re-write these URIs into resolvable `http://` URIs that point to the corresponding resources in the fedora 4 or 5 repository
* FOXML object properties will be expressed in terms of RDF according to the mapping defined in `${migration.mapping.file}`. See example [custom-mapping.properties](https://github.com/fcrepo4-exts/migration-utils/blob/master/src/main/resources/custom-mapping.properties).
* TODO: is there more?

Migrations to Fedora 6 may take a different approach, writing migrated objects directly to the filesystem as [OCFL](https://ocfl.io/draft/spec/)
objects.  Additionally, a "minimal" migration mode is available that performs fewer transformations to migrated content.
In particular:

* There is a 1:1 correspondence between fedora 3 objects and OCFL objects.  Fedora 3 datastreams appear as files within the resulting OCFL objects.
* RDF is not re-mapped, `info:fedora/` subjects and objects are kept intact as-is
* FOXML object and datastream properties are represented as triples in additional sidecar files as per the mapping defined in `${migration.mapping.file}`. See example [custom-mapping.properties](https://github.com/fcrepo4-exts/migration-utils/blob/master/src/main/resources/custom-mapping.properties).

## Status

Fedora 6 support is being actively developed, and is considered unstable until Fedora 6 is released.

A basic migration scenario is implemented that may serve as a starting point for
your own migration from Fedora 3.x to Fedora 4.x.

## How to use

Background work

* Determine the disposition of your FOXML files:
  * Will you be migrating from exported (archive or migration context) FOXML?
    * If so, you will need all of the export FOXML in a known directory.
  * Will you be migrating from from a native fcrepo3 filesystem?
    * If so, fcrepo3 should not be running, and you will need to determine if you're using legacy or akubra storage
* Determine your fcrepo4 url (ex: http://localhost:8080/rest/, http://yourHostName.ca:8080/fcrepo/rest/)

*Warning*: _The migration tool is under active development, so these instructions will change as the configuration process becomes more refined_  

General usage of the migration utils CLI is as follows:

```java -jar target/migration-utils-4.4.1-SNAPSHOT-driver.jar [various options | --help]```

*Note that the migration utility will only run under Java 11+.*  

The following CLI options for specifying details of a given migration are available:
```
Usage: migration-utils [-hrV] [--debug] -a=<targetDir> [-d=<f3DatastreamsDir>]
                       [-e=<f3ExportedDir>] [-l=<objectLimit>]
                       [-o=<f3ObjectsDir>] [-p=<pidFile>] -t=<f3SourceType>
                       [-y=<ocflLayout>]
  -h, --help                 Show this help message and exit.
  -V, --version              Print version information and exit.
  -t, --source-type=<f3SourceType>
                             Fedora 3 source type. Choices: akubra | legacy |
                               exported
  -d, --datastreams-dir=<f3DatastreamsDir>
                             Directory containing Fedora 3 datastreams (used
                               with --source-type 'akubra' or 'legacy')
  -o, --objects-dir=<f3ObjectsDir>
                             Directory containing Fedora 3 objects (used with
                               --source-type 'akubra' or 'legacy')
  -e, --exported-dir=<f3ExportedDir>
                             Directory containing Fedora 3 export (used with
                               --source-type 'exported')
  -a, --target-dir=<targetDir>
                             Directory where OCFL storage root and supporting
                               state will be written
  -y, --layout=<ocflLayout>  OCFL layout of storage root. Choices: flat |
                               pairtree | truncated
                               Default: flat
  -l, --limit=<objectLimit>  Limit number of objects to be processed.
                               Default: no limit
  -r, --resume               Resume from last successfully migrated Fedora 3
                               object
                               Default: false
  -p, --pid-file=<pidFile>   PID file listing which Fedora 3 objects to migrate
      --debug                Enables debug logging
```

### PID migration selection

The default migration configuration will migrate all of the Fedora 2/3 objects found in the source. Subsequent runs will simply re-migrate all of those objects.
However, there are circumstances when it is preferred that only a subset of all source objects be migrated.
There are three means by which a subset of objects may be selected for migration (noting that these means may also be combined).
* *Limit*: When setting the `limit` configuration (detailed above), the migration will be performed on first X-number of objects specified by the value of `limit`.
* *PID List*: When a pid-list is provided (detailed above), the migration will only be performed on the objects associated with the PIDs in the provided pid-list file.
* *Resume*: When enabling the `resume` configuration (detailed above), a file is maintained that keeps track of the last successfully migration object. Subsequent executions will only migrate objects following the last migrated object. Note, this capability is based on the assumption that the order of objects to be migrated is deterministic and the same from one execution to the next.


### Examples

Run a minimal fedora 6 migration from fedora3 legacy foxml

```shell
java -jar target/migration-utils-4.4.1-SNAPSHOT-driver.jar --source-type=legacy --limit=100 --working-dir=target/test/ocfl --objects-dir=src/test/resources/legacyFS/objects --datastreams-dir=src/test/resources/legacyFS/datastreams
```
Run a minimal fedora 6 migration from a fedora3 archival export
```shell
java -jar target/migration-utils-4.4.1-SNAPSHOT-driver.jar --source-type=exported --limit=100 --working-dir=target/test/ocfl --exported-dir=src/test/resources/exported

```

## Property Mappings

### fcrepo3 Object properties to fcrepo4

| fcrepo 3         | fcrepo4                             | Example                  |
|------------------|-------------------------------------|--------------------------|
| PID              | fedora3model:PID†                   | yul:328697               |
| state            | fedoraaccess:objState               | Active                   |
| label            | fedora3model:label†                 | Elvis Presley            |
| createDate       | fcrepo:created                      | 2015-03-16T20:11:06.683Z |
| lastModifiedDate | fcrepo:lastModified                 | 2015-03-16T20:11:06.683Z |
| ownerId          | fedora3model:ownerId†               | nruest                   |

### fcrepo3 Datastream properties to fcrepo4

| fcrepo3       | fcrepo4                                                      | Example                                                    |
|---------------|--------------------------------------------------------------|------------------------------------------------------------|
| DSID          | dcterms:identifier                                           | OBJ                                                        |
| Label         | dcterms:title‡                                               | ASC19109.tif                                               |
| MIME Type     | ebucore:hasMimeType†                                         | image/tiff                                                 |
| State         | fedoraaccess:objState                                        | Active                                                     |
| Created       | fcrepo:created                                               | 2015-03-16T20:11:06.683Z                                   |
| Versionable   | fedora:hasVersions‡                                          | true                                                       |
| Format URI    | premis:formatDesignation‡                                    | info:pronom/fmt/156                                        |
| Alternate IDs | dcterms:identifier‡                                          |                                                            |
| Access URL    | dcterms:identifier‡                                          |                                                            |
| Checksum      | cryptofunc:_hashalgorithm_‡                                  | cryptofunc:sha1 "c91342b705b15cb4f6ac5362cc6a47d9425aec86" |

### auditTrail mapping

| fcrepo3 event                      | fcrepo4 Event Type                              |
|------------------------------------|-------------------------------------------------|
| addDatastream                      | premis:ing‡                                     |
| modifyDatastreamByReference        | audit:contentModification/metadataModification‡ |
| modifyObject                       | audit:resourceModification‡                     |
| modifyObject (checksum validation) | premis:validation‡                              |
| modifyDatastreamByValue            | audit:contentModification/metadataModification‡ |
| purgeDatastream                    | audit:contentRemoval‡                           |

† The `fedora3model` namespace is not a published namespace. It is a representation of the fcrepo3 namespace `info:fedora/fedora-system:def/model`.

‡ Not yet implemented

**Note**: All fcrepo3 DC (Dublin Core) datastream values are mapped as dcterms properties on the Object in fcrepo4. The same goes for any properties in the RELS-EXT and RELS-INT datastreams.

## Additional Documentation

 * [wiki](https://wiki.duraspace.org/display/FF/Fedora+3+to+4+Data+Migration)

### Development

The migration-utils software is built with [Maven 3](https://maven.apache.org) and requires Java 11 and Maven 3.1+.
```bash
mvn clean install
```
The executable utility will be found in the `target` directory.

## Maintainers

Current maintainer

* [Andrew Woods](https://github.com/awoods)

