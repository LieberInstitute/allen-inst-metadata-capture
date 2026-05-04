# Data Lake vs Allen Institute Metadata Capture

**Can this metadata capture system help Lieber Institute with data management
and migration of petabytes of genomic data to an AWS data lake?
**

## Short Answer

The Allen Institute metadata-capture repository is not the answer to a multi-PB genomic data lake migration.

It is a useful pattern for human-in-the-loop metadata capture, not a petabyte-scale ingestion, crawling, or cataloging system.

It could help with:

- Curator UI for ambiguous datasets.
- Chat-assisted annotation after deterministic scanners have produced facts.
- Summarizing README files, manuscript folders, analysis notes, notebooks, and SLURM logs.
- Mapping messy user language to controlled metadata fields.
- Asking humans targeted follow-up questions when metadata is missing.

It should not be trusted to:

- Discover all LIBD data autonomously.
- Correctly annotate millions or billions of files without validation.
- Infer biological meaning from raw FASTQ, RDS, H5AD, Seurat, methylation, neuroimaging, or manuscript-analysis folders by itself.
- Become the system of record.
- Replace deterministic metadata extraction.

The LLM belongs near the curation layer, not the ingestion foundation.

## Recommended Shape

A better architecture is:

```text
JHPCE volumes
  -> deterministic crawler/inventory
  -> file/object manifest database
  -> format-specific probes
  -> dataset/study grouping logic
  -> human/LLM-assisted curation UI
  -> S3 data lake + metadata catalog
```

In this model, the Allen project is useful mainly as inspiration for the human/LLM-assisted curation UI.

## Why AWS Glue Was Disappointing

AWS Glue crawlers are built around files where a tabular schema can be inferred: CSV, JSON, Avro, Parquet, ORC, XML, logs, and similar data.

That does not naturally fit:

- FASTQ.
- BAM, CRAM, SAM.
- VCF, BCF.
- RDS and Seurat objects.
- AnnData H5AD objects.
- methylation IDATs, 450K/EPIC outputs, WGBS outputs.
- DICOM, NIfTI, BIDS neuroimaging folders.
- arbitrary lab directory trees.
- manuscript-specific analysis folders.

Glue can still be useful later for derived metadata tables:

```text
s3://libd-catalog/manifests/files.parquet
s3://libd-catalog/manifests/datasets.parquet
s3://libd-catalog/manifests/samples.parquet
s3://libd-catalog/manifests/assays.parquet
```

Then Athena, Glue, and DataZone can query those metadata tables. But Glue should not be expected to understand genomic objects directly.

Relevant AWS docs:

- AWS Glue crawlers classify data, infer schema, group tables/partitions, and write metadata to the Glue Data Catalog.
- AWS Glue built-in classifiers focus on common table-ish formats such as JSON, CSV, Avro, ORC, Parquet, XML, and logs.
- Custom classifiers are possible, but they still require deliberate engineering.

Docs:

- https://docs.aws.amazon.com/glue/latest/dg/add-crawler.html
- https://docs.aws.amazon.com/glue/latest/dg/add-classifier.html

## What LIBD Probably Needs First

The first layer should be a canonical file inventory.

For every discovered file, capture:

```text
path
storage system
size
mtime
owner/group
permissions
checksum if available
file extension
magic/type
assay guess
project/study guess
sample/library/run IDs parsed from path/name
related files
migration status
S3 URI
checksum verification status
curation status
```

This inventory should be generated deterministically and stored in a real database, for example PostgreSQL/RDS or Aurora, with Parquet snapshots exported to S3 for Athena/Glue/DataZone.

## Format-Specific Probes

After basic inventory, add probes that understand common scientific formats.

FASTQ:

- gzip validity.
- read count estimate.
- sample/lane naming.
- paired-end mate detection.
- sequencer/run identifiers when parseable.

BAM/CRAM:

- header extraction.
- reference genome.
- read groups.
- sample names.
- sort status.
- index status.
- basic QC flags.

VCF/BCF:

- sample list.
- contigs.
- variant count estimate.
- genome build hints.
- index status.

