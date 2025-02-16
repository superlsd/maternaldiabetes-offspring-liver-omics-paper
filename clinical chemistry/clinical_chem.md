statistical analysis of clinical chemical paramters data with
visualization
================
BS
19/02/2023

## load libraries

``` r
library(tidyverse)
library(cowplot)
library(ggpubr)
library(xlsx)
```

## load data

``` r
clinical_chem_data    <- read.delim("clinicalparams.txt", sep = "\t", header = T) 
conditions            <- read.delim("Conditions.txt", sep = "\t", header = T)
```

## differential abundance analysis using 2 way ANOVA

### log2 transform data

``` r
data_stat <- clinical_chem_data %>% 
      select(-Explanation) %>% 
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
                          compared_to  = "PHG", values_log = T, id_name = "Parameter")
```

    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Parameter'. You can override using the
    ## `.groups` argument.

    ## positive fold change means up in PHG

``` r
l2fc_sex <- fc_function(data_stat, condition = "Sex", conditions_data = conditions,
                          compared_to  = "F", values_log  = T, id_name = "Parameter")
```

    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Parameter'. You can override using the
    ## `.groups` argument.

    ## positive fold change means up in F

``` r
# merge fold-changes
l2fc_parameters <- l2fc_group %>% 
  rename("l2fc group (PHG/PNG)" = l2fc) %>% 
  left_join(l2fc_sex) %>% 
  rename("l2fc sex (F/M)" = l2fc) 
```

    ## Joining, by = "Parameter"

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

##### significance of clinical chemical parameter changes with 2 way anova

only row p-values will be used because these are separate/independent
measurements and multiple testing should not be an issue

``` r
anova_results <- two_way_anova_fn(data = data_stat, id_name = "Parameter", 
                                  conditions_file = conditions, 
                                  adjust_p_value = F,
                                  add_l2fc = T,
                                  l2fc=l2fc_parameters)
```

    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Parameter'. You can override using the
    ## `.groups` argument.
    ## Joining, by = "Parameter"

##### clean-up anova results

