statistical analysis of proteomics data and visualization
================
BS
18/02/2023

## load libraries

``` r
library(tidyverse)
library(missForest)
library(ggrepel)
library(ComplexHeatmap)
library(msigdbr)
library(circlize)
library(cowplot)
library(WebGestaltR)
library(ggpubr)
library(xlsx)
library(pepquantify)
library(seqinr)
library(biomartr)
library(protti)
```

## Convert precursor data to protein and peptide data (pepquantify package)

### read the main output of DIA-NN

implicit reading is necessary only if any modifications other than
supported by pepquantify package is necessary, e.g. filtering for
contaminants

``` r
raw_diann <- read.delim("Liver_DIA_precursors_piglets_3d_old.tsv", sep = "\t", header = T)
```

### remove contaminants (contaminants fasta file from MaxQuant common contaminants)

``` r
contamintants           <- read.fasta("contaminants.fasta")
contaminant.names       <- getName(contamintants) 
raw_diann_filtered      <- raw_diann %>% 
  filter(!str_detect(Protein.Group, str_c(contaminant.names, collapse="|")))

# save contaminants-removed data
write.table(raw_diann_filtered, "DIA_precursors_nocontaminants.tsv", quote = F, sep = "\t", row.names = F)
```

### load data using pepquantify (performs filtering and writes filtered output, peptides and protein groups files)

``` r
precursors_diann_3d_old <- pepquantify::read_diann(Q_Val = 0.01, 
                                                   diann_file_name = "DIA_precursors_nocontaminants.tsv",
                                                   Global_Q_Val = 0.01, 
                                                   Global_PG_Q_Val = 0.01, 
                                                   experimental_library = T, 
                                                   unique_peptides_only = TRUE, 
                                                   Quant_Qual = 0.5, 
                                                   include_mod_in_pepreport = T,
                                                   save_supplementary = T,
                                                   for_msempire = F, 
                                                   id_column = "Genes",
                                                   second_id_column = "Protein.Group", 
                                                   quantity_column = "Genes.MaxLFQ.Unique")
```

    ## [1] "R is located in C:/Users/shashikadze/Documents/GitHub/maternaldiabetes-offspring-liver-omics-paper/proteomics, if this path is wrong, change it from R studio, or by specifying the correct path using the directory command e.g. directory = name_of_the_path"
    ## [1] "file, with the name DIA_precursors_nocontaminants.tsv is currently loading"
    ## [1] "in the peptide output modifications will be included (so far (v.2.1.2) only works for Carbamidomethyl(C))"

    ## Joining, by = "Genes"
    ## Joining, by = "Stripped.Sequence"
    ## Joining, by = "Stripped.Sequence"
    ## Joining, by = "Stripped.Sequence"
    ## Joining, by = "Stripped.Sequence"
    ## Joining, by = "Genes"
    ## Joining, by = "Genes"

    ## the following files were saved in the txt folder:
    ##     1 - diann_output_filtered;
    ##     2 - peptides;
    ##     3 - proteingroups.

``` r
rm(raw_diann, raw_diann_filtered)
```

## calculate sequence coverage using protti R package

``` r
# read uniprot fasta file which was used during DIA-NN search
fasta_file <- read_proteome("uniprot-proteome_UP000008227_20012022_49792_1438.fasta", obj.type = "Biostrings")

# convert fasta file to a dataframe
fasta_df <- as.data.frame(fasta_file) %>% 
            tibble::rownames_to_column("name") %>% 
            dplyr::rename(sequence = x)
rm(fasta_file)

# list of identified proteins (only first accession)
seq_cov_data <- precursors_diann_3d_old[[1]] %>%
  select(Stripped.Sequence, Protein.Group) %>% 
  mutate(first_protein = str_remove(Protein.Group, ";.*")) 

# prepare fasta file to match proteins and get the sequence 
# protein name in uniprot fasta file is between two vertical lines, following extracts proteins in a new column from each entry
fasta_df <- fasta_df %>%
   mutate(protein = str_extract_all(name,"(?<=\\|).+(?=\\|)")) %>% 
   mutate(protein = as.character(protein))

# add full protein sequence to each identified protein 
seq_cov_data <- seq_cov_data %>% 
  left_join(fasta_df %>% select(sequence, protein), by = c("first_protein" = "protein")) %>% 
  select(Stripped.Sequence, sequence)

# calculate sequence coverage using protti package
seq_cov <- calculate_sequence_coverage(
  seq_cov_data,
  protein_sequence = sequence,
  peptides = Stripped.Sequence)

# retain max seq coverage for each gene
seq_cov <- seq_cov %>% 
  left_join(precursors_diann_3d_old[[1]] %>% select(Stripped.Sequence, Protein.Group, Genes)) %>% 
  select(coverage, Genes) %>% 
  group_by(Genes) %>%
  summarise(coverage = max(coverage))
```

    ## Joining, by = "Stripped.Sequence"

## load conditions file

``` r
conditions <- read.delim("Conditions.txt", sep = "\t", header = T)
```

## missing value imputation (random forest)

### data preparation

#### reoder columns (consistent with other data)

also prepare suppl table

