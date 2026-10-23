The ID mapping data is in the database, not in flat files. Each species' core database holds its own ID history.

**How it works:** the converter script is `ensembl-tools/scripts/id_history_converter/IDmapper.pl`. It finds the species' core DB through the registry, using the `gramene_web` account on `colden`. It then runs one SQL query per input ID:

```sql
SELECT old_version, old_release, new_stable_id, new_version, new_release, score
FROM   stable_id_event JOIN mapping_session USING (mapping_session_id)
WHERE  old_stable_id = ?
```

- `stable_id_event` holds one row per mapping: old ID and version → new ID and version, a score and a type. A missing new ID means the gene was retired.
- `mapping_session` holds one row per mapping run, with the old and new release and assembly.
- `gene_archive` and `peptide_archive` are part of the same system, but this script doesn't use them, and they're empty for maize.

**Maize in the release-116 database (`zea_mays_core_70_116_8`):**

| session | releases | assemblies                               | events |
| ------- | -------- | ---------------------------------------- | ------ |
| 1       | 84 → 85  | AGPv3 → AGPv4                            | 29,903 |
| 2       | 85 → 104 | B73_RefGen_v4 → Zm-B73-REFERENCE-NAM-5.0 | 32,927 |

**Why your test returned nothing:** `Zm00001eb000010` is a current NAM-5.0 ID, so it appears in the table only as a *new* ID (`Zm00001d027230.1 → Zm00001eb000010.1`). The script searches only by old ID. Submitting `Zm00001d027230` should return that mapping. The converter only maps old IDs forward; it can't look up where a current ID came from.

Other species work the same way: if a species' core DB has no rows in these tables, the converter will return empty results for it.