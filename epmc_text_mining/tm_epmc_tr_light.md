---
title: Evaluation text mining predictions EPMC
date: 2016-03-31
author: Pablo Porras
---


Text mining prediction EPMC: Technical report
========================================================

### Synopsis

We collaborated the EPMC team at EBI to use text-mining techniques that use a set of keywords (see Appendix) that describe potential interaction relationships to identify sentences that may depict interacting proteins. The sentences are extracted from the abstract or full text (when available, due to copyright restrictions) or publications listed in PubMed. The text-mining exercise was performed by Senay Kafkas (SK). Email exchange about this subject can be found in the folder ./source_files/emails. 


### Part 1: Evaluate and extract information from SK file

SK sent the result file from her search script on 2015/08/28 and an updated version with all PMIDs added on 2016/04/07. Here is the text of the email she sent on the first instance:  

*"I now adapted our gene-disease pipeline which we developed for the cttv project, for extracting the PPIs. I used the keyword list that we agreed on before.*  

*The data that I generated is big, hence I cannot attach it to the e-mail, but you can access it from:*

*/nfs/misc/literature/shenay/PPIExtraction/PPIData4Expr.txt*

*I provided the data in the format that we provide for the cttv project. The first line in the file describes the data format which is not complicated to understand. One thing is I provided only 3 publication IDs and sentences. However if you would need all of the publications and sentences, I can extract this data easily as well."*

It is important to note that the data contains human identifiers only. 

#### Pre-formatting the text-mined dataset

I re-process the dataset by removing the reference sentences, which make up for most of the file size. 
Further cleaning is required, since there are several UniProtKB accessions, comma-separated, per line. I need to have a single identifier per line in order to generate pair ids. 
Finally, the PMIDs and PMCIDs given in the PMCID field also need to be divided in single line records.

All tasks are tackled using PERL, which is more efficient than R in this case. 


```r
system("perl ./scripts/field_selector.pl ./source_files/PPIData4Expr_noLimit.txt ./processed_files/PPIData4Expr_sel_fields.txt")
system("rm ./source_files/PPIData4Expr_noLimit.txt")
system("perl ./scripts/multiplier.pl ./processed_files/PPIData4Expr_sel_fields.txt ./processed_files/tm_full.txt")
system("perl ./scripts/multiplier2.pl ./processed_files/tm_full.txt ./processed_files/tm_full_single.txt")
```

File size gets dramatically reduced (5.25 GB to ~94 MB), so it is now manageable and I load it up. 


```r
library(data.table)
tm_full <- fread("./processed_files/tm_full_single.txt", sep = "\t", header = T, colClasses="character",data.table=F)

tm_full <- data.frame(tm_full,colClasses="character")

system("rm ./processed_files/tm_full.txt")
system("rm ./processed_files/tm_full_single.txt")

tm_full$Nof.Docs <- as.numeric(tm_full$Nof.Docs)
tm_full$Nof.co.occr.in.Title.Abs <- as.numeric(tm_full$Nof.co.occr.in.Title.Abs)
tm_full$Nof.co.occr.in.Body <- as.numeric(tm_full$Nof.co.occr.in.Body)
```

```
## Warning: NAs introduced by coercion
```

```r
tm_full <- tm_full[order(as.numeric(tm_full$Nof.Docs),decreasing=T),]
```

I check out how many publications could be screened in full (not only the abstract).


```r
pub_nobody <- unique(subset(tm_full,is.na(Nof.co.occr.in.Body), select=c("PMCIDs")))
library(dplyr)
pub_total <- unique(select(tm_full,PMCIDs))
```

It was possible to check the body of the text in 70.41% of the total number of publications reported. 

Now I generate the pair ids for the text-mined datset.


```r
tm_full$pair_id <- apply(tm_full[,1:2], 1,function(i){
  paste(sort(i),collapse = "_")
})
tm_full$tm <- 1
```

#### Exploring the dataset

I do a tentative plot of the data, to see how many pairs, roughly, are represented in just a few publications and how many are represented in many of them. 