``` r
# final anova results
anova_results  <- anova_results %>% 
  left_join(clinical_chem_data %>% select(Parameter, Explanation)) %>% 
  select("Parameter", "Explanation", "l2fc group (PHG/PNG)", "p-value group (PHG/PNG)", 
         "l2fc sex (F/M)", "p-value sex (F/M)", 
         "p-value group:sex") %>% 
  arrange(-desc(`p-value group (PHG/PNG)`)) 


# separately significant factors and interactions
# significant by group
anova_results_group <- anova_results %>% 
  select(1:4) %>% 
  filter(`p-value group (PHG/PNG)` <= 0.05 & abs(`l2fc group (PHG/PNG)`) > log2(1)) %>% 
          mutate(`Differentially abundant` = case_when(
                 `l2fc group (PHG/PNG)` > log2(1) ~  "increased in PHG",
                 `l2fc group (PHG/PNG)` < -log2(1) ~ "decreased in PHG",
             TRUE ~ "n.s."
           )) %>% 
  arrange(desc(`l2fc group (PHG/PNG)`)) 


# significant by sex
anova_results_sex <- anova_results %>% 
  select(1:2, 5:6) %>% 
  filter(`p-value sex (F/M)` <= 0.05 & abs(`l2fc sex (F/M)`) > log2(1)) %>% 
          mutate(`Differentially abundant` = case_when(
                 `l2fc sex (F/M)` >  log2(1)  ~  "increased in F",
                 `l2fc sex (F/M)` < -log2(1)  ~  "decreased in F",
             TRUE ~ "n.s."
           )) %>% 
  arrange(desc(`l2fc sex (F/M)`)) 


# interaction
anova_results_int <- anova_results %>% 
  select(1:2, 7) %>% 
  filter(`p-value group:sex` <= 0.05) %>% 
  arrange(desc(`p-value group:sex`))
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

##### THSD of clinical chemical parameter interactions

``` r
anova_results_int_tuk <- tkhsd_fn(data = anova_results_int,  id_name = "Parameter", numeric_data = data_stat, conditions_file = conditions)
```

    ## Joining, by = "Parameter"
    ## Joining, by = "Bioreplicate"
    ## Joining, by = "Parameter"
    ## Joining, by = "Parameter"
    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Parameter'. You can override using the
    ## `.groups` argument.

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
  theme(panel.border = element_rect(linewidth =  1, colour = "black"),
       panel.grid.major = element_blank(), 
       panel.grid.minor = element_blank(), 
       panel.background = element_blank(), 
       axis.line = element_blank(),
  legend.position = "NONE") +
  xlab("")+
  theme(axis.title = element_text(size = 10), 
        axis.text.x = element_text(size= 9, colour = "black", vjust = -0.1), 
        axis.ticks = element_line(colour = "black"),
        axis.text.y = element_text(size = 9, colour = "black"))+
  theme(legend.position = "bottom", legend.title = element_text(size = 8.5), axis.text = element_text(size = 8.5))+
  facet_wrap(~factor(Parameter, levels=unique(anova_results_int_tuk[[1]]$Parameter)), scales = "free", ncol = 5) +
  ylab("log2")+
  geom_text(x = Inf, y = -Inf, 
            aes(label = paste("p=",`p-value group:sex`)),  
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

![](clinical_chem_files/figure-gfm/unnamed-chunk-8-1.png)<!-- -->

``` r
ggsave("anova_interactions_clinpar.svg", width = 3.5, height = 3.5)
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
bar_chart_fn(data_statistics = anova_results %>% 
                                   filter(`p-value group (PHG/PNG)`<= 0.1),
                              numeric_data = data_stat,
                              point_size = 2,
                              strip_text_size = 9,
                              n_widht = 0.7, 
                              jitt_widht = 0.05,
                              id_name = "Parameter", 
                              conditions_data = conditions)+
          ylab("")+
          geom_text(x = Inf, y = -Inf, 
          aes(label = paste("p=", `p-value group (PHG/PNG)`)),  
            size = 3, 
            hjust = -1.5, 
            angle = 90,
            vjust = -15, 
            data = anova_results %>% 
            filter(`p-value group (PHG/PNG)`<= 0.1) %>% 
            mutate(across(where(is.numeric), round, 3)), inherit.aes = F)+
  facet_wrap(~factor(Parameter, levels=c("Bilirubin (mg/dl)", "Albumin (g/dl)", 
                                         "NEFA (mmol/l)", "Glycerol (mmol/l)", "TG (mg/dl)", "ALT (U/l)")), scales = "free_y", ncol = 2)+
  guides(shape = guide_legend(order = 2, override.aes = list(stroke = 1, shape  = c(0,1))),
         fill = guide_legend(order = 1))+
  theme(legend.position = "bottom")+
  theme(plot.margin = margin(1,1,2,-3, "mm"))
```

    ## Joining, by = "Parameter"
    ## Joining, by = "Bioreplicate"
    ## `summarise()` has grouped output by 'Parameter'. You can override using the
    ## `.groups` argument.
    ## Joining, by = "Parameter"
    ## Joining, by = "Bioreplicate"
    ## Joining, by = c("Parameter", "Group")

![](clinical_chem_files/figure-gfm/unnamed-chunk-10-1.png)<!-- -->

``` r
ggsave("clinicalparameters.svg", width = 3.5, height = 5)
```

### save data in a supplementary tables

``` r
if (!file.exists("Supplementary table 4.xlsx")) {
  
# clinical chemical data (all)
write.xlsx(as.data.frame(clinical_chem_data), file = "Supplementary table 4.xlsx", sheetName = "Suppl table 4A", 
  col.names = TRUE, row.names = FALSE, append = T)
  
# anova results (all)
write.xlsx(as.data.frame(anova_results), file = "Supplementary table 4.xlsx", sheetName = "Suppl table 4B", 
  col.names = TRUE, row.names = FALSE, append = T)

# anova results (group)
write.xlsx(as.data.frame(anova_results_group), file = "Supplementary table 4.xlsx", sheetName = "Suppl table 4C", 
  col.names = TRUE, row.names = FALSE, append = T)

# anova results (sex)
write.xlsx(as.data.frame(anova_results_sex), file = "Supplementary table 4.xlsx", sheetName = "Suppl table 4D", 
  col.names = TRUE, row.names = FALSE, append = T)

# anova results (interaction) 
write.xlsx(as.data.frame(anova_results_int_tuk[[1]]), file = "Supplementary table 4.xlsx", sheetName = "Suppl table 4E", 
  col.names = TRUE, row.names = FALSE, append = T)
}
```

``` r
save.image()
```
