statistical analysis and visualization of metabolomics data
================
BS
19/02/2023

## load libraries

``` r
library(tidyverse)
library(ggrepel)
library(ComplexHeatmap)
library(circlize)
library(cowplot)
library(ggpubr)
library(ropls) 
library(xlsx)
```

## load data

``` r
metabolomics_data <- read.delim("metabolites.txt", sep = "\t", header = T) 
conditions        <- read.delim("Conditions.txt", sep = "\t", header = T)
```

### clean-up data

``` r
# ratios
ratios_data_tidy <- metabolomics_data %>% 
  filter(Class == "Metabolite ratio") %>% 
  select(Metabolite, starts_with("D"))

# metabolite concentrations
metabolomics_data_tidy <- metabolomics_data %>% 
  filter(Class != "Metabolite ratio") %>% 
  select(Metabolite, starts_with("D"))
```

## differential abundance analysis using 2 way ANOVA (metabolites)

### log2 transform data

``` r
data_stat <- metabolomics_data_tidy %>% 
      mutate(across(where(is.numeric), ~ log2(.x)))
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
                          compared_to  = "PHG", values_log= T, id_name = "Metabolite")
```

    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Metabolite'. You can override using the
    ## `.groups` argument.

    ## positive fold change means up in PHG

``` r
l2fc_sex <- fc_function(data_stat, condition = "Sex", conditions_data = conditions,
                          compared_to  = "F", values_log= T, id_name = "Metabolite")
```

    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Metabolite'. You can override using the
    ## `.groups` argument.

    ## positive fold change means up in F

``` r
# merge fold-changes
l2fc_metabolites <- l2fc_group %>% 
  rename("l2fc group (PHG/PNG)" = l2fc) %>% 
  left_join(l2fc_sex) %>% 
  rename("l2fc sex (F/M)" = l2fc) 
```

    ## Joining, by = "Metabolite"

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

##### significance of metabolite changes with 2 way anova

``` r
anova_results <- two_way_anova_fn(data = data_stat, id_name = "Metabolite", 
                                  conditions_file = conditions, 
                                  p_adj_method = "BH", adjust_p_value = T,  
                                  add_l2fc = T, l2fc=l2fc_metabolites)
```

    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Metabolite'. You can override using the
    ## `.groups` argument.
    ## Joining, by = "Metabolite"

##### clean-up anova results

``` r
# final anova results
anova_results  <- anova_results %>% 
  left_join(metabolomics_data %>% select(Metabolite, Class)) %>% 
  select("Metabolite", "Class", "l2fc group (PHG/PNG)", "p-value group (PHG/PNG)", 
         "Adjusted p-value group (PHG/PNG)", "l2fc sex (F/M)", "p-value sex (F/M)", 
         "Adjusted p-value sex (F/M)", "p-value group:sex", "Adjusted p-value group:sex") %>% 
  arrange(-desc(`Adjusted p-value group (PHG/PNG)`)) 


# separately significant factors and interactions
# significant by group
anova_results_group <- anova_results %>% 
  select(1:5) %>% 
  filter(`Adjusted p-value group (PHG/PNG)` <= 0.05 & abs(`l2fc group (PHG/PNG)`) > log2(1.5)) %>% 
          mutate(`Differentially abundant` = case_when(
                 `l2fc group (PHG/PNG)` >  log2(1.5)  ~  "increased in PHG",
                 `l2fc group (PHG/PNG)` < -log2(1.5)  ~  "decreased in PHG",
             TRUE ~ "n.s."
           )) %>% 
  arrange(desc(`l2fc group (PHG/PNG)`)) 


# significant by sex
anova_results_sex <- anova_results %>% 
  select(1:2, 6:8) %>% 
  filter(`Adjusted p-value sex (F/M)` <= 0.05 & abs(`l2fc sex (F/M)`) > log2(1.5)) %>% 
          mutate(`Differentially abundant` = case_when(
                 `l2fc sex (F/M)` >  log2(1.5)  ~  "increased in F",
                 `l2fc sex (F/M)` < -log2(1.5)  ~  "decreased in F",
             TRUE ~ "n.s."
           )) %>% 
  arrange(desc(`l2fc sex (F/M)`)) 


# interaction
anova_results_int <- anova_results %>% 
  select(1:2, 9, 10) %>% 
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

##### THSD of metabolite interactions

``` r
anova_results_int_tuk <- tkhsd_fn(data = anova_results_int,  id_name = "Metabolite", numeric_data = data_stat, conditions_file = conditions)
```

    ## Joining, by = "Metabolite"
    ## Joining, by = "Bioreplicate"
    ## Joining, by = "Metabolite"
    ## Joining, by = "Metabolite"
    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Metabolite'. You can override using the
    ## `.groups` argument.

## volcano plot

### prepare data for the volcano plot

``` r
data_volcano <- anova_results %>% 
                mutate(significant = case_when(
                `Adjusted p-value group (PHG/PNG)` < 0.05 & `l2fc group (PHG/PNG)` > log2(1.5) | `Adjusted p-value group (PHG/PNG)` < 0.05 & 
                `l2fc group (PHG/PNG)` < -log2(1.5) ~ "+", TRUE ~ "n.s."),
                diff_abundant = case_when(
                `Adjusted p-value group (PHG/PNG)` < 0.05 & `l2fc group (PHG/PNG)` > log2(1.5)  ~  "Increased_in_PHG",
                `Adjusted p-value group (PHG/PNG)` < 0.05 & `l2fc group (PHG/PNG)`     < -log2(1.5) ~  "Decreased_in_PHG",
             TRUE ~ "n.s."
           )) 
