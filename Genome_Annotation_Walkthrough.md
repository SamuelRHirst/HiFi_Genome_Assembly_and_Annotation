# 🧬 Genome Annotation Walkthrough

## 🔁 Repeat Modeling & Masking

Before predicting genes, it's critical to **identify and mask repetitive elements** in your genome. Repeats — like transposable elements, simple repeats, and satellite sequences — can **confuse gene predictors** and lead to incorrect annotations.

To handle this, we’ll use:

- **RepeatModeler**: builds a *de novo* repeat library from your genome
- **RepeatMasker**: uses that library to identify and mask repeats in the genome sequence

---

### 📥 Input

Use your **cleaned, scaffolded genome FASTA** as input — typically the file you generated from RagTag:

```bash
INPUT=your_genome.fasta
DB=your_genome_name
```

🧬 Run RepeatModeler & RepeatMasker

```bash
# Step 1: Build repeat database
BuildDatabase -name $DB $INPUT

# Step 2: Run RepeatModeler to identify repeat families
RepeatModeler -pa 50 -database $DB > repeatmodeler.log

# Step 3: Link repeat library
ln -s RM_*/consensi.fa.classified ./

# Step 4: Mask the genome using the custom library
RepeatMasker -pa 50 -xsmall -gff -lib consensi.fa.classified -dir MaskerOutput $INPUT
```

📄 Output Files
consensi.fa.classified: Your custom repeat library (used by RepeatMasker)

ragtag.scaffold.fasta.masked: The genome with repeats masked in lowercase (-xsmall)

*.gff: Repeat annotations in GFF format

✅ You’ll use the masked genome (*.masked) for all downstream annotation steps (e.g., GeMoMa, BRAKER, etc.).

💡 Masking prevents gene prediction tools from falsely identifying genes in repetitive regions and helps reduce the number of false positives.

🎯 Now your genome is ready for accurate structural annotation


Now that you’ve assembled and refined your genome, the next step is to identify where the **genes** are and what they **do** — a process known as **annotation**.

---

## 🧠 What is Genome Annotation?

Annotation involves two main components:
- **Structural annotation**: Identifying gene models — i.e., where genes are located in the genome and their exon-intron structure.
- **Functional annotation**: Predicting what those genes do based on homology, protein domains, or ontology terms.

---

## 🛠️ Available Annotation Tools

There are several tools and pipelines commonly used for genome annotation, each suited to different data types and research goals. There are others, but these are some commonly used ones:

| Tool        | Approach                        | Good For                                          | Link |
|-------------|----------------------------------|---------------------------------------------------|------|
| **MAKER**   | Evidence + ab initio hybrid      | Combining protein, transcript, and ab initio data | [MAKER](http://www.yandell-lab.org/software/maker.html) |
| **BRAKER**  | ab initio + RNA-seq              | When you have RNA-seq but no high-quality reference | [BRAKER](https://github.com/Gaius-Augustus/BRAKER) |
| **Funannotate** | Functional + structural       | Annotating eukaryotic genomes (originally for fungi but works for any eukaryotes| [Funannotate](https://funannotate.readthedocs.io/en/latest/) |
| **GeMoMa**  | Homology-based                   | When you have a **closely related, well-annotated reference genome** | [GeMoMa](https://github.com/JProf-GE/GeMoMa) |

---

## ✅ Why I Use GeMoMa

I primarily work with species that have **closely related organisms with chromosome-level assemblies and well-curated gene annotations**.

Because of that, **GeMoMa** works well for my research — it's a **reference-based gene prediction tool** that uses homology and optionally RNA-seq data to accurately project annotations from one species to another.

> ⚠️ If you **do not have a closely related reference**, you should consider using a hybrid pipeline like **MAKER** or **BRAKER**, which incorporate ab initio gene prediction and RNA-seq evidence.

---

## 📋 What You Need to Run GeMoMa

- A **reference genome** (FASTA) and its **annotation file** (GFF3) from a closely related species
- Your own **genome assembly** (FASTA)
- **Mapped transcriptomes** (BAM files) from your species — ideally from multiple tissues

> 🧠 High-quality transcriptome data improves GeMoMa's ability to detect UTRs, splice variants, and refine gene models.

---

## ▶️ Example GeMoMa Command

Below is an example of a general GeMoMa command, using multiple transcriptome BAMs and a reference genome:

```bash
GeMoMa GeMoMaPipeline \
  threads=20 \
  -Xmx64G \
  outdir=GeMoMa_out \
  GeMoMa.Score=ReAlign \
  AnnotationFinalizer.r=NO \
  o=false \
  t=your_genome.fasta \
  i=ReferenceSpeciesID \
  a=reference_annotations.gff3 \
  g=reference_genome.fasta \
  r=MAPPED \
  ERE.m=blood.bam \
  ERE.m=gonad.bam \
  ERE.m=heart.bam \
  ERE.m=kidney.bam \
  ERE.m=liver.bam \
  ERE.m=other_tissue.bam
```
Explanation:

t= → Target genome (your assembly)

i= → Short name/ID for reference species

a= → GFF3 annotation of the reference genome

g= → FASTA of the reference genome

ERE.m= → One or more mapped transcriptome BAM files

📦 Transcriptome Mapping Note:

You’ll need mapped transcriptomes (RNA-seq aligned to your genome) before running GeMoMa. Ideally, you should include multiple tissues to improve coverage of tissue-specific isoforms and UTR regions.

A separate repository will contain my basic transcriptome processing pipeline, from raw reads to aligned BAMs.

## 📤 GeMoMa Output

After GeMoMa finishes, the main output will be located in your `GeMoMa_out/` directory (or whatever `outdir=` you specified).

The most important file is:

final_annotation.gff


This file contains all predicted gene models based on homology and transcript evidence. However...

> ⚠️ **Heads up:** The raw `final_annotation.gff` file from GeMoMa can be somewhat **messy**, with:
> - Inconsistent ID formatting
> - Redundant or nested features
> - Irregular gene and transcript naming

---

## 🧼 Clean It Up: Use My Custom GFF Fixer Script

To produce a cleaner, more standardized GFF file suitable for downstream analyses or submission, I created a Python script that processes and refines GeMoMa’s output:

📁 [`Fix_GeMoMa_gff.py`](https://github.com/SamuelRHirst/Custom_Scripts/blob/main/Fix_GeMoMa_gff.py)

### ✅ Usage:

```bash
python Fix_GeMoMa_gff.py final_annotation.gff cleaned_annotation.gff
```

## 🧠 Functional Annotation with InterProScan

After predicting genes with GeMoMa, we can now ask: **what do these genes do?**  
To assign function to your predicted proteins, one of the most widely used tools is **[InterProScan](https://www.ebi.ac.uk/interpro/interproscan.html)**.

InterProScan compares your proteins to a wide range of databases (e.g., Pfam, SMART, CDD) and returns:
- Protein domains
- Gene Ontology (GO) terms
- Pathways (e.g., KEGG, MetaCyc)

---

### ✅ How to Run InterProScan

Use the predicted **protein FASTA** file from GeMoMa as input, and ask InterProScan to produce both **TSV** and **GFF3** output:

```bash
interproscan.sh \
  -i final_annotation_proteins.fasta \
  -f tsv,gff3 \
  -o interpro_results \
  -dp \
  -goterms \
  -pa \
  -cpu 16
```
💡 -f tsv,gff3 ensures you get both summary and functional GFF output.

📄 Outputs
interpro_results.tsv: Easy-to-parse summary table

interpro_results.gff3: Functional annotations in GFF3 format, linked to your protein IDs

You can now load this GFF3 file alongside your structural annotation in a genome browser or use it to enrich downstream gene tables.