``` r
# protein groups data
proteingroups  <- precursors_diann_3d_old[[2]] 

# reorder, first Genes, afterwards samples (from PNG -> PHG as in conditions file) finally additional columns
proteingroups_reordered <- proteingroups[c("Genes", str_c("LFQ.intensity_", conditions$Bioreplicate), colnames(proteingroups)[(nrow(conditions)+2):ncol(proteingroups)])]

# prepare protein groups for the suppl table and save
proteingroups_reordered_suppl <- proteingroups_reordered %>% 
  mutate(Intensity = rowSums(select(., starts_with("LFQ.")), na.rm = T)) %>% 
  arrange(desc(Intensity)) %>% 
  left_join(seq_cov) %>% 
  mutate(coverage = round(coverage, 2)) %>% 
  select(Genes, First.Protein.Description, second_ids, coverage, starts_with("LFQ."), n_pep, pg_Q_Val) %>% 
  rename("Protein names"   = First.Protein.Description,
         "Protein groups"  = second_ids,
         "Unique peptides" = n_pep,
         "Unique sequence coverage %" = coverage,
         "q-value"         = pg_Q_Val) %>%
          write.table("proteingroups.txt", sep = "\t", row.names = F, quote = F)
```

    ## Joining, by = "Genes"

#### count percentage of missing values

``` r
# data numeric
proteingroups  <- proteingroups_reordered %>% 
  column_to_rownames("Genes") %>% 
  select(starts_with("LFQ.")) %>% 
  mutate_all(log2)

# count missingness
n_total   <- nrow(proteingroups) * ncol(proteingroups)
n_missing <- sum(colSums(is.na(proteingroups)))

perc_missing <- (n_total/n_missing)
cat(paste0(round(perc_missing), " %"), "of the data is missing")
```

    ## 18 % of the data is missing

``` r
rm(n_total, n_missing, perc_missing)
```

#### filter for valid values (keep proteins with at least 60% of valid values)

``` r
proteingroups_filtered <-  proteingroups %>% 
  mutate(n_valid = 100 - (100 * rowSums(is.na(.))/ncol(proteingroups))) %>% 
  filter(n_valid >= 60) %>% 
  select(-n_valid) 

# count percentage of missing values after filtering
n_total_afterfiltering   <- nrow(proteingroups_filtered) * ncol(proteingroups_filtered)
n_missing_afterfiltering <- sum(colSums(is.na(proteingroups_filtered)))

perc_missing_afterfiltering <- (n_missing_afterfiltering/n_total_afterfiltering)
cat(paste0(perc_missing_afterfiltering, " %"), "of the data is missing after filtering")
```

    ## 0.0232595062316425 % of the data is missing after filtering

``` r
rm(n_total_afterfiltering, n_missing_afterfiltering, perc_missing_afterfiltering)
```

### missing value imputation

``` r
set.seed(12345)
system.time(data_imputed_RF <- missForest::missForest(as.matrix(proteingroups_filtered)))
```

    ##    user  system elapsed 
    ## 1564.35    4.34 1569.36

#### save imputed protein groups data

``` r
# clean-up the data
data_imputed <- data_imputed_RF[["ximp"]] %>% 
  as.data.frame() %>% 
  rownames_to_column("Genes") %>% 
  left_join(precursors_diann_3d_old[[2]] %>% select(-starts_with("LFQ."))) 

# save results
write.table(data_imputed, "proteingroups_imputed.txt", sep = "\t", row.names = F)
```

## differential abundance analysis using 2 way ANOVA

### prepare data

``` r
# clean up proteomics data (1. retain only id column and columns containing protein intensities; 2. remove the string LFQ.intensity_ from each column, this is to be able to properly match with conditions file)
data_stat <- data_imputed %>% 
  select(Genes, starts_with("LFQ.")) %>% 
  rename_all(~str_replace_all(., 'LFQ.intensity_', ''))
```

### fold-change calculation

``` r
fc_function <- function(data, conditions_data, condition_name, compared_to, id_name, values_log) {
 
   # should values be log transformed?
  if (values_log == TRUE) {
    data_long <- data %>% 
    pivot_longer(names_to = "Bioreplicate", values_to = "Intensity", -!!as.symbol(id_name)) %>%
    left_join(conditions_data)
  }
  
  if (values_log == FALSE) {
    data_long <- data %>% 
    pivot_longer(names_to = "Bioreplicate", values_to = "Intensity", -!!as.symbol(id_name)) %>%
    mutate(Intensity = log2(Intensity)) %>% 
    left_join(conditions_data)
  }
  
  #fold-change calculation    
    data_fc <- data_long %>% 
    group_by(!!as.symbol(id_name), !!as.symbol(condition_name)) %>% 
    summarise(grp_mean = mean(Intensity, na.rm=T)) %>% 
    ungroup() 
  
    
  if (compared_to %in% unique(data_fc[[2]][2])) {
    data_fc <- data_fc %>% 
     select(-all_of(condition_name)) %>% 
     group_by(!!as.symbol(id_name)) %>% 
     mutate(l2fc=grp_mean-lag(grp_mean)) %>% 
     select(-grp_mean)  %>% 
     drop_na()
    
     }
  
   else{
    data_fc <- data_fc  %>% 
    select(-all_of(condition_name)) %>%  
    group_by(!!as.symbol(id_name))  %>% 
    mutate(l2fc=grp_mean-lead(grp_mean)) %>% 
    select(-grp_mean)   %>% 
    drop_na()
     }
    
    cat(paste("positive fold change means up in", compared_to, sep=" ")) 
    
    return(data_fc)
    
}


# calculate fold-change separately for group and sex
l2fc_group <- fc_function(data_stat, condition = "Group", conditions_data = conditions,
                          compared_to  = "PHG", values_log= T, id_name = "Genes")
```

    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Genes'. You can override using the
    ## `.groups` argument.

    ## positive fold change means up in PHG

