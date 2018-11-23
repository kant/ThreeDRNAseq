3D RNA-seq pipeline
================
Wenbin Guo
21 November 2018

Introduction
------------

Instead of mouse click analysis in the <a href="xxx" target="_blank">3D RNA-seq App</a>, advanced R users also can use command line to do the same analysis. This file includes step-by-step codings, which correspond to the steps in the 3D RNA-seq App. Users can refer to the details in the manual:xxx.

To use our pipeline in your work, please cite:

<a href="http://www.plantcell.org/content/30/7/1424" target="_blank">Calixto,C.P.G., Guo,W., James,A.B., Tzioutziou,N.A., Entizne,J.C., Panter,P.E., Knight,H., Nimmo,H., Zhang,R., and Brown,J.W.S. (2018) Rapid and dynamic alternative splicing impacts the Arabidopsis cold response transcriptome. Plant Cell.</a>

Load R package
--------------

Assume all the required R packages are installed.

``` r
.libPaths('D:/R/win_library')
# library(ThreeDRNAseq)
## app.R ##
library(shiny)
library(shinydashboard)
library(rhandsontable)
library(shinyFiles)
library(tximport)
library(edgeR)
library(limma)
library(plotly)
library(RUVSeq)
library(eulerr)
library(gridExtra)
library(grid)
library(ComplexHeatmap)
library(Gmisc)
options(stringsAsFactors=F)

sourceDir <- function(path, trace = TRUE, ...) {
  for (nm in list.files(path, pattern = "[.][RrSsQq]$")) {
    #if(trace) cat(nm,":")
    source(file.path(path, nm), ...)
    #if(trace) cat("/n")
  }
}
sourceDir('D:/PhD project/R projects/test round 2018/ThreeDRNAseq/R')
```

Set pipeline parameters
-----------------------

``` r
################################################################################
### Set folder names to save results
##----->> save results to local folders
data.folder <- 'data' # for .RData format results
result.folder <- 'result' # for 3D analysis results in csv files
figure.folder <- 'figure' # for figures
report.folder <- 'report'
### Set the input data folder
##----->> input data folder of samples.csv, mapping.csv, contrast.csv, etc.
input.folder <- 'examples/Arabidopsis_cold'

################################################################################
### parameters of tximport to generate read counts and TPMs
abundance.from <- 'salmon' # abundance generator
tximport.method <- 'lengthScaledTPM' # method to generate expression in tximport

################################################################################
### parameter for low expression filters
cpm.cut <- 1
sample.n.cut <- 1

################################################################################
### data normalisation parameter
norm.method <- 'TMM' ## norm.method is one of 'TMM','RLE' and 'upperquartile'

################################################################################
### parameters for 3D analysis
p.adjust.method <- 'BH'
pval.cutoff <- 0.01
lfc.cutoff <- 1
DE.pipeline <- 'limma'

################################################################################
### parameters for clusters in heatmap
dist.method <- 'euclidean'
cluster.method <- 'ward.D'
cluster.number <- 10
```

Data generateion
----------------

Use the <a href="http://bioconductor.org/packages/release/bioc/html/tximport.html" target="_blank">tximport</a> R package (Soneson et al., 2016) to generate the transcript and gene level read counts and TPMs (transcript per million reads) from outputs of quantification tools, such as Salmon and Kallisto

``` r
################################################################################
##----->> Sample information, e.g. conditions, bio-reps, seq-reps, abundance paths, etc.
samples <- read.csv(paste0(input.folder,'/samples.csv'))
samples$condition <- gsub(pattern = '_','.',samples$condition)
samples$sample.name <- paste0(samples$condition,'_',samples$brep,'_',samples$srep)
save(samples,file=paste0(data.folder,'/samples.RData'))
data.files <-  as.character(samples$path) #Complete path of abundance, e.g. quant.sf files of salmon.

##----->> Transcript-gene mapping
mapping <- read.csv(paste0(input.folder,'/mapping.csv'))
rownames(mapping) <- mapping$TXNAME
save(mapping,file=paste0(data.folder,'/mapping.RData'))

################################################################################
##----->> Generate gene expression
##
txi_genes <- tximport(data.files,dropInfReps = T,
                      type = abundance.from, tx2gene = mapping,
                      countsFromAbundance = tximport.method)

## give colunames to the datasets
colnames(txi_genes$counts)<- samples$sample.name
colnames(txi_genes$abundance)<-samples$sample.name
colnames(txi_genes$length)<-samples$sample.name

## save the data
write.csv(txi_genes$counts,file=paste0(result.folder,'/counts_genes.csv'))
write.csv(txi_genes$abundance,file=paste0(result.folder,'/TPM_genes.csv'))
save(txi_genes,file=paste0(data.folder,'/txi_genes.RData'))

################################################################################
##----->> Generate transcripts expression
txi_trans<- tximport(data.files, type = abundance.from, tx2gene = NULL,
                     countsFromAbundance = tximport.method,
                     txOut = T,dropInfReps = T)

## give colunames to the datasets
colnames(txi_trans$counts)<- samples$sample.name
colnames(txi_trans$abundance)<-samples$sample.name
colnames(txi_trans$length)<-samples$sample.name

## save the data
write.csv(txi_trans$counts,file=paste0(result.folder,'/counts_trans.csv'))
write.csv(txi_trans$abundance,file=paste0(result.folder,'/TPM_trans.csv'))
save(txi_trans,file=paste0(data.folder,'/txi_trans.RData'))

################################################################################
genes_counts=txi_genes$counts
trans_counts=txi_trans$counts
```

Data pre-processing
-------------------

### Step 1: Merge sequencing replicates

If the RNA-seq data includes sequencing replicates (i.e. the same biological replicate/library is sequenced multiple times), the sequencing replicates are merged to increase the library depths, since they do not provided useful information for biological/technical variability.

``` r
##If no sequencing replicates, genes_counts and trans_counts remain the same by 
##runing the command
idx <- paste0(samples$condition,'_',samples$brep)
genes_counts <- sumarrays(genes_counts,group = idx)
trans_counts <- sumarrays(trans_counts,group = idx)

## update the samples
samples_new <- samples[samples$srep==samples$srep[1],]
save(samples_new,file=paste0(data.folder,'/samples_new.RData'))
```

### Step 2: Filter low expression transcripts/genes

To filter the low expressed transcripts across the samples, the expression mean-variance trend is used to determine the CPM (count per million reads) cut-off for low expression and the number of samples to cut. A gene is expressed if any of its transcript is expressed. Please see the details in the 3D RNA-seq App user manual.

``` r
################################################################################
##----->> Do the filters
target_high <- counts.filter(counts = trans_counts,
                             mapping = mapping,
                             cpm.cut = cpm.cut,
                             sample.n = sample.n.cut,
                             Log=F)
save(target_high,file=paste0(data.folder,'/target_high.RData'))

################################################################################
##----->> Mean-variance plot
## transcript level
condition <- paste0(samples_new$condition)
counts.raw = trans_counts[rowSums(trans_counts>0)>0,]
counts.filtered = trans_counts[target_high$trans_high,]
mv.trans <- check.mean.variance(counts.raw = counts.raw,
                                counts.filtered = counts.filtered,
                                condition = condition)
### make plot
fit.raw <- mv.trans$fit.raw
fit.filtered <- mv.trans$fit.filtered
mv.trans.plot <- function(){
  par(mfrow=c(1,2))
  plot.mean.variance(x = fit.raw$sx,y = fit.raw$sy,
                     l = fit.raw$l,lwd=2,fit.line.col ='gold',col='black')
  title('\n\nRaw counts (transcript level)')
  plot.mean.variance(x = fit.filtered$sx,y = fit.filtered$sy,
                     l = fit.filtered$l,lwd=2,col='black')
  title('\n\nFiltered counts (transcript level)')
  lines(fit.raw$l, col = "gold",lty=4,lwd=2)
  legend('topright',col = c('red','gold'),lty=c(1,4),lwd=3,
         legend = c('low-exp removed','low-exp kept'))
}
mv.trans.plot()
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-5-1.png" style="display: block; margin: auto;" />

``` r
### save to figure folder
png(filename = paste0(figure.folder,'/Transcript mean-variance trend.png'),
    width = 25/2.54,height = 12/2.54,units = 'in',res = 300)
mv.trans.plot()
dev.off()
```

    ## png 
    ##   2

``` r
pdf(file = paste0(figure.folder,'/Transcript mean-variance trend.pdf'),
    width = 25/2.54,height = 12/2.54)
