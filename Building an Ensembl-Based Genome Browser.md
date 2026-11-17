# Building an Ensembl-Based Genome Browser

Oct 27, 2026 · @Aisa

*Visual-first product description • Draft v0.2 • Gramene 70 (Ensembl 116 / EnsemblPlants 63)*

&#91;embedded content: Figure 1 · build execution map, 3 inputs, 3 phases, 4 gates\]

*Figure 1. Build execution map. Upstream data and code flow through three phases; each gate (G0–G3) must pass before the next phase starts. Arrows show each input's main entry point; Gramene configuration also shapes the web server phase.*

**Question:** How do we turn each EnsemblPlants release into a working, Gramene-branded genome browser with local analysis tools, repeatably and on our own hardware?

**Plan:**

- Pull Ensembl 116 code from GitHub and EnsemblPlants 63 databases from the EBI FTP site; verify every file against its published checksum.
- Rename and load the databases on Gramene MySQL servers, then layer Gramene configuration and local patches over the Ensembl web code on the ayinger VM.
- Stand up the tools pipeline (VEP, Assembly Converter, ID History Converter) on eHive, test each tool, then expose the site at ensembl-dev.gramene.org for review before go-live.

Source SOP: [GrameneGenomesBuildSOP58](https://docs.google.com/document/d/193rANAJR6-Hb2c5Mn-51Ws_K7v9KT7F_/edit). Steps confirmed in the Gramene 70 build are marked (G70).

## 1. Goal and scope

- Deliver Gramene 70: a self-hosted EnsemblPlants 63 browser on Ensembl 116 code, served from `/usr/local/ensembl-116` on ayinger (`/usr/local/ensembl-live` points to it).
- User-facing features: genome, variation and comparative views; species pages; BioMart; BLAST (EBI REST); HMMER (link to EBI phmmer); and three local tools — VEP, Assembly Converter and ID History Converter.
- Gramene adds what EnsemblPlants does not: its own branding and search endpoint (`data.gramene.org/ebeye<release>`), local VCF-backed variation tracks, and control over which tools run and where.
- Status (G70): code installed including ensembl-hive 2.7.0 and ensembl-taxonomy; assembly chain data refreshed; 989 species pages and 284 images installed; taxon tree regenerated (267 species); EVA VCFs for teff, bean and cacao in place; download links moved to the EnsemblPlants 63 archive; VEP on the read-only `vep_ro` account with fixed htslib, retest passed 2026-11-17; Assembly Converter jobs complete; beekeeper running; ID History Converter patched, awaiting web retest.
- Out of scope: producing new genome data, running the EnsemblPlants production pipelines, and BLAST execution (EBI hosts it). Track hub speed and the Compara cactus alignment view (HALXS not built) are known gaps, not build blockers.

## 2. Inputs and methods

- **Databases.** Download EnsemblPlants MySQL dumps (hidden `.release-63` path before public release) into \~800 GB of scratch on brie. Also fetch plants mart, pan-division (compara pan, ontology, website, metadata, production) and variation VCFs.
- **Verification.** Run `sum *.gz` against each directory's `CHECKSUMS`; every diff file must be empty. For G70 assembly chain data, all 57 files matched.
- **Renaming and loading.** `phh-file-rename` maps EG release numbers to Gramene numbers; `install_databases_from_egdump.sh` loads colden, `install_pan_databases_from_dump.sh` splits pan databases between colden and cabot. Then `OPTIMIZE TABLE` everywhere.
- **Metadata.** Update `dbname` and release columns in `ensembl_metadata` (`genome`, `genome_database`, `compara_analysis`, `data_release`) for `division = 'EnsemblPlants'`, so species pages resolve renamed databases.
- **Code.** `setup_ensembl_live.sh` clones Ensembl and EG packages; re-run `fetch_eg_packages.sh` and `fetch_ensembl_packages.sh` after the EG release until nothing new arrives. Add ensembl-hive (G70: `version/2.7.0`), ensembl-pipeline and ensembl-taxonomy.
- **Dependencies.** Build htslib, Bio::DB::HTS and `ensembl-variation/C_code`. Apache and the hive both need `/usr/local/bin/perl` 5.36, which has DBI, DBD::mysql and (G70) DateTime::Format::ISO8601.
- **Configuration.** Plugins load in order: public-plugins → eg-web-common → eg-web-plants → gramene-live. Gramene settings live in `gramene-live/gramene/conf/SiteDefs.pm` and `ini-files/` (DEFAULTS, MULTI, per species); `ENSEMBL_SITETYPE` must be `'Ensembl Plants'`.
- **Static content.** `copy_files_to_websites.pl --release=116-63 --division=plants` from ensembl-static (G70). This copies species text and images only; it does not build the taxon tree.
- **Taxon tree.** `gramene-live/gramene/htdocs/taxon_tree_data.js` feeds the species selector (BLAST and other tools). Regenerate it every release with `eg-web-common/utils/dump_taxon_tree.pl`, which reads the core databases and `ncbi_taxonomy_<Ensembl version>` on colden:
  ```
  /usr/local/bin/perl ensembl-webcode/utils/dump_ensembl_valid_species.pl \
   | /usr/local/bin/perl eg-web-common/utils/dump_taxon_tree.pl \
       --host colden.cshl.edu --port 3306 --user gramene_web --pass <DB password> \
       --thost colden.cshl.edu --tport 3306 --tuser gramene_web --tpass <DB password> \
       --version 116 --plugin-dir <output dir>
  ```
  It writes `<output dir>/htdocs/taxon_tree_data.js`; copy that into `gramene-live/gramene/htdocs/`. Check that every leaf key opens `http://localhost/<key>/Info/Index` with 200. Apache serves it as a static file, so no restart is needed. G70: 267 species (was 157), commit 39333a8.
- **Tools.** Create the tools database (`create_tools_db.pl`), initialize the hive (`init_pipeline.pl`), start `beekeeper_manager.pl` as `nobody`. Tool data lives in `tools_data/`: VEP cache named by Ensembl version (116), chain files, and toplevel FASTA symlinked for CrossMap.
  - **VEP database account.** `ENSEMBL_VEP_SCRIPT_DEFAULT_OPTIONS` in `gramene/conf/SiteDefs.pm` uses `vep_ro`, a passwordless SELECT-only account on colden limited to ayinger (143.48.220.243), with `password => undef`. VEP writes its full command line into each job's `stats.txt`, which users can see, so the account must have no password. Don't set a VEP password in `gramene-private` either; it would override this and the login would fail. Grants: SELECT on `%\_core\_%`, `%\_variation\_%`, `%\_otherfeatures\_%` (add `%\_funcgen\_%` if regulatory annotation is needed).
  - **htslib on `PATH`.** The VEP job and the results page both run plain `bgzip` and `tabix`. They must be working builds in `/usr/local/bin`. G70 found the 2017 builds broken (they need `libcrypto.so.10`); replace them with the htslib 1.17 build from `/usr/local/src/htslib` (`bgzip`, `tabix`, `htsfile`). `samtools`, `md5fa` and `md5sum-lite` in `/usr/local/bin` have the same problem.
- **Gramene customizations** (gramene-live, branch `gramene-ensembl-70`):
  - Gene-page search links: Gramene Search for all species, plus Sorghumbase Search (Sorghum_bicolor NCBIv3), GrameneOryza (Oryza_sativa IRGSP-1.0) and GrameneMaize Search (Zea_mays Zm-B73-REFERENCE-NAM-5.0). See `Gramene-species-search-skills.md`.
  - Download links on species home pages (`Component/Info/HomePage.pm`) and the FTP table (`Document/HTML/FTPtable.pm`) point to the EnsemblPlants archive, `https://ftp.ensemblgenomes.ebi.ac.uk/pub/plants/release-63/`. They use two new SiteDefs variables, `EG_FTP_RELEASE_VERSION = 63` and `EG_FTP_URL`. `SITE_RELEASE_VERSION_EG` stays 70 because other code uses it. Archive folders are named by production name, and VCF-only species (teff, bean, cacao) have no GVF/VCF folders.
  - The header Downloads link goes to `/info/data/ftp/index.html`; the logo goes to `https://dev.gramene.org/main/v70/` (switch back to www.gramene.org at go-live).
- **Housekeeping crons** (scripts in `gramene/utils/`, installed in `/usr/local/ensembl-116`): `rotate_logs.sh` at 01:00 every Monday keeps one week of Apache logs; `clean_tmp_img.sh` at midnight on the 1st deletes images older than 30 days, and empty folders, in `tmp/img/tmp/temporary/`.

### Data availability

| Resource | Location |
| --- | --- |
| Gramene config and scripts | [warelab/gramene-ensembl](https://github.com/warelab/gramene-ensembl) → `gramene-live/` |
| Species static pages | [EnsemblGenomes/ensembl-static](https://github.com/EnsemblGenomes/ensembl-static), branch `release/eg/63` |
| Upstream databases | `ftp.ensemblgenomes.org/pub/release-63/plants/mysql` |
| VEP cache | `.../release-63/plants/variation/indexed_vep_cache` → `tools_data/vep` |
| Assembly chains | `.../release-63/plants/assembly_chain` → `tools_data/assembly_chain` (update note: `update_note_2026-10-23.txt`) |
| Variation VCFs | `/usr/local/vcf` (`DATAFILE_BASE_PATH`), linked as `data_files`; paths set in `eg-web-plants/conf/json/<species>_vcf.json` |
| EVA VCFs (teff, bean, cacao) | `/usr/local/vcf/eva` → `/usr/local/tools_data/eva`. Teff from `ftp.ebi.ac.uk/pub/databases/eva/PRJEB62522/`, with the `Chromosome_` prefix stripped and re-indexed (`.tbi`); the original is in `eva_original/` |
| Server root | `ayinger:/usr/local/ensembl-116` |
| Tools and hive databases | `ensembl_web_tools_ayinger`, `ensembl_web_hive_ayinger` |
| Beekeeper fixes | [Tools Hive Beekeeper Fixes](https://claude.ai/artifact/BR9XpM9TAaKPq4hNeuXJmg) |

## 3. Deliverables and gates

- **D1 / G0 — Verified databases.** All dumps checksum-clean, renamed, loaded on colden (and cabot replica), optimized, metadata gramenized. Gate: empty diff files and clean install logs.
- **D2 / G1 — Running web server.** Code on the right release branches, configured, Apache starts with no errors in `preload-errors.log`; species pages, BioMart and BLAST tree render. Gate: spot-check gene, variant and compara pages for several species.
- **D3 / G2 — Working tools.** Tools and hive databases initialized (`hive_sql_schema_version` present), beekeeper running as `nobody`, one successful job for VEP, Assembly Converter and ID History Converter. Gate (G70): Assembly Converter passed; VEP passed 2026-11-17 (rice, 3 variants; results table filled, `.tbi` present, no password in `stats.txt`); ID History Converter pending retest with an old ID such as `Zm00001d027230`.
- **D4 / G3 — Public release.** Site reviewed through the ensembl-dev.gramene.org proxy, then switched live. Gate: team sign-off; local patches committed to gramene-ensembl; credentials removed from shared logs.

## 4. People and resources

- **Build lead:** @Aisa. Roles named in the SOP, to confirm for G70: Peter — system administration, htslib reinstalls and FTP archiving; Andrew — Gramene search server and site styling; Kapeel — FTP disk checks each release.
- **Upstream contact:** EnsemblPlants (EBI) for dump locations before public release and for bugs found in shared plugins.
- **Disk:** \~800 GB scratch per build; each release adds \~400 GB to the 7 TB Gramene FTP disk; archive old releases at 90% full, keeping the latest 10.
- **Effort:** the database download alone can take hours; analyst-day estimates are TBD.

The site runs on one of two VMs, ayinger and paulaner, alternating each release. The live databases are on colden, with cabot as a replica and development server.

## 5. Reproducibility and decisions

- **Pinned versions (G70):** ensembl-hive `version/2.7.0` (152be3d4); ensembl-taxonomy `main` (894bbe9); ensembl-static `release/eg/63` (fc5de5fb). Record the branch and commit of every other repo at go-live.
- **Local patches to carry forward:** `RunnableDB/IDMapper.pm` (skip ensembl-webcode on the include path), `httpd.conf` (modules and `User`/`Group`), `Vcf.pm` (`https` → `http`, run tabix from `tmp`), SSL-check bypasses in GXA, Literature and EBeye modules, HMMER link in `ToolsTable.pm`, logo CSS. Only the logo CSS is confirmed committed to gramene-ensembl.
- **Config caches:** delete `conf/*.config.packed` after code changes, `conf/packed/<Species>.packed` after core changes, `conf/packed/MULTI.packed` after compara changes; delete `beekeeper.config` and `.pid` before restarting the beekeeper.
- **Silent failures:** `init_pipeline.pl` and `beekeeper_manager.pl` hide errors. Check `hive_meta` after init and the job's `stderr.log` after a failure; adding `die $@ if $@` to the init wrapper is proposed.
- **Credentials:** database passwords appear in `beekeeper.config`, job logs, the hive `analysis_data` table and the source SOP. Restrict files with `chmod 600`, replace passwords in the SOP with placeholders, and rotate after sharing logs.
  - Since G70, no password is in git. They live in an untracked plugin, `/usr/local/ensembl-116/gramene-private` (`conf/SiteDefs.pm` for the user DB; `conf/ini-files/DEFAULTS.ini` and `MULTI.ini` for the DB passwords). It's listed first in `conf/Plugins.pm` so it overrides gramene-live, and is owned `weix:nobody`, mode 640. Copy it by hand to each new build.
  - Push gramene-live over SSH (`git@github.com:warelab/gramene-ensembl.git`) with `git push origin gramene-ensembl-70`. `push.default=matching` makes a bare `git push` also try the stale `master`.
  - VEP: see the `vep_ro` account in section 2. `stats.txt` files from earlier jobs still contain the old VEP password, so rotate it.
- **Open decisions:** whether to commit all local patches upstream-safe in gramene-ensembl; whether to build HALXS for cactus alignments; whether to keep alternating ayinger and paulaner or settle on one VM.

## Appendix A · system architecture

*Runtime view of the finished site. Page requests read MySQL directly; tool requests leave a ticket and a hive job that the beekeeper runs against `tools_data`.*

&#91;embedded content: Gramene 70 architecture · 6 components\]

## Appendix B · known issues and fixes

*Recurring problems from Gramene 58 through 70. Keep each fix in gramene-ensembl so the next build picks it up.*

| Problem | Cause | Fix |
| --- | --- | --- |
| Hive jobs fail with "Can't locate DBI.pm" (Gramene 70) | Hive scripts use `#!/usr/bin/env perl`; system perl 5.32 lacks DBI, and sudo drops `/usr/local/bin` from `PATH` | Put `/usr/local/bin` first on `PATH` and run with `/usr/local/bin/perl` 5.36 |
| `init_pipeline` appears to succeed but dispatch fails (Gramene 70) | A missing `DateTime::Format::ISO8601`, needed by the SLURM meadow, killed init silently | Install the module with cpanm, re-run init, check `hive_meta` |
| Beekeeper exits silently on restart (Gramene 70) | A stale `beekeeper.config` cached a `PATH` without `/usr/local/bin` | Delete `beekeeper.config` and `beekeeper.pid` before restarting |
| ID History Converter fails on `DBAdaptor->new` (Gramene 70, also 69) | A stub module in `ensembl-webcode` shadows the real one in `ensembl-variation` | Exclude `ensembl-webcode` from the include path in `RunnableDB/IDMapper.pm` |
| VEP reports "No cache found" | Cache directories named with EG or Gramene version | Rename them to the Ensembl web version |
| Apache won't start (Ensembl 114) | New `httpd.conf` lacks modules and the `User`/`Group` lines | Restore the missing `LoadModule` and `User`/`Group` lines from the previous release |
| Expression, literature and search widgets fail | EBI certificate checks fail from the VM | Set `PERL_LWP_SSL_VERIFY_HOSTNAME = 0` in the widget modules, or use Gramene search |
| HMMER form returns "method not allowed" | EBI changed the phmmer endpoint | Link directly to EBI phmmer from `ToolsTable.pm` |
| Gramene logo missing (Gramene 69) | Custom CSS was never committed | Commit `02_page_template.css` to gramene-ensembl |
| Species selector misses species or links to 404 pages (Gramene 70) | `taxon_tree_data.js` was left from an old release; EnsemblPlants renamed some species with assembly accessions (e.g. `Solanum_lycopersicum` → `Solanum_lycopersicum_GCA_000188115.5cm`) | Regenerate it with `dump_taxon_tree.pl` each release (see Taxon tree in section 2) |
| VEP job succeeds but the results table is empty (Gramene 70) | `/usr/local/bin/bgzip` can't load `libcrypto.so.10`. `bgzip -c tmp > output.vcf` empties the output, then bgzip crashes, leaving `output.vcf` at 0 bytes. | Install htslib 1.17 `bgzip`/`tabix`/`htsfile` in `/usr/local/bin` and re-submit the job; the lost output can't be recovered |
| DB password visible in VEP job `stats.txt` (Gramene 70) | VEP records its whole command line, including `--password` | Use the passwordless read-only account `vep_ro` (section 2); rotate the old password |
| VEP `.web` summary has fewer rows than variants (Gramene 70) | Still 1 row for 3 variants after the bgzip fix. Probably `--fork 4`: each forked worker opens the file with `>` (cause not confirmed). | Results table is unaffected (it reads `output.vcf`). Open: try `fork => undef` for small jobs if the job summary matters |
| Gramene Search panel: "Can't locate object method get_gene_display_link" (Gramene 70) | Lines copied from Gene::Summary into `GrameneSearch.pm`; that method is in Component::Summary, not Component::Gene. Fails only for genes with a display xref. | Removed the unused lookups (commit cf0a57e) |
| Gene pages 500 with "Reference node is same as the new node" after adding a menu node | `create_node(...)` was wrapped in `$menu->after(...)` | `create_node` already appends the node; don't wrap it |
| Teff, bean and cacao variant tracks empty (Gramene 70) | The `_vcf.json` paths point to `/usr/local/vcf/eva/...`, which didn't exist. Teff's EVA file names chromosomes `Chromosome_1A`; the site tries `1A`, then the GenBank synonym, never that name. | Symlink `vcf/eva` → `tools_data/eva`; download teff and strip the prefix (see Data availability) |
| FTP table links 404 (Gramene 70) | Links used URL names, not production names (gca genomes, Avena strains), plus GVF/VCF links for VCF-only species, and the Gramene release number | Fixed in `FTPtable.pm` with production names and `EG_FTP_*` (commit f27d428) |
| AlphaFold 3D viewer: "An error occurred while loading the 3D protein viewer" | The viewer runs in the browser against `rest.ensembl.org`; that REST service was returning 500 (even `info/ping`) | Nothing to fix locally; check `curl https://rest.ensembl.org/info/ping` and retry when it returns `{"ping":1}` |
| "ERROR FETCHING FILE https://interactions.rest.ensembl.org/..." in the log after restart | EBI's interactions REST returns 500 | Harmless warning; the page loads without it |
| Command-line `tabix`/`samtools` fail with `libcrypto.so.10` (Gramene 70) | 2017 builds in `/usr/local/bin` linked against OpenSSL 1.0 | Use `/usr/local/ensembl-live/htslib/tabix`, or rebuild |
| Compara cactus alignment view fails | HALXS XS module not built | Open: build HALXS in ensembl-compara |
