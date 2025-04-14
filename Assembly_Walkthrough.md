# 🚀 Assembly Walkthrough (HiFi Genomics)

So you sent off your tissue or library to be sequenced and finally (hopefully not after too much waiting) have your PacBio HiFi files ready to go. Congrats! 🎉  
Now let’s turn those beautiful long reads into a genome.

---

## 🛠️ Environment Setup: Miniconda is Your Best Friend

Most of this pipeline is run on a supercomputer (HPC), and we highly recommend using **[Miniconda3](https://www.anaconda.com/docs/getting-started/miniconda/install)** to manage environments and tools.

Why Miniconda?

> Because juggling bioinformatics software is like herding cats — and conda is the cat whisperer. 🐱  
> It handles versions, dependencies, and environments so you don’t scream into the void every time something breaks.

To install Miniconda3, follow the [official instructions here](https://www.anaconda.com/docs/getting-started/miniconda/install), or ask your HPC support if it's already installed. Most clusters have it available as a module or pre-installed in user space.

---

## 📦 Assumptions

I’ll assume you already have access to an environment where conda works and that you can install tools via:

```bash
conda create -n hifi_env hifiasm samtools fastqc seqkit -c bioconda -c conda-forge
conda activate hifi_env

```
For example

## 📂 Step 1: Check Your File Format
Your PacBio HiFi reads typically arrive in one of two formats:

.fastq.gz → You're ready to go! 🎯

.bam → You'll need to convert to .fastq.gz first.

```bash
bam2fastq -o sample_reads sample.bam
gzip sample_reads.subreads.fastq
```
This will produce:

sample_reads.subreads.fastq

And, after compression: sample_reads.subreads.fastq.gz

✅ Make sure you're working with HiFi (CCS) reads and not continuous long reads (CLR). If in doubt, ask your sequencing center or check the read lengths.

## 🔍 Step 2: Raw Read Quality Checks (Optional)
While PacBio HiFi reads are typically high-quality, it's still smart to run a few quality checks.

### 🧪 2.1 Run FastQC
```bash
fastqc sample_reads.subreads.fastq.gz
```
This will generate an .html report that you can open in your browser.

⚠️ Some warnings (e.g., about sequence length) are expected and can be ignored — FastQC is designed for Illumina reads.

### 📊 2.2 View Stats with SeqKit
```bash
seqkit stats sample_reads.subreads.fastq.gz
```
Example output:
```bash
file                          format  type  num_seqs  sum_len   min_len  avg_len  max_len  gc%
sample_reads.subreads.fastq.gz  FASTQ   DNA   150342    224506273  10500    14937    15624    42.8
```
### 📈 2.3 Visualize with NanoPlot (Optional)
```bash
NanoPlot --fastq sample_reads.subreads.fastq.gz -o nanoplot_output/
```
NanoPlot generates high-quality plots for:
Read length distribution
Quality scores
Total base yield

✅ Once your input data is confirmed and ready... it’s time to assemble.

➡️ Go to Step 3 – Assembly with Hifiasm

## 🧱 Step 3: Assemble with Hifiasm

Now that your HiFi reads are ready to go, it's time to assemble the genome using [`hifiasm`](https://github.com/chhylp123/hifiasm), a fast and accurate assembler designed for PacBio HiFi data.

---

### 📥 Input

Your input should be one or more `.fastq.gz` files containing HiFi reads.

You can provide multiple read files separated by spaces:

```bash
hifiasm -o sample_output -t 32 reads1.fastq.gz reads2.fastq.gz
```

🧬 Example Assembly Command
```bash
# Define your read file
read1=/path/to/your/reads/sample1.hifi_reads.fastq.gz

# Run hifiasm
hifiasm -o example_output.asm -t 50 $read1
```
Explanation:

-o example_output.asm: Sets the output prefix for all files.

-t 50: Allocates 50 threads (adjust to your HPC specs).

$read1: Input FASTQ file; you can also list multiple files directly.

📄 What Does Hifiasm Output?
After successful assembly, hifiasm will output several files:
```bash
example_output.asm.bp.p_ctg.gfa        # Primary contig graph file
example_output.asm.bp.hap1.p_ctg.gfa   # Hap1 assembly (if phased)
example_output.asm.bp.hap2.p_ctg.gfa   # Hap2 assembly (if phased)
```
example_output.asm.bp.p_ctg.gfa        # Primary contig graph file
example_output.asm.bp.hap1.p_ctg.gfa   # Hap1 assembly (if phased)
example_output.asm.bp.hap2.p_ctg.gfa   # Hap2 assembly (if phased)

🔄 Convert GFA to FASTA with awk
To use your assembly downstream, you’ll want to convert the .gfa file(s) to .fasta. Here's how:
📌 Primary Assembly to FASTA
```bash
awk '/^S/{print ">"$2"\n"$3}' example_output.asm.bp.p_ctg.gfa | fold > example_primary_assembly.fasta
```
📌 Haplotype-resolved Assemblies to FASTA (if available)
```bash
awk '/^S/{print ">"$2"\n"$3}' example_output.asm.bp.hap1.p_ctg.gfa | fold > example_hap1.fasta
awk '/^S/{print ">"$2"\n"$3}' example_output.asm.bp.hap2.p_ctg.gfa | fold > example_hap2.fasta
```

## 🧹 Step 4: (Optional) Purging Duplicates

Some users opt to run `purge_dups` after assembly to remove potential haplotypic duplications or redundant contigs. However, **Hifiasm already performs its own internal purging**, so this step is **not always recommended**.

> 💡 See this [Twitter thread by Heng Li](https://x.com/lh3lh3/status/1422917891675070468) (Hifiasm developer) explaining why **post-assembly purging may be unnecessary** and potentially harmful if applied blindly.

That said, if your **BUSCO duplication rate** is high (e.g., >2–5%) or you suspect unresolved haplotypic contigs, `purge_dups` can still be useful. Just proceed cautiously and back up your original assembly.

---

### 📊 When Should You Use `purge_dups`?

- If your genome shows **>2–5% duplicated BUSCOs**
- If your genome is known to be **highly heterozygous**
- If you're working with **CLR data** or assemblies from tools that don't already purge

---

📌 **Note:** If you do use `purge_dups`, the typical workflow involves:

1. Generating a read depth histogram
2. Identifying cutoff thresholds
3. Running the `purge_dups` commands


## 🧼 Step 5: Adapter & Contamination Screening

After assembly, it's good practice to screen your genome for any leftover adapters or foreign contamination (e.g., vectors, microbial sequences, etc.). These can sneak into your data during library prep, sequencing, or assembly.

There are several tools available for this, but one of the best is **[NCBI FCS](https://github.com/ncbi/fcs)** (Foreign Contamination Screen), which is used by NCBI to evaluate submissions to GenBank.

---

### ⚠️ The Catch

Setting up FCS on a cluster — downloading and configuring all the necessary **databases and dependencies** — can be a bit of a nightmare. It often requires a lot of time, system resources, and patience.

---

### 🌐 A Better Way: Use Galaxy!

[Galaxy](https://usegalaxy.org) is a free, web-based platform for accessible, reproducible bioinformatics. It hosts hundreds of tools and workflows and **comes pre-configured with all necessary databases** — including those used by FCS.

You don’t have to install anything. Just upload your files and run the workflow in your browser. ✨

---

### 🚀 Tutorial: Screen for Contamination with Galaxy + FCS

Follow this excellent step-by-step tutorial from the Galaxy team:

📚 [Galaxy Training: NCBI Foreign Contamination Screen](https://training.galaxyproject.org/training-material/topics/sequence-analysis/tutorials/ncbi-fcs/tutorial.html)

#### ✅ What You'll Do:

1. **Create a free account** at [usegalaxy.org](https://usegalaxy.org)
2. Open the tutorial and click to **import the workflow**
3. **Upload your assembly FASTA** file
4. **Run the workflow** on your assembly
5. Set the **contamination database** to match your organism group (e.g., vertebrates, prokaryotes, etc.)

---

### 📄 Output

The workflow will produce:
- A contamination report
- A cleaned version of your FASTA file (if contamination is detected)

You can then download the cleaned FASTA and move forward with a tidy genome. 🧼

## 📊 Step 6: Assembly Statistics

Before moving forward with annotation or scaffolding, it's important to evaluate the quality of your genome assembly. This includes:

- Total size
- Number of contigs
- N50/L50
- GC content
- Genome completeness

---
### 🐍 6.1 Use a Custom Python Script

You can use this Python script to generate quick summary stats (assembly size, N50, L50, etc.).

📁 Available here:  
👉 [`assembly_stats.py`](https://github.com/samhirst/custom-scripts/blob/main/assembly_stats.py)

#### ✅ Usage:

```bash
python assembly_stats.py cleaned_assembly.fasta
```
### 🧰 6.2 Other Tools You Can Use
If you'd prefer established third-party tools, here are a few good options:

🔹 seqkit (lightweight, fast)
```bash
seqkit stats cleaned_assembly.fasta
```
🔹 QUAST (more detailed)
```bash
quast cleaned_assembly.fasta -o quast_output/ -t 16
```

QUAST reports:

N50, L50, NG50

Total length

GC%

Misassemblies (if a reference is provided)

### 🧬 6.3 Genome Completeness with BUSCO
BUSCO (Benchmarking Universal Single-Copy Orthologs) estimates how complete your genome is by searching for conserved orthologous genes.

✅ Run BUSCO
```bash
busco -i cleaned_assembly.fasta -l <lineage_dataset> -m genome -o busco_output -c 1
```
Where:

-i: Your input FASTA

-l: Lineage dataset appropriate for your organism
(e.g., vertebrata_odb10, arthropoda_odb10, fungi_odb10)

-m genome: Run mode for genome assemblies

-o: Output folder

-c: Number of threads

✅ Be sure to choose the correct lineage dataset.
For vertebrates, use: -l vertebrata_odb10

📄 Example BUSCO Output
You'll find a file like:

```bash
busco_output/short_summary.specific.<lineage>.cleaned_assembly.txt
```
Example summary:

```
C:97.0%[S:96.0%,D:1.0%],F:0.7%,M:2.3%,n:3354
```
Where:

C = Complete (S + D)

S = Single-copy

D = Duplicated

F = Fragmented

M = Missing

n = Number of BUSCO groups searched

🟢 A good assembly typically has ≥95% completeness and low duplication (< 2-5%)