mv.trans.plot()
dev.off()
```

    ## png 
    ##   2

``` r
################################################################################
## gene level
condition <- paste0(samples_new$condition)
counts.raw = genes_counts[rowSums(genes_counts>0)>0,]
counts.filtered = genes_counts[target_high$genes_high,]
mv.genes <- check.mean.variance(counts.raw = counts.raw,
                                counts.filtered = counts.filtered,
                                condition = condition)
### make plot
fit.raw <- mv.genes$fit.raw
fit.filtered <- mv.genes$fit.filtered
mv.genes.plot <- function(){
  par(mfrow=c(1,2))
  plot.mean.variance(x = fit.raw$sx,y = fit.raw$sy,
                     l = fit.raw$l,lwd=2,fit.line.col ='gold',col='black')
  title('\n\nRaw counts (gene level)')
  plot.mean.variance(x = fit.filtered$sx,y = fit.filtered$sy,
                     l = fit.filtered$l,lwd=2,col='black')
  title('\n\nFiltered counts (gene level)')
  lines(fit.raw$l, col = "gold",lty=4,lwd=2)
  legend('topright',col = c('red','gold'),lty=c(1,4),lwd=3,
         legend = c('low-exp removed','low-exp kept'))
}
mv.genes.plot()
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-5-2.png" style="display: block; margin: auto;" />

``` r
### save to figure folder
png(filename = paste0(figure.folder,'/Gene mean-variance trend.png'),
    width = 25/2.54,height = 12/2.54,units = 'in',res = 300)
mv.genes.plot()
dev.off()
```

    ## png 
    ##   2

``` r
pdf(file = paste0(figure.folder,'/Gene mean-variance trend.pdf'),
    width = 25/2.54,height = 12/2.54)
mv.genes.plot()
dev.off()
```

    ## png 
    ##   2

### Step 3: Principal component analysis (PCA)

The PCA plot is a rough visualization of expression variation across conditions/samples and to determine whether the data has batch effects between different biological replicates.

``` r
################################################################################
##----->> trans level
data2pca <- trans_counts[target_high$trans_high,]
dge <- DGEList(counts=data2pca) 
dge <- calcNormFactors(dge)
data2pca <- t(counts2CPM(obj = dge,Log = T))
dim1 <- 'PC1'
dim2 <- 'PC2'
ellipse.type <- 'polygon' #ellipse.type=c('none','ellipse','polygon')

##--All Bio-reps plots
rownames(data2pca) <- gsub('_','.',rownames(data2pca))
groups <- samples_new$brep ## colour on biological replicates
# groups <- samples_new$condition ## colour on condtions
g <- plot.PCA.ind(data2pca = data2pca,dim1 = dim1,dim2 = dim2,
                  groups = groups,plot.title = 'Transcript PCA: bio-reps',
                  ellipse.type = ellipse.type,
                  add.label = T,adj.label = F)

g
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-6-1.png" style="display: block; margin: auto;" />

``` r
### save to figure
png(filename = paste0(figure.folder,'/Transcript PCA Bio-reps.png'),
    width = 15/2.54,height = 13/2.54,units = 'in',res = 300)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(file = paste0(figure.folder,'/Transcript PCA Bio-reps.pdf'),
    width = 15/2.54,height = 13/2.54)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
##################################################
##--average expression plot
rownames(data2pca) <- gsub('_','.',rownames(data2pca))
groups <- samples_new$brep
data2pca.ave <- rowmean(data2pca,samples_new$condition,reorder = F)
groups <- unique(samples_new$condition)
g <- plot.PCA.ind(data2pca = data2pca.ave,dim1 = 'PC1',dim2 = 'PC2',
                  groups = groups,plot.title = 'Transcript PCA: average expression',
                  ellipse.type = 'none',add.label = T,adj.label = F)

g
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-6-2.png" style="display: block; margin: auto;" />

``` r
### save to figure
png(filename = paste0(figure.folder,'/Transcript PCA Average expression.png'),
    width = 15/2.54,height = 13/2.54,units = 'in',res = 300)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(file = paste0(figure.folder,'/Transcript PCA Average expression.pdf'),
    width = 15/2.54,height = 13/2.54)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
################################################################################
##----->> genes level
data2pca <- genes_counts[target_high$genes_high,]
dge <- DGEList(counts=data2pca) 
dge <- calcNormFactors(dge)
data2pca <- t(counts2CPM(obj = dge,Log = T))
dim1 <- 'PC1'
dim2 <- 'PC2'
ellipse.type <- 'polygon' #ellipse.type=c('none','ellipse','polygon')

##--All Bio-reps plots
rownames(data2pca) <- gsub('_','.',rownames(data2pca))
groups <- samples_new$brep ## colour on biological replicates
# groups <- samples_new$condition ## colour on condtions
g <- plot.PCA.ind(data2pca = data2pca,dim1 = dim1,dim2 = dim2,
                  groups = groups,plot.title = 'genescript PCA: bio-reps',
                  ellipse.type = ellipse.type,
                  add.label = T,adj.label = F)

g
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-6-3.png" style="display: block; margin: auto;" />

``` r
### save to figure
png(filename = paste0(figure.folder,'/Gene PCA Bio-reps.png'),
    width = 15/2.54,height = 13/2.54,units = 'in',res = 300)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(file = paste0(figure.folder,'/Gene PCA Bio-reps.pdf'),
    width = 15/2.54,height = 13/2.54)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
##################################################
##--average expression plot
rownames(data2pca) <- gsub('_','.',rownames(data2pca))
groups <- samples_new$brep
data2pca.ave <- rowmean(data2pca,samples_new$condition,reorder = F)
groups <- unique(samples_new$condition)
g <- plot.PCA.ind(data2pca = data2pca.ave,dim1 = 'PC1',dim2 = 'PC2',
                  groups = groups,plot.title = 'genescript PCA: average expression',
                  ellipse.type = 'none',add.label = T,adj.label = F)

g
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-6-4.png" style="display: block; margin: auto;" />

``` r
### save to figure
png(filename = paste0(figure.folder,'/Gene PCA Average expression.png'),
    width = 15/2.54,height = 13/2.54,units = 'in',res = 300)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(file = paste0(figure.folder,'/Gene PCA Average expression.pdf'),
    width = 15/2.54,height = 13/2.54)
print(g)
dev.off()
```

    ## png 
    ##   2

### Step 3-continued: Batch effect estimation

Only if see clear batch effects on the PCA plots, otherwise skip this step. The <a href="https://bioconductor.org/packages/release/bioc/html/RUVSeq.html" target="_blank">RUVSeq</a> (Risso et al., 2014) is used to estimate the batch effects. It will generate modified read counts, where the batch effects have been removed, for batch-free PCA plots and batch effect term which can be passed to the linear regression model for 3D analysis.

``` r
ruvseq.method <- 'RUVr' # ruvseq.method is one of 'RUVr', 'RUVs' and 'RUVg'
design <- condition2design(condition = samples_new$condition,
                           batch.effect = NULL)


################################################################################
##----->> trans level
trans_batch <- remove.batch(read.counts = trans_counts[target_high$trans_high,],
                            condition = samples_new$condition,
                            design = design,
                            contrast=NULL,
                            group = samples_new$condition,
                            method = ruvseq.method)
save(trans_batch,file=paste0(data.folder,'/trans_batch.RData')) 

################################################################################
##----->> genes level
genes_batch <- remove.batch(read.counts = genes_counts[target_high$genes_high,],
                            condition = samples_new$condition,
                            design = design,
                            contrast=NULL,
                            group = samples_new$condition,
                            method = ruvseq.method)
save(genes_batch,file=paste0(data.folder,'/genes_batch.RData')) 


################################################################################
## DO the PCA again
################################################################################

##----->> trans level
data2pca <- trans_batch$normalizedCounts[target_high$trans_high,]
dge <- DGEList(counts=data2pca) 
dge <- calcNormFactors(dge)
data2pca <- t(counts2CPM(obj = dge,Log = T))
dim1 <- 'PC1'
dim2 <- 'PC2'
ellipse.type <- 'polygon' #ellipse.type=c('none','ellipse','polygon')

##--All Bio-reps plots
rownames(data2pca) <- gsub('_','.',rownames(data2pca))
groups <- samples_new$brep ## colour on biological replicates
# groups <- samples_new$condition ## colour on condtions
g <- plot.PCA.ind(data2pca = data2pca,dim1 = dim1,dim2 = dim2,
                  groups = groups,plot.title = 'Transcript PCA: bio-reps',
                  ellipse.type = ellipse.type,
                  add.label = T,adj.label = F)

g
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-7-1.png" style="display: block; margin: auto;" />

``` r
### save to figure
png(filename = paste0(figure.folder,'/Transcript PCA batch effect removed Bio-reps.png'),
    width = 15/2.54,height = 13/2.54,units = 'in',res = 300)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(file = paste0(figure.folder,'/Transcript PCA batch effect removed Bio-reps.pdf'),
    width = 15/2.54,height = 13/2.54)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
