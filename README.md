# Atividade Amostras Extras (Pipeline Variantes Somáticas)

# Antes de Começar:
## Link para acessar os arquivos BAM:
```
https://github.com/renatopuga/somatico
```
### Download dos cromossomos 13 e 19:

``` bash
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr19.fa.gz
```
``` bash
wget -c https://hgdownload.soe.ucsc.edu/goldenPath/hg19/chromosomes/chr13.fa.gz
```
### Conferindo se os dois arquivos foram salvos:

``` bash
ls -lh chr13.fa.gz chr19.fa.gz
```
#### Exemplo de Saída Esperada:
```
-rw-rw-rw- 1 codespace codespace 31M Mar 20  2009 chr13.fa.gz

-rw-rw-rw- 1 codespace codespace 17M Mar 20  2009 chr19.fa.gz
```
### Montar a referência hg19.fa:

``` bash
zcat chr13.fa.gz chr19.fa.gz | sed 's/^>chr/>/' > hg19.fa
```
### Conferir os dois cabeçalhos de uma vez:

``` bash
grep '^>' hg19.fa
```
#### Exemplo de Saída Esperada:
```
>13

>19
```
### Indexar a referencia:

``` bash
samtools faidx hg19.fa
```
### Conferir o indice gerado:

``` bash
cat hg19.fa.fai
```

#### Exemplo de Saída Esperada:
```
13      115169878       4       50      51

19      59128983        117473284       50      51
```
### Criar o dicionário da referencia:

``` bash
./gatk-4.2.2.0/gatk CreateSequenceDictionary -R hg19.fa -O hg19.dict
```
### Conferir as duas sequências no dicionário:

``` bash
grep '^@SQ' hg19.dict
```
#### Exemplo de Saída Esperada:
```
@SQ     SN:13   LN:115169878    M5:283f8d7892baa81b510a015719ca7b0b     UR:file:/workspaces/somatico/hg19.fa

@SQ     SN:19   LN:59128983     M5:1aacd71f30db8e561810913e0b72636d     UR:file:/workspaces/somatico/hg19.fa
```
### Gerar a lista de intervalos:

``` bash
./gatk-4.2.2.0/gatk ScatterIntervalsByNs \
  -R hg19.fa \
  -O hg19.interval_list \
  -OT ACGT
```
### Confirmar que os dois cromossomos têm intervalos:

``` bash
awk '!/^@/ {print $1}' hg19.interval_list | sort -n | uniq -c
```

#### Exemplo de Saída Esperada:
```
5 13

5 19
```
### Conferir os arquivos das amostras extras:

``` bash
ls -lh *wp190* *wp191* af-only-gnomad-chr13-chr19.vcf.gz*
```

#### Exemplo de Saída Esperada:
```
-rw-rw-rw- 1 codespace root 124K Nov 14 12:42 af-only-gnomad-chr13-chr19.vcf.gz

-rw-rw-rw- 1 codespace root  255 Nov 14 12:42 af-only-gnomad-chr13-chr19.vcf.gz.tbi

-rw-rw-rw- 1 codespace root 1.9M Nov 14 12:42 tumor_wp190.bam

-rw-rw-rw- 1 codespace root  15K Nov 14 12:42 tumor_wp190.bam.bai

-rw-rw-rw- 1 codespace root 605K Nov 14 12:42 tumor_wp191.bam

-rw-rw-rw- 1 codespace root  15K Nov 14 12:42 tumor_wp191.bam.bai
```
### Confirmar os IDs das amostras nos BAMs:

``` bash
samtools view -H tumor_wp190.bam | grep '^@RG'
samtools view -H tumor_wp191.bam | grep '^@RG'
```

#### Exemplo de Saída Esperada:
```
@RG     ID:test8        PL:illumina     LB:test_8_WP190 SM:WP190        CN:NGSColumbia

@RG     ID:03615        PL:illumina     LB:0361_5_WP191 SM:WP191        CN:NGSColumbia
```
### Rodar o Mutect2 para o par WP190/WP191:

``` bash
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp190.bam \
  -I tumor_wp191.bam \
  -normal WP191 \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP190.somatic.vcf.gz
```
### Gerar as tabelas de contagem do tumor e do normal:

``` bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp190.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP190.table
```
``` bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp191.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP191.table
```
### Conferindo os cabeçalhos e as primeiras contagens:

``` bash
head -n 5 WP190.table WP191.table
```

#### Exemplo de Saída Esperada:
```
==> WP190.table <==

#<METADATA>SAMPLE=WP190

contig  position        ref_count       alt_count       other_alt_count allele_frequency

13      28585519        2       0       0       0.027

13      28585589        2       0       0       0.038

13      28586085        0       0       0       0.021
```
```
==> WP191.table <==

#<METADATA>SAMPLE=WP191

contig  position        ref_count       alt_count       other_alt_count allele_frequency

13      28579332        1       0       0       0.029

13      28584281        1       0       0       0.028

13      28584707        0       2       0       0.031
```
### Estimar a contaminação no tumor WP190:

``` bash
./gatk-4.2.2.0/gatk CalculateContamination \
  -I WP190.table \
  -matched WP191.table \
  -O WP190.contamination.table
```
### Conferindo o resultado:

``` bash
cat WP190.contamination.table
```

#### Exemplo de Saída Esperada:
```
sample  contamination   error

WP190   0.0     4.76837158203125E-7
```
### Filtrar as chamadas do par WP190/WP191:

``` bash
./gatk-4.2.2.0/gatk FilterMutectCalls \
  -R hg19.fa \
  -V WP190.somatic.vcf.gz \
  --contamination-table WP190.contamination.table \
  -O WP190.filtered.vcf.gz
```
### Contar as chamadas por valor da coluna FILTER:

``` bash
zcat WP190.filtered.vcf.gz | awk '!/^#/ {print $7}' | sort | uniq -c
```
### Conferir os BAMs do par WP017/WP018:

``` bash
ls -lh *wp017* *wp018*
```

#### Exemplo de Saída Esperada:
```
-rw-rw-rw- 1 codespace root 791K Nov 14 12:42 tumor_wp017.bam

-rw-rw-rw- 1 codespace root 6.5K Nov 14 12:42 tumor_wp017.bam.bai

-rw-rw-rw- 1 codespace root 384K Nov 14 12:42 tumor_wp018.bam

-rw-rw-rw- 1 codespace root 6.5K Nov 14 12:42 tumor_wp018.bam.bai
```
### Conferindo os IDs nos cabeçalhos:

``` bash
samtools view -H tumor_wp017.bam | grep '^@RG'
samtools view -H tumor_wp018.bam | grep '^@RG'
```
#### Exemplo de Saída Esperada:
```
@RG     ID:1    PL:illumina     PU:WP017.sorted.ddup.bam        LB:tumor        SM:WP017

@RG     ID:1    PL:illumina     PU:WP018.sorted.ddup.bam        LB:normal       SM:WP018
```
### Rodar o Mutect2 para WP017/WP018:

``` bash
./gatk-4.2.2.0/gatk Mutect2 \
  -R hg19.fa \
  -I tumor_wp017.bam \
  -I tumor_wp018.bam \
  -normal WP018 \
  --germline-resource af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP017.somatic.vcf.gz
```
### Gerar as tabelas de contagem do tumor e do normal:

``` bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp017.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP017.table
```
``` bash
./gatk-4.2.2.0/gatk GetPileupSummaries \
  -I tumor_wp018.bam \
  -V af-only-gnomad-chr13-chr19.vcf.gz \
  -L hg19.interval_list \
  -O WP018.table
```
### Conferindo os IDs e as primeiras contagens:

``` bash
head -n 5 WP017.table WP018.table
```

#### Exemplo de Saída Esperada:
```
==> WP017.table <==

#<METADATA>SAMPLE=WP017

contig  position        ref_count       alt_count       other_alt_count allele_frequency

19      13049473        137     0       0       0.046

19      13050201        79      0       0       0.044

19      13050688        44      1       0       0.016
```
```
==> WP018.table <==

#<METADATA>SAMPLE=WP018

contig  position        ref_count       alt_count       other_alt_count allele_frequency

19      13049473        69      0       1       0.046

19      13050201        44      0       0       0.044

19      13050688        26      1       0       0.016
```
### Estimar a contaminação no tumor WP017:

``` bash
./gatk-4.2.2.0/gatk CalculateContamination \
  -I WP017.table \
  -matched WP018.table \
  -O WP017.contamination.table
```
### Conferindo o resultado:

``` bash
cat WP017.contamination.table
```

#### Exemplo de Saída Esperada:
```
sample  contamination   error

WP017   0.08101101749837979     0.23686839645163477
```
### Filtrar as chamadas do par WP017/WP018:

``` bash
./gatk-4.2.2.0/gatk FilterMutectCalls \
  -R hg19.fa \
  -V WP017.somatic.vcf.gz \
  --contamination-table WP017.contamination.table \
  -O WP017.filtered.vcf.gz
```
### Contar as chamadas por valor de FILTER:

``` bash
zcat WP017.filtered.vcf.gz | awk '!/^#/ {print $7}' | sort | uniq -c
```
### Identificar as três chamadas PASS:

``` 
for vcf in WP190.filtered.vcf.gz WP017.filtered.vcf.gz; do
  printf '\n%s\n' "$vcf"
  zcat "$vcf" | awk '/^#CHROM/ || (!/^#/ && $7 == "PASS")'
done
```