``` r
l2fc_sex <- fc_function(data_stat, condition = "Sex", conditions_data = conditions,
                          compared_to  = "F", values_log= T, id_name = "Genes")
```

    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Genes'. You can override using the
    ## `.groups` argument.

    ## positive fold change means up in F

``` r
# merge fold-changes
l2fc_genes <- l2fc_group %>% 
  rename("l2fc group (PHG/PNG)" = l2fc) %>% 
  left_join(l2fc_sex) %>% 
  rename("l2fc sex (F/M)" = l2fc) 
```

    ## Joining, by = "Genes"

### 2 way anova

#### perform anova analysis

##### define functions that performs 2 way anova

``` r
two_way_anova_fn <- function(data, id_name, conditions_file, adjust_p_value, p_adj_method, l2fc_data, add_l2fc) {
  
  data_anova  <- data %>% 
          pivot_longer(names_to = "Bioreplicate", 
          values_to = "Value", -all_of(id_name)) %>% 
          left_join(conditions_file) %>% 
          drop_na(Value) %>% 
          group_by(!!as.symbol(id_name)) %>% 
          summarise(`p-value` = 
          summary(aov(Value ~ Group*Sex))[[1]][["Pr(>F)"]][1:3]) 
  
  # correct all resulting p-values (pool) for multiple hypothesis testing
  if (adjust_p_value == TRUE) {
    
  data_anova$`Adjusted p-value`  <- p.adjust(data_anova$`p-value`, method = p_adj_method)
  # prepare empty data frame with proper comparisons
  anova_factors <- as.data.frame(rep(c("group (PHG/PNG)", "sex (F/M)", "group:sex"), length = nrow(data_anova)))
  # rename column 
  names(anova_factors) <- "Comparison"
  
  # final anova results
  anova_results  <- as.data.frame(cbind(data_anova, anova_factors)) %>% 
  pivot_wider(names_from = Comparison, 
                          values_from = c(`p-value`, `Adjusted p-value`), all_of(id_name), names_sep = " ") 
    
  }
  
  if (adjust_p_value == FALSE) {
  anova_factors <- as.data.frame(rep(c("p-value group (PHG/PNG)", "p-value sex (F/M)", 
                                       "p-value group:sex"), length = nrow(data_anova)))
  # rename column 
  names(anova_factors) <- "Comparison"
  
  # final anova results
  anova_results  <- as.data.frame(cbind(data_anova, anova_factors)) %>% 
  pivot_wider(names_from = Comparison, 
                          values_from = `p-value`, all_of(id_name), names_sep = " ") 
    
  }
  # add fold changes
  if (add_l2fc == TRUE) {
    anova_results  <- anova_results  %>% 
       left_join(l2fc_data)
  }
  return(anova_results)
  }
```

##### significance of protein changes with 2 way anova

``` r
anova_results <- two_way_anova_fn(data = data_stat, id_name = "Genes",                                  
                                  adjust_p_value = T,
                                  p_adj_method = "BH",
                                  add_l2fc = T,
                                  conditions_file = conditions, 
                                  l2fc=l2fc_genes)
```

    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Genes'. You can override using the
    ## `.groups` argument.
    ## Joining, by = "Genes"

#### clean-up anova results

``` r
# final anova results
anova_results  <- anova_results %>% 
  left_join(precursors_diann_3d_old[[2]] %>% select(Genes, First.Protein.Description, second_ids)) %>% 
  select("Genes", "First.Protein.Description", "second_ids", "l2fc group (PHG/PNG)", "p-value group (PHG/PNG)", 
         "Adjusted p-value group (PHG/PNG)", "l2fc sex (F/M)", "p-value sex (F/M)", 
         "Adjusted p-value sex (F/M)", "p-value group:sex", "Adjusted p-value group:sex") %>% 
  arrange(-desc(`Adjusted p-value group (PHG/PNG)`)) %>% 
  rename(`Protein names` = First.Protein.Description,
         `Protein group` = second_ids) 


# separately significant factors and interactions

# significant by group
anova_results_group <- anova_results %>% 
  select(1:6) %>% 
  filter(`Adjusted p-value group (PHG/PNG)` <= 0.05 & abs(`l2fc group (PHG/PNG)`) > log2(1.5)) %>% 
          mutate(`Differentially abundant` = case_when(
                 `l2fc group (PHG/PNG)` > log2(1.5) ~  "increased in PHG",
                 `l2fc group (PHG/PNG)` < -log2(1.5) ~ "decreased in PHG",
             TRUE ~ "n.s."
           )) %>% 
  arrange(desc(`l2fc group (PHG/PNG)`)) 


# significant by sex
anova_results_sex <- anova_results %>% 
  select(1:3, 7:9) %>% 
  filter(`Adjusted p-value sex (F/M)` <= 0.05 & abs(`l2fc sex (F/M)`) > log2(1.5)) %>% 
          mutate(`Differentially abundant` = case_when(
                 `l2fc sex (F/M)` > log2(1.5) ~  "increased in F",
                 `l2fc sex (F/M)` < -log2(1.5) ~ "decreased in F",
             TRUE ~ "n.s."
           )) %>% 
  arrange(desc(`l2fc sex (F/M)`)) 


# interaction
anova_results_int <- anova_results %>% 
  select(1:3, 10,11) %>% 
  filter(`Adjusted p-value group:sex` <= 0.05) %>% 
  arrange(desc(`Adjusted p-value group:sex`))
```