##################################################
##--average expression plot
rownames(data2pca) <- gsub('_','.',rownames(data2pca))
groups <- samples_new$brep
data2pca.ave <- rowmean(data2pca,samples_new$condition,reorder = F)
groups <- unique(samples_new$condition)
g <- plot.PCA.ind(data2pca = data2pca.ave,dim1 = 'PC1',dim2 = 'PC2',
                  groups = groups,plot.title = 'Transcript PCA: average expression',
                  ellipse.type = 'none',add.label = T,adj.label = F)

g
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-7-2.png" style="display: block; margin: auto;" />

``` r
### save to figure
png(filename = paste0(figure.folder,'/Transcript PCA Average expression.png'),
    width = 15/2.54,height = 13/2.54,units = 'in',res = 300)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(file = paste0(figure.folder,'/Transcript PCA Average expression.pdf'),
    width = 15/2.54,height = 13/2.54)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
################################################################################
##----->> genes level
data2pca <- genes_batch$normalizedCounts[target_high$genes_high,]
dge <- DGEList(counts=data2pca) 
dge <- calcNormFactors(dge)
data2pca <- t(counts2CPM(obj = dge,Log = T))
dim1 <- 'PC1'
dim2 <- 'PC2'
ellipse.type <- 'polygon' #ellipse.type=c('none','ellipse','polygon')

##--All Bio-reps plots
rownames(data2pca) <- gsub('_','.',rownames(data2pca))
groups <- samples_new$brep ## colour on biological replicates
# groups <- samples_new$condition ## colour on condtions
g <- plot.PCA.ind(data2pca = data2pca,dim1 = dim1,dim2 = dim2,
                  groups = groups,plot.title = 'genescript PCA: bio-reps',
                  ellipse.type = ellipse.type,
                  add.label = T,adj.label = F)

g
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-7-3.png" style="display: block; margin: auto;" />

``` r
### save to figure
png(filename = paste0(figure.folder,'/Gene PCA batch effect removed Bio-reps.png'),
    width = 15/2.54,height = 13/2.54,units = 'in',res = 300)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(file = paste0(figure.folder,'/Gene PCA batch effect removed Bio-reps.pdf'),
    width = 15/2.54,height = 13/2.54)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
##################################################
##--average expression plot
rownames(data2pca) <- gsub('_','.',rownames(data2pca))
groups <- samples_new$brep
data2pca.ave <- rowmean(data2pca,samples_new$condition,reorder = F)
groups <- unique(samples_new$condition)
g <- plot.PCA.ind(data2pca = data2pca.ave,dim1 = 'PC1',dim2 = 'PC2',
                  groups = groups,plot.title = 'genescript PCA: average expression',
                  ellipse.type = 'none',add.label = T,adj.label = F)

g
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-7-4.png" style="display: block; margin: auto;" />

``` r
### save to figure
png(filename = paste0(figure.folder,'/Gene PCA Average expression.png'),
    width = 15/2.54,height = 13/2.54,units = 'in',res = 300)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(file = paste0(figure.folder,'/Gene PCA Average expression.pdf'),
    width = 15/2.54,height = 13/2.54)
print(g)
dev.off()
```

    ## png 
    ##   2

### Step 4: Data normalisation

The same transcript/gene in deeplier sequenced samples have more mapped reads. To make genes/transcripts comparable across samples, the "TMM", "RLE" or "upperquantile" method can be used to normalize the expression.

``` r
################################################################################
##----->> trans level
dge <- DGEList(counts=trans_counts[target_high$trans_high,],
               group = samples_new$condition,
               genes = mapping[target_high$trans_high,])
trans_dge <- suppressWarnings(calcNormFactors(dge,method = norm.method))
save(trans_dge,file=paste0(data.folder,'/trans_dge.RData'))

################################################################################
##----->> genes level
dge <- DGEList(counts=genes_counts[target_high$genes_high,],
               group = samples_new$condition)
genes_dge <- suppressWarnings(calcNormFactors(dge,method = norm.method))
save(genes_dge,file=paste0(data.folder,'/genes_dge.RData'))

################################################################################
##----->> distribution plot
sample.name <- paste0(samples_new$condition,'.',samples_new$brep)
condition <- samples_new$condition

###--- trans level
data.before <- trans_counts[target_high$trans_high,]
data.after <- counts2CPM(obj = trans_dge,Log = T)
g <- boxplot.normalised(data.before = data.before,
                        data.after = data.after,
                        condition = condition,
                        sample.name = sample.name)
do.call(grid.arrange,g)
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-8-1.png" style="display: block; margin: auto;" />

``` r
### save to figure
png(filename = paste0(figure.folder,'/Transcript data distribution.png'),
    width = 20/2.54,height = 20/2.54,units = 'in',res = 300)
do.call(grid.arrange,g)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(file = paste0(figure.folder,'/Transcript data distribution.pdf'),
    width = 20/2.54,height = 20/2.54)
do.call(grid.arrange,g)
dev.off()
```

    ## png 
    ##   2

``` r
###--- genes level
data.before <- genes_counts[target_high$genes_high,]
data.after <- counts2CPM(obj = genes_dge,Log = T)
g <- boxplot.normalised(data.before = data.before,
                        data.after = data.after,
                        condition = condition,
                        sample.name = sample.name)
do.call(grid.arrange,g)
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-8-2.png" style="display: block; margin: auto;" />

``` r
### save to figure
png(filename = paste0(figure.folder,'/Gene data distribution.png'),
    width = 20/2.54,height = 20/2.54,units = 'in',res = 300)
do.call(grid.arrange,g)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(file = paste0(figure.folder,'/Gene data distribution.pdf'),
    width = 20/2.54,height = 20/2.54)
do.call(grid.arrange,g)
dev.off()
```

    ## png 
    ##   2

### Data information

``` r
data.info <- data.frame(
  raw.number = c(length(mapping$TXNAME),length(unique(mapping$GENEID))),
  sample = rep(nrow(samples),2),
  condition = rep(length(unique(samples$condition)),2),
  brep = rep(length(unique(samples$brep)),2),
  srep = rep(length(unique(samples$srep)),2),
  merged.sample = c(ncol(trans_counts),ncol(genes_counts)),
  cpm.cut = c(cpm.cut,'--'),
  sample.n.cut = c(sample.n.cut,'--'),
  after.filter = c(length(target_high$trans_high),length(target_high$genes_high))
)
rownames(data.info) <- c('Transcript','Gene')
data.info
```

    ##            raw.number sample condition brep srep merged.sample cpm.cut
    ## Transcript      82198     27         3    3    3             9       1
    ## Gene            34218     27         3    3    3             9      --
    ##            sample.n.cut after.filter
    ## Transcript            1        41871
    ## Gene                 --        19594

``` r
write.csv(data.info,file=paste0(result.folder,'/data.info.csv'),row.names = T)
```

DE, DAS and DTU analysis
------------------------

Three pipelines, <a href="https://bioconductor.org/packages/release/bioc/html/limma.html" target="_blank">limma</a> (Ritchie et al., 2015), <a href="https://bioconductor.org/packages/release/bioc/html/edgeR.html" target="_blank">glmQL (edgeR)</a> and <a href="https://bioconductor.org/packages/release/bioc/html/edgeR.html" target="_blank">glm (edgeR)</a> (Robinson et al., 2010) are provided for DE gene and transcripts analysis. "limma" and "edgeR" have comparable performance and they have stringent controls of false positives than the "glm".

### Step 1: Load contrast group

``` r
# The contrast groups passed to the analysis are in a vector object, e.g. \code{contrast <- c('B-A','C-A')} means compare conditions "B" and "C" to condition "A".
contrast <- read.csv(paste0(input.folder,'/contrast.csv'))
contrast <- paste0(contrast[,1],'-',contrast[,2])
###---generate proper contrast format
contrast <- gsub('_','.',contrast)
```

