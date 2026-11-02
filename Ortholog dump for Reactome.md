# Ortholog dump for Reactome  
  
  
1. Need to optimize the compara database. $dbname  
  
for tb in `mysql-cabot -Nq $dbname -e "show tables"`; do echo $tb; mysql-cabot -Nq $dbname -e "optimize table $tb"; done  
  
  
2 Dump ortholgs of other species to Rice with  
  
SELECT g1.name as species, gm1.stable_id,  g2.name as otherspecies, gm2.stable_id as other_stable_id, hm1.perc_id, hm2.perc_id as other_perc_id, h.is_high_confidence FROM method_link_species_set INNER JOIN homology h USING(method_link_species_set_id) INNER JOIN homology_member hm1 on (h.homology_id=hm1.homology_id) INNER JOIN gene_member gm1 on (hm1.gene_member_id=gm1.gene_member_id) INNER JOIN genome_db g1 ON gm1.genome_db_id = g1.genome_db_id INNER JOIN homology_member hm2 on (h.homology_id=hm2.homology_id) INNER JOIN gene_member gm2 on (hm2.gene_member_id=gm2.gene_member_id) INNER JOIN genome_db g2 ON gm2.genome_db_id = g2.genome_db_id WHERE method_link_id = 201 AND g1.genome_db_id = 1985 AND g2.genome_db_id != 1985 order by otherspecies  
  
  
```
mysql-colden -q $dbname -e "SELECT g1.name as species, gm1.stable_id,  g2.name as otherspecies, gm2.stable_id as other_stable_id, hm1.perc_id, hm2.perc_id as other_perc_id, h.is_high_confidence FROM method_link_species_set INNER JOIN homology h USING(method_link_species_set_id) INNER JOIN homology_member hm1 on (h.homology_id=hm1.homology_id) INNER JOIN gene_member gm1 on (hm1.gene_member_id=gm1.gene_member_id) INNER JOIN genome_db g1 ON gm1.genome_db_id = g1.genome_db_id INNER JOIN homology_member hm2 on (h.homology_id=hm2.homology_id) INNER JOIN gene_member gm2 on (hm2.gene_member_id=gm2.gene_member_id) INNER JOIN genome_db g2 ON gm2.genome_db_id = g2.genome_db_id WHERE method_link_id = 201 AND g1.genome_db_id = 1985 AND g2.genome_db_id != 1985 order by otherspecies" >Osj2allOrthologs.txt 

```
  
  
3. Convert to the format reactome want  
  
```
perl /usr/local/gramene-ensembl/scripts/misc-scripts/convert_sqldump2rmt.pl Osj2allOrthologs.txt 

```
```
perl /usr/local/gramene-ensembl/scripts/misc-scripts/convert_sqldump2rmt.pl Osj2allOrthologs.txt
Total number of records is 4344990

```
  
for build 116  
perl /usr/local/gramene-ensembl/scripts/misc-scripts/convert_sqldump2rmt.pl Osj2allOrthologs.txt  
Total number of records is 4514830  
  
Release build:  
  
ftp://ftp.ensemblgenomes.org/pub/release-42/plants/mysql/  
  
  
Post release syntney build for next release.  
  
Investigate what syntney pairs are in current compara db  
  
select mlss.method_link_species_set_id, mlss.name, g1.name, g2.name, count(*) from synteny_region sr join method_link_species_set mlss using(method_link_species_set_id) join species_set ss1 on ( mlss.species_set_id=ss1.species_set_id) join genome_db g1 on(ss1.genome_db_id=g1.genome_db_id) join species_set ss2 on(ss2.species_set_id=mlss.species_set_id) join genome_db g2 on(ss2.genome_db_id=g2.genome_db_id) where g1.genome_db_id < g2.genome_db_id    group by 1   
  
  
  
=======  
EG new ortholog dump readme (Apr 14, 2026)  
========  
https://raw.githubusercontent.com/Ensembl/ensembl-compara/refs/heads/release/116/docs/ftp/README.gene_trees.tsv_dumps.txt  
Directory 'tsv/ensembl-compara/homologies' contains tab-separated  
value (TSV) dumps of homologies inferred on gene trees.  
  
All homology TSV dump files have the same naming convention and fields, as follows.  
  
Compara.{release}.{protein|ncrna}_{gene_tree_collection}.homologies.tsv.gz  
  Each homology between a pair of genes is represented  
  on one tab-delimited line, with the following fields:  
    gene_stable_id : gene stable_id of the first homologous gene  
    protein_stable_id : sequence stable_id of the first homologous gene (may be a protein or transcript stable_id, depending on whether the gene is protein-coding)  
    species : name of the genome containing the first homologous gene  
    identity : identity between homologous sequences, expressed as a percentage of the length of the representative sequence of the first homologous gene  
    homology_type : homology type and cardinality (e.g. 'ortholog_one2one')  
    homology_gene_stable_id : gene stable_id of the second homologous gene  
    homology_protein_stable_id : sequence stable_id of the second homologous gene (may be a protein or transcript stable_id)  
    homology_species : name of the genome containing the second homologous gene  
    homology_identity : identity between homologous sequences, expressed as a percentage of the length of the representative sequence of the second homologous gene  
    dn : non-synonymous mutation rate (currently unused)  
    ds : synonymous mutation rate (currently unused)  
    goc_score : gene order conservation (GOC) score of the homology  
    wga_coverage : whole genome alignment (WGA) coverage of the homology  
    is_high_confidence : whether this is considered a 'high-confidence' homology  
    homology_id : unique internal ID of the homology  
  Note that within these files, the order of the first and second homologous genes within each row  
  is arbitrary, and should not be interpreted as conferring any special status on one or the other.  
  Both genes are co-equal participants in a homology relationship.  
  
The contents of these files differs depending on their location relative  
to the directory 'tsv/ensembl-compara/homologies'.  
  
Each homology TSV dump file at the top level in directory 'tsv/ensembl-compara/homologies' contains the complete  
set of available homologies for a given gene-tree collection (e.g. 'default') and member type (e.g. 'protein').  
It is recommended to download this complete homology TSV file if you need to access most or all of the  
homologies in a gene-tree collection.  
  
For those who need access to homologies for a subset of genomes in a gene-tree collection, genome-specific homology TSV dump files  
are available within a subdirectory of 'tsv/ensembl-compara/homologies' named for a particular genome. Subdirectories are structured  
in a way that mirrors the directory structure for single-species TSV data, with data for genomes from species-specific core  
databases in a single directory named for that species (e.g. 'tsv/ensembl-compara/homologies/saccharomyces_cerevisiae'),  
and data for genomes from collection core databases in a directory path which includes the name of the collection core  
(e.g. 'tsv/ensembl-compara/homologies/fungi_ascomycota2_collection/erysiphe_necator_gca_000798715').  
  
NOTE: To eliminate redundancy, each genome-specific homology TSV file contains an arbitrary  
subset of orthologies involving the given genome. To access all available orthologies between two genomes  
(e.g. 'drosophila_melanogaster' and 'saccharomyces_cerevisiae'), you will need to download the genome-specific  
files of both genomes (e.g. 'drosophila_melanogaster/Compara.116.protein_default.homologies.tsv.gz'  
and 'saccharomyces_cerevisiae/Compara.116.protein_default.homologies.tsv.gz').  
  
  
Last release 116, 2026  
https://ftp.ebi.ac.uk/pub/ensemblgenomes/current/plants/tsv/ensembl-compara/homologies/oryza_sativa/  