```r
library(dplyr)
tm_full_pair_info <- unique(select(tm_full,pair_id,Nof.Docs,Nof.co.occr.in.Title.Abs,Nof.co.occr.in.Body,tm))

library(ggplot2)

g <- ggplot(data=tm_full_pair_info, aes(tm_full_pair_info$Nof.Docs))
g <- g + geom_density()
g <- g + labs(title ="Publications behind interacting pairs", y = "Density of publications", x= "Number of publications supporting the interaction")

g2 <- g + coord_cartesian(ylim = c(0,0.0001))
g2 <- g2 + labs(title ="Zoomed to the y axis lowest region", x = "Number of publications supporting the interaction", y="")

multiplot(g,g2,cols=2)
```

![plot of chunk unnamed-chunk-6](figure/unnamed-chunk-6-1.png)

The graph is not perfect, but it is still informative. I now need to compare the text mining predictions with the IMEx set of interactions. 

### Part 2: Comparison with the IMEx dataset

#### Loading the IMEx dataset

I use a small pipeline to put together data from DIP and IntAct. The details can be found [here](https://github.com/pporrasebi/darkspaceproject/IMEx/IMEx_dsgen.md). 


```r
imex_full <- read.delim("../../darkspaceproject/IMEx/results//imex_full.txt", header=T, sep="\t",colClasses="character")
```

I select exclusively the human data (interactions where both proteins are human). 


```r
imex_human <- unique(subset(imex_full,taxid_a=="9606" & taxid_b=="9606"))
```

#### Comparison between IMEx and the text-mined dataset at the pair level

I compare both datasets using the pair ids.


```r
imex_sel <- unique(select(imex_human,pair_id_clean,pair_id_clean,id_a_clean,id_b_clean,taxid_a,taxid_b,pubid))
imex_sel$imex <- 1
imex_pairs <- unique(select(imex_sel, pair_id=pair_id_clean,imex))

comp <- unique(merge(tm_full_pair_info,imex_pairs,by="pair_id",all=T))

comp <- mutate(comp, db_pair =
                 ifelse(tm == 1 & is.na(imex), "tm",
                 ifelse(is.na(tm) & imex == 1, "imex",
                 ifelse(tm == 1 & imex == 1, "tm & imex",
                 "check"))))

comp$db_pair <- as.factor(comp$db_pair)

comp <- mutate(comp, nr_oc_group =
               ifelse(Nof.Docs <= 5, Nof.Docs, "over 5"))

table(comp$db_pair,useNA="ifany")
```

```

     imex        tm tm & imex 
   115667    826434       175 
```

```r
comp_simple <- unique(select(comp, pair_id,db_pair))

write.table(comp_simple,"./results/pairs_tm_vs_imex.txt",col.names=T,row.names=F,quote=F,sep="\t")
```

There is some overlap between the datasets when comparing the pairs. I save this file as a temporary output while the issues with the publication count (discussed below) are solved. 

#### Comparison between IMEx and the text-mined dataset at the publication level

Now I will check how many of the publications where the pairs have been reported in the text-mined dataset were curated in IMEx. For that I first need to translate the PMC ids given in the PMCIDs field to PMIDs, in order to be able to compare with the IMEx dataset. 

First I obtain the translation table. 


```r
if(!file.exists("./source_files/PMID_PMCID_DOI.csv.gz")){
  download.file(url="ftp://ftp.ebi.ac.uk/pub/databases/pmc/DOI/PMID_PMCID_DOI.csv.gz", destfile="./source_files/PMID_PMCID_DOI.csv.gz",method="curl")
}
```

```
## Warning in download.file(url = "ftp://ftp.ebi.ac.uk/pub/databases/pmc/DOI/
## PMID_PMCID_DOI.csv.gz", : download had nonzero exit status
```

```r
library("data.table")
system("gunzip ./source_files/PMID_PMCID_DOI.csv.gz")

map_pub <- fread("./source_files/PMID_PMCID_DOI.csv",header=T,sep=",",stringsAsFactors=F)
```

```
## Error in fread("./source_files/PMID_PMCID_DOI.csv", header = T, sep = ",", : File './source_files/PMID_PMCID_DOI.csv' does not exist. Include one or more spaces to consider the input a system command.
```

```r
map_pub_sel <- unique(select(map_pub,PMID,PMCID))
```

```
## Error in select_(.data, .dots = lazyeval::lazy_dots(...)): object 'map_pub' not found
```

```r
rm(map_pub)
```

```
## Warning in rm(map_pub): object 'map_pub' not found
```

```r
system("rm ./source_files/PMID_PMCID_DOI.csv")
```


```r
tm_pmcids <- unique(select(tm_full,PMCIDs))
tm_pmcids2pmids <- merge(tm_pmcids,map_pub_sel,by.x="PMCIDs",by.y="PMCID",all.x=T,all.y=F)
```

```
## Error in as.data.frame(y): object 'map_pub_sel' not found
```

```r
tm_pmcids2pmids <- mutate(tm_pmcids2pmids, PMID =
                 ifelse(is.na(PMID), PMCIDs,
                 PMID))
```

```
## Error in mutate_(.data, .dots = lazyeval::lazy_dots(...)): object 'tm_pmcids2pmids' not found
```

```r
rm(map_pub_sel)
```

```
## Warning in rm(map_pub_sel): object 'map_pub_sel' not found
```

Now I map the pairs to the cleaned-up PMIDs, so can generate a file that can be compared to other datasets. 


```r
tm_full_map <- unique(merge(tm_full,tm_pmcids2pmids,by="PMCIDs",all.x=T,all.y=F))
```

```
## Error in as.data.frame(y): object 'tm_pmcids2pmids' not found
```

```r
tm_sel <- unique(select(tm_full_map,pair_id,pmid=PMCIDs,tm,Nof.Docs,Nof.co.occr.in.Title.Abs,Nof.co.occr.in.Body))
```

```
## Error in select_(.data, .dots = lazyeval::lazy_dots(...)): object 'tm_full_map' not found
```

```r
write.table(tm_sel,"./results/pairs_pmids_tm.txt",col.names=T,row.names=F,quote=F,sep="\t")
```

```
## Error in is.data.frame(x): object 'tm_sel' not found
```

```r
system("gzip ./results/pairs_pmids_tm.txt")
```

### Part 3: Evaluate occurence of a pair as interaction predictor

Given the limited overlap, I have a quick look to check if the different parameters counted by SK in the EPMC dataset correlate somehow with their presence in the IMEx dataset. 


```r
tm_vs_imex <- unique(subset(comp, db_pair == "tm" | db_pair == "tm & imex"))

g3 <- ggplot(data=tm_vs_imex, aes(x=db_pair,y=Nof.Docs))
g3 <- g3 + geom_violin(aes(fill=db_pair))
g3 <- g3 + scale_fill_manual(values=c("#56B4E9","#E69F00"),
                             labels=c("Found in IMEx", "text-mined only"))
g3 <- g3 + scale_y_log10(breaks=c(10,100,1000,10000),
                         labels=function(n){format(n, scientific = FALSE)})
g3 <- g3 + labs(title ="Number of occurrences in EPMC in the IMEx and text-mined groups", y = "Number of occurrences (log scale)", x="group")
g3 <- g3 + geom_hline(yintercept=median(tm_vs_imex[tm_vs_imex$db_pair == "tm",2]), colour="#56B4E9", linetype="dashed")
g3 <- g3 + geom_hline(yintercept=median(tm_vs_imex[tm_vs_imex$db_pair == "tm & imex",2]), colour="#E69F00", linetype="dashed")
g3 <- g3 + annotate("text", x=0.75, median(tm_vs_imex[tm_vs_imex$db_pair == "tm",2])+0.5, label = "EPMC-only\nmedian", colour="#56B4E9")
g3 <- g3 + annotate("text", x=1.75, median(tm_vs_imex[tm_vs_imex$db_pair == "tm & imex",2])+0.7, label = "Found in IMEx\nmedian", colour="#E69F00")
g3 <- g3 + theme(legend.position="none")
g3
```

![plot of chunk unnamed-chunk-13](figure/unnamed-chunk-13-1.png)

Now I check the statistical significance of the difference in the number of publications between the set of predicted pairs found in IMEx and the rest. 

The data is clearly non-parametric, I compare if the variances of the two sets are at least similar to decide which test to use. I use the [Bartlett test of homogeneity of variances](https://en.wikipedia.org/wiki/Bartlett's_test). 


```r
bartlett.test(tm_vs_imex[,2], tm_vs_imex[,8])
```

```

	Bartlett test of homogeneity of variances

data:  tm_vs_imex[, 2] and tm_vs_imex[, 8]
Bartlett's K-squared = Inf, df = 5, p-value < 2.2e-16
```
The null hypothesis of homogeneous variances is rejected, so I decide to use the [Mood test](https://en.wikipedia.org/wiki/Median_test) for comparison of the median of two populations. I have to do the test with a sample of 20000 elements of the text0mined subset, since the fully-sized test throws an error and does not give any results. 


```r
nocs_tm <- sample(tm_vs_imex[tm_vs_imex$db_pair == "tm",2], 20000)
nocs_both <- sample(tm_vs_imex[tm_vs_imex$db_pair == "tm & imex",2])

mood.test(nocs_both,nocs_tm,alternative="two.sided")
```

```

	Mood two-sample test of scale

data:  nocs_both and nocs_tm
Z = 12.882, p-value < 2.2e-16
alternative hypothesis: two.sided
```

I reject the null hypothesis of identical medians. It seems that pairs represented in IMEx tend to be found in a higher number of publications, we can probably use the number of publications as a criterium to decide which pairs to explore first. 



************************************************************

## Appendix: Keywords describing interaction relationships

 - (de)acetylate
 - (co)activate
 - transactivate 
 - (dis)associate 
 - add ?
 - bind
 - link ?
 - catalyse
 - cleave
 - co(-)immunoprecipitate, co(-)ip
 - (de)methylate
 - (de)phosphorylate
 - (de)phosphorylase
 - produce ?
 - modify
 - impair 
 - inactivate 
 - inhibit 
 - interact
 - react
 - (dis)assemble
 - discharge ?
 - modulate 
 - stimulate
 - substitute ?
 - (de)ubiquitinate 
 - heterodimerize
 - heterotrimerize 
 - immunoprecipitate
 - (co)assemble
 - co-crystal
 - complex
 - copurifies
 - cross-link
 - two(-)hybrid, 2(-)hybrid, 2(-)H, yeast two(-) hybrid, yeast 2(-) hybrid), Y(-)2H, classical two(-)hybrid, classical 2(-) hybrid, Gal4 transcription regeneration
 - cosediment
 - comigrate
 - AP(-)MS
 - TAP-MS
 - Homo (dimer/trimer/tetramer…)
 - Oligomerize

********************************************************

## Appendix 2: Extra code

#### Multiplot function


```r
# Multiple plot function
#
# ggplot objects can be passed in ..., or to plotlist (as a list of ggplot objects)
# - cols:   Number of columns in layout
# - layout: A matrix specifying the layout. If present, 'cols' is ignored.
#
# If the layout is something like matrix(c(1,2,3,3), nrow=2, byrow=TRUE),
# then plot 1 will go in the upper left, 2 will go in the upper right, and
# 3 will go all the way across the bottom.
#
multiplot <- function(..., plotlist=NULL, file, cols=1, layout=NULL) {
  library(grid)

  # Make a list from the ... arguments and plotlist
  plots <- c(list(...), plotlist)

  numPlots = length(plots)

  # If layout is NULL, then use 'cols' to determine layout
  if (is.null(layout)) {
    # Make the panel
    # ncol: Number of columns of plots
    # nrow: Number of rows needed, calculated from # of cols
    layout <- matrix(seq(1, cols * ceiling(numPlots/cols)),
                    ncol = cols, nrow = ceiling(numPlots/cols))
  }

 if (numPlots==1) {
    print(plots[[1]])

  } else {
    # Set up the page
    grid.newpage()
    pushViewport(viewport(layout = grid.layout(nrow(layout), ncol(layout))))

    # Make each plot, in the correct location
    for (i in 1:numPlots) {
      # Get the i,j matrix positions of the regions that contain this subplot
      matchidx <- as.data.frame(which(layout == i, arr.ind = TRUE))

      print(plots[[i]], vp = viewport(layout.pos.row = matchidx$row,
                                      layout.pos.col = matchidx$col))
    }
  }
}
```