### Step 2: DE genes

``` r
batch.effect <- genes_batch$W
# batch.effect <- NULL ## if has no batch effects
design <- condition2design(condition = samples_new$condition,
                           batch.effect = batch.effect)

################################################################################
if(DE.pipeline == 'limma'){
##----->> limma pipeline
genes_3D_stat <- limma.pipeline(dge = genes_dge,
                                 design = design,
                                 deltaPS = NULL,
                                 contrast = contrast,
                                 diffAS = F,
                                 adjust.method = p.adjust.method)
}
```

    ## [1] "Contrast groups: T10-T2; T19-T2"

``` r
if(DE.pipeline == 'glmQL'){
##----->> edgeR glmQL pipeline
genes_3D_stat <- edgeR.pipeline(dge = genes_dge,
                                 design = design,
                                 deltaPS = NULL,
                                 contrast = contrast,
                                 diffAS = F,
                                 method = 'glmQL',
                                 adjust.method = p.adjust.method)
}

if(DE.pipeline == 'glm'){
##----->> edgeR glm pipeline
genes_3D_stat <- edgeR.pipeline(dge = genes_dge,
                                 design = design,
                                 deltaPS = NULL,
                                 contrast = contrast,
                                 diffAS = F,
                                 method = 'glm',
                                 adjust.method = p.adjust.method)
}
## save results
save(genes_3D_stat,file=paste0(data.folder,'/genes_3D_stat.RData'))
```

### Step 3: DAS genes, DE and DTU transcripts

``` r
################################################################################
##----->> generate deltaPS
transAbundance <- txi_trans$abundance[target_high$trans_high,]
deltaPS <- transAbundance2PS(transAbundance = transAbundance,
                             PS = NULL,
                             contrast = contrast,
                             condition = samples$condition,
                             mapping = mapping[target_high$trans_high,])
PS <- deltaPS$PS
deltaPS <- deltaPS$deltaPS
## save 
save(deltaPS,file=paste0(data.folder,'/deltaPS.RData')) 
save(PS,file=paste0(data.folder,'/PS.RData'))

################################################################################
##----->> DAS genes,DE and DTU transcripts
batch.effect <- genes_batch$W
# batch.effect <- NULL ## if has no batch effects
design <- condition2design(condition = samples_new$condition,
                           batch.effect = batch.effect)

################################################################################
p.adjust.method <- 'BH'
pval.cutoff <- 0.01
lfc.cutoff <- 1
deltaPS.cutoff <- 0.1

if(DE.pipeline == 'limma'){
##----->> limma pipeline
trans_3D_stat <- limma.pipeline(dge = trans_dge,
                                 design = design,
                                 deltaPS = deltaPS,
                                 contrast = contrast,
                                 diffAS = T,
                                 adjust.method = p.adjust.method)
}
```

    ## [1] "Contrast groups: T10-T2; T19-T2"
    ## Total number of exons:  41871 
    ## Total number of genes:  19594 
    ## Number of genes with 1 exon:  10233 
    ## Mean number of exons in a gene:  2 
    ## Max number of exons in a gene:  31

``` r
if(DE.pipeline == 'glmQL'){
##----->> edgeR glmQL pipeline
trans_3D_stat <- edgeR.pipeline(dge = trans_dge,
                                 design = design,
                                 deltaPS = deltaPS,
                                 contrast = contrast,
                                 diffAS = T,
                                 method = 'glmQL',
                                 adjust.method = p.adjust.method)
}

if(DE.pipeline == 'glm'){
##----->> edgeR glm pipeline
trans_3D_stat <- edgeR.pipeline(dge = trans_dge,
                                 design = design,
                                 deltaPS = deltaPS,
                                 contrast = contrast,
                                 diffAS = T,
                                 method = 'glm',
                                 adjust.method = p.adjust.method)
}
## save results
save(trans_3D_stat,file=paste0(data.folder,'/trans_3D_stat.RData'))
```

Result summary
--------------

### Significant 3D targets and testing statistics

``` r
################################################################################
##----->> Summary DE genes
DE_genes <- summary.DE.target(stat = genes_3D_stat$DE.stat,
                              cutoff = c(adj.pval=pval.cutoff,
                                         log2FC=lfc.cutoff))

################################################################################
## summary DAS genes, DE and DTU trans
##----->> DE trans
DE_trans <- summary.DE.target(stat = trans_3D_stat$DE.stat,
                              cutoff = c(adj.pval=pval.cutoff,
                                         log2FC=lfc.cutoff))

##----->> DAS genes
DAS.p.method <- 'F-test' ## DAS.p.method is one of 'F-test' and 
if(DAS.p.method=='F-test') {
  DAS.stat <- trans_3D_stat$DAS.F.stat
} else {
  DAS.stat <- trans_3D_stat$DAS.Simes.stat
}

lfc <- genes_3D_stat$DE.lfc
lfc <- reshape2::melt(as.matrix(lfc))
colnames(lfc) <- c('target','contrast','log2FC')
DAS_genes <- summary.DAS.target(stat = DAS.stat,
                                lfc = lfc,
                                cutoff=c(pval.cutoff,deltaPS.cutoff))

##----->> DTU trans
lfc <- trans_3D_stat$DE.lfc
lfc <- reshape2::melt(as.matrix(lfc))
colnames(lfc) <- c('target','contrast','log2FC')
DTU_trans <- summary.DAS.target(stat = trans_3D_stat$DTU.stat,
                                lfc = lfc,cutoff = c(adj.pval=pval.cutoff,
                                                     deltaPS=deltaPS.cutoff))

################################################################################
## save results
save(DE_genes,file=paste0(data.folder,'/DE_genes.RData'))
save(DE_trans,file=paste0(data.folder,'/DE_trans.RData')) 
save(DAS_genes,file=paste0(data.folder,'/DAS_genes.RData')) 
save(DTU_trans,file=paste0(data.folder,'/DTU_trans.RData')) 

## csv
write.csv(DE_genes,file=paste0(result.folder,'/DE genes.csv'),row.names = F)
write.csv(DAS_genes,file=paste0(result.folder,'/DAS genes.csv'),row.names = F)
write.csv(DE_trans,file=paste0(result.folder,'/DE transcripts.csv'),row.names = F)
write.csv(DTU_trans,file=paste0(result.folder,'/DTU transcripts.csv'),row.names = F)
```

### Significant 3D target numbers

``` r
################################################################################
##----->> target numbers
DDD.numbers <- summary.3D.number(DE_genes = DE_genes,
                                 DAS_genes = DAS_genes,
                                 DE_trans = DE_trans,
                                 DTU_trans=DTU_trans,
                                 contrast = contrast)
DDD.numbers
```

    ##       contrast DE genes DAS genes DE transcripts DTU transcripts
    ## 1       T10-T2      946       362           1271             598
    ## 2       T19-T2     2261       883           3079            1479
    ## 3 Intersection      433       244            599             377

``` r
write.csv(DDD.numbers,file=paste0(result.folder,'/DE DAS DTU numbers.csv'),
          row.names = F)

################################################################################
##----->> DE vs DAS
DEvsDAS.results <- DEvsDAS(DE_genes = DE_genes,
                           DAS_genes = DAS_genes,
                           contrast = contrast)
DEvsDAS.results
```

    ##       Contrast DEonly DE&DAS DASonly
    ## 1       T10-T2    928     18     344
    ## 2       T19-T2   2171     90     793
    ## 3 Intersection    408      4     216

``` r
write.csv(DEvsDAS.results,file=paste0(result.folder,'/DE vs DAS gene number.csv'),
          row.names = F)


################################################################################
##----->> DE vs DTU
DEvsDTU.results <- DEvsDTU(DE_trans = DE_trans,
                           DTU_trans = DTU_trans,
                           contrast = contrast)
DEvsDTU.results
```

    ##       Contrast DEonly DE&DTU DTUonly
    ## 1       T10-T2   1141    130     468
    ## 2       T19-T2   2600    479    1000
    ## 3 Intersection    483     59     201

``` r
write.csv(DEvsDTU.results,file=paste0(result.folder,'/DE vs DTU transcript number.csv'),row.names = F)
```

### Make plot

#### Up- and down-regulation

