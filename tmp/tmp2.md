I want a precise, evidence-based description of how our map-validation
pipeline stores spatial data and evaluates predicates over it. Read the
code and configs; cite file paths and line numbers for every claim. Where
you can't find the answer in the code, say "not determined" rather than
guessing. Don't summarise from READMEs alone — confirm against the
implementation.

## 1. Storage layout in S3

- What is the unit of storage: one file per what? How are the files
  named/keyed, and what metadata (manifest, catalog, path convention)
  records each file's spatial extent?
- How are the partition boundaries decided — fixed grid, adaptive (from a
  sample of the data?), something else? Where is that code, and what
  parameters control it (target records per partition, max bytes, depth)?
- Is each feature stored in exactly one file? If so, what rule picks the
  file for a feature whose geometry crosses a partition boundary
  (centroid, a corner of its bounding box, first vertex, ...)? If a
  feature can be in more than one file, is it stored whole in each, or
  clipped to the partition?
- Inside a file: what is the format (Parquet? GeoParquet version?), how is
  geometry encoded, is there a bounding-box column, are rows sorted
  spatially (by what curve or key), what is the row-group size? What
  exactly does "spatial index" refer to in this system?
- How are oversized features (very long lines, large polygons) handled at
  write time, if at all?

## 2. Predicate model

- How is a validation predicate represented in code — what's the
  interface or base type? List the categories that exist.
- Which predicates need only the feature itself, and which need other
  features nearby? Is that distinction explicit in the type/interface, or
  implicit?
- For the neighbour-dependent ones: how does a predicate declare what it
  needs — a distance/radius, a topological relation, a named other
  dataset? Is there a maximum neighbourhood distance anywhere, and what
  sets it?

## 3. Evaluation

- What executes the predicates (Spark? something else?), and what is the
  unit of parallelism — one task per S3 file, per something else?
- Does the engine evaluate directly on the stored partitions, or does it
  re-partition the data first? If it re-partitions: where, by what
  scheme, and is there any documented reason for doing so?
- For a feature near a partition edge whose predicate needs neighbours
  across that edge: exactly how does the engine get those neighbours?
  (Reads adjacent files with a margin? Replicates features into multiple
  partitions? Something else?) Where is that code, and what parameter
  controls the margin or replication rule?
- If features or results can be produced more than once, how are
  duplicates prevented or removed, and where?
- Walk through one concrete neighbour-dependent predicate end to end:
  from the S3 read to the emitted result, naming each Spark stage and
  whether it involves a shuffle (exchange). Do the same for one
  feature-local predicate.

## 4. Output

- Format and destination of results, and whether results are keyed by
  feature id, by partition, or both.

Finish with a short list of things you could not determine from the code
and where you'd look next (dashboards, runbooks, specific people).
