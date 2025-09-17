**How to run:**

```
singularity exec \
  --bind /lnx01_data2:/lnx01_data2 \
  /data/shared/programmer/simg/updfinder.sif \
  /opt/micromamba/envs/updfinder/bin/python /opt/updfinder/updfinder.py \
  -p /path/to/proband/vcf.gz \
  -m /path/to/mothers/vcf.gz \
  -f /path/to/fathers/vcf.gz \
  -o /path/to/output-folder \
  -x name of project
```


**Formål (hvad scriptet gør)**

Finder genomiske regioner med uniparental disomi (UPD) i en trio (barn, mor, far) ud fra VCF’er (helgenom). Det bruger tre uafhængige evidenstyper pr. vindue og samler dem til kald:

MIE (Mendelian Inheritance Errors) – når barnets genotype ikke kan forklares af forældrene.

PSB (Parental Specific B-allele) – bias i hvilke heterozygote sites der har B-allel fra mor vs. far.

SBV (Skewed B-allele VAF) – systematisk forskel i barnets VAF for maternelle vs. paternelle B-alleler.

Derudover beregnes dybde-ratio og overlap til evt. CNV-input for at flagge UPD-regioner der i virkeligheden ligner kopitalforandringer.

**Input (CLI-parametre)**

**Obligatoriske:**

```
-p/--vcf-patient – VCF for barn (bgz + .tbi anbefales)

-m/--vcf-mother – VCF for mor

-f/--vcf-father – VCF for far

-o/--output-folder – outputmappe (scriptet laver også igv/)

-x/--prefix – præfix til outputfilnavne
```

**Valgfrie:**
```
-c/--cnv – enkel TSV/bed-agtig fil: chrom start end [DEL|LOSS|DUP|GAIN]

-q/--gq – GQ cutoff (default 90)

-w/--window – vinduesstørrelse i bp (default 1,000,000)

-s/--step – glidende step i bp (default 500,000)

--mie-pval – FDR-tærskel for MIE (default 1e-6)

--sbv-pval – Holm-tærskel for SBV (default 1e-10)
```

**Forventninger til data**

hg38 kromnavne (chr1…chr22, chrX, chrY) – scriptet masker centromerer for hg38.

VCF’erne skal være bi-alleliske SNPs med formatfelter GT, GQ, DP, AD (i den rækkefølge scriptet slår dem op via FORMAT-kolonnen).

Kun PASS (eller tom FILTER) og kun 1-bp ref/alt tages med.

Sample-ID hentes fra hver enkelt VCF og bruges til at finde kolonner efter merge.




**Berig per-kald:**

Midlende statistik fra vinduer i regionen: MIE-p, PSB-log2ratio, SBV-p (gemmes i tabellen).

SNP_DEPTH_LOG2RATIO: median DP i region / median DP på autosomer (log2).

CNV overlap (valgfrit): andel af region der dækkes af LOSS/GAIN.

WARNING: “CNV-gain” ved dybde>~0.1 log2 eller DUP-overlap > 0.5; “CNV-loss” ved modsat.



**Output:**
```
<prefix>_upd_significant.txt – hovedtabel med alle UPD-kald + metadata og evidenskolonner.
```

**igv BED:**
```
<prefix>_upd_significant.bed (rød=mat, blå=pat; mosaik samme farver).
```


**WIG tracks:**
```
<prefix>_vaf.wig – alle VAF.
<prefix>_snp_dp.wig – SNP-dybder.
<prefix>_b_origin_mat.wig – VAF hvor B-allel er maternel.
<prefix>_b_origin_pat.wig – VAF hvor B-allel er paternel.
```

**MIE-SNPs BED:**
```
<prefix>_mie_snps.bed (enkelt-SNP evidens).
```

**IGV session:**
```
<prefix>_igv_session.xml (sat til genome="hg38"). # til nærmere undersøgelse af kald
```

**Figur:**
```
<prefix>_updfinder_plot.png – pr. kromosom: DP, VAF, PSB, MIE, og markerede UPD-regioner.
```



**Fortolkning**

En sand isodisomi (klassisk UPD på hele kromosomet) vil ofte vise:

Mange MIE i én retning (UPD-mat eller UPD-pat) → stærk MIE-evidens.

PSB-ratio stærkt skæv (mange flere b_mat eller b_pat).

SBV-signal (forskellig median VAF for b_mat vs b_pat) især ved mosaik.

Ingen dybdeafvigelse (SNP_DEPTH_LOG2RATIO ~ 0), da UPD ikke er kopitalændring.

Segmental/mosaik UPD: typisk stærk SBV over en del af kromosomet; MIE/PSB kan være svagere.

Hvis depth og/eller CNV overlap er afvigende → kan være CNV snarere end UPD (warning-feltet hjælper).

VCF’erne skal have GT:GQ:DP:AD i FORMAT (scriptet finder indeks via FORMAT, men forventer at de findes).

Kun SNPs (enkelt-base ref/alt) og PASS indgår.

Kromnavne hg38 med “chr”. (Centromer-masken er hardcodet til hg38.)




**Væsentlige felter:**

chrom, start, end, UPD, EVIDENCE (UPD-mat / UPD-pat / *_mosaic; EVIDENCE = MIE/PSB/SBV)

MIE_PVAL_ADJ, PSB_LOG2RATIO, SBV_PVAL_ADJ

SNP_DEPTH_LOG2RATIO (≈ 0 ved “ren” UPD), CNV_DEL_COV, CNV_DUP_COV, WARNING


**Performance / praktisk**

Store VCF’er → RAM/CPU; vindue/step kan øges (færre vinduer) for hurtigere kørsel.

Hvis du allerede har en korrekt multi-sample VCF, kan du springe merge-trin over ved at pege -p/-m/-f på samme multi-sample VCF (men scriptet forventer tre filer i koden, så det er lettest at beholde merge).

X-håndtering: scriptet udelader X hvis estimeret X-antal <2; for piger (2 X) beholdes X.