``` r
################################################################################
##----->> DE genes
idx <- factor(DE_genes$contrast,levels = contrast)
targets <-  split(DE_genes,idx)
data2plot <- lapply(contrast,function(i){
  if(nrow(targets[[i]])==0){
    x <- data.frame(contrast=i,regulation=c('down_regulate','up_regulate'),number=0)
  } else {
    x <- data.frame(contrast=i,table(targets[[i]]$up.down))
    colnames(x) <- c('contrast','regulation','number')
  }
  x
})
data2plot <- do.call(rbind,data2plot)
g.updown <- plot.updown(data2plot,plot.title = 'DE genes',contrast = contrast)

### save to figure
png(paste0(figure.folder,'/DE genes updown regulation numbers.png'),
    width = length(contrast)*5/2.54,10/2.54,units = 'in',res = 300)
print(g.updown)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(paste0(figure.folder,'/DE genes updown regulation numbers.pdf'),
    width = length(contrast)*5/2.54,10/2.54)
print(g.updown)
dev.off()
```

    ## png 
    ##   2

``` r
################################################################################
##----->> DAS genes
idx <- factor(DAS_genes$contrast,levels = contrast)
targets <-  split(DAS_genes,idx)
data2plot <- lapply(contrast,function(i){
  if(nrow(targets[[i]])==0){
    x <- data.frame(contrast=i,regulation=c('down_regulate','up_regulate'),number=0)
  } else {
    x <- data.frame(contrast=i,table(targets[[i]]$up.down))
    colnames(x) <- c('contrast','regulation','number')
  }
  x
})
data2plot <- do.call(rbind,data2plot)
g.updown <- plot.updown(data2plot,plot.title = 'DAS genes',contrast = contrast)

### save to figure
png(paste0(figure.folder,'/DAS genes updown regulation numbers.png'),
    width = length(contrast)*5/2.54,10/2.54,units = 'in',res = 300)
print(g.updown)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(paste0(figure.folder,'/DAS genes updown regulation numbers.pdf'),
    width = length(contrast)*5/2.54,10/2.54)
print(g.updown)
dev.off()
```

    ## png 
    ##   2

``` r
################################################################################
##----->> DE trans
idx <- factor(DE_trans$contrast,levels = contrast)
targets <-  split(DE_trans,idx)
data2plot <- lapply(contrast,function(i){
  if(nrow(targets[[i]])==0){
    x <- data.frame(contrast=i,regulation=c('down_regulate','up_regulate'),number=0)
  } else {
    x <- data.frame(contrast=i,table(targets[[i]]$up.down))
    colnames(x) <- c('contrast','regulation','number')
  }
  x
})
data2plot <- do.call(rbind,data2plot)
g.updown <- plot.updown(data2plot,plot.title = 'DE trans',contrast = contrast)

### save to figure
png(paste0(figure.folder,'/DE transcripts updown regulation numbers.png'),
    width = length(contrast)*5/2.54,10/2.54,units = 'in',res = 300)
print(g.updown)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(paste0(figure.folder,'/DE transcripts updown regulation numbers.pdf'),
    width = length(contrast)*5/2.54,10/2.54)
print(g.updown)
dev.off()
```

    ## png 
    ##   2

``` r
################################################################################
##----->> DTU trans
idx <- factor(DTU_trans$contrast,levels = contrast)
targets <-  split(DTU_trans,idx)
data2plot <- lapply(contrast,function(i){
  if(nrow(targets[[i]])==0){
    x <- data.frame(contrast=i,regulation=c('down_regulate','up_regulate'),number=0)
  } else {
    x <- data.frame(contrast=i,table(targets[[i]]$up.down))
    colnames(x) <- c('contrast','regulation','number')
  }
  x
})
data2plot <- do.call(rbind,data2plot)
g.updown <- plot.updown(data2plot,plot.title = 'DTU trans',contrast = contrast)

### save to figure
png(paste0(figure.folder,'/DTU transcripts updown regulation numbers.png'),
    width = length(contrast)*5/2.54,10/2.54,units = 'in',res = 300)
print(g.updown)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(paste0(figure.folder,'/DTU transcripts updown regulation numbers.pdf'),
    width = length(contrast)*5/2.54,10/2.54)
print(g.updown)
dev.off()
```

    ## png 
    ##   2

#### Flow chart

``` r
################################################################################
##----->> DE vs DAS genes
DE.genes <- unique(DE_genes$target) 
DAS.genes <- unique(DAS_genes$target) 

genes.flow.chart <- function(){
  plot.flow.chart(expressed = target_high$genes_high, 
                x = DE.genes, 
                y = DAS.genes, 
                type = 'genes', 
                pval.cutoff = pval.cutoff,
                lfc.cutoff = lfc.cutoff,
                deltaPS.cutoff = deltaPS.cutoff)
}

png(filename = paste0(figure.folder,'/Union set DE genes vs DAS genes.png'),
    width = 22/2.54,height = 13/2.54,units = 'in',res = 300)
genes.flow.chart()
dev.off()
```

    ## png 
    ##   2

``` r
pdf(file = paste0(figure.folder,'/Union set DE genes vs DAS genes.pdf'),
    width = 22/2.54,height = 13/2.54)
genes.flow.chart()
dev.off()
```

    ## png 
    ##   2

``` r
################################################################################
##----->> DE vs DTU transcripts
DE.trans<- unique(DE_trans$target)
DTU.trans <- unique(DTU_trans$target)

trans.flow.chart <- function(){
  plot.flow.chart(expressed = target_high$trans_high, 
                x = DE.trans, 
                y = DTU.trans, 
                type = 'transcripts', 
                pval.cutoff = pval.cutoff,
                lfc.cutoff = lfc.cutoff,
                deltaPS.cutoff = deltaPS.cutoff)
}

png(filename = paste0(figure.folder,'/Union set DE transcripts vs DTU transcripts.png'),
    width = 22/2.54,height = 13/2.54,units = 'in',res = 300)
trans.flow.chart()
dev.off()
```

    ## png 
    ##   2

``` r
pdf(file = paste0(figure.folder,'/Union set DE transcripts vs DTU transcripts.pdf'),
    width = 22/2.54,height = 13/2.54)
trans.flow.chart()
dev.off()
```

    ## png 
    ##   2

#### Comparisons between contrast groups

``` r
################################################################################
##----->> DE genes
targets <- lapply(contrast,function(i){
  subset(DE_genes,contrast==i)$target
})
names(targets) <- contrast
g <- plot.euler.diagram(x = targets)
grid.arrange(g,top=textGrob('DE genes', gp=gpar(cex=1.2)))
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-17-1.png" style="display: block; margin: auto;" />

``` r
################################################################################
##----->> DAS genes
targets <- lapply(contrast,function(i){
  subset(DAS_genes,contrast==i)$target
})
names(targets) <- contrast
g <- plot.euler.diagram(x = targets)
grid.arrange(g,top=textGrob('DAS genes', gp=gpar(cex=1.2)))
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-17-2.png" style="display: block; margin: auto;" />

``` r
################################################################################
##----->> DE transcripts
targets <- lapply(contrast,function(i){
  subset(DE_trans,contrast==i)$target
})
names(targets) <- contrast
g <- plot.euler.diagram(x = targets)
grid.arrange(g,top=textGrob('DE transcripts', gp=gpar(cex=1.2)))
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-17-3.png" style="display: block; margin: auto;" />

``` r
################################################################################
##----->> DTU transcripts
targets <- lapply(contrast,function(i){
  subset(DTU_trans,contrast==i)$target
})
names(targets) <- contrast
g <- plot.euler.diagram(x = targets)
grid.arrange(g,top=textGrob('DTU transcripts', gp=gpar(cex=1.2)))
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-17-4.png" style="display: block; margin: auto;" />

#### Comparison of transcription and AS targets

``` r
contrast.idx <- contrast[1]
################################################################################
##----->> DE vs DAS genes
x <- unlist(DEvsDAS.results[DEvsDAS.results$Contrast==contrast.idx,-1])
if(length(x)==0){
  message('No DE and/or DAS genes')
} else {
  names(x) <- c('DE','DE&DAS','DAS')
  g <- plot.euler.diagram(x = x,fill = gg.color.hue(2))
  g
  grid.arrange(g,top=textGrob('DE vs DAS genes', gp=gpar(cex=1.2)))
}
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-18-1.png" style="display: block; margin: auto;" />

``` r
################################################################################
##----->> DE vs DTU transcripts
x <- unlist(DEvsDTU.results[DEvsDTU.results$Contrast==contrast.idx,-1])
if(length(x)==0){
  message('No DE and/or DTU transcripts')
} else {
  names(x) <- c('DE','DE&DTU','DTU')
  g <- plot.euler.diagram(x = x,fill = gg.color.hue(2))
  g
  grid.arrange(g,top=textGrob('DE vs DTU transcripts', gp=gpar(cex=1.2)))
}
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-18-2.png" style="display: block; margin: auto;" />

