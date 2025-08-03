# Ben Atkinson's README

# HOW TO SET UP OMOP-LITE 

1) clone this repo
2) create a data folder in the root of this omop-lite directory
3) add your data to it (either the proper vocabs folder James emailed you or copy and paste the data in `omop-lite/synthetic/1000`)
4) replace the embeddings/embeddings.parquet file with the one James sent to you
5) challenges I faced when doing this on 26/07/25: the omop-lite main branch isnt up to date so have to replace the docker-compose in it with that which is in the `bugfix/postgres-user` branch - line 23 is missing `"-U", "postgres"` --> without this you get `"FATAL root not found error"`

- USING DOCKER
6) cd into the omop-lite root directory in a terminal and run `docker compose --profile text-search up`. If it has worked you will see `text-search-1 exited with code 0`.

- USING SINGULARITY:
6)   1. Pull and convert Docker image to Singularity image (only once)
  
        `singularity pull /home/apyba3/pgvector_pg17.sif docker://pgvector/pgvector:pg17`

     2. Create directories on host for data and runtime files (only once)
        
        `mkdir -p /home/apyba3/pgdata`
        
        `mkdir -p /home/apyba3/pgrun`

     4. Initialize the PostgreSQL data directory inside the container (only once)
        
        ```
        singularity exec \        
          --bind /home/apyba3/pgdata:/var/lib/postgresql/data \
          /home/apyba3/pgvector_pg17.sif \
          initdb -D /var/lib/postgresql/data`
        ```

     6. Start PostgreSQL server (run each time you want to start the DB)
 
        ```
        singularity exec \
          --bind /home/apyba3/pgdata:/var/lib/postgresql/data \
          --bind /home/apyba3/pgrun:/var/run/postgresql \
          /home/apyba3/pgvector_pg17.sif \
          /usr/lib/postgresql/17/bin/postgres -D /var/lib/postgresql/data
        ```

---
# omop-lite

![MIT License][license-badge]
[![omop-lite Releases][omop-lite-releases-badge]][omop-lite-releases]
[![omop-lite Tests][omop-lite-tests-badge]][omop-lite-tests]
![Python][python-badge]
[![omop-lite Containers][docker-badge]][omop-lite-containers]
[![omop-lite helm][helm-badge]][omop-lite-containers]

A small container to get an OMOP CDM database running quickly, with support for both PostgreSQL and SQL Server.

Drop your data into `data/`, and run the container.

## Configuration

You can configure the container or CLI using the following environment variables:

- `DB_HOST`: The hostname of the database. Default is `db`.
- `DB_PORT`: The port number of the database. Default is `5432`.
- `DB_USER`: The username for the database. Default is `postgres`.
- `DB_PASSWORD`: The password for the database. Default is `password`.
- `DB_NAME`: The name of the database. Default is `omop`.
- `DIALECT`: The type of database to use. Default is `postgresql`, but can also be `mssql`.
- `SCHEMA_NAME`: The name of the schema to be created/used in the database. Default is `public`.
- `DATA_DIR`: The directory containing the data CSV files. Default is `data`.
- `SYNTHETIC`: Load synthetic data (boolean). Default is `false`
- `SYNTHETIC_NUMBER`: Size of synthetic data, `100` or `1000`. Default is `100`.
- `DELIMITER`: The delimiter used to separate data. Default is `tab`, can also be `,`

## Usage

### CLI

`pip install omop-lite`
`python omop-lite --help`

### Docker

`docker run -v ./data:/data ghcr.io/health-informatics-uon/omop-lite`

```yaml
# docker-compose.yml
services:
  omop-lite:
    image: ghcr.io/health-informatics-uon/omop-lite
    volumes:
      - ./data:/data
    depends_on:
      - db

  db:
    image: postgres:latest
    environment:
      - POSTGRES_DB=omop
      - POSTGRES_PASSWORD=password
    ports:
      - "5432:5432"
```

### Helm

To install using Helm:

```bash
# Add the Helm repository
helm install omop-lite oci://ghcr.io/health-informatics-uon/charts/omop-lite --version 0.2.2
```

The Helm chart deploys OMOP Lite as a Kubernetes Job that creates an OMOP CDM in a database. You can customise the installation using a values file:

```yaml
# values.yaml
env:
  dbHost: postgres
  dbPort: "5432"
  dbUser: postgres
  dbPassword: postgres
  dbName: omop_helm
  dialect: postgresql
  schemaName: public
  synthetic: "false" 
```

Install with custom values:

```bash
helm install omop-lite omop-lite/omop-lite -f values.yaml
```

## Synthetic Data

If you need synthetic data, some is provided in the `synthetic` directory. It provides a small amount of data to load quickly.
To load the synthetic data, run the container with the `SYNTHETIC` environment variable set to `true`.

- 100 is fake data.
- 1000 is [Synthea 1k](https://registry.opendata.aws/synthea-omop/) data.

## Bring Your Own Data

You can provide your own data for loading into the tables by placing your files in the `data/` directory. This should contain `.csv` files matching the data tables (`DRUG_STRENGTH.csv`, `CONCEPT.csv`, etc.).

To match the vocabulary files from Athena, this data should be tab-separated, but as a `.csv` file extension.
You can override the delimiter with `DELIMITER` configuration.

## Text search OMOP

### Full-text search

Adding a tsvector column to the concept table and an index on that column makes full-text search queries on the concept table run much faster.
This can be configured by setting `FTS_CREATE` to be non-empty in the environment.

### Vector search

Postgres does vector search too!
To enable this on omop-lite, you can compose the `compose-omop-ts.yml` with

```bash
docker compose -f compose-omop-ts.yml
```

To do this, you need to have `embeddings/embeddings.parquet`, containing concept_ids and embeddings.
This uses [pgvector](https://github.com/pgvector/pgvector) to create an `embeddings` table.

## Testing

If you're a developer and want to iterate on omop-lite quickly, there's a small subset of the vocabularies sufficient to build in `synthetic/`.
If you wish to test the vector search, there are matching embeddings in `embeddings/embeddings.parquet`.

[omop-lite-containers]: https://github.com/orgs/Health-Informatics-UoN/packages?repo_name=omop-lite
[omop-lite-releases]: https://github.com/Health-Informatics-UoN/omop-lite/releases
[omop-lite-tests]: https://github.com/Health-Informatics-UoN/omop-lite/actions/workflows/check.test.python.yml
[omop-lite-releases-badge]: https://img.shields.io/github/v/tag/Health-Informatics-UoN/omop-lite
[omop-lite-tests-badge]: https://github.com/Health-Informatics-UoN/omop-lite/actions/workflows/check.test.python.yml/badge.svg

[license-badge]: https://img.shields.io/github/license/health-informatics-uon/omop-lite.svg
[python-badge]: https://img.shields.io/badge/Python-3776AB?style=flat-square&logo=python&logoColor=white
[docker-badge]: https://img.shields.io/badge/docker-%230db7ed.svg?style=flat-square&logo=docker&logoColor=white
[helm-badge]: https://img.shields.io/badge/Helm-0F1689?logo=helm&logoColor=fff&style=flat-square
