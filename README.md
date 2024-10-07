# mie2025-paper-e2e-data-quality

## Prerequisites

- Docker with the Compose plugin
- the Trino CLI from <https://trino.io/docs/current/client/cli.html> available in `$PATH` as `trino`
- curl

## Run

> [!IMPORTANT]
> This is not intended to demonstrate a production-like deployment!
> For that you'd want a resilient Kubernetes cluster and deploy all services higly-available.
> Anything done imperatively here, is better expressed as jobs and workflows instead.

```sh
docker compose up
```

once you see `DATABASE IS READY TO USE!` from the `oracle-1` container, you can run:

```sh
./deploy-connector.sh
```

this will eventually trigger `obds-to-fhir` to map data from the ONKOSTAR database to FHIR resources
and also import them to the Pathling server via the `fhir-to-pathling-server` service.

You can check the import status into Pathling by running:

```sh
curl http://localhost:8082/fhir/{Patient,Condition}?_summary=count
```

which should return a total of `3` and `4` respectively.

To register the Pathling-encoded Delta Tables for querying by Trino, run:

```sh
docker compose -f compose.yaml -f compose.warehousekeeper.yaml up warehousekeeper
```

after the resources are imported.

Next, to create the CSV to compare against, run:

```sh
sudo chown -R 1001:1001 ./csv-output/
docker compose -f compose.obds-fhir-to-opal.yaml -f compose.yaml up obds-fhir-to-opal
```

Once that completed, upload the CSV to MinIO:

```sh
docker compose -f compose.yaml -f compose.upload-csv.yaml up upload-csv
```

Finally, run:

```sh
trino -f queries/counts_cancer_diagnoses_2018-2023.sql
```

and

```sh
trino -f queries/counts_cancer_diagnoses_by_icd10code_2022.sql
```