Advanced plots
--------------

### Heatmap of 3D targets

The targets are firstly clustered into groups by using the <a href="https://cran.r-project.org/web/packages/fastcluster/index.html" target="_blank">fastcluster</a> R package, and then made into heatmap by using the <a href="https://bioconductor.org/packages/release/bioc/html/ComplexHeatmap.html" target="_blank">ComplexHeatmap</a> R package

``` r
################################################################################
##----->> DE genes
targets <- unique(DE_genes$target)
data2heatmap <- txi_genes$abundance[targets,]
column_title <- paste0(length(targets),' DE genes')
data2plot <- rowmean(x = t(data2heatmap),
                     group = samples$condition,
                     reorder = F)
data2plot <- t(scale(data2plot))

hc.dist <- dist(data2plot,method = dist.method)
hc <- fastcluster::hclust(hc.dist,method = cluster.method)
clusters <- cutree(hc, k = cluster.number)
clusters <- reorder.clusters(clusters = clusters,dat = data2plot)

### save the target list in each cluster to result folder
x <- split(names(clusters),clusters)
x <- lapply(names(x),function(i){
  data.frame(Clusters=i,Targets=x[[i]])
})
x <- do.call(rbind,x)
colnames(x) <- c('Clusters','Targets')
write.csv(x,file=paste0(result.folder,'/Target in each cluster heatmap ', column_title,'.csv'),
          row.names = F)
###############################

g <- Heatmap(as.matrix(data2plot), name = 'Z-scores', 
             cluster_rows = TRUE,
             clustering_method_rows=cluster.method,
             row_dend_reorder = T,
             show_row_names = FALSE, 
             show_column_names = ifelse(ncol(data2plot)>10,F,T),
             cluster_columns = FALSE,
             split=clusters,
             column_title= column_title)

draw(g,column_title='Conditions',column_title_side = "bottom")
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-19-1.png" style="display: block; margin: auto;" />

``` r
### save to figure
png(paste0(figure.folder,'/Heatmap DE genes.png'),
    width = pmax(10,1*length(unique(samples$condition)))/2.54,height = 20/2.54,units = 'in',res = 300)
draw(g,column_title='Conditions',column_title_side = "bottom")
dev.off()
```

    ## png 
    ##   2

``` r
pdf(paste0(figure.folder,'/Heatmap DE genes.pdf'),
    width = pmax(10,1*length(unique(samples$condition)))/2.54,height = 20/2.54)
draw(g,column_title='Conditions',column_title_side = "bottom")
dev.off()
```

    ## png 
    ##   2

``` r
################################################################################
##----->> DAS genes
dist.method <- 'euclidean'
cluster.method <- 'ward.D'
cluster.number <- 10
targets <- intersect(unique(DE_genes$target),unique(DAS_genes$target))
data2heatmap <- txi_genes$abundance[targets,]
column_title <- paste0(length(targets),' DE&DAS genes')
data2plot <- rowmean(x = t(data2heatmap),
                     group = samples$condition,
                     reorder = F)
data2plot <- t(scale(data2plot))

hc.dist <- dist(data2plot,method = dist.method)
hc <- fastcluster::hclust(hc.dist,method = cluster.method)
clusters <- cutree(hc, k = cluster.number)
clusters <- reorder.clusters(clusters = clusters,dat = data2plot)

### save the target list in each cluster to result folder
x <- split(names(clusters),clusters)
x <- lapply(names(x),function(i){
  data.frame(Clusters=i,Targets=x[[i]])
})
x <- do.call(rbind,x)
colnames(x) <- c('Clusters','Targets')
write.csv(x,file=paste0(result.folder,'/Target in each cluster heatmap ', column_title,'.csv'),
          row.names = F)
###############################

g <- Heatmap(as.matrix(data2plot), name = 'Z-scores', 
             cluster_rows = TRUE,
             clustering_method_rows=cluster.method,
             row_dend_reorder = T,
             show_row_names = FALSE, 
             show_column_names = ifelse(ncol(data2plot)>10,F,T),
             cluster_columns = FALSE,
             split=clusters,
             column_title= column_title)

draw(g,column_title='Conditions',column_title_side = "bottom")
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-19-2.png" style="display: block; margin: auto;" />

``` r
### save to figure
png(paste0(figure.folder,'/Heatmap DAS genes.png'),
    width = pmax(10,1*length(unique(samples$condition)))/2.54,height = 20/2.54,units = 'in',res = 300)
draw(g,column_title='Conditions',column_title_side = "bottom")
dev.off()
```

    ## png 
    ##   2

``` r
pdf(paste0(figure.folder,'/Heatmap DAS genes.pdf'),
    width = pmax(10,1*length(unique(samples$condition)))/2.54,height = 20/2.54)
draw(g,column_title='Conditions',column_title_side = "bottom")
dev.off()
```

    ## png 
    ##   2

``` r
################################################################################
##----->> DE trans
dist.method <- 'euclidean'
cluster.method <- 'ward.D'
cluster.number <- 10
targets <- unique(DE_trans$target)
data2heatmap <- txi_trans$abundance[targets,]
column_title <- paste0(length(targets),' DE trans')
data2plot <- rowmean(x = t(data2heatmap),
                     group = samples$condition,
                     reorder = F)
data2plot <- t(scale(data2plot))

hc.dist <- dist(data2plot,method = dist.method)
hc <- fastcluster::hclust(hc.dist,method = cluster.method)
clusters <- cutree(hc, k = cluster.number)
clusters <- reorder.clusters(clusters = clusters,dat = data2plot)

### save the target list in each cluster to result folder
x <- split(names(clusters),clusters)
x <- lapply(names(x),function(i){
  data.frame(Clusters=i,Targets=x[[i]])
})
x <- do.call(rbind,x)
colnames(x) <- c('Clusters','Targets')
write.csv(x,file=paste0(result.folder,'/Target in each cluster heatmap ', column_title,'.csv'),
          row.names = F)
###############################

g <- Heatmap(as.matrix(data2plot), name = 'Z-scores', 
             cluster_rows = TRUE,
             clustering_method_rows=cluster.method,
             row_dend_reorder = T,
             show_row_names = FALSE, 
             show_column_names = ifelse(ncol(data2plot)>10,F,T),
             cluster_columns = FALSE,
             split=clusters,
             column_title= column_title)

draw(g,column_title='Conditions',column_title_side = "bottom")
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-19-3.png" style="display: block; margin: auto;" />

``` r
### save to figure
png(paste0(figure.folder,'/Heatmap DE transcripts.png'),
    width = pmax(10,1*length(unique(samples$condition)))/2.54,height = 20/2.54,units = 'in',res = 300)
draw(g,column_title='Conditions',column_title_side = "bottom")
dev.off()
```

    ## png 
    ##   2

``` r
pdf(paste0(figure.folder,'/Heatmap DE transcripts.pdf'),
    width = pmax(10,1*length(unique(samples$condition)))/2.54,height = 20/2.54)
draw(g,column_title='Conditions',column_title_side = "bottom")
dev.off()
```

    ## png 
    ##   2

``` r
################################################################################
##----->> DTU trans
dist.method <- 'euclidean'
cluster.method <- 'ward.D'
cluster.number <- 10
targets <- intersect(unique(DE_trans$target),unique(DTU_trans$target))
data2heatmap <- txi_trans$abundance[targets,]
column_title <- paste0(length(targets),' DE&DTU trans')
data2plot <- rowmean(x = t(data2heatmap),
                     group = samples$condition,
                     reorder = F)
data2plot <- t(scale(data2plot))

hc.dist <- dist(data2plot,method = dist.method)
hc <- fastcluster::hclust(hc.dist,method = cluster.method)
clusters <- cutree(hc, k = cluster.number)
clusters <- reorder.clusters(clusters = clusters,dat = data2plot)

### save the target list in each cluster to result folder
x <- split(names(clusters),clusters)
x <- lapply(names(x),function(i){
  data.frame(Clusters=i,Targets=x[[i]])
})
x <- do.call(rbind,x)
colnames(x) <- c('Clusters','Targets')
write.csv(x,file=paste0(result.folder,'/Target in each cluster heatmap ', column_title,'.csv'),
          row.names = F)