#### define functions that performs 2 way anova Tukey’s honest significance difference (interaction significant proteins)

``` r
tkhsd_fn <- function(data, id_name, conditions_file, numeric_data) {
  
  # prepare data
  data_tukey <- data %>% 
              select(all_of(id_name)) %>%
              left_join(data_stat) %>% 
              pivot_longer(names_to = "Bioreplicate", values_to = "Intensity", 
                           -all_of(id_name)) %>% 
              left_join(conditions_file) %>% 
    drop_na(Intensity)
  
  # for loop. for each feature which showed significant interaction from
  # 2 way anova, THSD is calculated. significant pairs are extracted
  # make an empty list, where to each feature significant interactions will be assigned
 
   my_vec <- list()
  
  # for which feature
  
  sig_hits <- unique(data_tukey[[id_name]])
  
  # for loop
  for (i in sig_hits) {
    sub_df              <- data_tukey[data_tukey[[id_name]] %in% i,]
    anova_model         <- aov(data= sub_df,  Intensity ~ Group*Sex)
    anova_tukey         <- TukeyHSD(anova_model)
    tuk_interactions    <- anova_tukey[3][[1]] %>% 
    as.data.frame() %>% 
    rownames_to_column() %>% 
    filter(`p adj` < 0.05) %>% 
    select(rowname)
    tuk_interactions_t  <- t(tuk_interactions)
    sig_tuk_interaction <- matrix(apply(tuk_interactions_t,1,paste,collapse=";"),
                                nrow=1)
    my_vec[i] <- sig_tuk_interaction
  }
 
  # final data with anova and THSD statistics
  tkhsd <- list()
  tkhsd[[1]] <- anova_results_int %>% 
               left_join(my_vec %>% 
               unlist() %>% 
               as.data.frame() %>% 
               rownames_to_column("parameter") %>% 
               rename_all(~str_replace(., "parameter", id_name)) %>% 
               rename(`THSD pair` = 2))
  
  
  # data for interaction plot
  tkhsd[[2]] <- numeric_data %>%  
  left_join(tkhsd[[1]]) %>% 
  drop_na(`THSD pair`) %>% 
  select(1:ncol(data_stat)) %>% 
  pivot_longer(values_to = "Intensity", names_to = "Bioreplicate", -all_of(id_name)) %>% 
  left_join(conditions_file) %>% 
  mutate(joinedgr = str_c(Group, Sex, sep = "_")) %>% 
  group_by(!!as.symbol(id_name), joinedgr) %>% 
  summarise(mean = mean(Intensity, na.rm = T), 
            sd = sd(Intensity, na.rm = T), t.score = qt(p=0.05/2, 
            df=length(conditions_file$Bioreplicate),lower.tail=F), 
            ci = t.score * sd ) %>% 
  ungroup() %>% 
  separate(joinedgr, c("Group", "Sex")) 
  
  return(tkhsd)
}
```

##### THSD of protein interactions

``` r
anova_results_int_tuk <- tkhsd_fn(data = anova_results_int,  id_name = "Genes", 
                                  numeric_data = data_stat, 
                                  conditions_file = conditions) 
```

    ## Joining, by = "Genes"
    ## Joining, by = "Bioreplicate"
    ## Joining, by = "Genes"
    ## Joining, by = "Genes"
    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Genes'. You can override using the
    ## `.groups` argument.

## volcano plot

### prepare data for the volcano plot

``` r
data_volcano <- anova_results %>% 
                mutate(significant = case_when(
                `Adjusted p-value group (PHG/PNG)` < 0.05 & `l2fc group (PHG/PNG)` > log2(1.5) | `Adjusted p-value group (PHG/PNG)` < 0.05 & 
                `l2fc group (PHG/PNG)` < -log2(1.5) ~ "+", TRUE ~ "n.s."),
                diff_abundant = case_when(
                `Adjusted p-value group (PHG/PNG)` < 0.05 & `l2fc group (PHG/PNG)` > log2(1.5) ~ "Increased_in_PHG",
                `Adjusted p-value group (PHG/PNG)` < 0.05 & `l2fc group (PHG/PNG)` < -log2(1.5) ~ "Decreased_in_PHG",
             TRUE ~ "n.s."
           )) 
```

### plot volcano plot