AnnData/H5AD:

- obs/var dimensions.
- obs columns.
- var columns.
- layers.
- embeddings.
- raw matrix presence.
- species/genome hints.

Seurat/RDS:

- load through controlled R worker.
- object class.
- assays.
- reductions.
- metadata columns.
- dimensions.
- package/version compatibility.

Methylation:

- IDAT/sample sheet linkage.
- 450K/EPIC platform detection.
- WGBS pipeline outputs.
- genome build.
- sample-level QC artifacts.

Neuroimaging:

- DICOM/NIfTI/BIDS detection.
- subject/session/task/run fields.
- modality.
- acquisition parameters where extractable.

Manuscript and analysis folders:

- README files.
- scripts.
- notebooks.
- SLURM logs.
- pipeline config files.
- output manifests.
- figures/tables.
- package lockfiles or session info.

## Where The Allen Agent Fits

The Allen project can be adapted as a curator-facing layer.

Example:

```text
deterministic scanner says:
  "This folder looks like snRNA-seq, has 24 h5ad files,
   24 sample IDs, 3 notebooks, a README, and no study accession."

LLM/UI asks curator:
  "Is this study LIBD123? Which manuscript? Which genome build?
   Are these raw, processed, or publication-specific outputs?"

curator confirms:
  -> write structured metadata to catalog
```

This is the right role for an LLM:

- summarize messy text.
- suggest likely classifications.
- ask missing metadata questions.
- turn curator answers into structured records.
- flag uncertainty.
- reduce manual spreadsheet burden.

It is the wrong role for an LLM:

- initial file discovery.
- checksum verification.
- migration state tracking.
- authoritative sample mapping.
- blind autonomous annotation.
- compliance/audit provenance.

## AWS Components To Consider

Raw storage:

- S3 buckets with explicit prefix conventions.
- S3 object tags for stable operational metadata.
- S3 lifecycle policies.
- S3 checksums.
- S3 Inventory for object-level reports after migration.

S3 Inventory docs:

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/configure-inventory.html

Omics-native storage:

- AWS HealthOmics Sequence Store supports FASTQ gzip, uBAM, BAM, and CRAM.
- It should be evaluated carefully for cost, workflow fit, access patterns, and lock-in.
- HealthOmics variant and annotation stores are no longer open to new customers according to current AWS docs.

HealthOmics docs:

- https://docs.aws.amazon.com/omics/latest/dev/create-sequence-store.html
- https://docs.aws.amazon.com/omics/latest/dev/omics-analytics.html

Catalog/business layer:

- Glue Data Catalog for technical tables, especially derived Parquet metadata.
- Athena for querying manifests and metadata snapshots.
- DataZone for higher-level cataloging, business metadata, metadata forms, glossary terms, and access workflows.

DataZone docs:

- https://docs.aws.amazon.com/en_en/datazone/latest/userguide/datazone-concepts.html

## Proposed Division Of Labor

Use deterministic code for:

- discovery.
- checksums.
- format detection.
- file grouping.
- metadata extraction from known formats.
- migration state.
- audit logs.
- reproducibility.

Use relational/catalog storage for:

- file manifests.
- dataset manifests.
- sample manifests.
- assay metadata.
- study metadata.
- provenance.
- curation state.
- access state.

Use LLM assistance for:

- summarization.
- ambiguous classification.
- controlled vocabulary mapping.
- curator Q&A.
- documentation drafting.
- resolving messy legacy folders.
- explaining why a dataset is incomplete.

Use AWS Glue/DataZone for:

- cataloging the derived metadata products.
- querying Parquet manifests.
- connecting downstream analytics tools.
- publishing governed catalog assets.

## Bottom Line

Do not use the Allen chatbot repo as the basis for petabyte-scale ingestion.

Use it as inspiration for a later layer:

```text
crawler facts + file probes + metadata DB + curator UI + LLM assist
```

The hard problem is not asking an LLM what a dataset is. The hard problem is building a reliable, repeatable inventory and provenance system across messy scientific storage.

Once that exists, an LLM can help humans annotate the gray areas much faster.
