User guide for package m6ALogisticModel
================

Installation
------------

Currently, you could install this package using this command in R.

``` r
devtools::install_github("ZhenWei10/m6ALogisticModel")
```

<hr/>
Motivation
----------

The major reasons for me to create this package are:

1.  Deal with **confounding genomic features** that are common in RNA genomics.

2.  Reduce the traditional work load for creating and screening the highly explanatory RNA features one by one, while unable to account the dependencies between those features.

3.  Provide a template for the linear modeling on the relationship between **belonging (dummy), relative position, and length** for a given feature on transcript. The previous researches suggest that these 3 different properties of a region are important predictors for the behaviours of m6A in different biological context.

**Logistic regression modeling** seems to be a reasonable computational technique here, since it can efficiently quantify the scientific and statistical significance for all the features, while adjust their dependencies on each others. The logistic regression is coupled with bayesian model selection on genaralized linear model for reasons of reduced model predictability and robustness in the presence of large number of highly correlated features (high collinearity). Also, model selection method can yield highly interpretable results at the same time making the estimators unbiased.

The funcions for the transcript feature annotation and the logistic modeling are included in this package; while at the same time, users can introduce more genomic features defined by them self using GRanges object. Effective visualization for comparation between multiple selected models are also implemented in this package.

<hr/>
A case study with this package.
-------------------------------