```

### plot volcano plot

``` r
plot_volcano <- ggplot(data_volcano %>%                       
                       arrange(desc(diff_abundant)), 
                       mapping = aes(x = `l2fc group (PHG/PNG)`, y = -log10(`p-value group (PHG/PNG)`), 
                                     fill=Class, label = Metabolite))+
         geom_point(aes(shape = diff_abundant, size = diff_abundant), stroke = 0.25)+
         scale_fill_manual(values=c("Glycerophospholipids" = "#6A3D9A", 
                                    "Biogenic Amines"      = "#FB9A99",      
                                    "Sphingolipids"        = "#E31A1C",
                                    "Aminoacids"           = "#B2DF8A",
                                    "Acylcarnitines"       = "#FFFF99",
                                    "Sugars"               = "#33A02C"))+
         scale_shape_manual(values = c('Increased_in_PHG' = 24, 'Decreased_in_PHG' = 25, "n.s." = 21)) + 
         scale_size_manual(values = c('Increased_in_PHG' = 2.5, 'Decreased_in_PHG' = 2.5, "n.s." = 2)) + 
         theme_bw()+
         scale_x_continuous(limits = c(-1.9, 1.5), breaks = c(-2,-1.5, -1, -0.5, 0, 0.5, 1, 1.5)) +
         scale_y_continuous(limits = c(0,8), breaks = c(0,2,4,6,8)) +
         theme(panel.border = element_rect(linewidth  = 1, color = "black"), 
               panel.grid.major = element_line(), 
               panel.grid.minor = element_blank(),
               axis.text = element_text(size = 9, colour = "black"),
               axis.title = element_text(size = 9, colour = "black"),
               panel.background = element_blank(),
               axis.ticks = element_line(colour = "black"),
               axis.line = element_blank())+
        theme(legend.position=c("top"), 
                            legend.box.spacing = unit(0.5, 'mm'), 
                            legend.title = element_blank(), 
                            legend.text = element_text(size = 8),
                            legend.spacing.y  = unit(0.01, 'mm'),
                            legend.spacing.x  = unit(0.01, 'mm'),
                            legend.margin=margin(0,0,0,0),
                            legend.box.margin=margin(-1.5,0,0,0)) + 
        guides(shape = "none") +
        guides(size = "none") +
        guides(fill = guide_legend(nrow=3, byrow=TRUE, keyheight=0.15,
                 default.unit="inch", override.aes=list(shape=21, size = 2.8)))+
         xlab("log2 fold change (PHG/PNG)")+
         ylab("-log10 p-value")
```

## supervised clustering

### define function which calculates principal components

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
data_pca <- pca_fn(data_stat, conditions, id_name = "Metabolite", scale = T)
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

data_HM <- hm_prep_fn(data = data_stat, id_name = "Metabolite")
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

![](metabolomics_files/figure-gfm/unnamed-chunk-14-1.png)<!-- -->

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

##### ploting legends separatelly

``` r
# color legend
hm_legend = grid.grabExpr(color_mapping_legend(hmap@matrix_color_mapping, plot = T, title = NULL, legend_direction = c("horizontal"),  title_gp = gpar(fontsize = 8.5, fontface = "bold"), param = list(at = c(-1,  1), labels = c("low", "high"), legend_width = unit(2, "cm"), labels_gp = gpar(fontsize=8.5)),  labels_gp = gpar(fontsize = 7)))

# legend for the group
Glycemia = grid.grabExpr(color_mapping_legend(hmap@top_annotation@anno_list[["Group"]]@color_mapping, nrow = 1, plot = T,  title_gp = gpar(fontsize = 7, fontface = "bold"),  labels_gp = gpar(fontsize = 8.5)))

# legend for the sex
Sex = grid.grabExpr(scale = 1, color_mapping_legend(hmap@top_annotation@anno_list[["Sex"]]@color_mapping,  nrow = 1, plot = T, title_gp = gpar(fontsize = 7, fontface = "bold"),  labels_gp = gpar(fontsize = 8.5)))