``` r
plot_volcano <- ggplot(data_volcano %>%                       
                       arrange(desc(diff_abundant)), 
                       mapping = aes(x = `l2fc group (PHG/PNG)`, y = -log10(`p-value group (PHG/PNG)`), 
                                     fill=diff_abundant, label = Genes, 
                                     alpha = diff_abundant))+
         geom_point(aes(shape =diff_abundant, size = diff_abundant), stroke = 0.25)+
         scale_shape_manual(values = c(n.s. = 16, Decreased_in_PHG =21, Increased_in_PHG =21))+
         scale_size_manual(values=c(n.s. = 1, Decreased_in_PHG =1.6, Increased_in_PHG =1.6))+
         scale_fill_manual(values=c("n.s." = "#999999", 
                                    "Decreased_in_PHG"= "#0088AA", "Increased_in_PHG"="#e95559ff"))+
         scale_alpha_manual(values= c("n.s." = 0.3, "Decreased_in_PHG"= 1, "Increased_in_PHG"= 1))+
         geom_text_repel(data = subset(data_volcano, 
                                       significant == "+" & `l2fc group (PHG/PNG)` > 0.6 | significant == "+"  & `l2fc group (PHG/PNG)` < -0.7),
                                       aes(label = Genes),
                                       size = 1.8,
                                       seed = 1234,
                                       color = "black",
                                       box.padding = 0.3,
                                       max.overlaps = 17,
                                       alpha = 1,
                          min.segment.length = 0)+
         theme_bw()+
         scale_x_continuous(limits = c(-2, 2.4)) +
         scale_y_continuous(limits = c(0,8.2), breaks = c(0,2.5,5,7.5)) +
         theme(panel.border = element_rect(linewidth = 1, colour = "black"), 
               panel.grid.major = element_line(), 
               panel.grid.minor = element_blank(),
               panel.background = element_blank(), 
               axis.ticks = element_line(colour = "black"),
               axis.line = element_blank())+
                   theme(legend.position = "none", 
        legend.box.spacing = unit(0.8, 'mm'), 
        legend.title = element_blank(), 
        legend.text = element_blank())+
        theme(axis.title = element_text(size  = 9), 
               axis.text.x = element_text(size = 9, colour = "black", vjust = -0.1), 
               axis.text.y = element_text(size = 9, colour = "black"))+
         xlab("log2 fold change (PHG/PNG)")+
         ylab("-log10 p-value")
```

## plot anova interactions

``` r
# reorder data
anova_results_int_tuk[[2]]$Group <- factor(anova_results_int_tuk[[2]]$Group, levels = c("PNG", "PHG"))


# order of the facets (the most significant first)
order <- anova_results_int %>% arrange(-desc(`Adjusted p-value group:sex`)) %>% 
                       select(Genes)
order <- order$Genes

# plot
ggplot(anova_results_int_tuk[[2]], aes(x=Group, y=mean)) +
  geom_line(size = 0.8, aes(group = Sex, color = Sex))+
  geom_errorbar(aes(ymin=mean-sd, ymax=mean+sd), width=.2,
                 position=position_dodge(0.05))+
  geom_point(aes(color = Sex))+
  scale_color_manual(values = c("#F0E442", "#E69F00"))+
  theme_bw()+
  theme(panel.border = element_rect(linewidth = 1, colour = "black"),
       panel.grid.major = element_blank(), 
       panel.grid.minor = element_blank(), 
       panel.background = element_blank(), 
       axis.line = element_blank(),
  legend.position = "NONE") +
  theme(axis.title = element_text(size = 10), 
        axis.text.x = element_text(size= 9, colour = "black", vjust = -0.1), 
        axis.ticks = element_line(colour = "black"),
        axis.text.y = element_text(size = 9, colour = "black"))+
  theme(legend.position = c(0.6,0.05), legend.title = element_text(size = 8.5), axis.text = element_text(size = 8.5))+
  facet_wrap(~factor(Genes, levels=order), scales = "free_y", ncol = 5) +
  ylab("Protein intensity (log2)")+
  xlab("")+
  geom_text(x = Inf, y = -Inf, 
            aes(label = paste("p=",`Adjusted p-value group:sex`)),  
            size = 2.8, 
            angle = 90,
            hjust = -0.7, 
            vjust = -0.7, 
            data = anova_results_int_tuk[[1]] %>% 
                        mutate(across(where(is.numeric), round, 3)), inherit.aes = F)+
  theme(strip.background = element_blank(), strip.text = element_text())+
  guides(color=guide_legend(nrow=1, byrow=TRUE))
```

    ## Warning: Using `size` aesthetic for lines was deprecated in ggplot2 3.4.0.
    ## ℹ Please use `linewidth` instead.

![](proteomics_files/figure-gfm/unnamed-chunk-19-1.png)<!-- -->

``` r
ggsave("anova_interactions.svg", width = 7.1, height = 8.4)
```

## supervised clustering

### PCA

``` r
# pca function
pca_fn <- function(data, conditions_file, id_name, scale) {
  
          # caclulate principal components of transposed dataframe
          PCA <- prcomp(scale = scale, t(data %>% column_to_rownames(id_name)))
          
          # export results for ggplot
          # according to https://www.youtube.com/watch?v=0Jp4gsfOLMs (StatQuest: PCA in R)
          pca.data <- data.frame(Bioreplicate=rownames(PCA$x),
          X=PCA$x[,1],
          Y=PCA$x[,2])
          pca.var <- PCA$sdev^2
          pca.var.per <- round(pca.var/sum(pca.var)*100,1)
          
          # export loadings
          loadings <- as.data.frame(PCA[["rotation"]]) %>% 
            select(1:2)
          
          # assign conditions
          pca.data <- pca.data %>% 
             left_join(conditions_file) 
          # save the data in a list
          pca_data      <- list()
          pca_data[[1]] <- pca.data
          pca_data[[2]] <- pca.var.per[1:2]
          pca_data[[3]] <- loadings
          
          return(pca_data)
}
```

