# SQLite databases for gypsum metadata

[![RunTests](https://github.com/ArtifactDB/gypsum-to-sqlite/actions/workflows/run-tests.yaml/badge.svg)](https://github.com/ArtifactDB/gypsum-to-sqlite/actions/workflows/run-tests.yaml)
[![Updates](https://github.com/ArtifactDB/gypsum-to-sqlite/actions/workflows/update-indices.yaml/badge.svg)](https://github.com/ArtifactDB/gypsum-to-sqlite/actions/workflows/update-indices.yaml)

## Overview 

This repository contains schemas and code to generate SQLite files from metadata in the [**gypsum** backend](https://github.com/ArtifactDB/gypsum-worker).
The SQLite files can then be used in client-side searches to find interesting objects for further analysis.
We construct new indices by fetching metadata and converting them into records on one or more tables based on a JSON schema specification;
existing indices are updated by routinely scanning the [logs](https://github.com/ArtifactDB/gypsum-worker#parsing-logs) for new or deleted content.

This document is intended for system administrators or the occasional developer who wants to create a new search index for their package(s).
Users should not have to interact with these indices directly, as this should be mediated by client packages in relevant frameworks like R/Bioconductor.
For example, the [gypsum R client](https://github.com/ArtifactDB/gypsum-R) provides functions for obtaining the schemas and indices,
which are then called by more user-facing packages like the [scRNAseq](https://github.com/LTLA/scRNAseq) R package.

## From JSON schemas to SQLite

We expect that the metadata for any **gypsum** object can be represented as JSON, which allows for fairly complex metadata fields.
The expectations for the metadata are described by [JSON schemas](https://json-schema.org) - readers can find some examples in the [schemas/](schemas/) subdirectory.
The scripts in this repository use the JSON schemas to automtically initialize SQLite tables and to convert JSON metadata into table rows.
Each JSON schema is used to generate its a separate SQLite file that contains one or more tables depending on the schema's complexity.

### File contents

Each SQLite file contains a `core` table.
Each row corresponds to an indexed object in **gypsum**.
The table will have at least the following fields, all of which have the `TEXT` type:

- `_key`: the primary key, created by combining `_project`, `_version`, `_asset`, `_path`.
- `_project`: the name of the project.
- `_asset`: the name of the asset.
- `_version`: the name of the version.
- `_path`: the path to the object inside this versioned asset.
  This is only supplied if the object is located inside a subdirectory of the asset, otherwise it is set to `null`.
- `_object`: the object type.

The SQLite file may contain a `free_text` virtual FTS5 table where each row corresponds to an indexed **gypsum** object.
This contains at least the `_key` column (same as above), along with one or more additional columns corresponding to free-text metadata fields.
If no metadata fields are marked as free-text, the `free_text` table will not be present in the file.

The SQLite file may contain any number of `multi_<FIELD>` tables, where `<FIELD>` is a metadata field that is a JSON array.
This holds one-to-many mappings between a **gypsum** object and the array items, so an indexed object may have zero, one or many rows in this table.
The table contains at least the `_key` column (same as above), with one or more additional columns depending on the type of the array items.
If the items are booleans, integers, strings or numbers, the table contains exactly one `item` column of the corresponding type;
if the items are objects, the table contains columns corresponding to the properties of the object.

For `core` and `multi_<FIELD>` tables, we generate an index for each column.
Clients can perform most complex queries efficiently with the relevant inner joins.

### Table generation rules

Each JSON schema is used to generate a SQLite table according to some simple rules:

- The schema must be a top-level `"type": "object"`.
- An `integer` property is converted to an `INTEGER` field on the `core` table.
- A `boolean` property is converted to an `INTEGER` field on the `core` table.
- A `number` property is converted to a `REAL` field on the `core` table. 
- A `string` property is usually converted to a `TEXT` field.
  However, if its `_attributes` contain `"free_text"`, it will instead be converted into a column of `free_text`.
- An `array` property is converted to a separate `multi_<FIELD>` table, where `<FIELD>` is the name of the property.
  - If the items are booleans, integers, strings or numbers, the type of the `item` column is determined as described above.
    However, note that any `"free_text"` in the `_attributes` is ignored.
  - If the items are objects, one column is generated per property in the object, following the rules described above.
    Properties should be booleans, integers, strings or numbers; any `"free_text"` in the `_attributes` is again ignored.
- We do not support `object` properties.
  Schema authors should flatten their objects for table generation.
- Properties should not start with an underscore, as these are reserved for special use.

### Adding new schemas

Package developers with objects stored in the **gypsum** backend may wish to create custom SQLite files to enable package-specific searches.
This can be easily done by adding or modifying files in a few locations:

- Add a new JSON schema to [`schemas/`](schemas/).
  This file's name should have the `.json` file extension and should not contain any whitespace.
  - It is also recommended to add some tests to [`tests/schemas`](tests/schemas) to ensure that the schema behaves as expected.
- Modify the [`scripts/indexVersion.js`](scripts/indexVersion.js) file to gather metadata from the **gypsum** backend into a JSON object.
  This should only involve fetching and parsing some small files from the bucket;
  package developers should consider computing any complex metadata before or during the original upload to the bucket.

Once a new schema is added, the code in this repository will automatically create and update a SQLite file corresponding to the new schema.

## SQLite file generation and editing

The [`scripts/`](scripts/) subdirectory contains several scripts for generating and updating the SQLite files.
These expect to have a modestly recent version of Node.js (tested on 16.19.1) and required dependencies can be installed with the usual `npm install` process.

### Creating new files

The [`fresh.js`](scripts/fresh.js) script will generate one SQLite file corresponding to each JSON schema.
This is done by listing all projects and assets in the **gypsum** backend,
identifying the latest version of each asset,
extracting metadata for objects in the latest version,
and generating a SQLite file from the extracted metadata. 

```shell
# Older versions of Node.js may need --experimental-fetch.
node scripts/fresh.js -s SCHEMAS -o OUTPUTS
```

The script has the following options:

- `-s`, `--schemas`: the directory containing the JSON schema files.
  Defaults to `./schemas`.
- `-o`, `--outputs`: the directory in which to store the output SQLite files.
  Each file will have the same prefix as its corresponding JSON schema.
  Defaults to `./outputs`.
- `-x`, `--only`: name of a project, indicating that indexing should only be performed for that project.
  If not supplied, indexing is performed for all projects.
  Useful for debugging specific projects.
- `-a`, `--after`: any string such that indexing is only performed for projects with names that sort after that string.
  If not supplied, indexing is performed for all projects.
  Useful for debugging a set of projects.
- `-w`, `--overwrite`: boolean that specifies whether to overwrite existing SQLite files in the output directory.
  This can be turned off and combined with `--after` to iteratively construct the full index.
  Defaults to `true`.

In addition to creating new SQLite files, `fresh.js` will also add a `modified` file containing a Unix timestamp.
This will be used by `update.js` (see below) to determine which logs to consider during updates.

### Updating files from logs

The [`update.js`](scripts/update.js) script will modify each SQLite file based on recent changes in the **gypsum** backend.
It does so by scanning the logs in the backend, filtering for those generated after the `modified` timestamp.
Each log may be used to perform an update to the SQLite file based on its action type (see [here](https://github.com/ArtifactDB/gypsum-worker#parsing-logs)),
either by inserting rows corresponding to new objects or (more rarely) by deleting rows corresponding to deleted assets, versions or projects.
The `add-version` and `delete-version` actions will only have an effect if the affected version is the latest;
for `delete-version`, the script will insert metadata for objects in the currently-latest version into the SQLite file.

```shell
# Older versions of Node.js may need --experimental-fetch.
node scripts/update.js -s SCHEMAS -d DIR
```

The script has the following options:

- `-s`, `--schemas`: the directory containing the JSON schema files.
  Defaults to `./schemas`.
- `-d`, `--dir`: the directory containing the SQLite files to be modified.
  Each file will have the same prefix as its corresponding JSON schema.
  Defaults to the working directory.

In addition to modifying the SQLite files, `update.js` will update the `modified` file to the timestamp of the last log.

### Manual updates

The [`manual.js`](scripts/manual.js) script will modify each SQLite file based on its arguments.
It uses the same logic as `update.js` and is intended for testing/debugging the update code.
Any updates to SQLite files in production should still be performed by `update.js`.

```shell
# Older versions of Node.js may need --experimental-fetch.
node scripts/manual.js -t add-version -p PROJECT -a ASSET -v VERSION -l true
node scripts/manual.js -t delete-version -p PROJECT -a ASSET -v VERSION -l true
node scripts/manual.js -t delete-asset -p PROJECT -a ASSET
node scripts/manual.js -t delete-project -p PROJECT -a ASSET
```

The script has the following options:

- `-s`, `--schemas`: the directory containing the JSON schema files.
  Defaults to `./schemas`.
- `-d`, `--dir`: the directory containing the SQLite files to be modified.
  Each file will have the same prefix as its corresponding JSON schema.
  Defaults to the working directory.
- `-t`, `--type`: the type of action to perform.
  This is a required argument and should be one of `add-version`, `delete-version`, `delete-asset` or `delete-project`.
- `-p`, `--project`: the name of the project.
  This is a required argument.
- `-a`, `--asset`: the name of the project.
  This is a required argument for all `type` except for `delete-project`.
- `-v`, `--version`: the name of the project.
  This is a required argument for `add-version` and `delete-version`.
- `-l`, `--latest`: boolean indicating whether the specified version is the latest of its asset.
  This is a required argument for `add-version` and `delete-version`.

## Publishing SQLite files

The various GitHub Actions in this repository will publish the SQLite files as release assets.

- The [`fresh-build` Action](https://github.com/ArtifactDB/gypsum-to-sqlite/actions/workflows/fresh-build.yaml) will run the `fresh.js` script to create and publish a fresh build.
  This is manually triggered and can be used on the rare occasions where the existing release is irrecoverably out of sync with the **gypsum** bucket.
- The [`update-indices` Action](https://github.com/ArtifactDB/gypsum-to-sqlite/actions/workflows/update-indices.yaml) runs the `update.js` script daily to match changes to the bucket contents.
  This will only publish a new release if any changes were performed.
  - Note that cron jobs in GitHub Actions require some semi-routine nudging to indicate that the repository is still active, otherwise the workflow is disabled.

The latest version of the SQLite files are available [here](https://github.com/ArtifactDB/gypsum-to-sqlite/releases/tag/latest).
Clients can check the `modified` file to determine when the files were last updated (and whether local caches need to be refreshed).