# combine legends
plot_lg <- plot_grid(hm_legend, Glycemia, Sex, ncol = 3, rel_widths = c(1,1,1), rel_heights = c(1,1,1))
ggsave("legend_main_met.svg", width =2.8, height = 1)
```

### Orthogonal Projections to Latent Structures Discriminant Analysis (OPLS-DA) (function)

``` r
oplsda_function <- function (data, scaling = "pareto", n_perm = 200, 
                             n_crossval = 9, 
                             vip_thresh = 1.5,
                             conditions_file,
                             group, stat_data)  {
  Grouping     <- conditions_file[, group]
  oplsda_model <- opls(t(data), Grouping, predI = 1, 
                      orthoI = NA, permI = n_perm, 
                      scaleC = scaling, 
                      crossvalI = n_crossval, subset = NULL) 
          p1          <- round(oplsda_model@modelDF$`R2X(cum)`[1]*100)
    loadings          <- getLoadingMN(oplsda_model)
  oplsda_vip          <- oplsda_model@vipVn %>% 
    as.data.frame() %>% 
    rownames_to_column() %>% 
    rename(Metabolite = rowname, VIP = 2) %>% 
    left_join(stat_data) %>% 
    mutate(VIP_sig = case_when(VIP > vip_thresh ~ "+")) %>% 
    mutate(Regulation = case_when(
      `l2fc group (PHG/PNG)` > 0 & VIP_sig  == "+" ~ "Increased in PHG",
      `l2fc group (PHG/PNG)`  < 0 & VIP_sig == "+" ~ "Decreased in PHG",
      TRUE ~ "n.s."
    ))  
  oplsda_vip$Compound <- factor(oplsda_vip$Metabolite, levels = oplsda_vip$Metabolite[order(oplsda_vip$VIP)])
  oplsda_pred         <- as.data.frame(oplsda_model@scoreMN)
  oplsda_ortho        <- as.data.frame(oplsda_model@orthoScoreMN)
  oplsda_components   <- cbind(oplsda_pred, oplsda_ortho, conditions_file)
  data_oplsda         <- list()
  data_oplsda[[1]]    <- oplsda_components
  data_oplsda[[2]]    <- oplsda_vip
  data_oplsda[[3]]    <- p1
  return(data_oplsda)
}
```

#### OPLS-DA calculation

``` r
set.seed(123456) 
data_oplsda <- oplsda_function(data_stat %>% column_to_rownames("Metabolite"), stat_data = anova_results, conditions_file = conditions, group = "Group", scaling = "pareto", n_perm = 500, n_crossval = 19, vip_thresh = 1.5)
```

    ## OPLS-DA
    ## 19 samples x 166 variables and 1 response
    ## pareto scaling of predictors and standard scaling of response(s)
    ##       R2X(cum) R2Y(cum) Q2(cum)  RMSEE pre ort  pR2Y   pQ2
    ## Total    0.573    0.988   0.925 0.0614   1   2 0.002 0.002

    ## Joining, by = "Metabolite"

![](metabolomics_files/figure-gfm/unnamed-chunk-17-1.png)<!-- -->

``` r
# metabolomics data with VIP scores
anova_results <- anova_results %>% 
  left_join(data_oplsda[[2]] %>% select(Metabolite, VIP)) %>% 
  rename("VIP score (PHG/PNG)" = VIP) %>% 
  relocate("VIP score (PHG/PNG)", .after = `l2fc group (PHG/PNG)`)
```

    ## Joining, by = "Metabolite"

#### Orthogonal Projections to Latent Structures Discriminant Analysis (OPLS-DA) (plotting)

``` r
plot_oplsda <- ggplot(data=data_oplsda[[1]], aes(x=p1, y=o1, fill = Group))+
geom_point(size = 3, shape =21, stroke = 0.25)+
scale_fill_manual(values=c('PHG' = "#e95559ff", 'PNG' = "#0088AA"))+
xlab(paste("OPLS-DA axis 1 - ", data_oplsda[[3]], "%", sep=""))+
ylab("OPLS-DA axis 2")+
geom_hline(yintercept = 0, linetype = "dashed")+
geom_vline(xintercept = 0, linetype = "dashed")+
theme_bw() + 
  theme(panel.border = element_rect(linewidth = 1, colour = "black"),
                   axis.ticks = element_line(colour = "black"),
                   axis.title = element_text(size = 9, colour="black"), 
                   axis.text.x = element_text(size=9, colour="black"), 
                   axis.text.y = element_text(size = 9, colour="black"),
panel.grid.major = element_line(), panel.grid.minor = element_blank())+
theme(legend.position = "top", legend.box.spacing = unit(0.5, 'mm'), 
      legend.title = element_blank(), 
      legend.text = element_text(size = 8.5))
```

#### variance importance in projection from OPLS-DA (plotting)

``` r
plot_vip <- ggplot(data_oplsda[[2]] %>% 
                  filter(VIP_sig == "+"), aes(x = VIP, y= Compound, fill = Regulation)) +
geom_point(size = 3, shape = 21, stroke = 0.25) +
theme_bw()+
scale_fill_manual(values = c("#0088AA", "#e95559ff"))+
theme(panel.border = element_rect(linewidth = 1, color = "black"), 
      panel.grid.major = element_line(), 
      panel.grid.minor = element_blank(), 
      panel.background = element_blank(),
      axis.ticks = element_line(colour = "black"),
                            axis.text.x = element_text(colour = "black", size = 9),
                            axis.title.x = element_text(size= 9),
                            axis.title.y = element_blank(),
                            axis.text.y = element_text(size = 9, colour = "black"))+
