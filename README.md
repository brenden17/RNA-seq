## RNA seq 튜토리얼

https://docs.tinybio.cloud/docs/rna-seq-tutorial-from-fastq-to-figures


이 튜토리얼에서는 인간 병원성 효모 C_andida parapsilosis_의 RNA-Seq 데이터 분석.

C. 파라실리증은 주요 기회성 곰팡이 병원체 중 하나.

이 연구는 두 가지 상태(플랑크톤 및 생물막)에서 각각 세 개의 생물학적 복제본 C. 파라실리시스균을 평가함.

샘플은 가닥별 라이브러리 준비 프로토콜로 준비된 다음, 쌍단 2x90 bp 읽기 길이의 Illumina HiSeq 2000 장치를 사용하여 시퀀. 모든 샘플은 효모의 차등 유전자 발현 분석을 위해 매우 풍부한 약 1,300만 개의 페어링 엔드 판독에 대해 시퀀싱되었습니다.

이 튜토리얼의 목표는 인간 곰팡이 병원균인 C. 파라실리시스균의 병원성 형태와 비병원성 형태 간에 차별적으로 발현되는 유전자를 찾는 것입니다.


세포에서 RNA가 발생되면 빠르게 얼리거나 기타 작업으로 보존한다.
RNA를 cDNA로 만든다음, cDNA를 참조 지놈, ref로 활용하여 정렬한다.
이때, gff를 사용하는데, 특정 gene를 비교할 수 있다.

~~~~

conda create -n rnaseq_env -c bioconda -c conda-forge \
  sra-tools fastqc hisat2 samtools subread trimmomatic gffread star igv salmon 

conda activate rnaseq_env

# when adding tools
conda install salmon gffread --channel conda-forge --channel bioconda  --channel defaults --strict-channel-priority

conda deactivate

# pagekage list
conda list


# conda env list
conda env list

~~~~

Steps:
 1. 데이터 fastq 구함           SRA
 2. 데이터 fastq 평가           FASTQC, MultiQC
 3. 데이터 fastq 제거           Trimmomatic 
 4. 데이터 fastq 평가           FASTQC, MultiQC
 5. 참조 지놈 reference the genome
 6. 정렬, Alignement            Star
 6.1. 시각화                    igv
 7. Pseudo/quasi-mapping        Kallisto, Salmon, Alevin, RapMap
 8. RNA-Seq Differential Gene Expression Analysis

0. pre

~~~~

mkdir -p ./ref_gen/ ./mapping ./raw_data/trimming

~~~~


1. 데이터 fastq 구함           SRA

데이터 fastq 구하고 편하게 이름 변경

Go to https://sra-explorer.info/,  search "PRJNA246482"

Download 6 samples
 * GSM1382947: WT **planktonic** rep1; Candida parapsilosis; RNA-Seq   SRR1278968  Illumina HiSeq 2000   24700   22 Jul 2015
 * GSM1382948: WT **planktonic** rep2; Candida parapsilosis; RNA-Seq   SRR1278969  Illumina HiSeq 2000   24720   22 Jul 2015
 * GSM1382949: WT **planktonic** rep3; Candida parapsilosis; RNA-Seq   SRR1278970  Illumina HiSeq 2000   24220   22 Jul 2015
 * GSM1382950: WT **biofilm** rep1; Candida parapsilosis; RNA-Seq  SRR1278971  Illumina HiSeq 2000   23670   22 Jul 2015
 * GSM1382951: WT **biofilm** rep2; Candida parapsilosis; RNA-Seq  SRR1278972  Illumina HiSeq 2000   24617   22 Jul 2015
 * GSM1382952: WT **biofilm** rep3; Candida parapsilosis; RNA-Seq  SRR1278973  Illumina HiSeq 2000   24055   22 Jul 2015


~~~~

cd raw_data
bash sra_explorer_fastq_download.sh
bash sed "s/GSM.*Seq_//g" sra_explorer_fastq_download.sh > renamed_sra_explorer.sh # 편하게 id만 남긴 파일명 변경
bash renamed_sra_explorer.sh