###############################

g <- Heatmap(as.matrix(data2plot), name = 'Z-scores', 
             cluster_rows = TRUE,
             clustering_method_rows=cluster.method,
             row_dend_reorder = T,
             show_row_names = FALSE, 
             show_column_names = ifelse(ncol(data2plot)>10,F,T),
             cluster_columns = FALSE,
             split=clusters,
             column_title= column_title)

draw(g,column_title='Conditions',column_title_side = "bottom")
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-19-4.png" style="display: block; margin: auto;" />

``` r
### save to figure
png(paste0(figure.folder,'/Heatmap DTU transcripts.png'),
    width = pmax(10,1*length(unique(samples$condition)))/2.54,height = 20/2.54,units = 'in',res = 300)
draw(g,column_title='Conditions',column_title_side = "bottom")
dev.off()
```

    ## png 
    ##   2

``` r
pdf(paste0(figure.folder,'/Heatmap DTU transcripts.pdf'),
    width = pmax(10,1*length(unique(samples$condition)))/2.54,height = 20/2.54)
draw(g,column_title='Conditions',column_title_side = "bottom")
dev.off()
```

    ## png 
    ##   2

### Profile plot

``` r
################################################################################
##----->> generate annotation of DE and/or DAS, DE and/or DTU
DE.genes <- unique(DE_genes$target)
DAS.genes <- unique(DAS_genes$target)
DE.trans <- unique(DE_trans$target)
DTU.trans <- unique(DTU_trans$target)

genes.ann <- set2(DE.genes,DAS.genes)
names(genes.ann) <- c('DEonly','DE&DAS','DASonly')
genes.ann <- plyr::ldply(genes.ann,cbind)[,c(2,1)]
colnames(genes.ann) <- c('target','annotation')


trans.ann <- set2(DE.trans,DTU.trans)
names(trans.ann) <- c('DEonly','DE&DTU','DTUonly')
trans.ann <- plyr::ldply(trans.ann,cbind)[,c(2,1)]
colnames(trans.ann) <- c('target','annotation')

################################################################################
##give a gene name
gene <- 'AT5G24470'
##----->> profile plot of TPM or read counts
g.pr <- plot.abundance(data.exp = txi_trans$abundance[target_high$trans_high,],
                       gene = gene,
                       mapping = mapping[target_high$trans_high,],
                       genes.ann = genes.ann,
                       trans.ann = trans.ann,
                       trans.expressed = NULL,
                       reps = samples$condition,
                       y.lab = 'TPM')
g.pr
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-20-1.png" style="display: block; margin: auto;" />

``` r
################################################################################
##----->> PS plot
g.ps <- plot.PS(data.exp = txi_trans$abundance[target_high$trans_high,],
                gene = gene,
                mapping = mapping[target_high$trans_high,],
                genes.ann = genes.ann,
                trans.ann = trans.ann,
                trans.expressed = NULL,
                reps = samples$condition,
                y.lab = 'PS')
g.ps
```

<img src="Command_line_user_manul_files/figure-markdown_github/unnamed-chunk-20-2.png" style="display: block; margin: auto;" />

### GO annotation plot of DE/DAS genes

To make the plot, users must provide GO annotation table in csv file, with first column of "Category" (i.e. CC, BP and MF), second column of GO "Term" and the remaining columns of statistics of significance.

``` r
################################################################################
##----->> DE genes
go.table <- suppressMessages(readr::read_csv(paste0(input.folder,'/DE genes GO terms.csv')))
go.table <- go.table[,c(1,2,5)]
g <- plotGO(go.table = go.table,col.idx = '-log10(FDR)',
            plot.title = 'GO annotation: DE genes')

### save to figure
png(paste0(figure.folder,'/DE genes GO annotation plot.png'),
    width = 20/2.54,height = 21/2.54,
    units = 'in',res = 300)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(paste0(figure.folder,'/DE genes GO annotation plot.pdf'),
    width = 20/2.54,height = 21/2.54)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
################################################################################
##----->> DAS genes
go.table <- suppressMessages(readr::read_csv(paste0(input.folder,'/DAS genes GO terms.csv')))
go.table <- go.table[,c(1,2,5)]
g <- plotGO(go.table = go.table,col.idx = '-log10(FDR)',
         plot.title = 'GO annotation: DAS genes')

### save to figure
png(paste0(figure.folder,'/DAS genes GO annotation plot.png'),
    width = 20/2.54,height = 8/2.54,
    units = 'in',res = 300)
print(g)
dev.off()
```

    ## png 
    ##   2

``` r
pdf(paste0(figure.folder,'/DAS genes GO annotation plot.pdf'),
    width = 20/2.54,height = 8/2.54)
print(g)
dev.off()
```

    ## png 
    ##   2

Generate report
---------------

``` r
################################################################################
##----->> Summary parameters 
para <- rbind(
  c('Folders','Directory',getwd()),
  c('','Data',paste0(getwd(),'/',data.folder)),
  c('','Results',paste0(getwd(),'/',result.folder)),
  c('','Figures',paste0(getwd(),'/',figure.folder)),
   c('','Reports',paste0(getwd(),'/',report.folder)),
  c('Data generation','tximport method',tximport.method),
  c('Data pre-processing','Sequencing replicates',length(unique(samples$srep))),
  c('','Sequencing replicate merged',ifelse(nrow(samples)>nrow(samples_new), 'Yes','No')),
  c('','Low expression CPM cut-off',cpm.cut),
  c('','Sample number for CPM cut-off', sample.n.cut),
  c('','Batch effect estimation',
    ifelse(length(list.files(data.folder,'*batch.RData'))>0,'Yes','No')),
  c('','Batch effect estimation method',ruvseq.method),
  c('','Normalisation method',norm.method),
  c('DE DAS and DTU','Pipeline',DE.pipeline),
  c('','AS function',ifelse(DE.pipeline=='limma',
                            'limma::diffSplice','edgeR::diffSpliceDGE')),
  c('','P-value adjust method',p.adjust.method),
  c('','Adjusted p-value cut-off',pval.cutoff),
  c('','Log2 fold change cut-off',lfc.cutoff),
  c('','delta PS cut-off',deltaPS.cutoff),
  c('Heatmap','Distance method',dist.method),
  c('','Cluster method',cluster.method),
  c('','Number of clusters',cluster.number)
)
para <- data.frame(para)
colnames(para) <- c('Step','Description','Parameter (Double click to edit if not correct)')
para
```

    ##                   Step                    Description
    ## 1              Folders                      Directory
    ## 2                                                Data
    ## 3                                             Results
    ## 4                                             Figures
    ## 5                                             Reports
    ## 6      Data generation                tximport method
    ## 7  Data pre-processing          Sequencing replicates
    ## 8                         Sequencing replicate merged
    ## 9                          Low expression CPM cut-off
    ## 10                      Sample number for CPM cut-off
    ## 11                            Batch effect estimation
    ## 12                     Batch effect estimation method
    ## 13                               Normalisation method
    ## 14      DE DAS and DTU                       Pipeline
    ## 15                                        AS function
    ## 16                              P-value adjust method
    ## 17                           Adjusted p-value cut-off
    ## 18                           Log2 fold change cut-off
    ## 19                                   delta PS cut-off
    ## 20             Heatmap                Distance method
    ## 21                                     Cluster method
    ## 22                                 Number of clusters
    ##                                                               Parameter (Double click to edit if not correct)
    ## 1         D:/PhD project/R projects/test round 2018/DDD-GUI of DE DAS and DTU in case studies of RNA-seq data
    ## 2    D:/PhD project/R projects/test round 2018/DDD-GUI of DE DAS and DTU in case studies of RNA-seq data/data
    ## 3  D:/PhD project/R projects/test round 2018/DDD-GUI of DE DAS and DTU in case studies of RNA-seq data/result
    ## 4  D:/PhD project/R projects/test round 2018/DDD-GUI of DE DAS and DTU in case studies of RNA-seq data/figure
    ## 5  D:/PhD project/R projects/test round 2018/DDD-GUI of DE DAS and DTU in case studies of RNA-seq data/report
    ## 6                                                                                             lengthScaledTPM
    ## 7                                                                                                           3
    ## 8                                                                                                         Yes
    ## 9                                                                                                           1
    ## 10                                                                                                          1
    ## 11                                                                                                        Yes
    ## 12                                                                                                       RUVr
    ## 13                                                                                                        TMM
    ## 14                                                                                                      limma
    ## 15                                                                                          limma::diffSplice
    ## 16                                                                                                         BH
    ## 17                                                                                                       0.01
    ## 18                                                                                                          1
    ## 19                                                                                                        0.1
    ## 20                                                                                                  euclidean
    ## 21                                                                                                     ward.D
    ## 22                                                                                                         10

``` r
write.csv(para,file=paste0(result.folder,'/Parameter summary.csv'),row.names = F)