scale_x_continuous(breaks = c(1.5, 1.85,  2.2))+
xlab("VIP score")+
ggtitle("VIP plot") +
theme(plot.title = element_blank()) +
theme(legend.position = c(0.5, 1.1), 
      legend.box.spacing = unit(0.5, 'mm'), 
      legend.title = element_blank(), 
      legend.text = element_text(size = 8.5))+
guides(fill=guide_legend(nrow=1, byrow=TRUE))+
theme(plot.margin = margin(12.1,2,1,1, "mm"))
```

## plot anova interactions

``` r
# reorder data
anova_results_int_tuk[[2]]$Group <- factor(anova_results_int_tuk[[2]]$Group, levels = c("PNG", "PHG"))

# plot
ggplot(anova_results_int_tuk[[2]], aes(x=Group, y=mean)) +
  geom_line(size = 0.8, aes(group = Sex, color = Sex))+
  geom_errorbar(aes(ymin=mean-sd, ymax=mean+sd), width=.2,
                 position=position_dodge(0.05))+
  geom_point(aes(color = Sex))+
  scale_color_manual(values = c("#F0E442", "#E69F00"))+
  theme_bw()+
  xlab("")+
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
  theme(legend.position = "bottom", legend.title = element_text(size = 8.5), axis.text = element_text(size = 8.5))+
  facet_wrap(~factor(Metabolite, levels=unique(anova_results_int_tuk[[1]]$Metabolite)), scales = "free", ncol = 5) +
  ylab("log2 metabolite concentration (pmol/mg)")+
  geom_text(x = Inf, y = -Inf, 
            aes(label = paste("p=",`Adjusted p-value group:sex`)),  
            size = 2.8, 
            angle = 90,
            hjust = -0.7, 
            vjust = -0.7, 
            data = anova_results_int_tuk[[1]] %>% 
                        mutate(across(where(is.numeric), round, 3)), inherit.aes = F)+
  theme(strip.background = element_blank(), strip.text = element_text())
```

    ## Warning: Using `size` aesthetic for lines was deprecated in ggplot2 3.4.0.
    ## ℹ Please use `linewidth` instead.

![](metabolomics_files/figure-gfm/unnamed-chunk-20-1.png)<!-- -->

``` r
ggsave("anova_interactions_met.svg", width = 6.4, height = 4)
```

## statistical analysis of metabolite ratios

### calculate fold-changes

``` r
# prepare data
data_stat_ratios <- ratios_data_tidy %>% 
      mutate(across(where(is.numeric), ~ log2(.x)))


# calculate fold-change separately for group and sex
l2fc_group <- fc_function(data_stat_ratios, condition = "Group", conditions_data = conditions,
                          compared_to  = "PHG", values_log= T, id_name = "Metabolite")
```

    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Metabolite'. You can override using the
    ## `.groups` argument.

    ## positive fold change means up in PHG

``` r
l2fc_sex <- fc_function(data_stat_ratios, condition = "Sex", conditions_data = conditions,
                          compared_to  = "F", values_log= T, id_name = "Metabolite")
```

    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Metabolite'. You can override using the
    ## `.groups` argument.

    ## positive fold change means up in F

``` r
# merge fold-changes
l2fc_ratios <- l2fc_group %>% 
  rename("l2fc group (PHG/PNG)" = l2fc) %>% 
  left_join(l2fc_sex) %>% 
  rename("l2fc sex (F/M)" = l2fc) 
```

    ## Joining, by = "Metabolite"

### perform anova

``` r
anova_results_ratios <- two_way_anova_fn(data = data_stat_ratios, id_name = "Metabolite", 
                                         conditions_file = conditions, adjust_p_value = T,  
                                         p_adj_method = "BH", add_l2fc = T, l2fc = l2fc_ratios)
```

    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Metabolite'. You can override using the
    ## `.groups` argument.
    ## Joining, by = "Metabolite"

#### clean-up anova results

``` r
# final anova results
anova_results_ratios  <- anova_results_ratios %>% 
  left_join(metabolomics_data %>% select(Metabolite, Class)) %>% 
  select("Metabolite", "l2fc group (PHG/PNG)", "p-value group (PHG/PNG)", 
         "Adjusted p-value group (PHG/PNG)", "l2fc sex (F/M)", "p-value sex (F/M)", 
         "Adjusted p-value sex (F/M)", "p-value group:sex", "Adjusted p-value group:sex") %>% 
  arrange(-desc(`Adjusted p-value group (PHG/PNG)`)) 
```

    ## Joining, by = "Metabolite"

``` r
# significant by group
anova_results_ratios_group <- anova_results_ratios %>% 
  select(1:4) %>% 
  filter(`Adjusted p-value group (PHG/PNG)` <= 0.05) %>% 
          mutate(`Differentially abundant` = case_when(
                 `l2fc group (PHG/PNG)` > log2(1) ~  "increased in PHG",
                 `l2fc group (PHG/PNG)` < -log2(1) ~ "decreased in PHG",
             TRUE ~ "n.s."
           )) %>% 
  arrange(desc(`l2fc group (PHG/PNG)`)) 