#### calculate PCs

``` r
data_pca <- pca_fn(data_stat, conditions, id_name = "Genes", scale = F)
```

    ## Joining, by = "Bioreplicate"

#### PCA plot

``` r
plot_pca <- ggplot(data=data_pca[[1]], aes(x=X*-1, y=Y, fill= Group, label = ID))+
geom_point(size = 3, aes(shape = Sex), stroke = 0.25)+
scale_fill_manual(values= c('PHG' = "#e95559ff", 'PNG' = "#0088AA")) +
scale_shape_manual(values = c('F' = 21, 'M' = 22)) + 
xlab(paste("PC 1 - ", data_pca[[2]][1], "%", sep=""))+
ylab(paste("PC 2 - ", data_pca[[2]][2], "%", sep=""))+
geom_hline(yintercept = 0, linetype = "dashed")+
geom_vline(xintercept = 0, linetype = "dashed")+
theme_bw() + theme(panel.border = element_rect(linewidth = 1, colour = "black"), 
                   axis.ticks = element_line(colour = "black"),
                   axis.title = element_text(size = 9, colour="black"), 
                   axis.text.x = element_text(size=9, colour="black", vjust = -0.1), 
                   axis.text.y = element_text(size = 9, colour="black"),
panel.grid.major = element_line(), panel.grid.minor = element_blank())+
theme(legend.title = element_text(colour="black", size=9))+
guides(shape = guide_legend(order = 2, override.aes = list(stroke = 1, shape  = c(0,1))),
       col = guide_legend(order = 1))+
  theme(legend.position = "top", 
        legend.box.spacing = unit(0.8, 'mm'), 
        legend.title = element_blank(), 
        legend.text = element_text(size = 8.5))+
  guides(fill = guide_legend(override.aes=list(shape=21))) #https://github.com/tidyverse/ggplot2/issues/2322
```

### hierarchical clustering

#### data standardization

``` r
hm_prep_fn <- function(data, id_name) {
  
  # data standardization
  cal_z_score <- function(x){
  (x - mean(x)) / sd(x)
  }
 
  data_HM <- as.data.frame(t(apply(data_stat %>% column_to_rownames(id_name), 1, cal_z_score)))

  # make sure that order of colnames matchs to the order of samples in conditions file
  order   <- conditions$Bioreplicate
  data_HM <- data_HM[order]

  # replace long names with short id names
  colnames(data_HM) <- conditions$ID
  
  return(data_HM)
  
}

data_HM <- hm_prep_fn(data = data_stat, id_name = "Genes")
rm(hm_prep_fn)
```

#### ploting heatmap

``` r
#makes a list with all necessary data for heatmap
hmap_all_data      <- list()
hmap_all_data[[1]] <- colorRamp2(c(-1.3, 0, 1.3), c("#0088AA",  "white", "#e95559ff"))
hmap_all_data[[2]] <- list('Group'  = c('PNG' = "#0088AA", 'PHG' = "#e95559ff"),
                           'Sex'     = c('F' = "#F0E442", 'M' = "#E69F00"))
hmap_all_data[[3]] <- HeatmapAnnotation(df = conditions %>% select(ID, Group, Sex) %>% column_to_rownames("ID"), 
                            show_legend    = F, 
                            which          = 'col',
                            annotation_name_gp   = gpar(fontsize = 8),
                            annotation_name_side = "left",
                            col                  = hmap_all_data[[2]],
                            height               = unit(0.7, "cm"),
                            simple_anno_size_adjust = TRUE)
# plotting
hmap <- Heatmap(as.matrix(data_HM),
                show_heatmap_legend        = F,
                row_dend_width             = unit(0.5, "cm"),
                column_dend_height         = unit(0.5, "cm"),
                show_column_names = T,
                show_row_names = F,
                column_names_side = "top",
                row_title = NULL,
                clustering_distance_columns = "euclidean",
                clustering_distance_rows    = "euclidean",
                clustering_method_rows      = "ward.D",
                clustering_method_columns   = "ward.D",
                use_raster = F,
                top_annotation              = hmap_all_data[[3]],
                width                       = unit(3.2, "in"),
                height                      = unit(4.72, "in"), 
                col                         = hmap_all_data[[1]], 
                border                      = TRUE,
                column_names_gp             = gpar(fontsize = 8)) 
ht <- draw(hmap)
```

![](proteomics_files/figure-gfm/unnamed-chunk-23-1.png)<!-- -->

``` r
#calculate actual plot size
w1 = ComplexHeatmap:::width(ht)
w1 = convertX(w1, "inch", valueOnly = TRUE)
h1 = ComplexHeatmap:::height(ht)
h1 = convertY(h1, "inch", valueOnly = TRUE)
c(w1, h1)
```

    ## [1] 3.700355 5.686221

``` r
#for cowplot
plot_hmap = grid.grabExpr(draw(ht))
```

#### ploting legends separatelly

