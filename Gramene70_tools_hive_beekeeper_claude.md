# Tools Hive Beekeeper Fixes

Oct 23, 2026 · @Aisa

The ensembl-116 tools pipeline (ayinger) now dispatches and runs jobs: five fixes, mostly about hive running on the system Perl instead of /usr/local/bin/perl.

## Fixes

Each row was a separate blocker; they surfaced one after another.

| # | Symptom | Cause | Fix |
| --- | --- | --- | --- |
| 1 | No ensembl-hive in the server root | Never installed for 116 | Cloned ensembl-hive `version/2.7.0` (the version the release/116 compara and production configs pin) into `/usr/local/ensembl-116/ensembl-hive`. Also cloned ensembl-taxonomy (`main`; no release branches), which eg-web-common loads. |
| 2 | `Can't locate DBI.pm` | Hive scripts use `#!/usr/bin/env perl`. With `/usr/local/bin` missing from PATH they run `/usr/bin/perl` (5.32), which has no DBI or DBD::mysql. | Run every hive command with `/usr/local/bin` first on PATH, so it uses `/usr/local/bin/perl` (5.36), the Perl Apache uses. |
| 3 | Web submit fails: `Can't use an undefined value as a HASH reference at DBAdaptor.pm line 130` | `init_pipeline.pl` died partway: hive loads every meadow driver, and `Meadow/SLURM.pm` needs `DateTime::Format::ISO8601`, which was missing. The hive DB had tables but no schema version or analyses. The error was invisible because the tools\_hive wrapper runs hive's script with `do` and never checks `$@`. | `sudo /usr/local/bin/cpanm DateTime::Format::ISO8601`, then re-ran the init with `-hive_force_init`. 5 analyses stored: AssemblyConverter, Blast, Blat, IDMapper, VEP. |
| 4 | Beekeeper starts, then dies silently | `beekeeper_manager.pl` caches PATH in `beekeeper.config`. Under sudo, PATH is sudo's `secure_path`, without `/usr/local/bin`, so the forked beekeeper got `/usr/bin/perl`. | Deleted the cached config and restarted with PATH set inside the sudo (next section). |
| 5 | ID History Converter job fails: `Can't locate object method "new" via package "Bio::EnsEMBL::Variation::DBSQL::DBAdaptor"` | The IDMapper runnable builds the script's `-I` list from an unsorted directory listing. When `ensembl-webcode` came first, its stub DBAdaptor (no `new`) shadowed the real one in ensembl-variation. | Patched `public-plugins/tools_hive/modules/EnsEMBL/Web/RunnableDB/IDMapper.pm` to sort the list and skip `ensembl-webcode`. |

## Starting the beekeeper

PATH must be set inside the sudo; calling `/usr/local/bin/perl beekeeper_manager.pl` alone is not enough, because the beekeeper it starts inherits the cached PATH.

```bash
cd /usr/local/ensembl-116/public-plugins/tools_hive/utils
sudo rm -f beekeeper.config beekeeper.pid
sudo -H -u nobody env PATH=/usr/local/bin:/usr/bin:/bin EHIVE_ROOT_DIR=/usr/local/ensembl-116/ensembl-hive \
  /usr/local/bin/perl beekeeper_manager.pl --keep_alive --sleep=0.5
sudo chmod 600 beekeeper.config
```

Check it:

```bash
grep PATH beekeeper.config     # must start with /usr/local/bin
ps -fp $(cat beekeeper.pid)    # beekeeper.pl should be running
```

Re-initialising the pipeline (drops and recreates the hive DB, so only when it holds no live jobs):

```bash
cd /usr/local/ensembl-116
export PATH=/usr/local/bin:$PATH EHIVE_ROOT_DIR=/usr/local/ensembl-116/ensembl-hive
perl -e 'BEGIN{require "/usr/local/ensembl-116/ensembl-webcode/conf/includeSiteDefs.pl"} @ARGV=("EnsEMBL::Web::PipeConfig::Tools_conf","-hive_force_init",1); do "$ENV{EHIVE_ROOT_DIR}/scripts/init_pipeline.pl"; print STDERR "INIT ERROR: $@\n" if $@'
```

This form prints hive's error, which the `utils/init_pipeline.pl` wrapper swallows. It should end with "Pipeline successfully stored".

## Result

- Assembly Converter: two test jobs (hive jobs 1 and 2) finished DONE.
- ID History Converter: the failed job's exact command, re-run by hand without `ensembl-webcode`, exited cleanly. A new job has not yet been submitted through the web form since the patch.
- ID history data lives in each species' core DB (`stable_id_event` + `mapping_session`), not in flat files. The converter only maps old IDs forward: for maize, submit an old ID such as `Zm00001d027230`, not a current NAM-5.0 ID.

## Open items

- [ ] Submit a new ID History Converter job through the web form to confirm the IDMapper patch.
- [ ] `sudo chmod 600 beekeeper.config`: it stores the hive DB password and is world-readable.
- [ ] DB passwords also appear in IDMapper job logs under `tmp/temporary/tools/IDMapper/` and in the hive `analysis_data` table. Consider changing the passwords.
- [ ] Make `public-plugins/tools_hive/utils/init_pipeline.pl` print hive's errors (`die $@ if $@` after the `do`).
- [ ] Optionally commit the IDMapper patch in `public-plugins`, so a future `git pull` doesn't drop it.