```

### plot anova significant results (ratios)

#### function that obtains the data for error bar and plots bar diagrams

``` r
bar_chart_fn <- function(data_statistics, 
                         numeric_data,
                         conditions_data,
                         point_size,
                         id_name,
                         n_widht,
                         jitt_widht,
                         strip_text_size) {
  
  
  # prepare data for error bar calculation
  error_bar <- data_statistics %>% 
  left_join(numeric_data) %>% 
  select(contains(colnames(numeric_data))) %>% 
  pivot_longer(names_to = "Bioreplicate", values_to = "Value", -all_of(id_name)) %>% 
  left_join(conditions_data) %>% 
  group_by(!!as.symbol(id_name), Group) %>% 
  summarise(mean = mean(2^Value, na.rm=T), 
            sd = sd(2^Value, na.rm=T), 
            n = n(),
            sem = sd/sqrt(n),
            t.score = qt(p=0.05/2, 
            df=length(conditions$Bioreplicate),lower.tail=F), 
            ci = t.score * sd ) %>% 
  ungroup()
  # prepare data for plotting
  data_plot <- data_statistics %>% 
  left_join(numeric_data) %>% 
  select(contains(colnames(numeric_data))) %>%  
  pivot_longer(names_to = "Bioreplicate", values_to = "Value", -all_of(id_name)) %>% 
  left_join(conditions_data) %>% 
  ungroup() %>% 
    left_join(error_bar) 
  
  # reorder data
  data_plot$Group <- factor(data_plot$Group, levels = c("PNG", "PHG"))
  
  
  # plot
  plot_data <- ggplot(data_plot %>% mutate(Value = 2^Value), aes(x=Group, y=Value))+
  geom_bar(stat = "summary", fun = mean, aes(fill = Group), alpha = 1, color = "black", lwd=0.15,
           width=n_widht/length(unique(data_plot$Group))) + 
  geom_jitter(size = point_size,  aes(shape = Sex), fill = "grey", alpha = 0.8, stroke =0.25, width = jitt_widht)+
  geom_errorbar(aes(ymin=mean-sd, ymax=mean+sd), width=.2,
                position=position_dodge(0.05)) +
  scale_fill_manual(values=c('PNG' = "#0088AA", 'PHG' = "#e95559ff")) +
  scale_shape_manual(values = c('F' = 21, 'M' = 22)) + 
  theme_bw() +
  xlab("") +
  theme(panel.border = element_rect(linewidth = 1, colour = "black"),
        axis.title = element_text(size = 9, colour = "black"),
        axis.text = element_text(size = 9, colour = "black"),
        axis.ticks = element_line(colour = "black"),
        legend.box.spacing = unit(0.8, 'mm'), 
        legend.title = element_blank(), 
        legend.text = element_text(size = 8))+
  theme(strip.text = element_text(size = strip_text_size),
        strip.background = element_blank(),
        legend.margin=margin(0,0,0,0), 
        legend.box.margin=margin(-12,-12,-3,-12)) 
  
  
  return(plot_data)
 
 
}
```

#### plot data

``` r
# plot ratios
# to save the space "total" will be removed before the metabolite ratios that are displayed on the plot 
plot_ratios <- bar_chart_fn(data_statistics = anova_results_ratios %>% 
                              mutate(Metabolite = case_when(Metabolite == 'Total PC ae' ~ 'PC ae [pmol/mg]',
                                                            Metabolite == 'Total SM' ~ 'SM [pmol/mg]',
                                                            TRUE ~ Metabolite)) %>% 
                              filter(`Adjusted p-value group (PHG/PNG)` <= 0.05),
                              numeric_data = data_stat_ratios %>% 
                              mutate(Metabolite = case_when(Metabolite == 'Total PC ae' ~ 'PC ae [pmol/mg]',
                                                            Metabolite == 'Total SM'    ~ 'SM [pmol/mg]',
                                                            TRUE ~ Metabolite)), 
                              point_size = 2, 
                              strip_text_size = 9,
                              n_widht = 0.8, 
                              jitt_widht = 0.05,
                              id_name = "Metabolite", 
                              conditions_data = conditions)+
            ylab("")+
            geom_text(x = Inf, y = -Inf, 
            aes(label = paste("p=", `Adjusted p-value group (PHG/PNG)`)),  
            size = 3, 
            hjust = -1.55, 
            angle = 90,
            vjust = -0.6, 
            data = anova_results_ratios %>% 
                              mutate(Metabolite = case_when(Metabolite == 'Total PC ae' ~ 'PC ae [pmol/mg]',
                                                            Metabolite == 'Total SM'    ~ 'SM [pmol/mg]',
                                                            TRUE ~ Metabolite)) %>% 
            filter(`Adjusted p-value group (PHG/PNG)` <= 0.05)  %>% 
            mutate(across(where(is.numeric), round, 3)), inherit.aes = F) +
            facet_wrap(~factor(Metabolite, levels=c("PC ae [pmol/mg]", "PUFA (PC) / MUFA (PC)", "SM [pmol/mg]")), scales = "free_y", ncol = 3)+ 
            guides(shape = guide_legend(order = 2, override.aes = list(stroke = 1, shape  = c(0,1))),
            fill = guide_legend(order = 1))+
            theme(legend.position = "bottom")+
            theme(plot.margin = margin(1,2,-5,-2, "mm"))