``` r
# color legend
hm_legend = grid.grabExpr(color_mapping_legend(hmap@matrix_color_mapping, plot = T, title = NULL, legend_direction = c("horizontal"),  title_gp = gpar(fontsize = 7, fontface = "bold"), param = list(at = c(-1,  1), labels = c("low", "high"), legend_width = unit(2, "cm"), labels_gp = gpar(fontsize=8)),  labels_gp = gpar(fontsize = 7)))

# legend for the group
Glycemia = grid.grabExpr(color_mapping_legend(hmap@top_annotation@anno_list[["Group"]]@color_mapping, nrow = 1, plot = T,  title_gp = gpar(fontsize = 7, fontface = "bold"),  labels_gp = gpar(fontsize = 7)))

# legend for the sex
Sex = grid.grabExpr(scale = 1, color_mapping_legend(hmap@top_annotation@anno_list[["Sex"]]@color_mapping,  nrow = 1, plot = T, title_gp = gpar(fontsize = 7, fontface = "bold"),  labels_gp = gpar(fontsize = 7)))

# combine legends
lg <- plot_grid(hm_legend, Glycemia, Sex, ncol = 3, rel_widths = c(1,1,1), rel_heights = c(1,1,1))
ggsave("legend.svg", width =2.8, height = 1)
```

## over-representation analysis (ORA)

#### ORA with WebGestalt

### perform ora

``` r
ora_for_cluster_function <- function(data, regulation, fc_threshold, fdr_threshold) {
  
  # regulation string to significance threshold
  if (regulation == "Decreased") {
    
    data_ora <- data %>% 
      filter(`l2fc group (PHG/PNG)` < -log2(fc_threshold) & `Adjusted p-value group (PHG/PNG)` <= fdr_threshold) %>% 
      select(Genes) %>% 
      as.list()
   
  }
  
  if (regulation == "Increased") {
       data_ora <- anova_results %>% 
      filter(`l2fc group (PHG/PNG)` > log2(fc_threshold) & `Adjusted p-value group (PHG/PNG)` <= fdr_threshold) %>% 
      select(Genes) %>% 
      as.list() 
  
       }  

  
  # perform ora
  set.seed(1234)
  outputDirectory <- getwd()
  enrichResult    <- WebGestaltR(enrichMethod="ORA", organism="hsapiens",
  enrichDatabase="geneontology_Biological_Process_noRedundant", interestGene=data_ora[[1]],
  interestGeneType="genesymbol", referenceSet = "genome",
  referenceGeneType="genesymbol", isOutput=TRUE,
  outputDirectory=outputDirectory, projectName=paste0(regulation), sigMethod = "fdr", fdrThr = 0.05, fdrMethod = "BH", minNum = 10, maxNum = 300)
  
  # read ora output
  ora_results <- read.delim(paste0("Project_", regulation, "/", "enrichment_results_", regulation, ".txt"))
  
  # tidy ora output
  ora_results <- ora_results %>% 
    mutate(Regulation = paste(regulation, "in PHG"))
  
  return(ora_results)
}

# apply the function
increased_ora <- ora_for_cluster_function(anova_results, regulation = "Decreased", fc_threshold = 1.3, fdr_threshold = 0.05)
```

    ## Loading the functional categories...
    ## Loading the ID list...
    ## Loading the reference list...
    ## Summarizing the input ID list by GO Slim data...

    ## Performing the enrichment analysis...
    ## Begin affinity propagation...
    ## End affinity propagation...
    ## Begin weighted set cover...
    ## End weighted set cover...
    ## Generate the final report...
    ## Results can be found in the C:/Users/shashikadze/Documents/GitHub/maternaldiabetes-offspring-liver-omics-paper/proteomics/Project_Decreased!

``` r
decreased_ora <- ora_for_cluster_function(anova_results, regulation = "Increased", fc_threshold = 1.3, fdr_threshold = 0.05)
```

    ## Loading the functional categories...
    ## Loading the ID list...
    ## Loading the reference list...
    ## Summarizing the input ID list by GO Slim data...

    ## Performing the enrichment analysis...
    ## Begin affinity propagation...
    ## End affinity propagation...
    ## Begin weighted set cover...
    ## Remain is 0, ending weighted set cover
    ## Generate the final report...
    ## Results can be found in the C:/Users/shashikadze/Documents/GitHub/maternaldiabetes-offspring-liver-omics-paper/proteomics/Project_Increased!

``` r
# combine ora data
ora_data <- increased_ora %>% 
  bind_rows(decreased_ora) %>% 
  select(Regulation, geneSet, description, size, overlap, enrichmentRatio, pValue, FDR, userId) %>% 
  rename('GO id'                = geneSet, 
         'Biological process'   = description, 
         'Gene set size'        = size,
         'Gene count'           = overlap,
         'p-value'              = pValue,
         'Genes mapped'         = userId,
         'Fold enrichment'      = enrichmentRatio)

# save results
write.table(ora_data, "ora_results.txt", sep = "\t", row.names = F, quote = F)
```

### choose the processes that will be displayed on the plot

