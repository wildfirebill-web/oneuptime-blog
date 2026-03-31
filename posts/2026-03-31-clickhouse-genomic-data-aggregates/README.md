# How to Store and Query Genomic Data Aggregates in ClickHouse

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: ClickHouse, Genomics, Bioinformatics, Healthcare, Variant Analysis

Description: Use ClickHouse to store and query genomic variant aggregates, allele frequency tables, and population genetics summaries at biobank scale.

---

Genomic databases at biobank scale contain hundreds of millions of variant records across thousands of samples. While raw variant calling is handled by specialized bioinformatics tools, the aggregated outputs - allele frequencies, phenotype associations, and population summaries - are ideal workloads for ClickHouse.

## Variant Summary Table

```sql
CREATE TABLE variant_summary (
    variant_id      UInt64,
    chromosome      LowCardinality(String),
    position        UInt64,
    ref_allele      FixedString(4),
    alt_allele      String,
    gene_symbol     LowCardinality(String),
    consequence     LowCardinality(String),   -- 'missense', 'synonymous', 'stop_gained'
    af_global       Float64,   -- allele frequency
    af_eur          Float64,
    af_afr          Float64,
    af_eas          Float64,
    cadd_score      Float32,
    clinvar_class   LowCardinality(String),   -- 'benign', 'likely_pathogenic', etc.
    cohort_size     UInt32
) ENGINE = MergeTree()
ORDER BY (chromosome, position)
PARTITION BY chromosome;
```

## Allele Frequency Distribution by Consequence

```sql
SELECT
    consequence,
    count()                                AS variants,
    round(avg(af_global), 6)               AS mean_af,
    round(quantile(0.5)(af_global), 6)     AS median_af,
    countIf(af_global < 0.01)              AS rare_variants
FROM variant_summary
GROUP BY consequence
ORDER BY variants DESC;
```

## Rare Pathogenic Variants by Gene

```sql
SELECT
    gene_symbol,
    count()                         AS pathogenic_variants,
    round(avg(cadd_score), 2)       AS avg_cadd
FROM variant_summary
WHERE clinvar_class IN ('likely_pathogenic', 'pathogenic')
  AND af_global < 0.01
GROUP BY gene_symbol
ORDER BY pathogenic_variants DESC
LIMIT 30;
```

## Population Frequency Comparison

```sql
SELECT
    variant_id,
    chromosome,
    position,
    gene_symbol,
    af_eur,
    af_afr,
    af_eas,
    abs(af_eur - af_afr)  AS eur_afr_diff
FROM variant_summary
WHERE consequence = 'missense'
  AND af_global > 0.001   -- only common enough to compare
ORDER BY eur_afr_diff DESC
LIMIT 20;
```

## ClinVar Classification Summary

```sql
SELECT
    clinvar_class,
    count()                                              AS variants,
    round(100.0 * count() / sum(count()) OVER (), 2)    AS pct
FROM variant_summary
WHERE clinvar_class != ''
GROUP BY clinvar_class
ORDER BY variants DESC;
```

## CADD Score Distribution by Consequence

```sql
SELECT
    consequence,
    round(avg(cadd_score), 2)              AS mean_cadd,
    round(quantile(0.9)(cadd_score), 2)    AS p90_cadd,
    round(max(cadd_score), 2)              AS max_cadd
FROM variant_summary
GROUP BY consequence
ORDER BY mean_cadd DESC;
```

## Chromosome-Level Variant Density

```sql
SELECT
    chromosome,
    count()                           AS total_variants,
    countIf(af_global < 0.001)        AS ultra_rare,
    countIf(af_global >= 0.05)        AS common
FROM variant_summary
GROUP BY chromosome
ORDER BY total_variants DESC;
```

## Summary

ClickHouse complements bioinformatics pipelines by providing a fast query layer over aggregated genomic data. Partitioning by chromosome aligns with common query patterns, columnar compression handles the repetitive consequence and classification strings efficiently, and quantile functions support allele frequency distribution analysis that would be slow in traditional relational databases.