```

    ## Joining, by = "Metabolite"
    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Metabolite'. You can override using the
    ## `.groups` argument.
    ## Joining, by = "Metabolite"
    ## Joining, by = "Bioreplicate"
    ## Joining, by = c("Metabolite", "Group")

``` r
# plot significant metabolites (Glycerophospholipids will be plotted as heatmap)
plot_other_sig <- bar_chart_fn(data_statistics = anova_results_group %>% filter(Class != "Glycerophospholipids" &
                                                                                Class != "Sphingolipids"), 
                            numeric_data = data_stat, n_widht = 0.95, 
                            jitt_widht = 0.1,  
                            point_size = 2, 
                            strip_text_size = 9,
                            id_name = "Metabolite", 
                            conditions_data = conditions)+
            ylab("pmol/mg")+
            geom_text(x = Inf, y = -Inf, 
            aes(label = paste("p=", `Adjusted p-value group (PHG/PNG)`)),  
            size = 3, 
            hjust = -1.65, 
            angle = 90,
            vjust = -0.6, 
            data = anova_results_group %>% 
            filter(Class != "Glycerophospholipids" & Class != "Sphingolipids")  %>% 
            mutate(across(where(is.numeric), round, 3)), inherit.aes = F)+
            facet_wrap(~factor(Metabolite, levels=c('Pro',
                                          't4-OH-Pro', 
                                          'total DMA',
                                          'SDMA',
                                          'ADMA')), scales = "free_y", ncol = 3)+
  theme(legend.position = c(0.85, 0.2))+
  guides(fill=guide_legend(nrow=2, byrow=TRUE, order = 1), shape = guide_legend(override.aes = list(stroke = 1, shape  = c(0,1))))+
  theme(plot.margin = margin(1,2,-5,3, "mm"))
```

    ## Joining, by = "Metabolite"
    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Metabolite'. You can override using the
    ## `.groups` argument.
    ## Joining, by = "Metabolite"
    ## Joining, by = "Bioreplicate"
    ## Joining, by = c("Metabolite", "Group")

### plot Glycerophospholipids as a heatmap

``` r
# make row annotations
  # make row annotation
  stars_data <- rownames(data_HM %>% 
                            filter(rownames(data_HM) %in% anova_results_group$Metabolite[anova_results_group$Class == "Glycerophospholipids" |anova_results_group$Class == "Sphingolipids"])) %>% 
    as.data.frame() %>% 
    rename(Metabolite = 1) %>% 
    left_join(anova_results_group) %>% 
    select(1,5) %>% 
    column_to_rownames("Metabolite") %>% 
    rename(adj_p_value = `Adjusted p-value group (PHG/PNG)`) %>% 
    mutate(padj_star = case_when(adj_p_value < 0.001 &  adj_p_value >  0     ~ "***",
                         adj_p_value < 0.01         &  adj_p_value >  0.001  ~ "**",
                         adj_p_value < 0.05         &  adj_p_value >  0.01   ~ "*",
                         adj_p_value > 0.05                          ~ ""))
```

    ## Joining, by = "Metabolite"

``` r
    FDR = stars_data$adj_p_value
    is_sig = FDR < 0.05
    pch = stars_data$padj_star
    pch[!is_sig] = NA
    
    # color mapping for -log10(pvalue)
    pvalue_col_fun = colorRamp2(c(0, 2, 3), c("white", "white", "white")) 

    row_ann = HeatmapAnnotation(which = 'row', annotation_name_gp   = gpar(fontsize = 0),
    FDR = anno_simple(-log10(FDR), col = pvalue_col_fun, pch = pch))
    
    
# plotting
hmap_gl <- Heatmap(as.matrix(data_HM %>% 
                            filter(rownames(data_HM) %in% anova_results_group$Metabolite[anova_results_group$Class == "Glycerophospholipids" |anova_results_group$Class == "Sphingolipids"])),
                show_heatmap_legend        = F,
                row_dend_width             = unit(0.5, "cm"),
                column_dend_height         = unit(0.5, "cm"),
                show_column_names = T,
                show_row_names = T,
                show_row_dend = F,
                row_names_side = "left",
                column_names_side = "top",
                row_title = NULL,
                clustering_distance_columns = "euclidean",
                clustering_distance_rows    = "euclidean",
                clustering_method_rows      = "ward.D",
                clustering_method_columns   = "ward.D",
                use_raster = F,
                top_annotation              = hmap_all_data[[3]],
                right_annotation            = row_ann,
                width                       = unit(1.8, "in"),
                height                      = unit(3.135, "in"), 
                col                         = hmap_all_data[[1]], 
                border                      = TRUE,
                column_names_gp             = gpar(fontsize = 8),
                row_names_gp                = gpar(fontsize = 8)) 

