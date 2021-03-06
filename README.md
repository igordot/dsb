
<!-- README.md is generated from README.Rmd. Please edit that file -->

## dsb <a href='https://mattpm.github.io/dsb'><img src='man/figures/logo.png' align="right" height="150" /></a>

## An R package for normalizing and denoising CITEseq data

<!-- badges: start -->

<!-- [![Travis build status](https://travis-ci.org/MattPM/dsb.svg?branch=master)](https://travis-ci.org/MattPM/dsb) -->

<!-- badges: end -->

**please see vignettes in the “articles” tab at
<https://mattpm.github.io/dsb/> for a detailed workflow describing
reading in proper cellranger output and using the DSB normalizaiton
method**

[LINK TO FULL
VIGNETTE](https://mattpm.github.io/dsb/articles/dsb_normalizing_CITEseq_data.html)

This package was developed at [John Tsang’s
Lab](https://www.niaid.nih.gov/research/john-tsang-phd) by Matt Mulè,
Andrew Martins and John Tsang. The package implements our normalization
and denoising method for CITEseq data. The details of the method can be
found in [the biorxiv
preprint](https://www.biorxiv.org/content/10.1101/2020.02.24.963603v1.full.pdf)
We utilized the dsb package to normalize CITEseq data reported in [this
paper](https://doi.org/10.1038/s41591-020-0769-8).

As described in [the biorxiv
preprint](https://www.biorxiv.org/content/10.1101/2020.02.24.963603v1.full.pdf)
comparing unstained control cells and empty droplets we found that a
major contributor to background noise in protein expression data is
unbound antibodies captured and sequenced in droplets. DSB corrects for
this background by leveraging empty droplets, which serve as a “built
in” noise measurement in droplet capture single cell experiments
(e.g. 10X, dropseq, indrop). In addition, we define a per-cell
denoising covariate to account for several potential sources of
technical differences among single cells – see our preprint for details.

## installation

You can install the released version of dsb in your R session with the
command
below

``` r
# this is analagous to install.packages("package), you need the package devtools to install a package from a github repository like this one. 
# require(devtools)
# devtools::install_github(repo = 'MattPM/dsb')
```

## Quickstart - removing background as captured by data from empty droplets

``` r
# load package and normalize the example raw data 
library(dsb)
# normalize
normalized_matrix = DSBNormalizeProtein(cell_protein_matrix = cells_citeseq_mtx,
                                        empty_drop_matrix = empty_drop_citeseq_mtx)
```

## The full version (recommended) – removing background as above and correcting for per-cell technical factor as a covariate

**This is a quick summary Please see the detailed workflow vignette**

By default, dsb defines the per-cell technical covariate by fitting a
two-component gaussian mixture model to the log + 10 counts (of all
proteins) within each cell and defining the covariate as the mean of the
“negative” component. We recommend also to use the counts from the
isotype controls in each cell to compute the denoising covariate
(defined as the first principal component of the isotype control counts
and the “negative” count inferred by the mixture model above.)

``` r

# get the empty cells from demultiplexing with 


# define a vector of the isotype controls in the data 
isotypes = c("Mouse IgG2bkIsotype_PROT", "MouseIgG1kappaisotype_PROT","MouseIgG2akappaisotype_PROT", "RatIgG2bkIsotype_PROT")

normalized_matrix = DSBNormalizeProtein(cell_protein_matrix = cells_citeseq_mtx,
                                        empty_drop_matrix = empty_drop_citeseq_mtx,
                                        use.isotype.control = TRUE,
                                        isotype.control.name.vec = isotypes)
```

## Example: Visualize the distributions of CD4 and CD8

plot the DSB normalized CITEseq data.

**Note, there is NO jitter added to these points for visualization;
these are the unmodified normalized
counts**

``` r
# add a density gradient on the points () this is helpful when there are many thousands of cells )
# this density function is from this blog post: https://slowkow.com/notes/ggplot2-color-by-density/
get_density = function(x, y, ...) {
  dens <- MASS::kde2d(x, y, ...)
  ix <- findInterval(x, dens$x)
  iy <- findInterval(y, dens$y)
  ii <- cbind(ix, iy)
  return(dens$z[ii])
}

library(ggplot2)
data.plot = normalized_matrix %>% t %>%
  as.data.frame() %>% 
  dplyr::select(CD4_PROT, CD8_PROT, CD27_PROT, CD19_PROT) 



data.plot = data.plot %>% dplyr::mutate(density = get_density(data.plot$CD4_PROT, data.plot$CD8_PROT, n = 100)) 
p1 = ggplot(data.plot, aes(x = CD8_PROT, y = CD4_PROT, color = density)) +
  geom_point(size = 0.4) + theme_bw() + ggtitle("small example dataset") +
  geom_vline(xintercept = 0, color = "red", linetype = 2) + 
  geom_hline(yintercept = 0, color = "red", linetype = 2) + 
  theme(axis.text = element_text(face = "bold",size = 12)) +
  viridis::scale_color_viridis(option = "B") +  
  scale_shape_identity() 


data.plot = data.plot %>% dplyr::mutate(density = get_density(data.plot$CD19_PROT, data.plot$CD27_PROT, n = 100)) 
p2 = ggplot(data.plot, aes(x = CD19_PROT, y = CD27_PROT, color = density)) +
  geom_point(size = 0.4) + theme_bw() + 
  geom_vline(xintercept = 0, color = "red", linetype = 2) + 
  geom_hline(yintercept = 0, color = "red", linetype = 2) + 
  theme(axis.text = element_text(face = "bold",size = 12)) +
  viridis::scale_color_viridis(option = "B") +  
  scale_shape_identity() 

cowplot::plot_grid(p1,p2)
```

<img src="man/figures/README-unnamed-chunk-4-1.png" width="100%" />

## How do I get the empty droplets?

**This is covered more extensively in the detailed workflow vignette**

There are a number of ways to get the empty drops. If you are using cell
hashing, when you demultiplex the cells, you get a vector of empty or
Negative droplets.

HTODemux function in Seurat:
<https://satijalab.org/seurat/v3.1/hashing_vignette.html>

deMULTIplex function from Multiseq (this is now also implemented in
Seurat). <https://github.com/chris-mcginnis-ucsf/MULTI-seq>

In practice, you would want to confirm that the cells called as
“negative” indeed have low RNA / gene content to be certain that there
are no contaminating cells. Also, we recommend hash demultiplexing with
the *raw* output from cellranger rather than the processed output
(i.e. outs/raw\_feature\_bc\_matrix). This output contains all barcodes
and will have more empty droplets from which the HTODemux function will
be able to estimate the negative distribution. This will also have the
benefit of creating more empty droplets to use as built-in protein
background controls in the DSB function. **please see vignettes in the
“articles” tab at <https://mattpm.github.io/dsb/> for a detailed
workflow detailing these steps**

## Simple example workflow (Seurat Version 3)

``` r

# get the ADT counts using Seurat version 3 
seurat_object = HTODemux(seurat_object, assay = "HTO", positive.quantile = 0.99)
Idents(seurat_object) = "HTO_classification.global"
neg_object = subset(seurat_object, idents = "Negative")
singlet_object = subset(seurat_object, idents = "Singlet")


# non sparse CITEseq data actually store better in a regular materix so the as.matrix() call is not memory intensive.
neg_adt_matrix = GetAssayData(neg_object, assay = "CITE", slot = 'counts') %>% as.matrix()
positive_adt_matrix = GetAssayData(singlet_object, assay = "CITE", slot = 'counts') %>% as.matrix()


# normalize the data with dsb
# make sure you've run devtools::install_github(repo = 'MattPM/dsb')
normalized_matrix = DSBNormalizeProtein(cell_protein_matrix = positive_adt_matrix,
                                        empty_drop_matrix = neg_adt_matrix)


# now add the normalized dat back to the object (the singlets defined above as "object")
singlet_object = SetAssayData(object = singlet_object, slot = "CITE", new.data = normalized_matrix)
```

## example workflow Seurat version 2

``` r

# get the ADT counts using Seurat version 3 
seurat_object = HTODemux(seurat_object, assay = "HTO", positive.quantile = 0.99)

neg = seurat_object %>%
  SetAllIdent(id = "hto_classification_global") %>% 
  SubsetData(ident.use = "Negative") 

singlet = seurat_object %>%
  SetAllIdent(id = "hto_classification_global") %>% 
  SubsetData(ident.use = "Singlet") 

# get negative and positive ADT data 
neg_adt_matrix = neg@assay$CITE@raw.data %>% as.matrix()
pos_adt_matrix = singlet@assay$CITE@raw.data %>% as.matrix()


# normalize the data with dsb
# make sure you've run devtools::install_github(repo = 'MattPM/dsb')
normalized_matrix = DSBNormalizeProtein(cell_protein_matrix = pos_adt_matrix,
                                        empty_drop_matrix = neg_adt_matrix)


# add the assay to the Seurat object 
singlet = SetAssayData(object = singlet, slot = "CITE", new.data = normalized_matrix)
```

How to get empty droplets without cell hashing or sample demultiplexing?

If you didn’t run a multiplexing experiment you can simply get a vector
of negative droplets from the droplets that ould be QCd out of the
experiment due to very low mRNA counts as an estimation of droplets that
contain only ambient loading buffer and no cells. There is also an
excellent R package for
this.  
<https://genomebiology.biomedcentral.com/articles/10.1186/s13059-019-1662-y>

**please see vignettes in the “articles” tab at
<https://mattpm.github.io/dsb/> for a detailed workflow describing
reading in proper cellranger output** There robust ways to estimate
which cells are empty droplets:
<https://genomebiology.biomedcentral.com/articles/10.1186/s13059-019-1662-y>

Below is a quick method to get outlier empty droplets assuming
seurat\_object is a object with most cells (i.e. any cell expressing at
least a gene).

Get the nUMI from a seurat version 3 object

``` r
# get the nUMI from a seurat version 3 object 
umi = seurat_object$nUMI
```

Get the nUMI from a Seurat version 2 object

``` r
#  Get the nUMI from a Seurat version 2 object
umi = seurat_object@meta.data %>% select("nUMI")
```

``` r
mu_umi = mean(umi)
sd_umi = sd(umi)

# calculate a threshold for calling a cell negative 
sub_threshold = mu_umi - (5*sd_umi)

# define the negative cell object
Idents(seurat_object) = "nUMI"
neg = subset(seurat_object, accept.high = sub_threshold)
```

This negative cell object can be used to define the negative background
following the examples above.

**Please see the detailed workflow vignette for a full workflow and more
details**