################################################################################
##----->> download report.Rmd file from Github
# if(!file.exists('report.Rmd'))
#   download.file(url = 'https://github.com/wyguo/ThreeDRNAseq/blob/master/vignettes/report.Rmd',
#               destfile = 'report.Rmd',method = 'auto',mode = )
# generate.report(input.Rmd = 'report.Rmd',report.folder = report.folder,type = 'all')
# 
# report.url <- 'https://rmarkdown.rstudio.com/demos/1-example.Rmd'
# report.url <- 'https://github.com/wyguo/ThreeDRNAseq/blob/master/vignettes/report.Rmd'
# download.file(url = report.url,
#               destfile = 'example2.Rmd',method = 'auto')
# 
# knitrRmd <- paste(readLines(textConnection(getURL(report.url))), collapse="\n")
```

References
----------

Balaban,M.O., Unal Sengor,G.F., Soriano,M.G., and Ruiz,E.G. (2011) Quantification of gaping, bruising, and blood spots in salmon fillets using image analysis. J Food Sci, 76, E291-7.

Bray,N.L., Pimentel,H., Melsted,P., and Pachter,L. (2016) Near-optimal probabilistic RNA-seq quantification. Nat. Biotechnol., 34, 525–527.

Patro,R., Duggal,G., Love,M.I., Irizarry,R.A., and Kingsford,C. (2017) Salmon provides fast and bias-aware quantification of transcript expression. Nat. Methods, 14, 417–419.

Risso,D., Ngai,J., Speed,T.P., and Dudoit,S. (2014) Normalization of RNA-seq data using factor analysis of control genes or samples. Nat. Biotechnol., 32, 896–902.

Ritchie,M.E., Phipson,B., Wu,D., Hu,Y., Law,C.W., Shi,W., and Smyth,G.K. (2015) limma powers differential expression analyses for RNA-sequencing and microarray studies. Nucleic Acids Res, 43, e47.

Robinson,M., Mccarthy,D., Chen,Y., and Smyth,G.K. (2011) edgeR : differential expression analysis of digital gene expression data User ’ s Guide. Most, 23, 1–77.

Soneson,C., Love,M.I., and Robinson,M.D. (2016) Differential analyses for RNA-seq: transcript-level estimates improve gene-level inferences. F1000Research, 4, 1521.

Session information
-------------------

    ## R version 3.5.1 (2018-07-02)
    ## Platform: x86_64-w64-mingw32/x64 (64-bit)
    ## Running under: Windows 7 x64 (build 7601) Service Pack 1
    ## 
    ## Matrix products: default
    ## 
    ## locale:
    ## [1] LC_COLLATE=English_United Kingdom.1252 
    ## [2] LC_CTYPE=English_United Kingdom.1252   
    ## [3] LC_MONETARY=English_United Kingdom.1252
    ## [4] LC_NUMERIC=C                           
    ## [5] LC_TIME=English_United Kingdom.1252    
    ## 
    ## attached base packages:
    ##  [1] grid      stats4    parallel  stats     graphics  grDevices utils    
    ##  [8] datasets  methods   base     
    ## 
    ## other attached packages:
    ##  [1] Gmisc_1.6.4                 htmlTable_1.12             
    ##  [3] Rcpp_0.12.18                ComplexHeatmap_1.18.1      
    ##  [5] gridExtra_2.3               eulerr_4.1.0               
    ##  [7] RUVSeq_1.14.0               EDASeq_2.14.1              
    ##  [9] ShortRead_1.38.0            GenomicAlignments_1.16.0   
    ## [11] SummarizedExperiment_1.10.1 DelayedArray_0.6.6         
    ## [13] matrixStats_0.54.0          Rsamtools_1.32.3           
    ## [15] GenomicRanges_1.32.6        GenomeInfoDb_1.16.0        
    ## [17] Biostrings_2.48.0           XVector_0.20.0             
    ## [19] IRanges_2.14.11             S4Vectors_0.18.3           
    ## [21] BiocParallel_1.14.2         Biobase_2.40.0             
    ## [23] BiocGenerics_0.26.0         plotly_4.8.0.9000          
    ## [25] ggplot2_3.0.0               edgeR_3.22.3               
    ## [27] limma_3.36.3                tximport_1.11.0            
    ## [29] shinyFiles_0.7.1            rhandsontable_0.3.6        
    ## [31] shinydashboard_0.7.0        shiny_1.1.0                
    ## 
    ## loaded via a namespace (and not attached):
    ##   [1] backports_1.1.2        circlize_0.4.4         Hmisc_4.1-1           
    ##   [4] aroma.light_3.10.0     plyr_1.8.4             lazyeval_0.2.1        
    ##   [7] splines_3.5.1          digest_0.6.17          htmltools_0.3.6       
    ##  [10] magrittr_1.5           checkmate_1.8.5        memoise_1.1.0         
    ##  [13] cluster_2.0.7-1        fastcluster_1.1.25     readr_1.1.1           
    ##  [16] annotate_1.58.0        R.utils_2.7.0          prettyunits_1.0.2     
    ##  [19] colorspace_1.3-2       blob_1.1.1             dplyr_0.7.6           
    ##  [22] crayon_1.3.4           RCurl_1.95-4.11        jsonlite_1.5          
    ##  [25] genefilter_1.62.0      bindr_0.1.1            survival_2.42-6       
    ##  [28] glue_1.3.0             polyclip_1.9-1         gtable_0.2.0          
    ##  [31] zlibbioc_1.26.0        GetoptLong_0.1.7       shape_1.4.4           
    ##  [34] abind_1.4-5            scales_1.0.0           DESeq_1.32.0          
    ##  [37] DBI_1.0.0              viridisLite_0.3.0      xtable_1.8-3          
    ##  [40] progress_1.2.0         foreign_0.8-71         bit_1.1-14            
    ##  [43] Formula_1.2-3          htmlwidgets_1.2        httr_1.3.1            
    ##  [46] RColorBrewer_1.1-2     acepack_1.4.1          pkgconfig_2.0.2       
    ##  [49] XML_3.98-1.16          R.methodsS3_1.7.1      nnet_7.3-12           
    ##  [52] locfit_1.5-9.1         reshape2_1.4.3         labeling_0.3          
    ##  [55] tidyselect_0.2.4       rlang_0.2.2            later_0.7.4           
    ##  [58] AnnotationDbi_1.42.1   munsell_0.5.0          tools_3.5.1           
    ##  [61] RSQLite_2.1.1          evaluate_0.11          stringr_1.3.1         
    ##  [64] yaml_2.2.0             knitr_1.20             bit64_0.9-7           
    ##  [67] fs_1.2.6               forestplot_1.7.2       purrr_0.2.5           
    ##  [70] bindrcpp_0.2.2         mime_0.5               R.oo_1.22.0           
    ##  [73] biomaRt_2.36.1         compiler_3.5.1         rstudioapi_0.7        
    ##  [76] tibble_1.4.2           geneplotter_1.58.0     stringi_1.1.7         
    ##  [79] GenomicFeatures_1.32.2 lattice_0.20-35        Matrix_1.2-14         
    ##  [82] pillar_1.3.0           GlobalOptions_0.1.0    data.table_1.11.6     
    ##  [85] bitops_1.0-6           httpuv_1.4.5           rtracklayer_1.40.6    
    ##  [88] R6_2.2.2               latticeExtra_0.6-28    hwriter_1.3.2         
    ##  [91] promises_1.0.1         gtools_3.8.1           MASS_7.3-50           
    ##  [94] assertthat_0.2.0       rprojroot_1.3-2        rjson_0.2.20          
    ##  [97] withr_2.1.2            GenomeInfoDbData_1.1.0 hms_0.4.2             
    ## [100] rpart_4.1-13           tidyr_0.8.1            rmarkdown_1.10        
    ## [103] base64enc_0.1-3