ht_gl <- draw(hmap_gl)
```

![](metabolomics_files/figure-gfm/unnamed-chunk-26-1.png)<!-- -->

``` r
#calculate actual plot size
w1_gl = ComplexHeatmap:::width(ht_gl)
w1_gl = convertX(w1_gl, "inch", valueOnly = TRUE)
h1_gl = ComplexHeatmap:::height(ht_gl)
h1_gl = convertY(h1_gl, "inch", valueOnly = TRUE)
c(w1_gl, h1_gl)
```

    ## [1] 3.039057 4.140591

``` r
#for cowplot
plot_Hmap_gl = grid.grabExpr(draw(ht_gl))
```

#### ploting legends separatelly for Glycerophospholipids

``` r
# color legend
hm_legend = grid.grabExpr(color_mapping_legend(hmap_gl@matrix_color_mapping, plot = T, title = NULL, legend_direction = c("horizontal"),  title_gp = gpar(fontsize = 8.5, fontface = "bold"), param = list(at = c(-1,  1), labels = c("low", "high"), legend_width = unit(2, "cm"), labels_gp = gpar(fontsize=8.5)),  labels_gp = gpar(fontsize = 7)))

# legend for the group
Glycemia = grid.grabExpr(color_mapping_legend(hmap_gl@top_annotation@anno_list[["Group"]]@color_mapping, nrow = 1, plot = T,  title_gp = gpar(fontsize = 7, fontface = "bold"),  labels_gp = gpar(fontsize = 8.5)))

# legend for the sex
Sex = grid.grabExpr(scale = 1, color_mapping_legend(hmap_gl@top_annotation@anno_list[["Sex"]]@color_mapping,  nrow = 1, plot = T, title_gp = gpar(fontsize = 7, fontface = "bold"),  labels_gp = gpar(fontsize = 8.5)))

# combine legends
plot_lg <- plot_grid(hm_legend, Glycemia, Sex, ncol = 3, rel_widths = c(1,1,1), rel_heights = c(1,1,1))
ggsave("legend_gl_met.svg", width =2.8, height = 1)
```

## correlation of significant metabolites (adj p-value \< 0.05 & fc \> 1.5) with MIDY data (Blutke et al. 2017 doi: 10.1016/j.molmet.2017.06.004)

``` r
# download metabolomics results of MIDY vs WT 
#Blutke, A., et al., The Munich MIDY Pig Biobank - A unique resource for studying organ crosstalk in diabetes. Molecular metabolism, 2017. #6(8): p. 931-940.
url      <- "https://ars.els-cdn.com/content/image/1-s2.0-S2212877817303150-mmc2.docx"
destfile <- "C:/Users/shashikadze/Documents/GitHub/maternaldiabetes-offspring-liver-omics-paper/metabolomics/MIDY_metabolomics_data.docx"
download.file(url, destfile, quiet = TRUE, mode = "wb")


# load MIDY data which was cleaned up so only metabolite column, significance (student's t-test p value and l2fc is kept)
data_midy <- read.delim("blutke_metabolomics.txt", sep = "\t", header = T)


# prepare data
MIDY_offspring_data <- anova_results_group %>% 
  select(Metabolite, `l2fc group (PHG/PNG)`, `Adjusted p-value group (PHG/PNG)`, Class) %>% 
  left_join(data_midy) %>% 
  filter(`Adjusted p-value group (PHG/PNG)` <= 0.05) %>% 
  mutate(sig_midy = case_when(p.value <= 0.05 ~ "+",
                              TRUE ~ "-")) %>% 
  mutate(label = case_when(Metabolite == "PC aa C36:6"  | Metabolite == "PC ae C38:0"  | Metabolite == "PC aa C32:1"  ~ Metabolite,
                           Class ==  "Biogenic Amines"  ~ Metabolite,
                           Class ==  "Sphingolipids"  ~ Metabolite,
                           Class ==  "Aminoacids"  ~ Metabolite))
```

    ## Joining, by = "Metabolite"

``` r
# plot
plot_corr <- ggplot(data = MIDY_offspring_data, mapping = aes(x=`l2fc group (PHG/PNG)`, y = l2fc, label = label)) +
  geom_hline(yintercept = 0, linetype = "dashed") +
  geom_vline(xintercept = 0, linetype = "dashed") +
  geom_point(aes(shape = sig_midy, fill = Class), size = 2.7) +
  xlab(paste0("log2 fold change", " (PHG/PNG)"))+
  ylab(paste0("log2 fold change", " (MIDY/WT)"))+
  scale_x_continuous(limits = c(-0.85, 1.5)) +
  scale_y_continuous(limits = c(-0.85, 1.5)) +
  scale_shape_manual(values= c("+" = 24, "-" = 21))+
  scale_fill_manual(values=c("Glycerophospholipids"        = "#6A3D9A", 
                                    "Biogenic Amines"      = "#FB9A99",      
                                    "Sphingolipids"        = "#E31A1C",
                                    "Aminoacids"           = "#B2DF8A"))+
  theme_bw() + theme(panel.border = element_rect(linewidth = 1, color = "black"), 
               panel.grid.major = element_line(), 
               axis.ticks = element_line(color = "black"),
               panel.grid.minor = element_blank(),
               panel.background = element_blank(), 
               axis.line = element_blank(), 
               legend.position = "NONE")+
  theme(axis.title = element_text(size  = 9), 
               axis.text.x = element_text(size = 9, colour = "black", vjust = -0.1), 
               axis.text.y = element_text(size = 9, colour = "black"))+
  theme(legend.position=c("bottom"), 
                            legend.box.spacing = unit(0.5, 'mm'), 
                            legend.title = element_blank(), 
                            legend.text = element_text(size = 8),
                            legend.spacing.y  = unit(0.01, 'mm'),
                            legend.spacing.x  = unit(0.01, 'mm'),
                            legend.margin=margin(0,0,0,0),
                            legend.box.margin=margin(1,0,0,0)) + 
  guides(shape = "none") +
  geom_text_repel(size = 2.5, 
                  box.padding = unit(0.6, "lines"),
                  point.padding = unit(0.6, "lines"),
                  min.segment.length = 0, 
                  show.legend = F, 
                  seed = 1234, 
                  max.overlaps = Inf) +
  guides(fill = guide_legend(nrow=2, byrow=TRUE, keyheight=0.15,
                 default.unit="inch", override.aes=list(shape=21, size = 2.8)))