To demonstrate the application of this package, I provide a case analysis on finding the transcriptomic features that can predict the evolutionary conservation of m6A locations. The m6A locations used were reported by [RMBase2](http://rna.sysu.edu.cn/rmbase/), and the genomic conservation scores used are extracted from the bioconductor packages [phastCons100way.UCSC.hg19](http://www.bioconductor.org/packages/release/data/annotation/html/phastCons100way.UCSC.hg19.html) and [fitCons.UCSC.hg19](http://www.bioconductor.org/packages/release/data/annotation/html/fitCons.UCSC.hg19.html).

``` r
library(m6ALogisticModel)
library(fitCons.UCSC.hg19)
library(phastCons100way.UCSC.hg19)
```

The m6A location information is stored within data `SE_example` of the package `m6ALogisticMode`.

The rowRanges contains the conserved m6A sites between hg19 and mm10 reported by RMBase2.

``` r
library(SummarizedExperiment)
RMBase2_hg19_gr <- rowRanges( SE_example )
```

The hg19 `TxDb` and `BSgenome` are loaded for the purpose of extracting the transcriptomic features.

``` r
library(TxDb.Hsapiens.UCSC.hg19.knownGene)
library(BSgenome.Hsapiens.UCSC.hg19)
```

First, I subset the m6A sites by overlapping with exons, because most of m6A sites on RMBase2 are overlapped with exons.

``` r
mean(RMBase2_hg19_gr %over% exons(TxDb.Hsapiens.UCSC.hg19.knownGene))
RMBase2_hg19_gr <- subsetByOverlaps(RMBase2_hg19_gr,exons(TxDb.Hsapiens.UCSC.hg19.knownGene))
Num <- length(RMBase2_hg19_gr) 
Num #There are around 0.137 million sites within it
```

Then, for purpose of control, I sample the same number of A sites and RRACH sites on the exonic regions of the transcripts mapped with the m6A sites above.

``` r
exbytx_hg19 <- exonsBy(TxDb.Hsapiens.UCSC.hg19.knownGene, by = c("tx"))
Subset_regions <- unlist( subsetByOverlaps( exbytx_hg19 , RMBase2_hg19_gr) )

set.seed(1)
Random_RRACH <-  m6ALogisticModel::Sample_sequence("RRACH", reduce( Subset_regions ), Hsapiens, N = Num) - 2
Random_A <-  m6ALogisticModel::Sample_sequence("A", reduce( Subset_regions ), Hsapiens, N = Num)
```

Merge the 3 GRanges into a single `SummarizedExperiment` object.

``` r
Row_Ranges <-
  reduce(c(RMBase2_hg19_gr, 
  Random_RRACH,
  Random_A), min.gapwidth=0L)

SE <- SummarizedExperiment( matrix(rep(NA,6*length(Row_Ranges)) ,ncol = 6) )

rowRanges(SE) = Row_Ranges
```

<hr/>
Define targets (response variables)
-----------------------------------

The PhastCons scores and fitCons scores of the As underlying sequences are extracted, each scores are calculated on locations of RMBase2 m6A, random RRACH, and random A without specific context.

``` r
library(dplyr)

GR_list <- list(RMBase2_hg19_gr,Random_RRACH,Random_A)

PhastCons_scores_all <- scores(phastCons100way.UCSC.hg19, rowRanges(SE))$scores

FitCons_scores_all <- scores(fitCons.UCSC.hg19, rowRanges(SE))$scores
  
for (i in 1:3) {
indx <- rowRanges(SE) %over% GR_list[[i]]
assay(SE)[indx,i] = PhastCons_scores_all[indx]
assay(SE)[indx,i+3] = FitCons_scores_all[indx]
}


colnames(SE) = paste0( rep(c("PastCons","FitCons"),
                                           each = 3),"_",
                                     rep(c("m6A","RRACH","A"),2) )
```

Visualize the distribution of the scores, and choose the cut off values to convert the scores into binary variables.

``` r
Plot_df <- reshape2::melt(assay(SE))
Plot_df$X_intercept = NA
cut_offs <- c(.9,.9,.9,.4,.4,.4)
for (i in 1:(2*3)){
Plot_df$X_intercept[Plot_df$Var2 == colnames(SE)[i]] <- cut_offs[i]
}
library(ggplot2)
ggplot(Plot_df) + geom_histogram(aes(x = value),fill = "grey") + facet_wrap(~Var2, nrow = 2, scales = "free_y") + theme_classic() + geom_vline(aes(xintercept = X_intercept),colour = "blue")
```

We could easily observed that the m6A locus in RMBase2 are more conserved than other locus for both scores.

``` r
#Convert the matrix with only entries of 0, 1, and NA using the cut_off defined above
for(i in 1:6) {
assay(SE)[,i] <- as.numeric(assay(SE)[,i] > cut_offs[i])
}

#Calculate proportions of positive instances in Target
for(i in 1:6) {
 cat( paste0(colnames(SE)[i],": ", round( mean( assay(SE)[,i],na.rm = T) ,3) , "\n"))
}
```

The targets still face the issue of not having balanced classes, but for now, we will just omit it and plug it into our logistic regression functions.

<hr/>
Generate features (predictors)
------------------------------

``` r
Feature_List_hg19 = list(
HNRNPC_eCLIP = eCLIP_HNRNPC_gr,
YTHDC1_TREW = YTHDC1_TREW_gr,
YTHDF1_TREW = YTHDF1_TREW_gr,
YTHDF2_TREW = YTHDF2_TREW_gr,
miR_targeted_genes = miR_targeted_genes_grl,
TargetScan = TargetScan_hg19_gr,
Verified_miRtargets = verified_targets_gr
)

SE <- m6ALogisticModel::predictors.annot(se = SE,
                                   txdb = TxDb.Hsapiens.UCSC.hg19.knownGene,
                                   bsgnm = BSgenome.Hsapiens.UCSC.hg19,
                                   struct_hybridize = Struc_hg19,
                                   feature_lst = Feature_List_hg19,
                                   HK_genes_list = HK_hg19_eids)
```

45 Transcriptomic features on hg19 are automatically attached by the command and data defined in `m6ALogisticModel`.

<hr/>
Model selection and inference
-----------------------------

-   Run the following commands, you will find the results saved under `save_dir` after some running times.
-   You could make this process much more efficient with reduced `MCMC_interations` number and reduced instance number in your matrix.

``` r
SE <- readRDS("SE.rds")
library(m6ALogisticModel)
library(SummarizedExperiment)
Group_list <- group_list_default[names(group_list_default) != "Evolution"]
set.seed(2)
m6ALogisticModel::logistic.modeling(SE,
                                    save_dir = "Conservation_scores",
                                    group_list = Group_list,
                                    MCMC_iterations = 100000)
```

<hr/>
``` r
sessionInfo()
```