``` r
# replace zero fdr with lowest reported fdr (if any)
ora_data$FDR[ora_data$FDR == 0] <- min(ora_data$FDR[ora_data$FDR > 0])

# choose processes to be displayed on plot
ora_data_plot <- ora_data %>% 
  filter(Regulation == "Decreased in PHG" &  `GO id` %in% c("GO:0006260", "GO:0032200", "GO:0006289", "GO:0055088", "GO:0006643")|
         Regulation == "Increased in PHG" &  `GO id` %in% c("GO:0016051", "GO:0051188", "GO:0033865", "GO:0006575", "GO:0042180"))
```

## ORA dot plot

``` r
# facet labels
f_labels <- data.frame(Cluster = c("Decreased in PHG", "Increased in PHG"), label = c("Decreased in PHG", "Increased in PHG"))


# order rows based on enrichment score for SKM
ora_data_plot$`Biological process` <- factor(ora_data_plot$`Biological process`, levels = ora_data_plot$`Biological process`
                             [order(ora_data_plot$`Fold enrichment`)])
# plot
plot_ora <- ggplot(ora_data_plot, aes(x = `Fold enrichment`, y= `Biological process`, fill = -log10(FDR), size = `Gene count`)) +
 geom_point(shape = 21)+
 theme_bw() +
 theme(panel.border = element_rect(linewidth = 1),
                            axis.text.x = element_text(colour = "black", size = 9),
                            axis.title  = element_text(size = 9),
                            axis.text.y = element_text(size = 9, colour = "black"))+
                      xlab("Enrichment score")+
                      ylab("")+
                      theme(plot.title = element_text(size = 8, hjust=0.5,
                                                      face = "bold")) +
                      theme(panel.border = element_rect(linewidth = 1, colour = "black"),
                            panel.grid.major = element_line(),
                            axis.ticks = element_line(colour = "black"),
                            panel.grid.minor = element_blank())+
                      scale_size_continuous(name = "Count", range = c(1.8, 4.2), breaks = c(5,10,15))+
                      scale_fill_gradient(name = "-log10(FDR)", low = "#007ea7", high = "#003459")+
                      theme(legend.position = "bottom", 
                            legend.box.spacing = unit(0.8, 'mm'), 
                            legend.title = element_text(size = 8), 
                            legend.text = element_text(size = 8))+
                      theme(legend.key.size = unit(4, 'mm'))+
                      facet_grid(~Regulation, scales = "free")+
    geom_text(x = Inf, y = -Inf, aes(label = label),  size = 3, hjust = 1.05, vjust = -0.7, data = f_labels, inherit.aes = F)+
    theme(strip.background = element_blank(), strip.text = element_blank())
```

## save results

### combine plots

``` r
p0     <- rectGrob(width = 1, height = 1)
plot_1 <- ggarrange(plot_pca, plot_volcano, labels = c("B", "C"), font.label = list(size = 17),
          ncol = 1, nrow = 2, widths = c(7.1-w1,7.1-w1), heights = c(7.1-w1,7.1-w1))
plot_2 <- ggarrange(plot_hmap, NA, nrow = 2, heights = c(h1, 5.82-h1), widths = c(w1,w1))
```

    ## Warning in as_grob.default(plot): Cannot convert object of class logical into a
    ## grob.

``` r
plot_3 <- plot_grid(plot_2, plot_1, rel_widths = c(w1, 7.1-w1), rel_heights = c(h1+(5.82-h1), h1+(5.82-h1)), labels = c('A'), label_size = 17)
plot_4 <- plot_grid(plot_3, p0, ncol = 1, rel_widths = c(1,1), rel_heights = c(h1+(5.82-h1), 0.1))
plot_5 <- plot_grid(plot_4, plot_ora, ncol = 1, rel_widths = c(w1+7.1-w1, w1+7.1-w1), 
                    rel_heights = c(h1+0.5+(5.82-h1), 2.6), labels = c('','D'),  label_size = 17)
ggsave("proteomics_main.svg", width = 7.1, height = h1+0.5+(5.82-h1)+2.6)
```

    ## Warning: ggrepel: 58 unlabeled data points (too many overlaps). Consider
    ## increasing max.overlaps

### save data in a supplementary tables

``` r
if (!file.exists("Supplementary table 1.xlsx")) {
  
# anova results (all)
write.xlsx(as.data.frame(anova_results), file = "Supplementary table 1.xlsx", sheetName = "Suppl table 1B", 
  col.names = TRUE, row.names = FALSE, append = T)

# anova results (group)
write.xlsx(as.data.frame(anova_results_group), file = "Supplementary table 1.xlsx", sheetName = "Suppl table 1C", 
  col.names = TRUE, row.names = FALSE, append = T)

# anova results (ora)
write.xlsx(as.data.frame(ora_data), file = "Supplementary table 1.xlsx", sheetName = "Suppl table 1D", 
  col.names = TRUE, row.names = FALSE, append = T)

# anova results (sex)
write.xlsx(as.data.frame(anova_results_sex), file = "Supplementary table 1.xlsx", sheetName = "Suppl table 1E", 
  col.names = TRUE, row.names = FALSE, append = T)

# anova results (interaction) 
write.xlsx(as.data.frame(anova_results_int_tuk[[1]] %>% arrange(-desc(`Adjusted p-value group:sex`))), file = "Supplementary table 1.xlsx", sheetName = "Suppl table 1F", 
  col.names = TRUE, row.names = FALSE, append = T)

}
```

``` r
save.image()
```