```

## save results

### combine plots

#### main figure

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
plot_5 <- plot_grid(plot_oplsda, plot_vip, ncol = 2, rel_widths = c(7.1/2, 7.1/2), rel_heights = c(3.35, 3.35), labels = c('D','E'), label_size = 17)
plot_6 <- plot_grid(plot_4, plot_5, nrow = 2, rel_widths = c(w1+7.1-w1, w1+7.1-w1), rel_heights = c(h1+0.5+(5.82-h1), 3.35))
ggsave("metabolomics_main.svg", width = 7.1, height = h1+0.5+(5.82-h1)+3.35)
```

#### metabolomics figure 2

``` r
p1 <-   plot_grid(p0, plot_Hmap_gl, p0, rel_widths = c(1,1,1), rel_heights = c(0.05, 3.9, 0.11), ncol = 1)
p2 <-   plot_grid(p0, plot_ratios, rel_widths = c(1,1), rel_heights = c(1.5, 1), ncol = 1, labels = c("", "C"), label_size = 17)
p3 <-   plot_grid(p1, p2, labels = c("A", "B"), rel_widths = c(w1_gl, 7.1-w1_gl), rel_heights = c(1,1), label_size = 17)
p4 <-   plot_grid(p3, p0, rel_widths = c(1,1), rel_heights = c(4.1,0.3), nrow = 2, label_size = 17)
p5 <-   plot_grid(plot_other_sig, p0, nrow = 2, rel_widths = c(1,1), rel_heights = c(3.8,0.12), labels = c("D"), label_size = 17)
p6 <-   plot_grid(p0, plot_corr, nrow = 2, rel_widths = c(1,1), rel_heights = c(0.24,3.76), labels = c("E"), label_size = 17)
```

    ## Warning: Removed 21 rows containing missing values (`geom_text_repel()`).

``` r
p7 <-   plot_grid(p5, p6, rel_widths = c(3.7,3.4), rel_heights = c(1,1))
plot_grid(p4, p7, nrow = 2, rel_widths = c(1,1), rel_heights = c(4.5, 3.5))
```

![](metabolomics_files/figure-gfm/unnamed-chunk-30-1.png)<!-- -->

``` r
ggsave("metabolomics_figure_2.svg", width = 7.1, height = 7.9)
```

### save data in a supplementary tables

``` r
if (!file.exists("Supplementary table 3.xlsx")) {
  
# metabolomics data (all)
write.xlsx(as.data.frame(metabolomics_data), file = "Supplementary table 3.xlsx", sheetName = "Suppl table 3A", 
  col.names = TRUE, row.names = FALSE, append = T)

# anova results (all)
write.xlsx(as.data.frame(anova_results), file = "Supplementary table 3.xlsx", sheetName = "Suppl table 3B", 
  col.names = TRUE, row.names = FALSE, append = T)

# anova results (group)
write.xlsx(as.data.frame(anova_results_group), file = "Supplementary table 3.xlsx", sheetName = "Suppl table 3C", 
  col.names = TRUE, row.names = FALSE, append = T)

# anova results (sex)
write.xlsx(as.data.frame(anova_results_sex), file = "Supplementary table 3.xlsx", sheetName = "Suppl table 3D", 
  col.names = TRUE, row.names = FALSE, append = T)

# anova results (interaction) 
write.xlsx(as.data.frame(anova_results_int_tuk[[1]] %>% arrange(-desc(`Adjusted p-value group:sex`))), file = "Supplementary table 3.xlsx", sheetName = "Suppl table 3E", 
  col.names = TRUE, row.names = FALSE, append = T)

# anova results (ratios)
write.xlsx(as.data.frame(anova_results_ratios), file = "Supplementary table 3.xlsx", sheetName = "Suppl table 3F", 
  col.names = TRUE, row.names = FALSE, append = T)

# anova results (ratios significant by group)
write.xlsx(as.data.frame(anova_results_ratios_group), file = "Supplementary table 3.xlsx", sheetName = "Suppl table 3G", 
  col.names = TRUE, row.names = FALSE, append = T)


}
```

``` r
save.image()
```
