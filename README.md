SMAGEXP (Statistical Meta Analysis for Gene EXPression) for Galaxy
========

SMAGEXP (Statistical Meta-Analysis for Gene EXPression) for Galaxy is a Galaxy tool suite providing a unified way to carry out meta-analysis of gene expression data, while taking care of their specificities. It handles microarray data from Gene Expression Omnibus (GEO) database or custom data from affymetrix microarrays. These data are then combined to carry out meta-analysis using metaMA package. SMAGEXP also offers to combine Next Generation Sequencing (NGS) RNA-seq analysis from Deseq2 results thanks to metaRNASeq package. In both cases, key values, independent from the technology type, are reported to judge the quality of the meta-analysis. 

Table of Contents <a name="toc" />
------------------------
- [How to install SMAGEXP](#how-to-install-smagexp)
	- [From the galaxy toolshed](#from-the-galaxy-toolshed)
	- [Using docker](#using-docker)
- [How to analyse data with SMAGEXP](#how-to-analyse-data-with-smagexp)
	- [Micro-array meta-analysis](#micro-array-meta-analysis)
		- [Data from GEO database](#data-from-geo-database)
		- [Data from affymetrix .CEL files](#Data-from-affymetrix-.CEL-files) 
		- [Custom matrix data](#Custom-matrix-data)
		- [Limma Analysis](#Limma-Analysis)
		- [Running a meta analysis](#Running-a-meta-analysis)
	- [Rna-seq meta analysis](#Rna-seq-meta-analysis) 
- [Step by step example of a micro-array meta-analysis](#Step-by-step-example-of-a-micro-array-meta-analysis)
-  [Step by step example of a RNA-seq meta-analysis](#Step-by-step-example-of-a-RNA-seq-meta-analysis)




How to install SMAGEXP  <a name="how-to-install-smagexp" /> [[toc]](#toc)
------------------------

### From the galaxy toolshed <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)

[SMAGEXP is available on the galaxy main toolshed ](https://toolshed.g2.bx.psu.edu/view/sblanck/smagexp/58052f8bc987)

SMAGEXP dependencies are available through conda either on bioconda or r conda channels.

If you want to manually install the SMAGEXP dependencies, without conda, these are the required R packages.

* From bioconductor : 
	* GEOquery 
	* limma
	* affy
	* annaffy
	* HTSFilter
	* GEOmetadb
	* affyPLM
	* recount

* From CRAN :  
	* metaMA
	* metaRNASeq
	* jsonlite
	* VennDiagram
	* dplyr
	* optparse
	* UpSetR

### Using Docker  <a name="using-docker" /> [[toc]](#toc)

A dockerized version of Galaxy containing SMAGEXP, based on [bgruening galaxy-stable](https://github.com/bgruening/docker-galaxy-stable) is also available.

At first you need to install docker. Please follow the [very good instructions](https://docs.docker.com/installation/) from the Docker project.

After the successful installation, all you need to do is:

```
docker run -d -p 8080:80 -p 8021:21 -p 8022:22 sblanck/galaxy-smagexp
```
Then, you just need to open a web browser (chrome or firefox are recommanded) and type 
```
localhost:8080
```
into the adress bar to access Galaxy running smagexp

The Galaxy Admin User has the username `admin@galaxy.org` and the password `admin`. In order to use some features of Galaxy, like import history, one has to be logged in.

Docker images are "read-only", all your changes inside one session will be lost after restart. This mode is useful to present Galaxy to your colleagues or to run workshops with it. To install Tool Shed repositories or to save your data you need to export the calculated data to the host computer.

Fortunately, this is as easy as:
```
docker run -d -p 8080:80 \
    -v /home/user/galaxy_storage/:/export/ \
    sblanck/galaxy-smagexp
```
For advanced users who wants to explore Rdata objects generated by SMAGEXP thanks to Galaxy's Rstudio Interactive Environment, we need to be able to launch Docker containers inside our Galaxy Docker container. At least docker 1.3 is needed on the host system.
```
docker run -d -p 8080:80 -p 8021:21 -p 8800:8800 \
    --privileged=true \
    -v /home/user/galaxy_storage/:/export/ \
    bgruening/galaxy-stable
```

For more information about the parameters and docker usage, please refer to https://github.com/bgruening/docker-galaxy-stable/blob/master/README.md#Usage


How to analyse data with SMAGEXP  <a name="how-to-analyse-data-with-smagexp" /> [[toc]](#toc)
------------------------

###  Micro-array meta-analysis  <a name="micro-array-meta-analysis" /> [[toc]](#toc)

SMAGEXP is able to perform analysis from 3 different data source :
* From GEO database
* From .CEL files
* From custom matrix text files

#### Data from GEO database  <a name="data-from-geo-database" /> [[toc]](#toc)

SMAGEXP can fetch data directly from [GEO database](https://www.ncbi.nlm.nih.gov/geo/), thanks to the GEOQuery R package. 

The inputs are : 

* The GEO Series Accession ID of the microarray experiment
* log2 transformation option : Limma expects data values to be in log space. If the values of the experiments are not in log space, SMAGEXP is able to check and to transform them accordingly (option auto). The user can also choose to force the transformation (option yes) or to override the auto detect feature (option no)

The outputs are :

* A tabular file containing the values of each probes (lines) for each samples (columns) of the experiment
* A .rdata file containing a bioconductor eset object. This file is required for further differential analysis.
* A tabular text file (.cond extension) summarizing the conditions of the experiment.

Exemple of a .cond file
```

GSM80460	series of 16 tumors		GSM80460 OSCE-2T SERIES OF 16 TUMORS
GSM80461	series of 16 tumors		GSM80461 OSCE-4T Series of 16 Tumors
GSM80461	series of 16 tumors		GSM80462 OSCE-6T Series of 16 Tumors
GSM80476	series of 4 normals		GSM80476 OSCE-2N Series of 4 Normals
GSM80477 	series of 4 normals		GSM80477 OSCE-9N Series of 4 Normals

```
.cond file is a text file, containing 3 columns, separate by tabs, summarizing the conditions of the experiment. 
* 1st column is the ID of the sample
* 2nd column is the condition of the sample
* 3rd column is a description of the sample

When extracting data from GEO database, SMAGEXP automatically generates a .cond files based on the metadata of the experiment. 

#### Data from affymetrix .CEL files  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
SMAGEXP handles affymetrix .CEL files. .CEL files have to be normalized with QCnormalization tool. This tool normalizes data and allows the user to check quality.

The inputs are
* A list of .CEL files
* The normalization methods
	- rma normalization
	- quantile normalization + log2
	- background correction + log2
	- log2 only

The outputs are 
- Several quality figures : microarray images, boxplots and MA plots
- Rdata object containing the normalized data for further analysis
- Text file containing normalized data

#### Custom matrix data  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
Import custom data tool imports data stored in a tabular text file. A few  normalization methods are proposed, but it is possible to skip the normalization step, by choosing "none" in the normalization methods options. Therefore this tool is of special interest when the input dataset has been previously normalized.

The inputs are :
* A text file, with probes in row and samples in columns. Column titles (chip IDs) must match the IDs of the .cond file.
* a .cond file, previously manually generated by the user.
* Normalization method (choose none if the data is already normalized)
* A GPL annotation code to fetch annotations from GEO.

Example of a header of input tabular text file
```
""			"GSM80460"			"GSM80461"			"GSM80462"			"GSM80463"			"GSM80464"
"1007_s_at"	-0.0513991525066443	0.306845500314283	0.0854246562526777	-0.142417044615852	0.0854246562526777
"1053_at"	-0.187707155126729	-0.488026018218199	-0.282789700980404	0.160920188181103	0.989865622866287
"117_at"	0.814755482809874	-2.15842936260448	-0.125006361067033	-0.256700472111743	0.0114956388378294
"121_at"	-0.0558912008920451	-0.0649174766813385	0.49467161164755	-0.0892673380970874	0.113700499164728
"1294_at"	0.961993677420255	-0.0320936297098533	-0.169744675832317	-0.0969617298870879	-0.181149439104566
"1316_at"	0.0454429707611671	0.43616183931445	-0.766111939825723	-0.182786075741673	0.599317793698226
"1405_i_at"	2.23450132056221	0.369606070031838	-1.06190243892591	-0.190997225060914	0.595503660502742
```

The according .cond file should look like this 

```

GSM80460	series of 16 tumors		GSM80460 OSCE-2T SERIES OF 16 TUMORS
GSM80461	series of 16 tumors		GSM80461 OSCE-4T Series of 16 Tumors
GSM80461	series of 16 tumors		GSM80462 OSCE-6T Series of 16 Tumors
GSM80476	series of 4 normals		GSM80476 OSCE-2N Series of 4 Normals
GSM80477 	series of 4 normals		GSM80477 OSCE-9N Series of 4 Normals

```
.cond file is a text file, containing 3 columns, separate by tabs, summarizing the conditions of the experiment. 
* 1st column is the ID of the sample
* 2nd column is the condition of the sample
* 3rd column is a description of the sample
				
The outputs are
 - Boxplots and MA plots 
 - Rdata object containing the data for further analysis.

#### Limma Analysis  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
The Limma analysis tool performs single analysis either of data previously retrieved from GEO database or normalized affymetrix .CEL files data. 
Given a .cond file, it runs a standard limma differential expression analysis. 

The inputs are 
* A Rdata object from GEOQuery tool, QCnormalization tool or Import custom data tool
* A .cond file,
* 2 conditions to compare
	
The outpouts are :
		
- Boxplots, p-value histograms and a volcano plot 
- Table summarizing the differentially expressed genes and their annotations. This table is sortable and requestable and annotated. When a line is expended, link to NCBI gene annotations and Gene Ontology functions are available
- .rdata file to perform further meta-analysis. 
- Text file containing the annotated results of the differential analysis

![Plots generated by limma analysis tool](https://raw.githubusercontent.com/sblanck/smagexp/master/images/fig5.png)
![Table generated by limma analysis tool](https://raw.githubusercontent.com/sblanck/smagexp/master/images/fig6.png)

#### Running a meta analysis  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
Given several Rdata object from the limma analysis tool the microarray meta-analysis tool run a meta-analysis using the metaMA R package.
		
The Inputs are :
* .rdata files from limma analysis tool.
		
The outputs are  :		
- Venn Diagram or upsetR diagram (when the number of studies is greater than 3) summarizing the results of the meta-analysis
- A list of indicators to evaluate the quality of the performance of the meta-analysis
		
	- DE : Number of differentially expressed genes 
	- IDD (Integration Driven discoveries) : number of genes that are declared differentially expressed in the meta-analysis that were not identified in any of the single studies alone
	- Loss : Number of genes that are identified differentially expressed in single studies but not in meta-analysis 
	- DR (Integration-driven Discovery Rate) : corresponding proportion of IDD
	- IRR (Integration-driven Revision) : corresponding proportion of Loss
		
- Fully sortable and requestable table, with gene annotations and hypertext links to NCBI gene database.

![Plots and results generated by the microarray meta-analysis tool ](https://raw.githubusercontent.com/sblanck/smagexp/master/images/fig7.png)

*Plots and results generated by the microarray meta-analysis tool*

### Rna-seq meta analysis  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
The RNA-seq data meta-analysis tool relies on DESeq2 results.

It outputs a Venn diagram or an upsetR diagram (when the number of studies is greater than 2) and the same indicators as in the microarray meta-analysis tool for both Fisher and inverse normal p-values combinations.

The inputs are :
- At least 2 studies, and for each study
	- Results of DESeq2 study
	- Number of replicates of the study	
- A FDR Threshold	

The outputs are  :		
- Venn Diagram or upsetR diagram (when the number of studies is greater than 3) summarizing the results of the meta-analysis
- A list of indicators to evaluate the quality of the performance of the meta-analysis
		
	- DE : Number of differentially expressed genes 
	- IDD (Integration Driven discoveries) : number of genes that are declared differentially expressed in the meta-analysis that were not identified in any of the single studies alone
	- Loss : Number of genes that are identified differentially expressed in single studies but not in meta-analysis 
	- DR (Integration-driven Discovery Rate) : corresponding proportion of IDD
	- IRR (Integration-driven Revision) : corresponding proportion of Loss

It also generates a text file containing summarization the results of each single analysis and meta-analysis. Potential conflicts between single analyses are indicated by zero values in the "signFC" column. 

![Example of RNA-seq data meta-analysis plots](https://raw.githubusercontent.com/sblanck/smagexp/master/images/metaRNAseq_results.png)

![Header of RNA-seq data meta-analysis text results](https://raw.githubusercontent.com/sblanck/smagexp/master/images/fig9.png)

*Header of RNA-seq data meta-analysis text results*

Step by step example of a micro-array meta-analysis  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
------------------------
### Data used in this example  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)

The full history of this example is available at  : 
```
https://github.com/sblanck/smagexp/blob/master/examples/Galaxy-History-Micro-array-meta-analysis-history.tar.gz
```
.CEL files used in this examples are extracted from the GEO dataset GSE13601. We picked up 6 .CEL files (to simplify the example) which can be found here :
```
https://github.com/sblanck/smagexp/raw/master/examples/GSM342582.CEL
https://github.com/sblanck/smagexp/raw/master/examples/GSM342583.CEL
https://github.com/sblanck/smagexp/raw/master/examples/GSM342584.CEL
https://github.com/sblanck/smagexp/raw/master/examples/GSM342585.CEL
https://github.com/sblanck/smagexp/raw/master/examples/GSM342586.CEL
https://github.com/sblanck/smagexp/raw/master/examples/GSM342587.CEL
```
We also manually generated a .cond file according to these 6 .cel files.
```
https://raw.githubusercontent.com/sblanck/smagexp/master/examples/Celfiles.cond
```
To easily upload these data on galaxy, it is possible to load an existing history containing all these data : 
```
https://github.com/sblanck/smagexp/raw/master/examples/Galaxy-History-Example-Data.tar.gz 
```
Download this history on your computer and import it in galaxy. If you choose to manually upload these data on Galaxy don't forget to specify the type of each file (.CEL or .cond) as Galaxy won't auto-detect them.

### First analysis: from GEO database  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)

#### Run The GEOQuery Tool  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
The GSE accession ID is needed (i.e GSE3524). The log2 transformation is set to auto in this example.
![GEOQuery tool form](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_geoqueryID_form.png)

*GEOQuery tool form*

The tool produce 
* A tabular text files containing normalized expression value for each probe (in row) and each sample (in column).
![Header of the tabular text file generated by GEOquery tool](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_geogeuryID_text_results.png)

*Header of the tabular text file generated by GEOquery tool*

* A .cond file summarizing the conditions of the experiment.
![.cond file generated by the](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_geoqueryID_cond.png)

*Condition file generated by GEOquery tool*

* A .rdata for further analysis

#### Run limma analysis  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)

The limma analysis tool takes an rdata and a .cond file as inputs.
![Limma analysis tool form](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_limma_form.png)

*Limma analysis tool form*

It generates a html report with boxplots, p-value histogram a volcano plot and a table listing the differentially expressed genes. 
![Limma analysis tool graphic outputs](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_limma_result_graph.png)

*Limma analysis tool graphic outputs*

![Limma analysis tool table output](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_limma_result_table.png)

*Limma analysis tool table output*

This table gives access to gene annotation on ncbi and gene ontology website.
![ncbi gene annotations](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_limma_result_ncbi.png)

*NCBI gene annotations*

### 2nd analysis : from raw .CEL files  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)

#### Run QC normalisation tool  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
The QC normalisation tool only needs a list of .CEL files and a normalization method. 

![QCnormalization tool form](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_QC_form.png)

*QCnormalization tool form*

It will generate an html report showing chip pseudo images, boxplots and MA plot for raw and normalized data. It also generates a .rdata file containing normalized data in a eset object for further analysis with limma.

![QCnormalization tool (partial) results](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_QC_report.png)

*QCnormalization tool (partial) results with chip pseudo-images, boxplots and MA-plots for raw data*

#### Run limma analysis  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
The limma analysis tool takes an rdata and a .cond file as inputs.
![Limma analysis tool form](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_limma_form2.png)

*Limma analysis tool form*

It generates a html report with boxplots, p-value histogram a volcano plot and a table listing the differentially expressed genes. 
![Limma analysis tool graphic outputs](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_limma_graph2.png)

*Limma analysis tool graphic outputs*

![Limma analysis tool table output](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_limma_table2.png)

*Limma analysis tool table output*


### Meta-analysis  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
The meta analysis tool only needs the rdata files produced by the limma tool. 
![MetaMA tool form](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_metaMA_form.png)

*MetaMA tool form*

The outputs are  :		
- Venn Diagram summarizing the results of the meta-analysis
- A list of indicators to evaluate the quality of the performance of the meta-analysis
		
	- DE : Number of differentially expressed genes 
	- IDD (Integration Driven discoveries) : number of genes that are declared differentially expressed in the meta-analysis that were not identified in any of the single studies alone
	- Loss : Number of genes that are identified differentially expressed in single studies but not in meta-analysis 
	- DR (Integration-driven Discovery Rate) : corresponding proportion of IDD
	- IRR (Integration-driven Revision) : corresponding proportion of Loss
		
- Fully sortable and requestable table, with gene annotations and hypertext links to NCBI gene database.

![MetaMA tools results](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_metaMA_results.png)

*MetaMA tools results*

Step by step example of a RNA-seq meta-analysis  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
------------------------

### Data used in this example  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)

The full history of this example is available at  : 
```
https://github.com/sblanck/smagexp/raw/master/examples/Galaxy-History-Example-of-RNA-seq-meta-analysis.tar.gz
```
Two dataset from the recount database are used in this example :
* [SRP032833](https://trace.ncbi.nlm.nih.gov/Traces/sra/?study=SRP032833)
* [SRP028180](https://trace.ncbi.nlm.nih.gov/Traces/sra/?study=SRP028180)

### First Analysis  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)

#### Recount  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
The first step is to fetch raw count data from Recount. The recount wrap the recount bioconductor package. It only needs the accession ID of the experiment. 
![Recount tool form](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_recount_form.png)

*Recount tool form*

The recount tool generate one count file per sample of the experiment, in order to be analysed with DESeq2.

![Example of header of a count file generated by recount tool](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_recount_result.png)

*Example of header of a count file generated by recount tool*

In this example 17 count files are generated

#### DESeq2  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
The [ DESeq2 tool is available on the galaxy toolshed](https://toolshed.g2.bx.psu.edu/repository?repository_id=1f158f7565dc70f9&changeset_revision=9a616afdbda5). It take the count files generated by the recount tool as inputs. It also wraps others DESeq2 parameters (see DESeq2 tool help section for more information).
In this example we keep the 6 invasive lung cancer samples to compare with the 5 normal samples.

![DESeq2 form](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_deseq2_form.png) 

*DESeq2 form*

It will generate a pdf report and a tabular text results 

![DESeq2 results header](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_deseq2_results.png)

*DESeq2 results header*


### Second Analysis  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
The same kind of analysis on an the other recount dataset

#### Recount  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)
![Recount tool from](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_recount_form2.png)

*Recount tool from*

In this example 24 count files are generated
#### DESeq2  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)

In this example we keep the 10 tumor samples to compare with the 6 normal samples.

![DESeq2 tool form](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_deseq2_form2.png)

*DESeq2 tool form*


### Meta Analysis with metaRNASeq  <a name="from-the-galaxy-toolshed" /> [[toc]](#toc)

MetaRNASeq tool takes several results from DESeq2 tool and perform a meta-analysis.
It requires text results file from DESeq2 and the number of replicates of each analysis. In this example we have 11 replicates for the first analysis and 16 for the second analysis.
It also requires a FDR threshold for genes to be declared differentially expressed (default is 0.05)

![MetaRNAseq tool form](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_metarnaseq_form.png)

*MetaRNAseq tool form*

The tool outputs 2 datasets :

![Venn diagram and statistical indicators of the meta-analysis](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_metarnaseq_diagram.png)

*Venn diagram and statistical indicators of the meta-analysis*

![Header of the text file generated by the metaRNAseq tool](https://raw.githubusercontent.com/sblanck/smagexp/master/images/smagexp_metaranseq_summary.png)

*Header of the text file generated by the metaRNAseq tool*

It summarizes the results of each single analysis and meta-analysis. Potential conflicts between single analyses are indicated by zero values in the "signFC" column. 

<!--stackedit_data:
eyJoaXN0b3J5IjpbLTIwMTA1NTI3NzksNzU4MTA3MDk0LDEyNz
U5NDgzNTEsLTE0MTg1Mzg1ODAsLTE5NDE0NTc2NjIsLTIwOTk0
NDMxMDEsLTE3Nzk5NzcxMzAsLTE0OTk3ODQyNTAsLTEyMDUyNz
A0NjcsLTE2MjgzODc4NDksLTIyNjQ4MDk0Myw2NjAyMzc3MjAs
LTk1Mzg5MzgxNiwtMjE0MTg1Nzk4MCwzMDA3MjA0OTgsLTQyMD
Q1NjQ3MSwtNTc0ODEyOTksLTgzNzIwOTIzMCwtNzA1MDA1NjA0
LDE2OTcxMTAxMDNdfQ==
-->