ls SRR\*gz | cut -f 1 -d "\_" | sort | uniq  > sample_ids.txt

~~~~

2. 데이터 fastq 평가           FASTQC, MultiQC

fastqc_initial에 평가 html 생성. multiqc로 하나로 만듬

FastQC는 각 fastq 파일별로 리포트 생성,

~~~~

mkdir ./fastqc_initial
fastqc *fastq.gz -o ./fastqc_initial -t 2

cd ./fastqc_initial
multiqc .

~~~~


3. 데이터 fastq 제거           Trimmomatic 

Trimmomatic은 4 가지 파일을 생성:
 * Forward-paired reads
 * Forward-unpaired reads
 * Reverse-paired reads
 * Reverse-unpaired reads.

~~~~

cd ./trimming
./run.sh

~~~~

4. 데이터 fastq 평가           FASTQC, MultiQC

~~~~

mkdir fastqc_trimming
fastqc \*P.fastq.gz -o fastqc_trimming/ -t 2
multiqc .

~~~~

5. 참조 지놈 reference the genome

* "고전적인" 접근 방식: 참조 게놈에 대한 읽기 매핑과 각 유전자에서 매핑된 읽기의 후속 계산 - Star
* 의사 매핑 접근 방식: fastq 파일과 참조 전사본에서 발현 수준을 직접 계산 - Salmon

~~~~

cd ./ref_gen/
wget http://www.candidagenome.org/download/sequence/C_parapsilosis_CDC317/current/C_parapsilosis_CDC317_current_chromosomes.fasta.gz
wget http://www.candidagenome.org/download/gff/C_parapsilosis_CDC317/C_parapsilosis_CDC317_current_features.gff
gunzip C_parapsilosis_CDC317_current_chromosomes.fasta.gz

~~~~

6. 정렬, Alignement           Star

"고전적인" 접근 방식: 참조 게놈에 대한 읽기 매핑과 각 유전자에서 매핑된 읽기의 후속 계산

~~~~

cd ./mapping
./run.sh

multiqc . #

~~~~

 * Aligned.sortedByCoord.out.bam - bam 형식의 정렬된 읽기 정렬 파일
 * Log.final.out - STAR 매핑의 최종 간결한 로그 파일
 * ReadsPerGene.out.tab - 차등 유전자 발현 분석에 사용될 읽기 횟수 데이터가 포함된 파일입니다.

SRR1278968_ReadsPerGene.out.tab는 4개의 Column이 있음

 * Column 1 – gene name
 * Column 2 – counts for unstranded RNA-seq
 * Column 3 – counts for forward-stranded data
 * Column 4 – counts for the reverse-stranded data



6.1. 시각화                    igv

~~~~

samtools index SRR1278973_Aligned.sortedByCoord.out.bam
igv

~~~~


7. Pseudo/quasi-mapping       Kallisto, Salmon, Alevin, RapMap

의사 매핑 접근 방식: fastq 파일과 참조 전사본에서 발현 수준을 직접 계산 - Salmon

Pseudo/quasi-mapping은 RNA-seq에서 read들을 정렬하지 않고, transcriptome에 기반한 빠르고 효율적인 발현량 추정 방식

 * index 만들기
 *

~~~~

cd ./ref_gen
gffread -w transcripts.fasta -W -F -g C_parapsilosis_CDC317_current_chromosomes.fasta C_parapsilosis_CDC317_current_features.gff
#  transcripts.fasta, C_parapsilosis_CDC317_current_chromosomes.fasta.fai 생성함

# cd top dir
mkdir -p salmon/ref_gen salmon/mapping
cd salmon/ref_gen/


salmon index -t ../../ref_gen/transcripts.fasta -i cpar_index
# created an index of the transcripts.fasta file called cpar_index

multiqc .

~~~~


8. RNA-Seq Differential Gene Expression Analysis

정규화는 RNA-Seq 데이터 분석에서 중요






