# vcf2maf

This repository lets you set up a vcf2maf (and hence maf2maf) Docker image.  It is based on the [MSKCC's vcf2maf repository](https://github.com/mskcc/vcf2maf) repository - the original README follows these notes.  In contrast to the official distribution, this Docker image makes VEP available, with its offline cache.

# Notes

To build the Docker file added the code set up vep to the end of the Dockerfile, following the instructions in this gist https://gist.github.com/ckandoth/f265ea7c59a880e28b1e533a6e935697.

To build a new Docker image, run, from this directory,

```
docker build . -t user/vcf2maf
```
I previously used `mawds` as `user`.

You will need to decide where to put the vep cache data on the host machine - I used `/opt/vepdata`.  

Having built the image, you will need to download and unpack the (large) offline caches to this directory:

(this is taken from the gist referenced above)

```
export VEP_DATA=/opt/vepdata
rsync -zvh rsync://ftp.ensembl.org/ensembl/pub/release-86/variation/VEP/homo_sapiens_vep_86_GRCh37.tar.gz $VEP_DATA
rsync -zvh rsync://ftp.ensembl.org/ensembl/pub/release-86/variation/VEP/homo_sapiens_vep_86_GRCh38.tar.gz $VEP_DATA
rsync -zvh rsync://ftp.ensembl.org/ensembl/pub/release-86/variation/VEP/mus_musculus_vep_86_GRCm38.tar.gz $VEP_DATA
cat $VEP_DATA/*_vep_86_GRC{h37,h38,m38}.tar.gz | tar -izxf - -C $VEP_DATA
unset VEP_DATA 
```

These then need to be installed from within the Docker container:

```
docker run -it -v $PWD/vepdata:/root/.vep \
	mawds/vcf2maf /bin/bash -c \
	'perl INSTALL.pl --AUTO af --SPECIES homo_sapiens \
	--ASSEMBLY GRCh37 --DESTDIR $VEP_PATH --CACHEDIR $VEP_DATA' 

docker run -it -v $PWD/vepdata:/root/.vep \
	mawds/vcf2maf /bin/bash -c \
	'perl INSTALL.pl --AUTO af --SPECIES homo_sapiens \
	--ASSEMBLY GRCh38 --DESTDIR $VEP_PATH --CACHEDIR $VEP_DATA' 

docker run -it -v $PWD/vepdata:/root/.vep \
	mawds/vcf2maf /bin/bash -c \
	'perl INSTALL.pl --AUTO af --SPECIES mus_musculus \
	--ASSEMBLY GRCm38 --DESTDIR $VEP_PATH --CACHEDIR $VEP_DATA' 
```
etc.  Note single quotes around command - `$VEP_PATH` and `$VEP_DATA` are defined inside the container

These run the have the effect of running the following lines from the gist within the container:

```
perl INSTALL.pl --AUTO af --SPECIES homo_sapiens --ASSEMBLY GRCh37 --DESTDIR $VEP_PATH --CACHEDIR $VEP_DATA
perl INSTALL.pl --AUTO af --SPECIES homo_sapiens --ASSEMBLY GRCh38 --DESTDIR $VEP_PATH --CACHEDIR $VEP_DATA
perl INSTALL.pl --AUTO af --SPECIES mus_musculus --ASSEMBLY GRCm38 --DESTDIR $VEP_PATH --CACHEDIR $VEP_DATA
```

You will then need to run the following commands to covert the offline cache.  Each command takes several hours to run:

```
docker run -it -v $PWD/vepdata:/root/.vep \
	mawds/vcf2maf /bin/bash -c \
	'perl convert_cache.pl --species homo_sapiens \
	--version 86_GRCh37 --dir $VEP_DATA'

docker run -it -v $PWD/vepdata:/root/.vep \
	mawds/vcf2maf /bin/bash -c \
	'perl convert_cache.pl --species homo_sapiens \
	--version 86_GRCh38 --dir $VEP_DATA'

docker run -it -v $PWD/vepdata:/root/.vep \
	mawds/vcf2maf /bin/bash -c \
	'perl convert_cache.pl --species mus_musculus \
	--version 86_GRCm38 --dir $VEP_DATA'
```

Finally, donwload ExAC on the host machine:
```
curl -L ftp://ftp.broadinstitute.org:/pub/ExAC_release/release0.3.1/subsets/ExAC_nonTCGA.r0.3.1.sites.vep.vcf.gz > /opt/vepdata/ExAC_nonTCGA.r0.3.1.sites.vep.vcf.gz
```

(Substituting /opt/vepdata if you stored the vep data elsewhere). The file is ~3GB

## To run:

Example invocation:

```
docker run -it -v $PWD/vepdata:/root/.vep -v $PWD/testmaf:/opt/maf mawds/vcf2maf /bin/bash -c 'perl ./maf2maf.pl --input-maf=/opt/maf/simplemaf.maf --output-maf=/opt/maf/processedmaf.maf --ref-fasta /root/.vep/homo_sapiens/86_GRCh37/Homo_sapiens.GRCh37.75.dna.primary_assembly.fa.gz --vep-path=/opt/variant_effect_predictor_86/ensembl-tools-release-86/scripts/variant_effect_predictor/'
```


The original README follows:

vcf<img src="http://i.giphy.com/R6X7GehJWQYms.gif" width="30">maf
=======

To convert a [VCF](http://samtools.github.io/hts-specs/) into a [MAF](https://wiki.nci.nih.gov/x/eJaPAQ), each variant must be mapped to only one of all possible gene transcripts/isoforms that it might affect. But even within a single isoform, a `Missense_Mutation` close enough to a `Splice_Site`, can be labeled as either in MAF format, but not as both. **This selection of a single effect per variant, is often subjective. And that's what this project attempts to standardize.** The `vcf2maf` and `maf2maf` scripts leave most of that responsibility to [Ensembl's VEP](http://useast.ensembl.org/info/docs/tools/vep/index.html), but allows you to override their "canonical" isoforms, or use a custom ExAC VCF for annotation. Though the most useful feature is the **extensive support in parsing a wide range of crappy MAF-like or VCF-like formats** we've seen out in the wild.

Quick start
-----------

Find the [latest stable release](https://github.com/mskcc/vcf2maf/releases), download it, and view the detailed usage manuals for `vcf2maf` and `maf2maf`:

    export VCF2MAF_URL=`curl -sL https://api.github.com/repos/mskcc/vcf2maf/releases | grep -m1 tarball_url | cut -d\" -f4`
    curl -L -o mskcc-vcf2maf.tar.gz $VCF2MAF_URL; tar -zxf mskcc-vcf2maf.tar.gz; cd mskcc-vcf2maf-*
    perl vcf2maf.pl --man
    perl maf2maf.pl --man

If you don't have [VEP](http://useast.ensembl.org/info/docs/tools/vep/index.html) installed, then [follow this gist](https://gist.github.com/ckandoth/f265ea7c59a880e28b1e533a6e935697). Of the many annotators out there, VEP is preferred for its large team of active coders, and its CLIA-compliant [HGVS formats](http://www.hgvs.org/mutnomen/recs.html). After installing VEP, you can test the script like so:

    perl vcf2maf.pl --input-vcf tests/test.vcf --output-maf tests/test.vep.maf

To fill columns 16 and 17 of the output MAF with tumor/normal sample IDs, and to parse out genotypes and allele counts from matched genotype columns in the VCF, use options `--tumor-id` and `--normal-id`. Skip option `--normal-id` if you didn't have a matched normal:

    perl vcf2maf.pl --input-vcf tests/test.vcf --output-maf tests/test.vep.maf --tumor-id WD1309 --normal-id NB1308

VCFs from variant callers like [VarScan](http://varscan.sourceforge.net/somatic-calling.html#somatic-output) use hardcoded sample IDs TUMOR/NORMAL in the genotype columns of the VCF. To have this script correctly parse the correct genotype columns, while still printing the proper IDs in the output MAF:

    perl vcf2maf.pl --input-vcf tests/test_varscan.vcf --output-maf tests/test_varscan.vep.maf --tumor-id WD1309 --normal-id NB1308 --vcf-tumor-id TUMOR --vcf-normal-id NORMAL

If you have the VEP script in a different folder like `/opt/vep`, and its cache in `/srv/vep`, there are options available to use those instead:

    perl vcf2maf.pl --input-vcf tests/test.vcf --output-maf tests/test.vep.maf --vep-path /opt/vep --vep-data /srv/vep

maf2maf
-------

If you have a MAF or a MAF-like file that you want to reannotate, then use `maf2maf`, which simply runs `maf2vcf` followed by `vcf2maf`:

    perl maf2maf.pl --input-maf tests/test.maf --output-maf tests/test.vep.maf

After tests on variant lists from many sources, `maf2vcf` and `maf2maf` are quite good at dealing with formatting errors or "MAF-like" files. It even supports VCF-style alleles, as long as `Start_Position == POS`. But it's OK if the input format is imperfect. Any variants with a reference allele mismatch are kept aside in a separate file for debugging. The bare minimum columns that `maf2maf` expects as input are:

    Chromosome	Start_Position	Reference_Allele	Tumor_Seq_Allele2	Tumor_Sample_Barcode
    1	3599659	C	T	TCGA-A1-A0SF-01
    1	6676836	A	AGC	TCGA-A1-A0SF-01
    1	7886690	G	A	TCGA-A1-A0SI-01

See `data/minimalist_test_maf.tsv` for a sampler. Addition of `Tumor_Seq_Allele1` will be used to determine zygosity. Otherwise, it will try to determine zygosity from variant allele fractions, assuming that arguments `--tum-vad-col` and `--tum-depth-col` are set correctly to the names of columns containing those read counts. Specifying the `Matched_Norm_Sample_Barcode` with its respective columns containing read-counts, is also strongly recommended. Columns containing normal allele read counts can be specified using argument `--nrm-vad-col` and `--nrm-depth-col`.

License
-------

    Apache-2.0 | Apache License, Version 2.0 | https://www.apache.org/licenses/LICENSE-2.0
