# ChIP-Seq Differential Peaks
Stephen Kelly  
9/13/2016  



# DiffBind 

DiffBind [@DiffBind] is used to determine which peaks in a ChIP-Seq experiment are differential bound between sample data sets. For this report, we have subset the standard DiffBind data to plot only the single peak per gene which is closest to the gene's start site (e.g. lowest 'shortestDistance' value).



```r
# function for formatting text in the report
mycat <- function(text){
    cat(gsub(pattern = "\n", replacement = "  \n", x = text))
}

# function to process the DiffBind data
Diff_process_data <- function(diff_df, signif_p_value=0.05,
                              fold_change_value=1, 
                              top_gene_number=10){
    
    # browser() # use this for debugging
    suppressPackageStartupMessages(library("data.table"))
    # this function makes the custom DiffBind volcano plots
    # only plots protein coding promoter genes from DiffBind output
    # highlights top x fold changed genes
    
    
    # ~~~~~~ PROCESS DATA ~~~~~~~~ # 
    # get the colname order for later
    original_colnames <- colnames(diff_df)
    
    
    # GET THE CLOSEST DIFFBIND PEAK PER GENE 
    # in Diffbind subset, getthe peaks closest to each gene
    # # for each gene name, find the peak with the lowest "shortestDistance" value
    # make it a data.table type
    setDT(diff_df)
    # get min value per factor, also remove duplicates
    diff_df_min <- as.data.frame(diff_df[, .SD[which.min(shortestDistance)], by=external_gene_name])
    # fix the colname order
    diff_df_min <- diff_df_min[,original_colnames]
    
    # get sample ID's from 7th, 8th colnames
    sampleID_1 <- colnames(diff_df_min)[7]
    sampleID_2 <- colnames(diff_df_min)[8]
    
    
    # subset for significant genes
    diff_df_min_signiff <- as.data.frame(subset(diff_df_min, p.value<signif_p_value & abs(Fold)>fold_change_value))
    
    # subset for protein coding genes
    diff_df_min_signiff_protein <- as.data.frame(subset(diff_df_min_signiff, gene_biotype=="protein_coding"))
    
    # subset top up/down protein coding genes
    diff_df_min_signiff_UpFold <- diff_df_min_signiff_protein[with(diff_df_min_signiff_protein, 
                                                                   order(Fold, p.value)), 
                                                              c("Fold","p.value","external_gene_name")][1:top_gene_number,]
    diff_df_min_signiff_DnFold <- diff_df_min_signiff_protein[with(diff_df_min_signiff_protein, 
                                                                   order(-Fold, p.value)), 
                                                              c("Fold","p.value","external_gene_name")][1:top_gene_number,]
    # ~~~~~~~~~~~~~~ # 
    
    
    
    
    diff_df_min <-diff_df_min[c("external_gene_name", "gene_biotype", "Fold", "p.value")]
    diff_df_min["group"] <- "not_signif"
    diff_df_min[which(diff_df_min['p.value'] < signif_p_value & abs(diff_df_min['Fold']) < fold_change_value ),"group"] <- "signif"
    diff_df_min[which(diff_df_min['p.value'] > signif_p_value & abs(diff_df_min['Fold']) > fold_change_value ),"group"] <- "fc"
    diff_df_min[which(diff_df_min['p.value'] < signif_p_value & abs(diff_df_min['Fold']) > fold_change_value ),"group"] <- "signif_fc"
        # browser() # use this for debugging

    
    suppressPackageStartupMessages(library(dplyr))

    
    diff_df_min <- diff_df_min %>%
        arrange(desc(gene_biotype == "protein_coding"), desc(Fold), p.value) %>% 
        mutate(group = ifelse(row_number() < top_gene_number + 1 & abs(Fold) > fold_change_value, "top_signif_fc", group))
    
    diff_df_min <- diff_df_min %>%
        arrange(desc(gene_biotype == "protein_coding"), Fold, p.value) %>% 
        mutate(group = ifelse(row_number() < top_gene_number + 1 & abs(Fold) > fold_change_value, "top_signif_fc", group))
    
    return(diff_df_min)
    
}


# Function for Plot.ly volcano plot
DiffBind_volcano_plotly_top_protein_coding_promoters <- function(diff_df, plot_colors=c("red","gray","orange", "blue", "green"), ... ){
    
    
    # diff_df_min <- Diff_process_data(diff_df)
    diff_df_min <- diff_df

    # ~~~~~~ MAKE PLOT ~~~~~~~~ # 
    mycat('### Volcano Plot: Plot.ly\n')
    suppressPackageStartupMessages(library("plotly"))
    
    plot_ly(data = diff_df_min, x = Fold, y = -log10(p.value), text = external_gene_name, mode = "markers", color = as.ordered(group), colors = plot_colors)
    
    # my_plot <- # can also save it to an object to be returned
    # print(my_plot)
    # print(htmltools::tagList(list(as.widget(my_plot))))
    # return(my_plot)
    
    
}


Diff_stats <- function(diff_df, signif_p_value=0.05, fold_change_value=1, top_gene_number=10, ... ){
    
    # process the DiffData
    diff_df_min <- Diff_process_data(diff_df)
    
    # print some gene table stats into the report; this could use optimization
    mycat('\n\n')
    mycat('\n******\n')
    # FULL DATASET STATS
    mycat('### Overall DiffBind Stats\n')
    mycat(paste0('Total number of DiffBind peaks:\n', length(as.data.frame(diff_df)[["external_gene_name"]]), '\n\n'))
    mycat(paste0('Total number of DiffBind genes:\n', length(unique(diff_df_min[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0('Total number positive fold change genes:\n', 
                 length(unique(subset(diff_df_min, Fold > 0 )[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0('Total number negative fold change genes:\n', 
                 length(unique(subset(diff_df_min, Fold < 0 )[["external_gene_name"]])), '\n\n'))
    
    # significant 
    mycat('\n******\n')
    mycat(paste0('Total number of p <' ,signif_p_value,' genes:\n', 
                 length(unique(subset(diff_df_min, p.value<signif_p_value )[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0('Total number of p <' ,signif_p_value,' genes (pos. FC):\n', 
                 length(unique(subset(diff_df_min, p.value<signif_p_value &
                                          Fold > 0)[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0('Total number of p <' ,signif_p_value,' genes (neg. FC):\n', 
                 length(unique(subset(diff_df_min, p.value<signif_p_value &
                                          Fold < 0)[["external_gene_name"]])), '\n\n'))
    
    # fold change
    mycat('\n******\n')
    mycat(paste0("Total number of log2(Fold Change) > ",fold_change_value, ' genes:\n',
                 length(unique(subset(diff_df_min, abs(Fold)>fold_change_value)[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0("Total number of log2(Fold Change) > ",fold_change_value, ' genes (pos. FC):\n',
                 length(unique(subset(diff_df_min, abs(Fold)>fold_change_value & 
                                          Fold > 0)[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0("Total number of log2(Fold Change) > ",fold_change_value, ' genes (neg. FC):\n',
                 length(unique(subset(diff_df_min, abs(Fold)>fold_change_value & 
                                          Fold < 0)[["external_gene_name"]])), '\n\n'))
    
    # siginificant & fold change
    mycat('\n******\n')
    mycat(paste0("Total number of p < ",signif_p_value," & log2(Fold Change) > ",fold_change_value, ' genes:\n',
                 length(unique(subset(diff_df_min, p.value<signif_p_value &
                                          abs(Fold)>fold_change_value)[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0("Total number of p < ",signif_p_value," & log2(Fold Change) > ",fold_change_value, ' genes (pos. FC):\n',
                 length(unique(subset(diff_df_min, p.value<signif_p_value & 
                                          Fold > 0 &
                                          abs(Fold)>fold_change_value)[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0("Total number of p < ",signif_p_value," & log2(Fold Change) > ",fold_change_value, ' genes (neg. FC):\n',
                 length(unique(subset(diff_df_min, p.value<signif_p_value & 
                                          Fold < 0 &
                                          abs(Fold)>fold_change_value)[["external_gene_name"]])), '\n\n'))
    
    
    mycat('\n\n')
    mycat('\n******\n')
    # ONLY PROTEIN CODING GENES
    mycat('### Protein Coding Gene Stats\n') # gene_biotype=="protein_coding"
    mycat(paste0('Total number of DiffBind peaks:\n', 
                 length(subset(as.data.frame(diff_df), gene_biotype=="protein_coding")[["external_gene_name"]]), '\n\n'))
    mycat(paste0('Total number of DiffBind genes:\n', 
                 length(unique(subset(diff_df_min, gene_biotype=="protein_coding")[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0('Total number positive fold change genes:\n', 
                 length(unique(subset(diff_df_min, gene_biotype=="protein_coding" & 
                                          Fold > 0 )[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0('Total number negative fold change genes:\n', 
                 length(unique(subset(diff_df_min, gene_biotype=="protein_coding" &
                                          Fold < 0 )[["external_gene_name"]])), '\n\n'))
    
    # significant 
    mycat('\n******\n')
    mycat(paste0('Total number of p <' ,signif_p_value,' genes:\n', 
                 length(unique(subset(diff_df_min, gene_biotype=="protein_coding" &
                                          p.value<signif_p_value )[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0('Total number of p <' ,signif_p_value,' genes (pos. FC):\n', 
                 length(unique(subset(diff_df_min, gene_biotype=="protein_coding" &
                                          p.value<signif_p_value &
                                          Fold > 0)[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0('Total number of p <' ,signif_p_value,' genes (neg. FC):\n', 
                 length(unique(subset(diff_df_min, gene_biotype=="protein_coding" &
                                          p.value<signif_p_value &
                                          Fold < 0)[["external_gene_name"]])), '\n\n'))
    
    # fold change
    mycat('\n******\n')
    mycat(paste0("Total number of log2(Fold Change) > ",fold_change_value, ' genes:\n',
                 length(unique(subset(diff_df_min, gene_biotype=="protein_coding" &
                                          abs(Fold)>fold_change_value)[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0("Total number of log2(Fold Change) > ",fold_change_value, ' genes (pos. FC):\n',
                 length(unique(subset(diff_df_min, gene_biotype=="protein_coding" &
                                          abs(Fold)>fold_change_value & 
                                          Fold > 0)[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0("Total number of log2(Fold Change) > ",fold_change_value, ' genes (neg. FC):\n',
                 length(unique(subset(diff_df_min, gene_biotype=="protein_coding" &
                                          abs(Fold)>fold_change_value & 
                                          Fold < 0)[["external_gene_name"]])), '\n\n'))
    
    # siginificant & fold change
    mycat('\n******\n')
    mycat(paste0("Total number of p < ",signif_p_value," & log2(Fold Change) > ",fold_change_value, ' genes:\n',
                 length(unique(subset(diff_df_min, gene_biotype=="protein_coding" &
                                          p.value<signif_p_value &
                                          abs(Fold)>fold_change_value)[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0("Total number of p < ",signif_p_value," & log2(Fold Change) > ",fold_change_value, ' genes (pos. FC):\n',
                 length(unique(subset(diff_df_min, gene_biotype=="protein_coding" &
                                          p.value<signif_p_value & 
                                          Fold > 0 &
                                          abs(Fold)>fold_change_value)[["external_gene_name"]])), '\n\n'))
    
    mycat(paste0("Total number of p < ",signif_p_value," & log2(Fold Change) > ",fold_change_value, ' genes (neg. FC):\n',
                 length(unique(subset(diff_df_min, gene_biotype=="protein_coding" &
                                          p.value<signif_p_value & 
                                          Fold < 0 &
                                          abs(Fold)>fold_change_value)[["external_gene_name"]])), '\n\n'))
    
    
    
    mycat('\n')
    mycat('\n******\n')
    mycat('\n******\n')
    
}

# function for the base R plots:
DiffBind_volcano_plot_top_protein_coding_promoters <- function(diff_df, signif_p_value=0.05, 
                                                               fold_change_value=1, 
                                                               plot_colors=c("grey","red","orange","blue", "green"), 
                                                               top_gene_number=10){
    # browser() # use this for debugging
    
    # this function makes the custom DiffBind volcano plots
    # only plots protein coding promoter genes from DiffBind output
    # highlights top x fold changed genes
    
    
    # ~~~~~~ PROCESS DATA ~~~~~~~~ # 
    # get the colname order for later
    original_colnames <- colnames(diff_df)
    
    suppressPackageStartupMessages(library("data.table"))
    # GET THE CLOSEST DIFFBIND PEAK PER GENE 
    # in Diffbind subset, getthe peaks closest to each gene
    # # for each gene name, find the peak with the lowest "shortestDistance" value
    # make it a data.table type
    setDT(diff_df)
    # get min value per factor, also remove duplicates
    diff_df_min <- as.data.frame(diff_df[, .SD[which.min(shortestDistance)], by=external_gene_name])
    # fix the colname order
    diff_df_min <- diff_df_min[,original_colnames]
    
    # get sample ID's from 7th, 8th colnames
    sampleID_1 <- colnames(diff_df_min)[7]
    sampleID_2 <- colnames(diff_df_min)[8]
    
    
    # subset for significant genes
    diff_df_min_signiff <- as.data.frame(subset(diff_df_min, p.value<signif_p_value & abs(Fold)>fold_change_value))
    
    # subset for protein coding genes
    diff_df_min_signiff_protein <- as.data.frame(subset(diff_df_min_signiff, gene_biotype=="protein_coding"))
    
    # subset top up/down protein coding genes
    diff_df_min_signiff_UpFold <- diff_df_min_signiff_protein[with(diff_df_min_signiff_protein, 
                                                                   order(Fold, p.value)), 
                                                              c("Fold","p.value","external_gene_name")][1:top_gene_number,]
    diff_df_min_signiff_DnFold <- diff_df_min_signiff_protein[with(diff_df_min_signiff_protein, 
                                                                   order(-Fold, p.value)), 
                                                              c("Fold","p.value","external_gene_name")][1:top_gene_number,]
    # ~~~~~~~~~~~~~~ # 
    
    # suppressPackageStartupMessages(library("plotly"))
    # browser() # use this for debugging
    
    # ~~~~~~ MAKE PLOT ~~~~~~~~ # 
    mycat('### Volcano Plot: base R\n')
    # set the multi-pane plot layout matrix
    plot_layout_matrix<-structure(c(1L, 2L, 2L, 2L, 2L, 2L, 3L, 3L, 3L, 3L, 
                                    3L, 1L, 2L, 2L, 2L, 2L, 2L, 3L, 3L, 3L, 
                                    3L, 3L, 1L, 2L, 2L, 2L, 2L, 2L, 3L, 3L, 
                                    3L, 3L, 3L, 1L, 2L, 2L, 2L, 2L, 2L, 3L, 
                                    3L, 3L, 3L, 3L), 
                                  .Dim = c(11L,4L), 
                                  .Dimnames = list(NULL, c("V1", "V2", "V3", "V4")))
    
    # subset the matrix for two panels # keep this because I like this code sample
    plot_layout_matrix2 <- plot_layout_matrix[(rowSums(plot_layout_matrix < 3) > 0), , drop = FALSE]
    # x[which(x < 3, arr.ind = TRUE)[,1],]
    
    # SET PLOT LAYOUT
    layout(plot_layout_matrix2) 
    
    
    # FIRST PLOT PANEL
    # adjust first panel margins; these need to be adjusted to intended plot size
    # currently configured for 8 inch x 8 inch plota
    # plot margins # c(bottom, left, top, right) # default is c(5, 4, 4, 2) + 0.1
    par(mar=c(1,3,1,0))
    # call blank plot to fill the first panel
    plot(1,type='n',axes=FALSE,xlab="",ylab="")
    # set up the Legend in the first panel; 1st legend only title, 2nd legend only legend
    legend("center",legend = "",title=paste0("Volcano plot: ", sampleID_1, " vs. ",sampleID_2,"\n"),cex=1.3, bty='n') 
    legend("bottom",legend=c("Not Significant",
                             paste0("p < ",signif_p_value),
                             paste0("log2(Fold Change) > ",fold_change_value),
                             paste0("p < ",signif_p_value," & log2(Fold Change) > ",fold_change_value),
                             paste0("Top log2(Fold Change) genes")),
           fill=plot_colors,bty = "n",ncol=2,cex=0.9)
    
    
    
    # SECOND PLOT PANEL
    # adjust second panel margins
    par(mar=c(6,4,0,3)+ 0.1)
    
    # add base volcano plot; all points
    with(diff_df_min, plot(Fold, -log10(p.value), pch=20, xlim=c(min(Fold)-1,max(Fold)+1),col=plot_colors[1],xlab = "log2(Fold Change)"))
    
    # Add colored points for data subsets
    with(subset(diff_df_min, p.value<signif_p_value ), points(Fold, -log10(p.value), pch=20, col=plot_colors[2]))
    with(subset(diff_df_min, abs(Fold)>fold_change_value), points(Fold, -log10(p.value), pch=20, col=plot_colors[3]))
    with(subset(diff_df_min, p.value<signif_p_value & abs(Fold)>fold_change_value), points(Fold, -log10(p.value), pch=20, col=plot_colors[4]))
    
    # add points and gene labels for top genes
    suppressPackageStartupMessages(library("calibrate"))
    suppressPackageStartupMessages(library("reshape2"))
    # up genes
    with(na.omit(diff_df_min_signiff_UpFold), points(Fold, -log10(p.value), pch=20, col="green"))
    with(na.omit(diff_df_min_signiff_UpFold), textxy(Fold, -log10(p.value), labs=external_gene_name, cex=.8))
    # down genes
    with(na.omit(diff_df_min_signiff_DnFold), points(Fold, -log10(p.value), pch=20, col="green"))
    with(na.omit(diff_df_min_signiff_DnFold), textxy(Fold, -log10(p.value), labs=external_gene_name, cex=.8))
    
    
    
}
```



```r
#
suppressPackageStartupMessages(library("plotly"))
mycat('# Plots and Results {.tabset}\n\n') # .tabset-fade .tabset-pills
```

# Plots and Results {.tabset}  
  

```r
# out_dir <- "/ifs/home/kellys04/projects/Bioinformatics/DiffBind_scripts_reports/DiffBind_Volcano_Plot_report/input"
out_dir <- "/Users/kellys04/Bioinformatics/DiffBind_scripts_reports/DiffBind_Volcano_Plot_report/input"

# mycat(paste0("Project dir:\n",out_dir,'\n'))
# mycat('\n******\n')

diff_cl4_ctrl_file <- paste0(out_dir,"/diff_bind.Treatment4-ChIPSeq-vs-Control-ChIPSeq.p100.csv")
diff_cl4_cl5_file <- paste0(out_dir,"/diff_bind.Treatment4-ChIPSeq-vs-Treatment5-ChIPSeq.p100.csv")
diff_cl5_ctrl_file <- paste0(out_dir,"/diff_bind.Treatment5-ChIPSeq-vs-Control-ChIPSeq.p100.csv")

sample_file_list <- setNames(c(diff_cl4_ctrl_file, diff_cl4_cl5_file, diff_cl5_ctrl_file),
                             c("Sample4_Control", "Sample4_Sample5", "Sample5_Control"))


 
sample_file <- sample_file_list[1]
mycat(paste0("## ", names(sample_file), ' {.tabset}\n'))
```

## Sample4_Control {.tabset}  

```r
diff_df <- read.delim(file = sample_file,header = TRUE,sep = ',')
DiffBind_volcano_plotly_top_protein_coding_promoters(diff_df = Diff_process_data(diff_df))
```

### Volcano Plot: Plot.ly  
<!--html_preserve--><div id="htmlwidget-ffc5cb17a313996f77a2" style="width:768px;height:768px;" class="plotly html-widget"></div>
<script type="application/json" data-for="htmlwidget-ffc5cb17a313996f77a2">{"x":{"data":[{"type":"scatter","inherit":false,"x":[-6.2,-6.1,-6.09,-5.55,-5.52,-5.33,-5.31,-5.3,-5.27,-5.26,4.97,4.98,4.99,5.03,5.14,5.27,5.31,5.32,5.43,5.46],"y":[14.7189666327523,14.2118316288588,14.197226274708,11.4259687322723,11.2388241868443,10.3053948010664,10.2218487496164,10.2125395254816,9.99139982823808,10.0078885122131,11.0034883278458,8.49620931694282,8.54821356447571,8.73992861201492,14.4621809049267,9.89962945488244,9.99567862621736,10.0883098412461,10.6819366650372,13.4634415574285],"text":["UQCC1","PMEPA1","CHD1L","C8orf46","ADPGK","CLDN11","FAM78B","FCRLA","LSAMP","TNIK","CSPG5","RNF180","ARHGAP10","DDX3Y","SGK223","HEPACAM","RRP8","PITPNC1","KCNA2","CROCC"],"mode":"markers","name":"top_signif_fc","marker":{"color":"#00FF00"}},{"type":"scatter","inherit":false,"x":[-5.2,-5.11,-5.06,-5.05,-5.04,-5.03,-5.02,-4.94,-4.94,-4.87,-4.84,-4.81,-4.78,-4.76,-4.69,-4.67,-4.64,-4.62,-4.56,-4.56,-4.56,-4.55,-4.49,-4.48,-4.47,-4.42,-4.42,-4.32,-4.31,-4.31,-4.31,-4.29,-4.29,-4.28,-4.27,-4.26,-4.26,-4.25,-4.23,-4.21,-4.19,-4.18,-4.18,-4.17,-4.15,-4.15,-4.12,-4.09,-4.08,-4.08,-4.07,-4.07,-4.06,-4.04,-4.04,-4.03,-3.99,-3.98,-3.97,-3.95,-3.91,-3.9,-3.88,-3.87,-3.84,-3.82,-3.79,-3.79,-3.79,-3.79,-3.77,-3.76,-3.76,-3.76,-3.74,-3.72,-3.71,-3.67,-3.67,-3.64,-3.63,-3.62,-3.62,-3.61,-3.61,-3.59,-3.59,-3.58,-3.57,-3.56,-3.56,-3.54,-3.52,-3.52,-3.51,-3.5,-3.47,-3.44,-3.44,-3.43,-3.41,-3.4,-3.37,-3.36,-3.35,-3.33,-3.31,-3.31,-3.22,-3.21,-3.2,-3.17,-3.14,-3.13,-3.07,-3.06,-3.04,-3.02,-3.01,-3,-3,-2.99,-2.96,-2.95,-2.95,-2.91,-2.86,-2.86,-2.86,-2.82,-2.81,-2.81,-2.8,-2.75,-2.73,-2.72,-2.71,-2.69,-2.68,-2.65,-2.65,-2.63,-2.62,-2.56,-2.54,-2.54,-2.53,-2.49,-2.48,-2.48,-2.47,-2.42,-2.42,-2.32,-2.3,-2.09,-1.99,-1.93,-1.87,-1.84,-1.76,-1.72,-1.67,-1.6,-1.53,-1.39,-1.37,-1.13,-1.11,-1.08,-1.04,1.02,1.07,1.14,1.14,1.15,1.15,1.16,1.17,1.19,1.2,1.2,1.22,1.22,1.23,1.24,1.24,1.24,1.25,1.27,1.28,1.28,1.29,1.31,1.31,1.32,1.32,1.34,1.35,1.35,1.36,1.37,1.38,1.38,1.39,1.39,1.41,1.43,1.44,1.44,1.45,1.46,1.47,1.48,1.48,1.5,1.5,1.5,1.51,1.52,1.53,1.54,1.54,1.55,1.56,1.56,1.57,1.58,1.58,1.58,1.59,1.59,1.6,1.61,1.63,1.64,1.64,1.65,1.65,1.66,1.66,1.66,1.66,1.68,1.7,1.7,1.7,1.7,1.71,1.72,1.73,1.73,1.74,1.74,1.74,1.77,1.77,1.78,1.78,1.79,1.79,1.79,1.8,1.82,1.82,1.82,1.85,1.85,1.87,1.87,1.88,1.88,1.88,1.89,1.89,1.9,1.9,1.9,1.9,1.91,1.92,1.92,1.93,1.93,1.93,1.94,1.95,1.95,1.95,1.95,1.95,1.96,1.96,1.96,1.97,1.97,1.97,1.97,1.97,2,2.01,2.01,2.02,2.02,2.02,2.02,2.03,2.03,2.03,2.04,2.04,2.05,2.06,2.07,2.08,2.08,2.08,2.08,2.08,2.08,2.08,2.09,2.09,2.09,2.1,2.1,2.1,2.11,2.11,2.11,2.11,2.11,2.11,2.12,2.12,2.12,2.13,2.13,2.13,2.13,2.13,2.15,2.16,2.16,2.16,2.17,2.17,2.17,2.17,2.18,2.19,2.19,2.19,2.2,2.22,2.22,2.22,2.22,2.22,2.23,2.24,2.25,2.25,2.26,2.26,2.26,2.26,2.26,2.27,2.27,2.27,2.28,2.29,2.29,2.29,2.29,2.31,2.31,2.31,2.31,2.32,2.32,2.33,2.33,2.33,2.34,2.34,2.34,2.35,2.36,2.36,2.37,2.37,2.37,2.38,2.38,2.38,2.39,2.39,2.39,2.39,2.39,2.4,2.4,2.4,2.4,2.4,2.41,2.41,2.42,2.42,2.43,2.43,2.43,2.43,2.44,2.44,2.44,2.44,2.44,2.44,2.44,2.44,2.45,2.45,2.45,2.45,2.46,2.46,2.47,2.47,2.48,2.48,2.49,2.49,2.49,2.5,2.51,2.51,2.52,2.52,2.52,2.52,2.52,2.52,2.52,2.54,2.54,2.55,2.55,2.55,2.55,2.55,2.56,2.57,2.57,2.57,2.57,2.58,2.58,2.59,2.59,2.6,2.6,2.6,2.6,2.6,2.62,2.62,2.62,2.62,2.62,2.63,2.63,2.64,2.64,2.65,2.66,2.66,2.67,2.67,2.67,2.67,2.67,2.68,2.71,2.72,2.72,2.72,2.72,2.73,2.73,2.73,2.74,2.74,2.75,2.75,2.76,2.76,2.77,2.78,2.78,2.79,2.79,2.79,2.8,2.8,2.8,2.81,2.82,2.82,2.83,2.83,2.83,2.83,2.83,2.83,2.84,2.84,2.85,2.85,2.86,2.86,2.86,2.86,2.86,2.87,2.87,2.88,2.88,2.89,2.89,2.89,2.89,2.89,2.91,2.91,2.91,2.91,2.92,2.92,2.93,2.93,2.93,2.93,2.93,2.94,2.94,2.94,2.95,2.95,2.95,2.95,2.95,2.96,2.96,2.96,2.96,2.96,2.96,2.97,2.97,2.98,2.99,2.99,2.99,3,3,3,3,3.01,3.01,3.01,3.01,3.02,3.02,3.02,3.03,3.03,3.03,3.04,3.04,3.05,3.05,3.05,3.05,3.06,3.06,3.06,3.07,3.07,3.08,3.08,3.09,3.09,3.09,3.1,3.11,3.11,3.12,3.12,3.13,3.14,3.15,3.15,3.16,3.17,3.17,3.18,3.19,3.19,3.2,3.22,3.22,3.22,3.22,3.22,3.22,3.22,3.22,3.23,3.23,3.23,3.23,3.24,3.24,3.25,3.26,3.26,3.26,3.26,3.26,3.26,3.26,3.27,3.27,3.28,3.28,3.28,3.29,3.29,3.3,3.3,3.31,3.33,3.33,3.34,3.34,3.34,3.34,3.34,3.35,3.35,3.37,3.37,3.37,3.37,3.37,3.37,3.38,3.39,3.39,3.39,3.4,3.41,3.41,3.42,3.42,3.43,3.43,3.43,3.43,3.43,3.43,3.44,3.44,3.44,3.45,3.45,3.45,3.45,3.46,3.46,3.46,3.46,3.47,3.47,3.47,3.48,3.48,3.48,3.48,3.48,3.49,3.49,3.49,3.5,3.5,3.5,3.51,3.51,3.51,3.51,3.52,3.53,3.53,3.53,3.53,3.53,3.54,3.54,3.54,3.54,3.54,3.54,3.55,3.55,3.56,3.57,3.57,3.57,3.57,3.58,3.58,3.58,3.58,3.58,3.58,3.59,3.59,3.59,3.59,3.6,3.6,3.61,3.61,3.62,3.62,3.62,3.63,3.63,3.64,3.64,3.64,3.64,3.65,3.65,3.65,3.66,3.66,3.66,3.67,3.67,3.67,3.67,3.68,3.68,3.69,3.69,3.69,3.69,3.7,3.7,3.71,3.71,3.71,3.71,3.72,3.73,3.73,3.74,3.74,3.74,3.75,3.75,3.76,3.78,3.78,3.78,3.78,3.78,3.79,3.79,3.79,3.8,3.8,3.8,3.81,3.81,3.81,3.81,3.82,3.82,3.82,3.82,3.83,3.83,3.83,3.84,3.84,3.84,3.85,3.85,3.85,3.86,3.86,3.86,3.86,3.86,3.86,3.87,3.87,3.88,3.88,3.88,3.88,3.89,3.89,3.89,3.89,3.9,3.9,3.9,3.9,3.9,3.9,3.9,3.91,3.92,3.92,3.94,3.94,3.94,3.95,3.96,3.96,3.98,3.99,3.99,3.99,3.99,3.99,4.01,4.01,4.02,4.02,4.02,4.03,4.03,4.03,4.04,4.04,4.04,4.05,4.05,4.05,4.06,4.06,4.06,4.06,4.06,4.07,4.07,4.07,4.07,4.07,4.08,4.08,4.08,4.08,4.08,4.08,4.08,4.09,4.09,4.09,4.1,4.11,4.11,4.12,4.12,4.12,4.13,4.13,4.13,4.14,4.14,4.15,4.17,4.17,4.17,4.17,4.17,4.18,4.18,4.19,4.2,4.21,4.21,4.21,4.21,4.22,4.23,4.23,4.24,4.24,4.26,4.26,4.26,4.27,4.27,4.27,4.28,4.28,4.28,4.29,4.29,4.29,4.33,4.34,4.35,4.35,4.37,4.38,4.39,4.41,4.42,4.43,4.43,4.43,4.43,4.46,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.49,4.54,4.54,4.55,4.56,4.56,4.56,4.57,4.58,4.58,4.59,4.6,4.61,4.62,4.62,4.69,4.69,4.69,4.7,4.71,4.74,4.74,4.77,4.78,4.79,4.8,4.81,4.85,4.86,4.87,4.87,4.91,4.95,4.96,-6.39,-5.93,-5.65,-5.61,-5.35,-5.21,-5.19,-5.18,-5.14,-5,-4.95,-4.94,-4.88,-4.87,-4.86,-4.86,-4.81,-4.81,-4.8,-4.77,-4.76,-4.75,-4.75,-4.73,-4.72,-4.63,-4.59,-4.59,-4.59,-4.57,-4.55,-4.55,-4.54,-4.53,-4.52,-4.49,-4.48,-4.47,-4.47,-4.42,-4.42,-4.39,-4.37,-4.35,-4.33,-4.32,-4.3,-4.28,-4.21,-4.2,-4.15,-4.14,-4.13,-4.13,-4.13,-4.11,-4.09,-4.08,-4.08,-4.07,-4.07,-4.03,-4.03,-4.03,-4.03,-4.02,-4,-3.96,-3.94,-3.93,-3.93,-3.92,-3.83,-3.79,-3.75,-3.71,-3.71,-3.68,-3.67,-3.65,-3.64,-3.63,-3.62,-3.61,-3.6,-3.58,-3.57,-3.56,-3.56,-3.56,-3.55,-3.51,-3.5,-3.46,-3.42,-3.41,-3.37,-3.35,-3.34,-3.32,-3.32,-3.31,-3.27,-3.24,-3.23,-3.18,-3.15,-3.14,-3.14,-3.13,-3.12,-3.11,-3.11,-3.05,-2.99,-2.96,-2.96,-2.93,-2.92,-2.88,-2.88,-2.86,-2.8,-2.71,-2.7,-2.65,-2.62,-2.53,-2.53,-2.5,-2.43,-2.42,-2.4,-2.25,-2.19,-2.18,-2.05,-2,-1.9,-1.77,-1.74,-1.67,-1.67,-1.65,-1.52,-1.36,-1.36,-1.28,-1.27,1.1,1.11,1.16,1.19,1.2,1.2,1.22,1.22,1.27,1.27,1.28,1.28,1.28,1.29,1.29,1.31,1.31,1.31,1.34,1.36,1.38,1.4,1.4,1.4,1.43,1.44,1.45,1.45,1.46,1.5,1.51,1.51,1.51,1.52,1.54,1.54,1.54,1.54,1.55,1.58,1.6,1.6,1.61,1.65,1.69,1.71,1.74,1.76,1.78,1.79,1.79,1.8,1.81,1.83,1.83,1.85,1.85,1.89,1.92,1.93,1.93,1.97,1.98,1.99,1.99,1.99,2,2,2.01,2.02,2.05,2.06,2.06,2.07,2.11,2.11,2.11,2.11,2.12,2.12,2.13,2.13,2.14,2.15,2.16,2.16,2.16,2.18,2.2,2.21,2.22,2.22,2.24,2.26,2.27,2.27,2.28,2.28,2.29,2.31,2.31,2.31,2.33,2.33,2.34,2.35,2.35,2.36,2.37,2.38,2.39,2.4,2.4,2.4,2.41,2.41,2.41,2.42,2.42,2.43,2.45,2.46,2.47,2.47,2.47,2.48,2.48,2.5,2.5,2.54,2.54,2.54,2.54,2.56,2.56,2.57,2.58,2.58,2.6,2.6,2.61,2.61,2.61,2.62,2.62,2.62,2.62,2.62,2.63,2.63,2.63,2.64,2.64,2.66,2.67,2.67,2.68,2.68,2.71,2.71,2.72,2.72,2.73,2.74,2.74,2.75,2.76,2.81,2.82,2.83,2.84,2.84,2.85,2.86,2.86,2.87,2.87,2.87,2.88,2.89,2.9,2.91,2.91,2.91,2.92,2.92,2.92,2.94,2.94,2.95,2.95,2.96,2.96,2.96,2.96,2.96,2.96,2.97,2.97,2.99,2.99,2.99,3,3,3.01,3.01,3.02,3.02,3.03,3.03,3.04,3.04,3.05,3.06,3.08,3.09,3.09,3.1,3.12,3.13,3.14,3.15,3.16,3.16,3.16,3.17,3.17,3.17,3.18,3.19,3.19,3.19,3.19,3.21,3.22,3.23,3.23,3.23,3.24,3.24,3.24,3.24,3.25,3.25,3.26,3.26,3.27,3.27,3.28,3.28,3.3,3.3,3.3,3.31,3.31,3.31,3.32,3.32,3.32,3.32,3.32,3.32,3.33,3.33,3.33,3.34,3.34,3.34,3.34,3.34,3.34,3.35,3.36,3.38,3.38,3.38,3.38,3.38,3.39,3.4,3.4,3.4,3.4,3.4,3.41,3.42,3.43,3.44,3.45,3.45,3.45,3.46,3.46,3.47,3.47,3.48,3.48,3.48,3.48,3.49,3.49,3.51,3.51,3.51,3.51,3.51,3.53,3.53,3.53,3.54,3.56,3.56,3.56,3.56,3.57,3.57,3.58,3.58,3.58,3.59,3.59,3.59,3.61,3.61,3.61,3.61,3.61,3.61,3.62,3.62,3.62,3.62,3.63,3.63,3.63,3.64,3.64,3.65,3.65,3.66,3.66,3.67,3.67,3.68,3.68,3.7,3.71,3.71,3.72,3.73,3.73,3.74,3.74,3.74,3.74,3.74,3.76,3.77,3.77,3.77,3.78,3.78,3.79,3.79,3.79,3.79,3.8,3.81,3.81,3.81,3.81,3.82,3.83,3.83,3.83,3.83,3.84,3.84,3.84,3.85,3.85,3.86,3.86,3.87,3.88,3.88,3.88,3.88,3.89,3.9,3.9,3.91,3.93,3.93,3.93,3.94,3.94,3.94,3.94,3.96,3.97,3.98,3.99,3.99,4,4.02,4.02,4.02,4.02,4.02,4.03,4.03,4.03,4.05,4.05,4.05,4.05,4.05,4.06,4.07,4.07,4.07,4.07,4.07,4.08,4.08,4.09,4.09,4.09,4.09,4.1,4.1,4.1,4.11,4.11,4.11,4.12,4.15,4.15,4.15,4.16,4.16,4.16,4.16,4.16,4.17,4.17,4.18,4.18,4.18,4.18,4.19,4.2,4.21,4.21,4.21,4.22,4.23,4.23,4.24,4.25,4.25,4.25,4.25,4.25,4.26,4.27,4.27,4.29,4.29,4.3,4.31,4.31,4.32,4.32,4.35,4.35,4.35,4.38,4.43,4.47,4.55,4.58,4.59,4.59,4.59,4.62,4.62,4.63,4.63,4.66,4.66,4.67,4.7,4.71,4.71,4.71,4.71,4.73,4.75,4.78,4.79,4.84,4.87,4.93,4.94,4.95,4.95,4.98,5.03,5.04,5.06,5.07,5.12,5.19,5.19,5.28,5.49,5.86],"y":[9.59006687666871,11.7212463990472,9.01863449092146,14.966576244513,8.97881070093006,8.85698519974591,8.50723961097316,8.5003129173816,8.46980030179692,8.16685288808721,7.86966623150499,10.0268721464003,7.67366413907125,7.64206515299955,7.28988263488818,7.10182351650232,10.966576244513,7.014573525917,16.7447274948967,6.71444269099223,6.52143350440616,6.6345120151091,10.0245681914907,6.30364361126667,6.27572413039921,7.79317412396815,6.11690664142431,5.72353819582676,9.07727454200674,8.09044397075882,5.82681373158773,7.2020403562628,5.73754891026957,10.5767541260632,10.8632794328436,12.8326826652518,5.71219827006977,5.64975198166584,11.48280410205,9.63827216398241,11.1140736601986,8.78781239559604,5.37263414340727,6.58169870868025,10.8827287043442,5.37263414340727,6.32790214206428,7.64016451766011,6.2958494831602,4.90308998699194,6.19859628998265,5.01278077009199,10.5243288116756,6.61978875828839,6.1180450286604,10.279840696594,8.60906489289662,5.82102305270683,12.8153085691824,4.6345120151091,5.64397414280688,6.72353819582676,14.1830961606243,5.27490547891853,6.70774392864352,7.92811799269387,6.16685288808721,5.95078197732982,5.95078197732982,4.51999305704285,5.98716277529483,6.03432802877989,6.03432802877989,4.49894073778225,7.10237290870956,8.03479829897409,5.84771165561694,11.0560111249262,11.0560111249262,7.64397414280688,5.16178077809237,11.5718652059712,6.28903688100472,8.54975089168064,7.56703070912559,10.2189630613789,6.92811799269387,6.15366288787019,7.69897000433602,6.08144546944973,5.97469413473523,5.26121944151563,7.75202673363819,5.32605800136591,4.85078088734462,5.73518217699046,11.5883802940368,19.1186153432294,19.1186153432294,7.66154350639539,5.82681373158773,6.02872415126189,6.71444269099223,6.77728352885242,6.09908693226233,4.80410034759077,5.04624030826677,4.75448733218585,4.32422165832592,5.89279003035213,13.3726341434073,13.112945621949,4.77728352885242,4.16241156176449,14.3605135107314,15.8728952016352,5.20760831050175,7.28735029837279,6.72353819582676,6.83564714421556,4.25884840114821,3.73754891026957,7.89962945488244,5.48945498979339,4.59006687666871,5.57839607313017,13.3575354797579,8.02410886359821,4.00348832784582,4.86966623150499,4.4907974776689,4.22040350874218,2.74714696902011,9.13906337929991,7.14996674231023,8.51286162452281,6.59176003468815,6.45469288353418,4.82390874094432,4.94309514866353,4.24336389175415,3.87614835903291,6.57186520597121,5.09528445472132,7.21752737583371,7.21752737583371,5.19178902707578,6.77469071827414,6.80966830182971,3.67366413907125,6.38510278396687,6.85387196432176,6.85387196432176,4.88941028970075,13.1636758842932,6.70996538863748,3.39147396642281,2.99139982823808,3.03715731879876,9.77728352885242,3.72124639904717,4.58502665202918,5.38933983691012,3.95078197732982,4.57186520597121,1.75202673363819,2.20760831050175,2.73754891026957,2.52578373592374,2.34775365899668,3.21395878975745,1.39254497678533,2.71896663275227,3.74957999769111,1.57511836336893,1.54211810326601,1.39685562737982,2.97061622231479,3.21896306137887,1.81247927916354,2.92445303860747,1.32148162095989,1.6345120151091,1.4907974776689,3.1438755557577,4.17005330405836,1.73282827159699,1.73282827159699,1.46852108295774,4.07160414774329,3.40450377817443,2.1007268126824,1.35753547975788,2.54668165995296,2.10902040301031,1.95078197732982,1.42712839779952,3.47886191629596,2.59345981956604,1.75448733218585,1.56383735295924,2.81815641205523,2.82102305270683,1.62342304294349,2.27002571430044,1.58169870868025,2.23957751657679,3.1791420105603,3.04143611677803,1.78781239559604,2.48545224733971,5.17069622716897,2.25492520841794,6.17848647159523,1.84771165561694,3.53610701101409,2.30189945437661,1.36855623098683,1.35359627377693,2.06398920428479,1.67162039656126,2.23284413391782,1.97881070093006,2.21183162885883,2.1073489661227,1.44249279809434,3.57348873863542,3.38827669199266,2.03245202378114,2.01999662841625,3.79317412396815,3.27327279097343,1.77728352885242,4.03668448861389,2.22621355501881,5.62160209905186,2.88272870434424,2.69897000433602,2.38510278396687,3.22402566887063,3.00833099262005,2.74472749489669,2.42596873227228,1.99567862621736,4.30189945437661,3.53017798402184,2.61618463401957,2.27327279097343,2.90657831483776,2.31158017799729,2.37468754903833,2.19246497193115,4.85078088734462,4.16621562534352,2.36251027048749,3.10846254232744,2.0204516252959,3.69680394257951,2.87289520163519,3.34969247686806,2.18708664335714,2.14691047014813,2.17587416608345,4.47755576649368,2.39469495385889,2.30803489723264,2.59006687666871,2.52287874528034,2.86646109162978,2.27490547891853,4.61261017366127,3.29157909986529,2.02502800570193,4.79317412396815,2.03857890593355,3.13667713987954,2.76447155309245,2.34390179798717,2.27654432796481,2.83564714421556,4.40340290437354,3.50863830616573,5.69464863055338,3.50723961097316,2.40230481407449,2.44490555142168,3.61083391563547,2.65560772631489,2.28819277095881,2.16941133131486,2.11520463605102,3.05256627811295,2.52578373592374,1.76447155309245,3.80966830182971,3.80966830182971,3.09745322068601,3.07935499859321,2.52287874528034,4.13430394008393,2.98716277529483,2.82390874094432,7.95078197732982,5.88941028970075,2.97881070093006,2.5003129173816,7.44733178388781,4.35261702988538,2.96257350205938,5.56224943717961,3.85078088734462,11.0250280057019,3.28483264215154,2.51999305704285,9.48678239993206,6.68402965454308,6.28650945690606,2.83863199776503,2.79860287567955,2.78251605578609,1.54515513999149,6.83564714421556,3.44249279809434,2.88605664769316,3.14508697769214,2.43770713554353,2.34198860334289,4.86966623150499,4.33629907461035,3.0670191780768,3.04095860767891,2.85078088734462,2.13846558914096,9.29843201494407,6.53760200210104,6.30364361126667,7.18309616062434,4.41680122603138,4.41680122603138,3.26440110030182,2.85078088734462,6.98296666070122,3.97881070093006,3.06148027482351,2.1505805862031,6.33913452199613,5.59345981956604,5.46092390120722,2.7281583934635,3.01637371287547,4.48811663902113,2.81247927916354,2.73518217699046,3.12842706445412,7.97061622231479,7.75202673363819,3.58004425151024,2.9100948885606,2.28650945690606,2.70774392864352,3.28316227670048,6.81247927916354,6.23433144524099,8.25414480482627,8.25414480482627,6.87942606879415,5.82102305270683,4.28988263488818,7.97881070093006,5.10790539730952,3.91364016932525,2.72584215073632,5.3767507096021,4.44977164694491,4.31158017799729,3.5543957967264,5.18309616062434,4.29157909986529,3.04866248120408,2.38827669199266,3.90657831483776,2.53461714855158,4.26042765554991,3.28650945690606,3.12090412049993,5.34872198600186,5.26921772433361,3.41907502432438,3.75448733218585,5.77728352885242,3.83863199776503,9.04191415147891,7.74472749489669,2.67366413907125,12.2160964207273,6.93930215964639,2.97061622231479,4.3429441471429,4.10347378251045,4.00217691925427,3.28149831113273,3.28149831113273,12.1023729087096,6.97881070093006,6.23210238398191,4.21609642072726,3.50584540598156,3.92445303860747,2.93554201077308,5.40782324260413,3.24336389175415,4.21681130892474,3.81247927916354,3.53313237964589,3.3585258894959,10.5833594926617,9.2020403562628,7.41907502432438,6.92445303860747,5.82973828460504,4.05060999335509,3.48280410205003,2.77469071827414,5.43770713554353,3.45842075605342,3.42829116819131,2.99139982823808,4.6903698325741,4.29670862188134,5.00744648216786,3.69680394257951,4.47886191629596,2.78251605578609,6.89962945488244,3.53313237964589,3.53313237964589,4.31875876262441,4.45842075605342,3.91721462968355,14.2924298239021,11.7212463990472,6.27245874297144,4.33818731446274,3.59859945921846,3.01233373507373,3.00130484168834,3.65560772631489,3.55129368009492,9.42596873227228,5.76447155309245,4.72584215073632,3.65364702554936,3.32790214206428,3.44249279809434,7.94309514866353,6.25258819211358,4.39040559077478,3.19044028536473,4.02779716162094,3.14996674231023,8.0268721464003,3.28988263488818,7.59859945921846,5.92445303860747,4.74957999769111,3.73992861201493,3.68402965454308,8.83863199776503,5,4.71219827006977,3.94692155651658,3.1681302257195,3.6345120151091,3.52432881167557,4.33348201944512,3.58335949266172,6.09044397075882,4.71669877129645,3.14327110961712,9.56224943717961,4.24872089601666,4.05109823902979,3.87614835903291,3.44249279809434,6.05354773498693,8.99567862621736,8.87942606879415,7.20481541031758,3.90657831483776,3.68402965454308,5.60380065290426,4.9100948885606,3.84771165561694,4.25336580106242,4.06298389253519,7.53165266958784,7.28316227670048,6.87289520163519,4.92811799269387,5.91721462968355,4.44009337496389,2.47495519296315,6.65364702554936,4.48945498979339,3.02456819149074,6.64397414280688,5.52870828894106,4.77728352885242,6.82681373158773,6.67162039656126,2.57186520597121,5.58335949266172,3.99139982823808,3.90657831483776,3.61978875828839,3.36451625318509,3.11690664142431,4.69680394257951,3.84771165561694,4.35556141053216,4.35556141053216,9.02641037657274,4.36151074304536,4.19517932127884,3.31425826139774,2.6458915608526,4.59516628338006,4.27083521030723,11.0644927341753,4.21824462534753,6.51999305704285,5.48148606012211,5.23807216157947,4.00261361560269,3.14086170270547,14.6216020990519,4.94309514866353,4.73992861201492,3.98296666070122,3.23284413391782,3.22914798835786,4.49214412830417,4.48811663902113,4.48811663902113,4.22329881601159,4.09854167860389,7.35654732351381,5.23957751657679,2.69680394257951,5.35163998901907,5.00568284733036,3.95860731484178,2.81247927916354,2.71444269099223,4.4907974776689,3.96657624451305,3.89279003035213,3.80410034759077,3.23882418684427,2.82390874094432,9.65364702554936,6.6345120151091,3.7619538968712,9.36957212497498,5.43297363384094,3.86646109162978,13.1617807780924,6.85698519974591,4.59516628338006,4.42712839779952,8.88605664769316,7.30539480106643,2.89619627904404,2.88605664769316,3.81247927916354,2.94309514866353,2.89962945488244,10.1605219526258,3.67571754470231,3.67571754470231,7.33068311943389,5.84771165561694,14.8013429130456,7.91721462968355,7.91721462968355,3.77728352885242,4.65955588515988,4.34969247686806,3.36855623098683,3.63264407897398,2.90308998699194,9.65955588515988,4.08196966321512,4.96257350205938,4.82390874094432,3.02733440773389,4.68193666503724,4.59516628338006,4.58335949266172,8.06248210798265,4.41680122603138,6.32790214206428,3.96257350205938,5.93181413825384,3.15614457737684,5.31785492362617,8.44009337496389,4.19997064075587,3.13135556160517,4.93554201077308,4.24949160514865,3.83564714421556,8.24336389175415,8.24336389175415,7.48811663902113,6.29073003902417,6.13548891894161,4.82390874094432,4.61083391563547,3.79588001734407,6.30980391997149,5.54975089168064,3.83863199776503,3.76447155309245,5.65560772631489,3.82390874094432,3.82973828460504,6.50584540598156,4.97881070093006,4.75696195131371,4.75696195131371,4.44129142946683,4.40011692792631,3.81247927916354,5.59687947882418,3.86327943284359,8.86646109162978,5.90657831483777,4.36251027048749,3.31605286924849,3.22257317761069,13.397940008672,6.12901118623942,5.77989191195995,5.57511836336893,4.20411998265593,6.1438755557577,6,4.74957999769111,4.0783135245164,3.42136079003193,13.3990271043133,10.0609802235513,10.2034256667896,5.93181413825384,5.58169870868025,5.57186520597121,5.42596873227228,5.28819277095881,5.3269790928711,12.8326826652518,8.95078197732982,4.91364016932525,5.02918838912748,5.11294562194904,5.10513034325475,5.67366413907125,4.25336580106242,9.2958494831602,7.73518217699046,4.95467702121334,4.71444269099223,4.31605286924849,3.66958622665081,14.0333890133181,4.24184537803261,3.57675412606319,5.00217691925427,4.90657831483777,4.90657831483777,3.61798295742513,5.69897000433602,5.66354026615147,4.59516628338006,4.45593195564972,4.37059040089728,4.29073003902417,3.55129368009492,7.17005330405836,5.07160414774329,4.86966623150499,3.7594507517174,3.72124639904717,5.12436006299583,4.96257350205938,4.64206515299955,4.41341269532824,3.82973828460504,3.79588001734407,13.8601209135988,8.00744648216786,5.52578373592374,5.52578373592374,12.48280410205,6.15242734085789,5.28650945690606,4.58670023591875,3.91364016932525,3.84771165561694,5.51004152057517,4.71219827006977,3.90308998699194,3.88272870434424,3.87289520163519,3.55909091793478,8.62160209905186,4.65757731917779,8.39361863488939,5.53017798402184,3.91364016932525,3.89619627904404,3.80410034759077,8.40340290437354,4.7851561519523,4.71896663275227,4.71896663275227,4.57024771999759,4.01322826573375,7.94692155651658,6.29413628771608,5.26760624017703,3.92445303860747,3.98296666070122,3.96657624451305,11.542118103266,9.24565166428898,11.5086383061657,7.92445303860747,6.33913452199613,10.0141246426916,9.63827216398241,7.4089353929735,6.48412615628832,4.14569395819892,4.03574036980315,5.71444269099223,4.86646109162978,4.09151498112135,7.92811799269387,4.84771165561694,4.08460016478773,5.92081875395237,5.53910215724345,4.77728352885242,4.16749108729376,8.17263072694617,7.81247927916354,8.69680394257951,5.49894073778225,4.84163750790475,4.26600071346161,6.01099538430146,5.92811799269387,7.1007268126824,5.96657624451305,5.64975198166584,4.24565166428898,5.95860731484177,7.08830984124614,5.84466396253494,5.51144928349956,4.29157909986529,4.16685288808721,8.50863830616573,5.09691001300806,6.12726117252733,7.81815641205523,5.20901152491118,5.20901152491118,5.20901152491118,4.24565166428898,5.32057210338788,4.82973828460504,4.49349496759513,9.1232050237993,6.01637371287547,5.30189945437661,6.35261702988538,5.27736607746619,5.16621562534352,4.50584540598156,7.99139982823808,6.04914854111145,5.86012091359876,5.40782324260413,4.55909091793478,4.55129368009492,4.52143350440616,8.21896306137887,4.59345981956604,4.52724355068279,8.21609642072726,6.29929628285498,6.26201267366657,7.91721462968355,7.87614835903291,6.72584215073632,6.1505805862031,6.01144104312138,4.60380065290426,5.58502665202918,5.52724355068279,8.41005039867429,5.56703070912559,4.70553377383841,4.69897000433602,12.3635121036466,12.3635121036466,11.7077439286435,4.67985371388895,5.62160209905186,5.62160209905186,5.53017798402184,4.72124639904717,4.71219827006977,4.63078414258986,4.63078414258986,4.75448733218585,8.5543957967264,4.74232142513082,6.62342304294349,4.83564714421556,4.79048498545737,4.71219827006977,4.89279003035213,4.78781239559604,4.93930215964639,10.4122890349811,9.01727661233145,5.87289520163519,4.93181413825384,4.87942606879415,11.8041003475908,4.95467702121334,6.18508681872493,5.92081875395237,5.64016451766011,5.07520400420209,5.07520400420209,5.06600683616876,7.26440110030182,4.98296666070122,4.98296666070122,5.11013827874181,4.92081875395237,4.91721462968355,8.83564714421556,6.95467702121334,6.27164621797877,5.99139982823808,5.03715731879876,14.2388241868443,9.12090412049993,8.48945498979339,7.83564714421556,6.25570701687732,9.34390179798717,7.95078197732982,7.39040559077478,6.03526907894637,5.14266750356873,5.12842706445412,5.05354773498693,6.47237009912866,5.16749108729376,5.12784372725171,5.17848647159523,5.25336580106242,5.17848647159523,6.16430942850757,5.24795155218056,5.14508697769214,5.30803489723264,5.30364361126667,5.11238269966426,5.33254704711005,5.30277065724028,22.698970004336,9.86327943284359,8.15739076038944,6.45469288353418,6.45469288353418,5.30102999566398,6.40671393297954,5.38195190328791,8.19586056766465,6.79048498545737,7.01772876696043,6.93930215964639,6.91364016932525,5.51286162452281,11.9469215565166,5.56224943717961,5.50445566245355,6.77728352885242,5.44977164694491,6.79588001734407,5.65560772631489,5.59345981956604,9.35066514128786,5.54668165995296,5.54668165995296,5.69464863055338,5.63264407897398,5.46218090492673,8.69897000433602,7.46852108295774,5.46980030179692,5.80966830182971,5.80687540164554,7.34008379993015,5.87942606879415,11.3477536589967,16.3516399890191,7.5543957967264,6.03151705144607,6.04866248120408,8.26760624017703,6.13667713987954,6.07987667370928,5.86327943284359,8.3840499483436,11.593459819566,8.27002571430044,8.16304326294045,6.33348201944512,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.32975414692588,6.29670862188134,11.0501222959631,6.41907502432438,6.51427857351842,14.4111682744058,8.79860287567955,6.41116827440579,21.1765257708297,6.63078414258986,6.63078414258986,10.5214335044062,6.67571754470231,10.7099653886375,6.68824613894425,6.67366413907125,13.2873502983728,9.64206515299955,7.08460016478773,13.3062730510764,7.17783192063198,7.23507701535011,7.18111458540599,7.36451625318509,7.57839607313017,7.60205999132796,7.47495519296315,7.59516628338006,7.80966830182971,7.86646109162978,7.89279003035213,7.83268266525182,8.05551732784983,8.3777859770337,11.1438755557577,15.6946486305534,13.2941362877161,11.8761483590329,11.7189666327523,10.3946949538589,9.76955107862173,9.63638802010786,12.1911141326402,9.45593195564972,8.73754891026957,8.54060751224077,8.47625353318844,7.99139982823808,8.06600683616876,11.0830199526796,8.12609840213554,18.2471835688117,9.96657624451305,7.83863199776502,7.71219827006977,15.4461169733561,7.61978875828839,7.26841123481326,9.54060751224077,15.2724587429714,6.97061622231479,15.0168249279622,8.76955107862173,6.78781239559604,8.69464863055338,12.4710832997223,8.72124639904717,10.2572748686953,6.48811663902113,6.43770713554353,10.0245681914907,6.21467016498923,6.3840499483436,6.27572413039921,6.16115090926274,6.04000516167158,5.97881070093006,9.82102305270683,5.94692155651658,5.89962945488244,9.70333480973847,5.78251605578609,10.5767541260632,5.51712641639125,5.43770713554353,5.3585258894959,7.86646109162978,7.79588001734407,5.3777859770337,5.31336373073771,5.14569395819892,7.64016451766011,5.14206473528057,4.90308998699194,6.06550154875643,5.0433514207948,10.279840696594,7.39254497678533,7.39254497678533,5.63264407897398,5.10568393731556,5.02826040911222,11.4672456210075,10.8761483590329,6.86327943284359,6.74957999769111,4.82390874094432,4.57186520597121,16.6635402661515,5.46092390120722,4.99567862621736,4.99567862621736,5.72353819582676,9.17005330405836,6.17979854051436,5.45717457304082,5.48678239993206,4.80966830182971,4.84466396253494,5.19178902707578,4.66554624884907,8.50863830616573,5.97469413473523,4.97469413473523,4.74472749489669,5.77728352885242,3.77728352885242,5.73518217699046,7.29929628285498,8.57839607313017,10.4736607226102,6.71444269099223,4.64397414280688,7.71444269099223,9.1681302257195,5.48017200622428,10.0639892042848,9.18842499412941,5.66354026615147,4.34872198600186,4.82681373158773,7.78781239559604,11.6270879970299,4.77728352885242,6.63638802010786,6.51286162452281,6.33068311943389,4.47366072261016,12.4056074496246,5.45099673797421,6.09854167860389,5.91721462968355,5.06905096883248,4.92811799269387,13.896196279044,3.61439372640169,11.9956786262174,8.65169513695184,3.29242982390206,4.38510278396687,3.51712641639125,6.57186520597121,5.68402965454308,5.19178902707578,3.60205999132796,8.61978875828839,5.86327943284359,5.07007043991541,5.89279003035213,3.65955588515988,2.97881070093006,10.7033348097385,2.95467702121334,5.45593195564972,5.07314329105031,2.96657624451305,5.38933983691012,5.38933983691012,3.34775365899668,1.63827216398241,6.19654288435159,3.74232142513082,2.40782324260413,2.57348873863542,1.44855000202712,1.4294570601181,1.95467702121334,1.7851561519523,2.92445303860747,2.10127481841051,6.07727454200674,2.36051351073141,2.23284413391782,1.98716277529483,2.1007268126824,1.71219827006977,1.49620931694282,2.82973828460504,2.61439372640169,3.02181948306259,2.95078197732982,2.77469071827414,1.95860731484178,4.19246497193115,3.63264407897398,2.51144928349956,2.47625353318844,1.75448733218585,1.51570016065321,1.5654310959658,2.52578373592374,2.48545224733971,4.18309616062434,5.29756946355447,2.02136305161553,2.02136305161553,1.78781239559604,4.30891850787703,2.30803489723264,1.95078197732982,1.64975198166584,1.63638802010786,3.60730304674033,2.18442225167573,2.80410034759077,1.80687540164554,4.80134291304558,3.72584215073632,2.83564714421556,1.97469413473523,4.35359627377693,2.16557929631847,2.72124639904717,7.19178902707578,2.82102305270683,4.05749589383192,4.46218090492673,2.39469495385889,1.81247927916354,4.26201267366657,2.31069114087638,3.90308998699194,2.66958622665081,2.35753547975788,1.84163750790475,2.59006687666871,2.59687947882418,5.90657831483777,5.35163998901907,3.28483264215154,2.88272870434424,2.6252516539899,2.98716277529483,2.26680273489343,2.11520463605102,3.28483264215154,2.95467702121334,2.80134291304558,6.26520017041115,5,2.85078088734462,2.43297363384094,6.30364361126667,4.6252516539899,11.299296282855,6.95467702121334,10.2403321553104,4.07883394936226,3.06148027482351,2.71896663275227,2.36351210364663,6.05551732784983,3.00305075150462,3.22184874961636,3.03763066432998,3.03763066432998,3.21824462534753,5.82102305270683,3.77211329538633,3.70333480973847,4.03810452633215,2.72584215073632,5.26121944151563,8.57839607313017,4.74472749489669,3.1890957193313,5.60730304674033,2.50307035192679,2.8153085691824,3.58004425151024,1.84466396253494,4.03479829897409,9.04191415147891,5.41116827440579,4.10347378251045,12.1023729087096,6.23210238398191,5.22548303427145,6.80134291304558,3.25103713874384,3.11918640771921,3.24336389175415,3.11975822410452,3.69897000433602,2.56224943717961,1.98296666070122,7.57186520597121,6.0515870342214,2.78781239559604,3.52578373592374,2.7851561519523,4.31875876262441,2.64016451766011,6.43179827593301,4,3.75202673363819,3.75202673363819,6.14508697769214,3.62160209905186,3.42712839779952,3.67162039656126,3.02273378757271,4.34969247686806,3.68402965454308,8.02045162529591,4.80687540164554,4.23882418684427,6.77728352885242,5.05749589383192,5,3.89279003035213,3.53910215724345,5.40011692792631,3.99139982823808,3.79860287567955,3.58335949266172,3.52143350440616,4.61978875828839,9.56224943717961,4.17848647159523,7.5543957967264,2.34486156518862,3.61439372640169,2.43651891460559,7.20481541031758,4.92081875395237,12.6675615400844,4.97881070093006,2.80687540164554,7.53165266958784,7.07675598136972,7.19111413264019,2.95860731484178,4.18575240426808,3.69250396208679,3.35753547975788,4.0017406615763,8.67778070526608,2.60906489289662,4.59516628338006,4.1890957193313,2.65560772631489,4.10237290870956,5.39254497678533,4.01908806222316,12.4710832997223,4.73992861201492,3.27654432796481,5.83863199776502,4.79588001734407,3.95467702121334,7.35654732351381,2.77989191195995,5.35163998901907,4.61978875828839,4.97061622231479,4.43889861635094,3.77989191195995,3.32057210338788,2.79860287567955,2.79860287567955,9.80687540164554,3.95860731484178,11.9136401693253,4.33913452199613,4.25258819211358,4.74232142513082,3.38510278396687,4.88941028970075,2.88605664769316,2.94309514866353,2.94309514866353,4.7851561519523,4.72124639904717,3.94309514866353,3.28232949699774,3.32239304727951,4.34969247686806,2.96257350205938,3.96657624451305,2.92445303860747,3.05998184499234,4.70553377383841,10.0599818449923,4.80687540164554,8.68402965454308,10.7011469235903,8.36754270781528,3.1438755557577,4.50445566245355,4.26841123481326,3.70553377383841,9.31695296176115,5.34775365899668,5.34775365899668,5.01818139282934,3.20620961530918,4.40782324260413,7.02319166266193,12.1095789811991,5.54975089168064,4.47108329972234,4.29843201494407,3.79317412396815,3.29843201494407,3.29413628771608,8.65364702554936,4.73282827159699,3.91364016932525,3.11520463605102,5.59687947882418,4.49485002168009,9.86327943284359,3.3269790928711,7.24033215531037,4.86646109162978,4.04191415147891,7.32239304727951,5.77989191195995,3.45099673797421,6.82681373158773,6.75202673363819,6.75202673363819,6.21324857785444,3.42136079003193,3.42136079003193,7.29929628285498,4.26600071346161,3.41341269532824,12.4948500216801,11.896196279044,9.01908806222316,6.93930215964639,4.22841251911874,3.2958494831602,3.41116827440579,3.53313237964589,9.93554201077308,8.61618463401957,8.3585258894959,5.3269790928711,3.45222529461218,3.58502665202918,19.1771783546969,6.12033079436795,6.04914854111145,5.2298847052129,3.52432881167557,5.01412464269161,6.44490555142168,4.50445566245355,5.21041928783557,9.57839607313017,7.7594507517174,3.51286162452281,7.86966623150499,3.69897000433602,6.40011692792631,4.5003129173816,8.75696195131371,5.01144104312138,4.49214412830417,3.72124639904717,6.23062267392386,3.75696195131371,8.00744648216786,5.52578373592374,4.62160209905186,3.72124639904717,3.69250396208679,8.19517932127884,5.20760831050175,3.89962945488244,5.48678239993206,6.3777859770337,5.48545224733971,4.53610701101409,3.95078197732982,5.90657831483777,4.65560772631489,4.87942606879415,4.70114692359029,4,4.70996538863748,4.66354026615147,4.57186520597121,10.3746875490383,8.76955107862173,6.43062609038495,4.78251605578609,4.77469071827414,3.95078197732982,10.3840499483436,6.33913452199613,4.07109230975605,4.00392634551472,7.76700388960785,5.00744648216786,4.83268266525182,6.48412615628832,5.67366413907125,7.70333480973847,4.86646109162978,4.94309514866353,4.88605664769316,4.17005330405836,4.10679324694015,8.99567862621736,5.58335949266172,4.28483264215154,6.10790539730952,5.65955588515988,8.39361863488939,5.01682492796219,4.32239304727951,6.92445303860747,5.09963287134353,5.09963287134353,4.29157909986529,4.20411998265593,5.19928292171762,14.2848326421515,7.39469495385889,4.45345733652187,5.23210238398191,4.24565166428898,6.34008379993015,5.17327747983101,4.82973828460504,4.35654732351381,21.4485500020271,20.8326826652518,4.50863830616573,4.50584540598156,4.44129142946683,15.829738284605,6.19111413264019,5.47237009912866,4.57675412606319,4.43415218132648,9.3429441471429,6.13727247168203,4.41680122603138,17.9871627752948,4.48811663902113,9.14874165128093,5.33629907461035,5.68613277963085,6.42136079003193,6.32975414692588,5.51144928349956,4.70553377383841,8.39254497678533,4.72353819582676,4.57511836336893,4.73754891026957,6.81815641205523,5.63078414258986,4.76955107862173,6.62342304294349,5.43533393574791,4.82390874094432,4.80966830182971,13.2924298239021,6.75448733218585,8.74232142513082,4.95860731484177,4.87942606879415,5.97469413473523,7.5003129173816,7.44733178388781,6.16941133131486,5.87942606879415,4.79048498545737,5.06803388527183,5.0515870342214,5.0395292224657,13.3062730510764,6.10846254232744,5.09691001300806,5.0893755951108,5.05403929642243,4.99139982823808,9.3840499483436,9.12090412049993,5.98296666070122,5.04817696468409,4.94309514866353,5.14266750356873,5.05354773498693,9.72353819582676,8.65560772631489,7.51712641639125,5.19997064075587,14.3882766919927,10.5406075122408,5.1337126609158,5.21467016498923,5.20971483596676,5.1337126609158,4.93930215964639,13.7644715530925,12.0530567293022,5.35163998901907,6.3936186348894,5.38195190328791,5.35066514128786,5.33068311943389,5.28650945690606,9.86327943284359,5.34486156518862,5.4145392704915,5.38195190328791,5.26201267366657,5.2298847052129,8.19586056766465,5.50445566245355,6.91364016932525,5.52287874528034,5.40450377817443,5.51999305704285,8.41116827440579,5.56224943717961,13.4907974776689,8.52432881167557,6.84466396253494,5.56066730616974,5.47625353318844,5.47625353318844,5.59345981956604,5.67162039656126,5.62342304294349,5.71444269099223,5.69464863055338,5.74714696902011,7.27002571430044,5.77989191195995,5.80410034759077,5.7619538968712,12.838631997765,5.87942606879415,5.76700388960785,9.30803489723264,6.1444808443322,12.9065783148378,6.44611697335613,9.03198428600636,10.5702477199976,10.5214335044062,6.61439372640169,25.2479515521806,10.8569851997459,9.13667713987954,8.89962945488244,9.25103713874384,6.96657624451305,9.10402526764094,13.3062730510764,9.31336373073771,7.20342566678957,7.20342566678957,7.18309616062434,7.31875876262441,9.73992861201492,7.54363396687096,7.58670023591875,7.7619538968712,7.89279003035213,8.27654432796481,8.31605286924849,10.6968039425795,8.37986394502624,8.47755576649368,8.69897000433602,8.64975198166584,8.88272870434424,8.94309514866353,9.14206473528057,9.49620931694282,9.48678239993206,12.5638373529592,10.8664610916298,12.8239087409443],"text":["TG","HIPK2","TMC2","PDP1","C10orf107","FRMD4A","C10orf85","PARP6","PRKAB2","AOAH","ADAM12","GALNT15","WFDC3","DGKI","IL1R1","EFTUD1","DLEU1","PDCD1LG2","ENSA","CD58","AGBL1","KHDRBS3","LEPREL1","FAM49B","CAV1","OTUD7B","DEC1","CPQ","LPP","COMMD7","TJP1","RCL1","ST8SIA6","KIAA1199","ASAP1","C15orf26","PHF8","LYSMD4","MRPL46","S100A10","VTCN1","ATP13A5","DIRC1","C1orf110","DPT","NTRK3","CYP7B1","NREP","NOL10","SORBS2","PDE8A","IPMK","WDR72","GDAP2","NSMAF","ZBTB38","AMPH","PRUNE2","TSC22D2","RAB27A","GPC6","IFI16","PI4KB","PAPSS2","LRRC28","MCTP2","SLC39A12","AKAP2","PALM2-AKAP2","CD47","TES","DDX47","APOLD1","YAP1","MB21D2","SERINC3","CP","PLCXD2","PHLDB2","PALM2","FBN1","PRUNE","TPRG1","MTMR3","SEMA6D","HUS1","CLIC5","C20orf187","PKIG","PHC2","BACH1","MME","UBE2H","BNC2","EFEMP1","NPSR1","ITGB5","AP3S2","C15orf38-AP3S2","MMP7","SPTSSB","ZNF639","ZNF474","PAK2","DMRT3","EFR3A","SLC4A1AP","RUNX2","LIPA","WHAMM","HIBADH","PEAK1","ME3","ADAMTSL3","METTL13","CYP24A1","RUNX1","SVIL","CUBN","MYC","BCAT1","GLIS3","PACS1","CD101","CADM1","SSPN","IQGAP1","GUCY1A2","PPFIBP1","PLEKHA5","KCNAB1","DIEXF","FAM180A","CHD6","DCBLD2","CRTC3","CNIH3","ILDR2","FAM214A","PRCC","RECK","TMOD3","CFLAR","NYX","MED12L","P2RY14","RALY","RPRD1B","RIN2","MMP20","PPM1L","CMSS1","FILIP1L","SRGAP2","POLR3GL","ANXA2","NF2","MS4A4E","MYPN","AKAP13","RGS20","RNF115","MSTO1","PIGV","PTGFRN","TOX","FRMD3","DNM3","IL1RAP","IGF1R","AL590452.1","ZBTB2","COX20","POMT2","WDR17","SCRN1","TRAM2","SREBF2","ATP5G2","STIM1","NR6A1","BCAR3","TRAK1","PARK2","FBXO42","TNN","ASF1A","MCM9","SSH1","CALM1","SPTBN1","RP11-159D12.5","ARHGAP22","RP11-458D21.5","ETV5","UNC5CL","SH3PXD2B","MYCBP2","WRNIP1","CRTAP","SNAP23","HIST1H1C","TEAD1","ACSL3","VEPH1","ATP2B4","ABLIM2","SEPT7","CDK6","TTC28","SPAG9","DCP2","MED8","SAP30BP","CHN1","EXT1","TMEM178A","ARHGAP29","KIF1A","WWTR1","RRAS2","ZC3H7A","ALG8","AL162389.1","MEIS2","NDUFAF5","TRIM66","ARRDC3","IFI44L","SPECC1","ARNTL","ARID4B","SLC28A3","ID3","GRB10","CHMP4B","DCLK1","CSRP1","TLE1","RAD51B","THNSL1","IL33","STMND1","TCP11L2","MCCC2","HDAC9","SLC10A7","SPRED2","PACRG","SOX9","SEC14L1","TSPAN5","B3GALTL","RNF216","ZNF521","MOXD1","CCDC80","GDF11","CARS","SUMF2","C1orf143","PRPF40B","GAS1","NOTCH2","FAM64A","SHQ1","AC022431.2","ZNF703","TARBP1","FAM111A","SEC31A","NCALD","ZBTB16","NIM1K","POU3F2","ARHGAP35","TCF7L2","ETF1","LITAF","RALGAPA2","MTHFD1L","DBT","TMEM108","MTHFD2L","BCHE","BOC","ANKRD40","FOXJ3","TRIO","CDKAL1","FARP1","VOPP1","PLXNA2","STARD13","TMX2-CTNND1","RP11-691N7.6","LDLRAD3","SYBU","CAP2","FAM134B","SLC16A4","ENC1","CPVL","RP11-770J1.4","LRRC23","APCDD1","BBOX1","METAP1D","AKAP1","OPTC","LMO2","SYT11","IFT74","USH2A","EIF4G3","MRPL48","FAM212B","EXTL3","FMO6P","PRICKLE2","CSMD1","USP36","BEND3","ADAMTS3","TRAF3IP2","TRIM9","ZFAT","ZMYND8","YPEL2","SLC15A2","DIP2B","ACTR6","FYN","PPM1D","CLIP2","C12orf65","PRELP","IQCJ-SCHIP1","SCHIP1","JAKMIP2","SAMD13","SCARB2","RHOJ","AVL9","SLX4IP","DIRC3","SDK2","AUTS2","LRRFIP1","UBALD2","SNX31","TCF4","NRG1","PLCB4","NEGR1","RPN2","GOLM1","EMC8","KIAA0355","ALK","SACS","SLC38A9","TRIT1","GCDH","SYCE2","NAV3","MYO18B","FNBP1L","MICALCL","MAP2K6","ABTB2","NEDD9","CSGALNACT1","ABCD2","SOGA1","COL28A1","NCMAP","SLC44A3","TSNAX-DISC1","DARS","CALN1","OOSP1","CELF1","AMBRA1","TCEB3","SWAP70","MAP1B","PXDNL","FAT1","UGP2","GLI3","ITPKB","LRP4","SPATA5","ACER3","PTPRJ","FAM117A","ATXN1","KCNJ10","NOTCH2NL","DDX60L","PALLD","LIMA1","KLF9","TTC7B","CHD7","SMNDC1","TRIM2","DKK3","UBE4B","ADORA1","PLCH1","ADORA3","PSMA1","AGAP1","ATXN7L1","IGFBP2","MICAL2","CCDC41","EGFR","PBX3","RNF182","ATP13A4","C5orf64","GRID2","GPD1L","DOCK4","TMC1","RP11-210M15.2","BICD1","USP10","ARHGAP31","SLC1A2","CEP164","SPATA13","RP11-307N16.6","TOP1","C20orf26","FIBIN","ZCCHC3","GFAP","DPYSL5","CUX1","ZSWIM6","CLASP2","CACNG5","ANKRD44","MFSD6","TMCC1","SORT1","CBFA2T2","FOXK2","ANKRD50","MAP6","PNMAL1","EPHA7","CPNE5","SH3RF3","MARCH6","CACHD1","POU2F1","LIMCH1","CPSF4L","BANF2","AC017081.1","CDKN1A","SP2","CUEDC1","NFIB","PC","FAM107B","AFF1","FOXO6","SLC6A11","MATN2","LRP1B","APBA1","TULP4","SARS","SRGAP1","ABL2","CNTN5","MYO5B","MAGI2","UBXN2B","HP1BP3","CHIT1","KCNQ1","RASA3","GABRB1","CDC14B","PSD3","C8orf56","UBIAD1","RAB3B","DST","FHAD1","YIPF1","POLR1A","CTNNBL1","RAD23A","ABCA8","PTPRZ1","ALKBH3","ELP4","KCNE4","MGAT5","BTG2","CSRP2","NFIX","NHSL1","COL4A1","ABRA","SOBP","KIF26B","FOXO3","SCEL","GMPR","RP11-144F15.1","LRRN3","IMMP2L","ADCY8","C5","STX6","DPP4","MARCH4","ZBTB20","SCML4","C1orf61","APC","PCDH7","DISC1","LAMB3","SPARCL1","TAMM41","SPATA6","SEPT9","KIAA0226L","TGFBR2","GPR126","FHL2","MKL1","FIP1L1","LNX1","RGL1","IQSEC3","KIRREL3","ZNF804A","ANGPT1","NAV1","PBX1","CYTH1","KAT2B","ARPP21","UBASH3B","FAM182B","BMPER","CDK14","KIAA1614","MEOX2","CDH4","ETV1","ZNF277","ADAM22","PRKCA","HEXB","PAMR1","WNT5B","ADRB1","SLC1A3","SPSB4","GAB2","AFAP1","PPP2R2B","SRGAP3","KCNJ16","CSNK1A1L","AP000708.1","AXIN2","CTD-2535L24.2","EBAG9","DRAXIN","ENOX1","B3GALT2","CDC73","C9orf3","BPTF","SMARCC1","DPYSL2","NCAM1","PREX2","FOXN3","ABAT","MERTK","KCNJ6","PPM1H","IGF2BP1","RGS12","SLC17A8","FARS2","GPM6A","CCDC15","RNF157","SMG7","SOX6","ZZZ3","NUFIP1","MYO10","SNTG1","CKB","PTPRD","KCTD6","LRTOMT","LAMTOR1","ID4","COLGALT2","GPR156","NDUFA10","NKAIN3","GNG2","FAM65B","MKRN3","ESRRG","GAS2","PPFIBP2","DOCK9","RPS6KC1","RP11-286N22.8","AC062017.1","RP11-1084J3.4","C1QTNF3","CDKL1","SERPINI1","KIF21A","TANC2","KIAA1598","SEMA5A","JMJD1C","CHST11","LYZL1","ULK4","NPTX2","GJC1","DPYD","PTCH1","MRAP2","RFX2","MRPS18B","NTSR2","THSD7A","C11orf44","MAST2","TAOK3","SIAE","CLHC1","WWC1","PTCHD2","ZDHHC13","ZNF880","A2ML1","C11orf49","SNX11","NGDN","PDZRN3","CTHRC1","SPTAN1","KAZN","ELAVL4","JAZF1","LARS2","TTLL4","ADAMTSL1","NIN","NR1H4","PRCP","ARHGAP15","WDR25","SHROOM3","MAPT","NTRK2","CATSPERB","SERINC5","PLCB1","FAM110B","LRRC3B","NPAS3","ICA1","UTY","SLC15A3","FAM184B","GRIK5","ISM1","ODF1","GRIA2","C12orf75","COG6","SEPT11","MFHAS1","TMTC2","SKAP2","SESN3","ZNF321P","ZNF816","SORL1","LRRN2","INTS9","IQGAP2","ARHGEF10L","DNAH11","EFNB2","LRRC16A","MAPK10","SNTB1","RAB3GAP2","PDE4B","BAI2","NUAK2","TCF7L1","AGO4","EDNRB","PTCHD4","ICOS","RFTN2","PRKD1","RP11-47I22.4","RP11-47I22.3","RFX4","PCDH18","ASS1","ADD2","BCL2L14","HOPX","GPR98","FBXL17","LARP1B","SLC39A11","TCP10","ANKRD17","VWC2L","AK4","NOS1AP","LMAN2L","BAALC","IL1R2","TMEM161B","ALCAM","SND1","PSEN1","NFIA","UBE2U","TOMM7","FGFR3","RSPH3","TRAF3","GATM","RDX","LEMD1","KDM3B","SCD5","ST6GAL2","ACTN2","RPTOR","C14orf166B","SLC22A23","SP8","GPNMB","C12orf79","ARHGEF28","PTPRS","OPCML","RNF217","DSEL","GCNT2","CACNA1E","AKAP7","NR1D2","IGSF11","TNNI3K","FPGT-TNNI3K","LRRC53","OLFML1","AC016757.3","WIPF1","VIT","PTPRO","SSBP3","VRK1","NCKAP5","ETS1","SHANK2","TLE4","KIF5C","UTRN","MOB1B","C2orf80","ATG7","ZFHX4","ITGA2","SASH1","MAP2","TMPRSS5","RBMS3","ASCL1","C1orf198","SDCCAG8","DENND1A","GBAS","FGFR1","SLITRK3","AKAP6","NMT1","RBM24","FOXP2","MRPS27","FHIT","XPR1","FXYD6","FXYD6-FXYD2","TBC1D16","PSRC1","ZSCAN5A","AC006116.20","YTHDC2","CDH1","CELF2","SUMF1","LRRN1","DCLK2","TNR","LPAR1","ZNRF3","C3orf55","STON2","ANKRD6","WIPF3","CPM","DLG2","NRCAM","NSF","HAPLN1","LACE1","FOXP1","ADCYAP1R1","C14orf64","C16orf62","AGBL4","BMP7","FAT3","MGAT5B","PDE4D","TMEM51","ZNF112","CTC-512J12.6","MAGI1","TIAM2","PNMA2","IQCE","ABCA5","PGBD5","C8orf4","PRTFDC1","FREM2","CCDC13","ECE1","CHD9","MPPED2","ACSS1","CCND2","KAT7","ELMO1","SLC35F1","BTBD3","FAM181A","FREM1","TMEM181","RGS6","DAB1","MSRA","FBXL7","ANKFN1","LINGO2","LHFP","ZNF184","PLEKHA2","RAPGEF4","EIF1AY","NXPH1","MSI2","KIAA1549L","JARID2","LIX1","CTD-2215E18.1","DGKB","HERC3","SH3RF1","LRP2BP","ENTPD2","ROBO2","RP11-65D24.2","LHFPL3","ROBO1","CAPN9","PCDH9","FGF2","SLC24A3","NRXN1","VPS13D","LRIG1","CHST9","OLIG3","LRRC8D","RP11-302M6.4","AGMO","LPHN3","LPHN2","POLN","MAPKAP1","GLI2","PRRX1","ARHGEF7","FAM181B","SLC4A4","KCNA10","SLC9A1","SDC3","C1orf87","XKR4","DCDC2","GPR75-ASB3","MOB3B","FABP7","NAV2","ST3GAL4","TMEM100","DOCK10","SDK1","PCDHGC3","PCDHGA11","PCDHGB4","PCDHGC5","PCDHGC4","PCDHGA10","PCDHGB3","PCDHGA5","PCDHGA9","PCDHGB2","PCDHGA2","PCDHGA1","PCDHGA3","PCDHGA8","PCDHGB6","PCDHGA6","PCDHGB7","PCDHGA7","PCDHGA12","PCDHGB1","PCDHGA4","TMSB4Y","PPAP2B","DOK5","THRB","LRRC4C","PHF21A","BOD1","PCNXL2","TMEM63C","RP11-463C8.4","STX18","CTNND2","GRIA1","KCND3","NTM","ANK2","PTPRK","GRIK3","USP2","FTCDNL1","HEPN1","RPH3A","TRPS1","KLHDC8A","SMOC2","EBF2","OTOS","DSCAML1","ATP9B","GAB1","C1orf21","XKR6","MTSS1","LAP3","MIR1208","RP11-1069G10.2","RP11-296O14.3","RP13-631K18.3","RP11-9L18.2","RP11-770E5.1","RP11-296O14.1","AF124730.4","RNU1-70P","TDGF1P2","RP11-244F12.1","RP11-600K15.1","AL079339.1","RNU6-413P","MIR4419B","RP4-666F24.3","RPL29P19","RP13-653N12.2","hsa-mir-490","RP11-572P18.2","AP001607.1","RP11-143P4.2","SLC25A38P1","RP5-860P4.2","CCDC26","CTD-2576F9.1","PVT1","AC083843.1","LINC00649","LINC00507","RP11-479J7.1","AF064858.6","TM4SF1-AS1","SNORA63","CDKN2B-AS1","LEPREL1-AS1","EXTL2P1","RP11-550P17.5","AC006159.5","RP11-464C19.3","RP11-542A14.1","RP11-10O17.3","U8","RP11-191N8.2","MTND1P24","AC005022.1","AC013448.1","RP11-351M8.2","RP13-526J3.1","LINC00113","RP11-115J16.1","AC009518.3","RPL7AL2","PSMA2P2","RP11-431K24.1","RP11-230G5.2","NREP-AS1","RP11-194G10.3","RP11-301L8.2","LINC01033","RP11-307O10.1","RP11-438D8.2","RP11-283G6.5","RP11-283G6.4","KRT18P3","RP11-289F5.1","RP11-284G10.1","RP4-782G3.1","RN7SKP148","AP000797.4","RP5-1069C8.2","RP11-408N14.1","ANKRD18DP","RP1-15D23.2","RP11-33A14.1","AC012370.3","AC074391.1","RP11-25E2.1","RNU1-35P","RN7SKP206","AJ006998.2","RP11-88H10.2","RP11-127O4.3","RP11-889D3.2","CASC17","NPHP3-AS1","LMCD1-AS1","LINC00189","RP11-90B22.1","GS1-410F4.4","RP11-89M16.1","RN7SKP93","NPSR1-AS1","RN7SKP191","RP13-631K18.2","DLG1-AS1","CTC-441N14.4","CTD-2021J15.1","RP11-149I23.3","RNU6-369P","AC002480.5","RP11-358M11.4","RP11-145A3.1","AC079613.1","CALR4P","RP11-680E19.2","RP11-246K15.1","RNU6-710P","RP11-317J19.1","CYP1B1-AS1","AC144449.1","PPP1R10P1","AC108105.1","AC003090.1","RP11-224P11.1","CTD-2005H7.2","RP11-359H3.4","KRT18P13","AC090945.1","GBAP1","RP11-103J8.1","KB-1568E2.1","RP11-523L1.2","RP11-120B7.1","RP11-359H3.1","AC005029.1","CFLAR-AS1","RP11-1018N14.5","RP5-1125A11.4","RP11-338H14.1","RP5-1033K19.2","RP3-388N13.3","RP11-148B18.4","AC006482.1","AC007091.1","TEX41","RP11-3P17.4","AC005019.3","RN7SKP226","RP11-798K3.2","AC006076.1","RP11-29H23.5","MSTO2P","SLC31A1P1","AC098617.1","RN7SL44P","RP11-570H19.2","AC079248.1","LINC00578","RP11-385M4.1","LEMD1-AS1","RP11-6L6.7","RPS11P1","MIR181A2HG","RP11-1007G5.2","RP11-309G3.3","CTD-2337A12.1","LINC00431","RP5-1011O1.2","RP11-159D12.2","AC004520.1","PGAM1P1","RP11-329A14.2","RP11-443B7.3","SUMO1P1","TOB1-AS1","RP11-528I4.2","RP11-317M11.1","ANKRD26P1","RP1-28H20.3","RP11-478K15.6","AC091729.9","RNY1P5","AP000770.1","AC106732.1","RP11-732M18.3","RP11-481C4.1","RP1-155D22.2","RP11-6N17.10","RP11-167H9.4","RP11-167H9.5","RP4-734C18.1","TRAF3IP2-AS1","AC003092.2","RP11-705O24.1","B4GALT4-AS1","AC010148.1","FTLP3","RP11-692D12.1","RP11-7F17.5","PCDH9-AS2","RP11-278H7.1","RP5-899E9.1","LINC00533","NDUFS5P2","AC005740.4","ELMO1-AS1","RP11-646I6.5","RPL7P8","AC012457.2","EXTL3-AS1","AC004448.5","RP11-511B23.2","RPL5P20","RP11-279F6.1","CTB-178M22.1","RP11-388C12.1","RP11-166D19.1","AC118653.2","SNORA64","RP11-417B4.2","LINC00478","AC073626.2","AC009313.2","RP11-81H14.2","AC009302.4","RP11-630C16.1","LAMTOR5-AS1","RP11-625L16.1","RP11-93B21.1","RP11-337A23.3","IGFBP7-AS1","CTD-2530H12.7","SRGAP2B","RP11-130C6.1","RP11-175P13.3","RP11-102J14.1","RP11-282O18.3","CTD-2187J20.1","LARS2-AS1","RP11-533E19.5","RP11-191L17.1","RP11-612J15.3","DPY19L1P1","AC013410.1","AL357519.1","LINC01059","TDPX2","RP1-249H1.2","RP4-723E3.1","RP5-1022J11.2","RP11-572M11.4","CTA-125H2.1","RP11-64D24.2","RP11-589C21.6","AC004012.1","RP3-510L9.1","RP11-1008C21.2","LINC01066","RP11-1082L8.4","RP11-644C3.1","FENDRR","RP11-639B1.1","RP11-444D3.1","ACA59","RP11-618P13.1","AC097724.3","ITPKB-IT1","RNA5SP144","RP11-536C5.2","RP3-405J10.3","RP11-661G16.2","AC022182.3","CASC18","CECR3","LOC124685","RP11-335O13.8","LINC01105","SNORA40","AC004538.3","RP11-506B6.6","RNU6-1096P","RP11-437L7.1","RNY3P9","RP5-1065J22.8","GAPDHP53","RP1-1J6.2","RP11-187O7.3","RN7SL607P","RP5-827O9.1","AP001258.5","AP001258.4","CTA-390C10.10","Y_RNA","RP11-179A10.1","RP11-222K16.1","RP11-160O5.1","RP11-10L7.1","AC003665.1","RP5-1061H20.5","RP5-1092L12.2","RP11-629G13.1","AC093609.1","RP11-82L7.4","RP11-120J1.1","RP11-469N6.3","RP11-472N13.3","RP11-197K3.1","RP11-159H3.1","AP000797.3","AC078882.1","RNU6-1246P","AC007358.1","RP11-196H14.2","AC114752.1","RP11-84A19.2","RP5-1177M21.1","RP11-38P22.2","RP11-8L2.1","KCNQ1OT1","RP11-167N24.5","HMGN2P19","CTB-57H20.1","Z83001.1","RP11-472M19.2","RP11-503E24.3","RNU6-1051P","MIR663A","RP11-232A1.2","RP1-155D22.1","BMPR1APS2","RP11-155G15.2","RP11-698N11.2","AL391538.1","ZBTB20-AS1","AC008753.6","AC007128.1","RP1-153P14.8","AC020606.1","ZNF123P","AC083864.3","PPP1R2P4","AC069154.4","RP4-737E23.2","RP11-489E7.1","SMIM2-AS1","KIRREL3-AS1","RP11-434D9.1","IPO9-AS1","RP11-486M23.1","SOX9-AS1","RP11-698N11.4","SPATA42","RP11-51G5.1","EEF1A1P9","RP11-394O9.1","RP11-75C10.9","NAV2-AS4","RP1-193H18.3","SOX2-OT","RNU6-256P","AC079586.1","HMGN1P17","RP11-452H21.1","CTB-99A3.1","LINC01028","AC002539.1","RP11-848P1.9","NAP1L1P1","RP11-20B7.1","RP11-90K6.1","RP11-531H8.1","RP11-717D12.1","RP11-141M1.3","RNF5P1","RP11-198M11.2","CTC-550M4.1","AC104088.1","AC074011.2","RP11-122C5.1","LIFR-AS1","RP11-344L21.1","RP11-86H7.6","GUCY1B2","CASC15","CTD-2572N17.1","AL442639.1","RNA5SP181","AE000661.37","TRDD3","LINC00882","RP11-109P6.2","RP11-222N13.1","RPL7L1P8","RN7SL366P","RP11-73C9.1","HNRNPA1P58","RP11-118E18.2","CTC-462L7.1","RP11-423J7.1","RP11-293B20.2","RN7SL178P","RP11-214L13.1","CAHM","HOXC13-AS","AC037445.1","RP11-159K7.2","LINC00856","RP1-309H15.2","RP5-1180C18.1","RP11-323I1.1","OACYLP","RP11-308N19.1","DPYD-AS1","CASC8","PLK1S1","AC004158.2","AC004158.3","RP11-134G8.7","RP11-456O19.2","CTD-2008L17.2","RP11-65M17.3","RP11-413N13.1","TTTY19","CTD-2290P7.1","FAM58BP","RP11-536O18.1","RNU6-1114P","AL162419.1","RP11-41O4.2","OFD1P3Y","SEPT7P3","AC008694.2","RP11-527N22.2","LINC00222","A2ML1-AS1","ALDH7A1P2","RP11-89K10.1","RP1-40E16.9","MIR4739","CTD-2245E15.3","RP11-201M22.1","ARHGEF26-AS1","RP11-255G12.3","RP11-286N22.10","RP11-11L12.2","RP11-315F22.1","RN7SKP2","RP11-699L21.1","RP11-991C1.1","RP3-471C18.2","RP5-837I24.2","RP11-60A8.1","RP5-1010E17.1","AC087269.2","RP11-406A20.1","RP11-437J19.1","RP3-359N14.2","RP11-666A8.8","AC092646.1","RP11-712B9.2","CTD-2620I22.1","AC003051.1","AC013406.1","RP11-291C6.1","RP11-408A13.2","RP1-111D6.3","RP11-456O19.4","RNU7-165P","RP11-820L6.1","RP11-367G18.2","LINC01135","U3","SLC7A15P","AC097372.1","RP11-476O21.1","AC004875.1","RP11-109E10.2","RP11-718L23.1","CTC-340D7.1","RP11-814M22.2","PNPT1P2","RP1-305G21.1","SNORA73","AC073255.1","RP11-613M5.2","CTBP2P1","RP11-706J10.2","AC107218.3","TPT1P9","RP11-331K15.1","RP11-118E18.4","RN7SL542P","RP11-118B18.2","RP11-318M2.2","RP11-6F6.1","AC005013.5","RP11-718B12.2","HMGB3P11","RP11-163N6.2","RP11-89M20.1","CTD-2316B1.2","RP11-344F13.1","RNA5SP75","RP11-438C19.2","RP11-492M23.2","RPL21P41","CRYBB2P1","PSPHP1","RP11-563J2.2","AC022909.1","CTC-297N7.9","CTC-297N7.5","CTD-2541J13.2","RP11-99J16__A.2","AC009410.1","AP003900.6","GRIFIN","RP11-121E16.1","AC079135.1","CTD-2516F10.2","KCND3-IT1","RP11-351J23.1","AC018890.6","RP11-428C19.5","RP11-343K8.3","AC018647.3","RP4-668G5.1","EDNRB-AS1","RP11-56I23.1","RNU6-27P","AC005592.2","CTD-2008L17.1","RP11-184M15.2","RP11-669M16.1","MTND5P21","RP11-788A4.1","RP11-1141N12.1","LINC01137","CTC-458G6.2","LINC01038","RP11-146I2.1","RP11-654C22.2","RP5-945F2.2","RP3-453D15.1","NR2F1-AS1","RNU6ATAC19P","RP11-664D1.1","RNU7-62P","AC112693.1","RP5-936J12.1","RNU6-753P","RP11-292D4.3","RP11-90C4.2","ZNRF3-IT1","CTB-47B8.1","NPM1P4","SNORD112","AC099778.1","RP11-394I13.2","LINC01162","RP11-414B7.1","FOXP1-AS1","AC090133.1","RP11-347L18.1","RP11-1078H9.6","RP11-1018N14.2","AC016712.2","EEF1B2P4","RP11-715C4.1","RP11-630C16.2","AF212831.2","RNU6-487P","LINC-ROR","AC011747.6","RP11-564P9.1","RP11-436K8.1","TMEM161B-AS1","RPL21P67","CCDC13-AS1","RNU7-190P","RP11-40G16.1","RP11-541G9.1","LINC01036","FAM181A-AS1","RP11-1325J9.1","RP3-495O10.1","RP11-114H23.1","RP11-396O20.1","RP5-978I12.1","EIF4BP3","AC012451.1","RP1-177I10.1","RP11-147G16.1","LINC01158","RP11-588F10.1","RP11-620J15.2","LINC00474","RP11-708B6.2","RP11-977P2.1","RP11-303G3.6","AC005152.3","RP11-94H18.2","RP11-724M22.1","RP4-541C22.5","NPM1P14","RP11-300M24.1","RNA5SP293","CASC6","AC007193.6","RP11-714G18.1","RNU6-356P","LHFPL3-AS1","AC133633.1","RP11-745C15.2","RP11-1003J3.1","RP5-1027O11.1","LINC00404","GS1-122H1.2","RP11-776H12.1","RP11-317N12.1","RP11-33N16.3","LINC01091","CHCHD4P2","AQP4-AS1","RP11-398G24.2","AC091969.1","CTC-498M16.4","AHCYP3","CTD-2008N3.1","AC068057.1","RN7SL318P","RNA5SP350","TTTY18","RP11-3L10.3","AC009498.1","RP11-254A17.1","AC010145.4","RP11-434H14.1","CYCSP26","AC003985.1","RP11-863K10.2","LINC00511","STX18-IT1","RNA5SP349","RP11-448P19.1","KCND3-AS1","RP11-329N22.1","RN7SKP279","RP11-66N24.4","RP11-429O1.1","AC007682.1","RP11-334E6.3","CHIAP2","RP11-693J15.4","RP11-693J15.5","RP11-67P15.1","RP11-365H23.1","RP11-32B5.1","TXLNG2P","RP11-1058G23.1","RP11-222A5.1","MIR3139","RMST","RP11-388N2.1","LINC00461","HMGB3P3","RP11-179A16.1","RP11-174J11.1","RP11-540K16.1","GS1-122H1.1","PARP4P1","RP11-462G22.1","RP11-395D3.1","RP11-541G9.2","RP11-14O19.2","LINC00466","CTB-70G10.1"],"mode":"markers","name":"signif_fc","marker":{"color":"#0000FF"}},{"type":"scatter","inherit":false,"x":[-0.93,-0.8,-0.79,-0.77,-0.62,0.62,0.7,0.77,0.78,0.8,0.82,0.84,0.85,0.86,0.87,0.87,0.87,0.89,0.92,0.93,0.95,0.95,0.96,0.96,0.98,0.99,0.72,0.82,0.86,0.91,0.94,0.95,0.99],"y":[1.92081875395238,1.87614835903291,2.27002571430044,1.43889861635094,1.31605286924849,1.72584215073632,1.39147396642281,2.70553377383841,2.05060999335509,1.47625353318844,1.61798295742513,1.59345981956604,2.32513885926219,1.90657831483776,3.28819277095881,1.60906489289662,1.54821356447571,1.34872198600186,2.14752000636314,2.46852108295775,1.75448733218585,1.61798295742513,1.36754270781528,1.36754270781528,1.57511836336893,1.51286162452281,1.42481215507234,1.84466396253494,1.63078414258986,1.55909091793478,2.58004425151024,1.75448733218585,1.78781239559604],"text":["CREM","NAALADL2","BCAS1","SETD5","ARHGAP20","AKNAD1","RAI14","ARHGAP42","DUX4L2","RUSC2","SIK3","ABL1","ASAP2","IFI6","JAM3","CDHR3","RTN3","VTI1A","STOM","IRF2BP2","PPP2R5C","C1orf122","FAM187A","CCDC103","GPR110","SATB2","MALAT1","CTD-2282P23.1","MIR607","RPS4XP2","snoU13","CTD-2017C7.2","CTC-497E21.5"],"mode":"markers","name":"signif","marker":{"color":"#FFA500"}},{"type":"scatter","inherit":false,"x":[-0.74,-0.71,-0.21,-0.18,-0.08,-0.07,-0.04,-0.02,-0.01,0.02,0.06,0.08,0.13,0.2,0.23,0.27,0.32,0.37,0.39,0.42,0.43,0.43,0.45,0.48,0.49,0.51,0.52,0.53,0.54,0.56,0.58,0.59,0.6,0.61,0.62,0.63,0.63,0.64,0.64,0.64,0.66,0.66,0.66,0.68,0.69,0.7,0.75,0.75,0.76,0.76,0.76,0.78,0.78,0.79,0.81,0.82,0.82,0.85,0.86,0.87,0.88,0.89,0.91,0.93,0.94,0.96,-0.67,-0.52,-0.48,-0.14,-0.06,-0.04,0.06,0.09,0.31,0.34,0.36,0.39,0.42,0.45,0.48,0.48,0.54,0.57,0.58,0.58,0.62,0.62,0.64,0.64,0.68,0.7,0.75,0.76,0.88,0.9,0.9,0.91,0.91,0.98],"y":[0.863279432843593,0.655607726314889,0.266802734893431,0.112945621949043,0.0545314148681803,0.0655015487564323,0.0472075569559079,0.00436480540245009,0.0145735259169983,0.0366844886138887,0.0639892042847904,0.0969100130080564,0.102372908709559,0.371611069949688,0.254925208417942,0.405607449624573,0.442492798094342,0.422508200162775,0.309803919971486,0.336299074610352,0.659555885159882,0.504455662453552,0.281498311132726,0.408935392973501,0.847711655616944,1.0329202658555,0.534617148551582,0.42021640338319,1.18309616062434,0.623423042943488,0.752026733638193,0.85078088734462,0.440093374963888,0.812479279163537,0.910094888560602,0.97061622231479,0.598599459218456,1.01144104312138,0.863279432843593,0.723538195826756,0.655607726314889,0.522878745280338,0.522878745280338,0.943095148663527,0.954677021213343,1.11294562194904,1.04287180232319,1.04287180232319,1.01863449092146,0.847711655616944,0.669586226650809,0.707743928643524,0.707743928643524,0.978810700930062,0.742321425130815,0.856985199745905,0.769551078621726,0.657577319177794,0.863279432843593,0.610833915635468,0.756961951313706,0.7594507517174,1.07987667370928,1.2839966563652,1.09528445472132,1.01055018233331,1.29157909986529,0.528708288941061,0.667561540084395,0.133122185662501,0.0888423912600234,0.0159229660971692,0.111259039317107,0.0931264652779296,0.167491087293764,0.576754126063192,0.474955192963155,0.806875401645538,1.02594909720712,0.694648630553376,0.777283528852417,0.450996737974212,1.20273245916928,0.517126416391246,0.612610173661271,0.505845405981557,0.752026733638193,0.609064892896621,0.939302159646388,0.586700235918748,0.609064892896621,0.602059991327962,0.661543506395395,1.14569395819892,0.872895201635192,0.931814138253838,0.931814138253838,1.03338901331807,0.920818753952375,1.06905096883248],"text":["PPP1R3C","MECOM","ANKRD28","PLEKHA8","CPNE4","NFE2L3","WNK1","HMGCS1","GFPT1","FSTL1","MACF1","RP11-362K2.2","FMNL3","PDE4DIP","NDUFC1","DUSP10","TNFRSF19","PAPPA2","THADA","HIST1H3E","CORO1C","ISPD","BRD2","SH3GL2","MTNR1B","VGLL4","CTNND1","ZNRF2","CLCN3","PTK2","GAS8","ADAMTS6","C6orf100","CNKSR3","CD300LB","CD59","ABLIM1","CREB5","DIAPH1","TENM2","MAML2","STRADB","TRAK2","SCMH1","SPAG17","TM4SF20","LRRTM2","CTNNA1","CACNA2D3","SPRY1","NUDT2","STK17A","COA1","PI3","DFNA5","TSHZ2","IGF2BP3","SNX9","PDE1C","PRKCE","METTL7A","FGGY","TYRO3","HMMR","ATPAF1","CDC27","RP11-644L4.1","AC013410.2","LINC00352","AC068570.1","RPL26P31","RP1-167F1.2","CTB-164N12.1","NALCN-AS1","AC017080.1","RP11-470E16.1","RSL24D1P1","RP11-340M11.1","PRICKLE2-AS3","AL357519.2","RP11-753H16.3","LINC00382","LINC01132","AL672294.1","AC096559.1","RP11-167H9.6","RP11-420A23.1","LINC00886","RP11-1L12.3","RP11-180P8.1","RNU6-793P","RPL23AP46","RP11-361D14.2","RP5-855F14.1","AC006227.1","RP11-1080G15.1","RP11-1080G15.2","AC068535.2","MPPE1P1","TSEN15P1"],"mode":"markers","name":"not_signif","marker":{"color":"#BEBEBE"}},{"type":"scatter","inherit":false,"x":[1.02,1.03,1.05,1.05,1.09,1.1,1.11,1.11,1.18,1.78,1.16,1.27],"y":[1.26440110030182,0.903089986991944,1.22040350874218,1.10846254232744,1.29073003902417,1.0893755951108,0.749579997691106,0.749579997691106,1.15864052954514,1.28988263488818,1.26520017041115,0.71669877129645],"text":["LIMD1","FNBP1","PKN2","TBC1D22B","ANLN","RAP1GAP","C8orf76","ZHX1-C8ORF76","WASF2","EFCAB4B","RP11-731C17.1","RP11-404J23.1"],"mode":"markers","name":"fc","marker":{"color":"#FF0000"}}],"layout":{"xaxis":{"title":"Fold"},"yaxis":{"title":"-log10(p.value)"},"hovermode":"closest","margin":{"b":40,"l":60,"t":25,"r":10}},"url":null,"width":null,"height":null,"source":"A","config":{"modeBarButtonsToRemove":["sendDataToCloud"]},"base_url":"https://plot.ly"},"evals":[],"jsHooks":[]}</script><!--/html_preserve-->

```r
# DiffBind_volcano_plot_top_protein_coding_promoters(diff_df = diff_df)
Diff_stats(diff_df = diff_df)
```

  
  
  
******  
### Overall DiffBind Stats  
Total number of DiffBind peaks:  
3192  
  
Total number of DiffBind genes:  
1824  
  
Total number positive fold change genes:  
1474  
  
Total number negative fold change genes:  
350  
  
  
******  
Total number of p <0.05 genes:  
1712  
  
Total number of p <0.05 genes (pos. FC):  
1377  
  
Total number of p <0.05 genes (neg. FC):  
335  
  
  
******  
Total number of log2(Fold Change) > 1 genes:  
1691  
  
Total number of log2(Fold Change) > 1 genes (pos. FC):  
1361  
  
Total number of log2(Fold Change) > 1 genes (neg. FC):  
330  
  
  
******  
Total number of p < 0.05 & log2(Fold Change) > 1 genes:  
1679  
  
Total number of p < 0.05 & log2(Fold Change) > 1 genes (pos. FC):  
1349  
  
Total number of p < 0.05 & log2(Fold Change) > 1 genes (neg. FC):  
330  
  
  
  
  
******  
### Protein Coding Gene Stats  
Total number of DiffBind peaks:  
2030  
  
Total number of DiffBind genes:  
1113  
  
Total number positive fold change genes:  
918  
  
Total number negative fold change genes:  
195  
  
  
******  
Total number of p <0.05 genes:  
1037  
  
Total number of p <0.05 genes (pos. FC):  
851  
  
Total number of p <0.05 genes (neg. FC):  
186  
  
  
******  
Total number of log2(Fold Change) > 1 genes:  
1021  
  
Total number of log2(Fold Change) > 1 genes (pos. FC):  
840  
  
Total number of log2(Fold Change) > 1 genes (neg. FC):  
181  
  
  
******  
Total number of p < 0.05 & log2(Fold Change) > 1 genes:  
1011  
  
Total number of p < 0.05 & log2(Fold Change) > 1 genes (pos. FC):  
830  
  
Total number of p < 0.05 & log2(Fold Change) > 1 genes (neg. FC):  
181  
  
  
  
******  
  
******  

```r
sample_file <- sample_file_list[2]
mycat(paste0("## ", names(sample_file), ' {.tabset}\n'))
```

## Sample4_Sample5 {.tabset}  

```r
diff_df <- read.delim(file = sample_file,header = TRUE,sep = ',')
DiffBind_volcano_plotly_top_protein_coding_promoters(diff_df = Diff_process_data(diff_df))
```

### Volcano Plot: Plot.ly  
<!--html_preserve--><div id="htmlwidget-6aa8fd3aee287acbfb00" style="width:768px;height:768px;" class="plotly html-widget"></div>
<script type="application/json" data-for="htmlwidget-6aa8fd3aee287acbfb00">{"x":{"data":[{"type":"scatter","inherit":false,"x":[-0.99,-0.8,-0.76,-0.75,-0.7,-0.65,-0.64,-0.63,-0.63,-0.61,-0.61,-0.61,-0.6,-0.59,-0.58,-0.58,-0.53,-0.51,-0.46,-0.46,-0.46,-0.39,-0.89,-0.72,-0.7,-0.7,-0.7,-0.68,-0.67,-0.64,-0.58,-0.57,-0.55,-0.51,-0.49,-0.48,-0.43],"y":[4.3840499483436,2.69464863055338,2.37161106994969,2.87289520163519,2.53313237964589,2.31069114087638,1.74472749489669,1.97469413473523,1.97469413473523,2.0017406615763,1.58335949266172,1.58335949266172,2.34582345812204,1.46092390120722,2.00305075150462,1.84466396253494,1.50584540598156,1.36552272983927,1.65955588515988,1.33724216831843,1.33441900898205,1.31515463835559,2.84163750790475,2.24336389175415,2.52578373592374,2.32975414692588,2.16749108729376,1.86012091359876,2.32605800136591,1.74472749489669,2.25492520841794,1.58169870868025,1.34969247686806,1.64781748188864,1.54975089168064,1.40340290437354,1.57511836336893],"text":["PARK2","ETS1","VPS13D","BBOX1","BMP7","ADCYAP1R1","C12orf65","DISC1","TSNAX-DISC1","LRIG1","B3GALT2","CDC73","SKAP2","PRKCA","TEAD1","CPVL","DCP2","PPAP2B","NCAM1","C1orf61","DCLK2","SGK223","AC009313.2","RPL21P67","AC087269.2","LINC00856","RNA5SP181","RP11-1L12.3","ARHGEF26-AS1","RP11-282O18.3","RP5-978I12.1","ANKRD26P1","RNU6-1096P","RNU6-487P","RP11-863K10.2","RN7SL366P","SRGAP2B"],"mode":"markers","name":"signif","marker":{"color":"#00FF00"}},{"type":"scatter","inherit":false,"x":[-0.52,-0.52,-0.5,-0.5,-0.5,-0.5,-0.49,-0.49,-0.46,-0.45,-0.43,-0.43,-0.41,-0.4,-0.4,-0.4,-0.39,-0.38,-0.38,-0.37,-0.36,-0.35,-0.34,-0.34,-0.33,-0.33,-0.33,-0.33,-0.32,-0.32,-0.32,-0.32,-0.32,-0.31,-0.3,-0.3,-0.3,-0.29,-0.29,-0.29,-0.28,-0.27,-0.27,-0.27,-0.27,-0.27,-0.26,-0.26,-0.26,-0.26,-0.26,-0.25,-0.25,-0.25,-0.24,-0.24,-0.24,-0.23,-0.23,-0.23,-0.23,-0.22,-0.21,-0.21,-0.2,-0.2,-0.19,-0.19,-0.19,-0.19,-0.18,-0.17,-0.17,-0.17,-0.16,-0.15,-0.15,-0.15,-0.14,-0.14,-0.14,-0.13,-0.12,-0.11,-0.11,-0.1,-0.08,-0.08,-0.08,-0.04,-0.03,-0.03,-0.03,-0.01,0,0.02,0.05,0.08,0.09,0.1,0.12,0.19,0.22,0.26,-0.5,-0.49,-0.47,-0.46,-0.46,-0.46,-0.45,-0.44,-0.41,-0.4,-0.4,-0.4,-0.38,-0.38,-0.35,-0.35,-0.34,-0.34,-0.34,-0.32,-0.32,-0.31,-0.31,-0.31,-0.31,-0.3,-0.3,-0.29,-0.29,-0.27,-0.27,-0.25,-0.24,-0.23,-0.23,-0.22,-0.22,-0.22,-0.22,-0.2,-0.19,-0.19,-0.17,-0.15,-0.15,-0.14,-0.14,-0.13,-0.11,-0.08,-0.06,-0.06,-0.06,-0.04,-0.03,-0.01,0.03,0.03,0.05,0.07,0.08,0.1,0.19,0.2,0.32,0.39],"y":[1.26280735729526,1.08144546944973,1.29499204066666,1.2839966563652,1.2580609222708,1.19586056766465,1.13548891894161,1.12436006299583,1.1944991418416,1.2298847052129,1.17522353752445,0.943095148663527,0.896196279044043,1.17198493577602,1.12262865413023,1.040481623027,0.886056647693163,0.79317412396815,0.769551078621726,0.844663962534938,0.721246399047171,0.709965388637482,1.09799710864927,0.718966632752272,1.00877392430751,1.00305075150462,0.876148359032914,0.804100347590766,1.0783135245164,0.913640169325252,0.906578314837765,0.667561540084395,0.667561540084395,0.749579997691106,0.847711655616944,0.718966632752272,0.568636235841013,0.809668301829709,0.583359492661719,0.539102157243452,0.531652669587843,0.950781977329818,0.924453038607469,0.573488738635425,0.505845405981557,0.504455662453552,0.657577319177794,0.619788758288394,0.512861624522813,0.505845405981557,0.474955192963155,0.643974142806877,0.576754126063192,0.468521082957745,0.616184634019569,0.3585258894959,0.3585258894959,0.605548319173784,0.460923901207223,0.404503778174426,0.351639989019068,0.607303046740334,0.539102157243452,0.473660722610156,0.341988603342888,0.341988603342888,0.42712839779952,0.373659632624958,0.35359627377693,0.326058001365912,0.294136287716081,0.328827157284917,0.289882634888184,0.272458742971444,0.332547047110046,0.298432014944073,0.268411234813261,0.242603971206976,0.27083521030723,0.254925208417942,0.213248577854439,0.276544327964814,0.189767482004916,0.1944991418416,0.190440285364732,0.122628654130226,0.11975822410452,0.110138278741812,0.10790539730952,0.0457574905606751,0.0814454694497265,0.0660068361687577,0.0199966284162537,0.0114410431213845,0.0213630516155257,0.0264103765727431,0.0570004066339595,0.116338564846382,0.17134010346468,0.214670164989233,0.193141970481183,0.188424994129407,0.617982957425132,0.431798275933005,1.19586056766465,1.25649023527157,1.0329202658555,1.12959609472097,1.06600683616876,1.06198090252379,1.08618614761628,1.30102999566398,0.910094888560602,1.18375870000822,1.12262865413023,1.04191415147891,0.991399828238082,0.847711655616944,0.772113295386326,0.772113295386326,1.00568284733036,0.818156412055227,0.709965388637482,1,0.913640169325252,0.749579997691106,0.723538195826756,0.632644078973981,0.612610173661271,1.07883394936226,0.718966632752272,0.677780705266081,0.549750891680639,1.03245202378114,0.950781977329818,0.98296666070122,0.397940008672038,0.435333935747911,0.351639989019068,0.490797477668897,0.462180904926726,0.446116973356126,0.42021640338319,0.38933983691012,0.442492798094342,0.368556230986828,0.289882634888184,0.326979092871104,0.24184537803261,0.274905478918531,0.216096420727265,0.216811308924742,0.185086818724926,0.158015195409886,0.145086977692144,0.118045028660399,0.115771230367396,0.0877779434675845,0.0670191780768018,0.0114410431213845,0.0716041477432862,0.0362121726544447,0.0545314148681803,0.111820506081675,0.161780778092374,0.126679398184601,0.281498311132726,0.419075024324381,0.707743928643524,0.71669877129645],"text":["COA1","CLCN3","HEPACAM","CHMP4B","ATP5G2","SERINC5","SCARB2","ASAP2","ETV1","AKNAD1","PPM1D","SEPT7","ETV5","FREM2","PRICKLE2","LAP3","NRCAM","SIAE","SCRN1","CHIT1","LRP4","ADAM22","KIAA1614","FYN","ARHGAP35","ZCCHC3","GLIS3","ATXN7L1","RRP8","LARS2","ACER3","NAV2","NTM","MAP2K6","ENOX1","ITPKB","IRF2BP2","KAT7","TCP10","BMPER","KCNA10","CUEDC1","DOK5","EIF4G3","RAB3GAP2","DUX4L2","SPATA6","NPTX2","GPM6A","NEGR1","POMT2","MAML2","PAMR1","POU3F2","NFIA","SYCE2","GCDH","SLC9A1","ST3GAL4","IGFBP2","ELMO1","ACSS1","MSI2","C11orf49","FXYD6-FXYD2","FXYD6","FAM181B","PDE4DIP","ANK2","RAD51B","PITPNC1","PRCP","USP2","GFAP","CROCC","DSCAML1","MICALCL","KLHDC8A","MAST2","JAM3","SSBP3","BOC","TMCC1","KCND3","MYO10","TBC1D16","CAPN9","CSPG5","SAP30BP","KAZN","SYT11","ID3","LRRC4C","LIMA1","CDK6","ARHGAP42","IQCE","SORL1","PDE4B","CREB5","PHC2","ANKRD44","PCNXL2","COLGALT2","snoU13","PNPT1P2","CTB-164N12.1","RP11-413N13.1","RP11-715C4.1","RP11-533E19.5","RNU6-256P","RP11-3L10.3","MALAT1","RPL26P31","PRICKLE2-AS3","AC099778.1","TRAF3IP2-AS1","RP11-6N17.10","CTA-390C10.10","CRYBB2P1","RP5-1065J22.8","PSPHP1","LINC00461","CTC-497E21.5","LARS2-AS1","RP1-193H18.3","LINC00474","CTD-2008N3.1","RP11-511B23.2","RP1-40E16.9","ITPKB-IT1","CYCSP26","Y_RNA","SOX9-AS1","RP11-343K8.3","RP11-448P19.1","RP11-344L21.1","AC083864.3","ELMO1-AS1","RPL21P41","AP003900.6","RP11-664D1.1","FAM58BP","RPL7P8","RP11-540K16.1","GS1-122H1.1","RP11-334E6.3","RP11-309G3.3","LINC01105","CTBP2P1","LINC01066","RP11-718B12.2","GS1-122H1.2","AC018647.3","RNU6-27P","CTB-70G10.1","RP11-693J15.5","RP11-620J15.2","SOX2-OT","RP3-405J10.3","RP11-14O19.2","LINC01137","CTD-2290P7.1","RP11-191L17.1","RP11-712B9.2","RP11-317N12.1","RP11-1008C21.2","RP11-134G8.7","HMGN2P19","RP11-32B5.1"],"mode":"markers","name":"not_signif","marker":{"color":"#FF0000"}}],"layout":{"xaxis":{"title":"Fold"},"yaxis":{"title":"-log10(p.value)"},"hovermode":"closest","margin":{"b":40,"l":60,"t":25,"r":10}},"url":null,"width":null,"height":null,"source":"A","config":{"modeBarButtonsToRemove":["sendDataToCloud"]},"base_url":"https://plot.ly"},"evals":[],"jsHooks":[]}</script><!--/html_preserve-->

```r
# DiffBind_volcano_plot_top_protein_coding_promoters(diff_df = diff_df)
Diff_stats(diff_df = diff_df)
```

  
  
  
******  
### Overall DiffBind Stats  
Total number of DiffBind peaks:  
279  
  
Total number of DiffBind genes:  
207  
  
Total number positive fold change genes:  
19  
  
Total number negative fold change genes:  
187  
  
  
******  
Total number of p <0.05 genes:  
37  
  
Total number of p <0.05 genes (pos. FC):  
0  
  
Total number of p <0.05 genes (neg. FC):  
37  
  
  
******  
Total number of log2(Fold Change) > 1 genes:  
0  
  
Total number of log2(Fold Change) > 1 genes (pos. FC):  
0  
  
Total number of log2(Fold Change) > 1 genes (neg. FC):  
0  
  
  
******  
Total number of p < 0.05 & log2(Fold Change) > 1 genes:  
0  
  
Total number of p < 0.05 & log2(Fold Change) > 1 genes (pos. FC):  
0  
  
Total number of p < 0.05 & log2(Fold Change) > 1 genes (neg. FC):  
0  
  
  
  
  
******  
### Protein Coding Gene Stats  
Total number of DiffBind peaks:  
176  
  
Total number of DiffBind genes:  
126  
  
Total number positive fold change genes:  
9  
  
Total number negative fold change genes:  
116  
  
  
******  
Total number of p <0.05 genes:  
22  
  
Total number of p <0.05 genes (pos. FC):  
0  
  
Total number of p <0.05 genes (neg. FC):  
22  
  
  
******  
Total number of log2(Fold Change) > 1 genes:  
0  
  
Total number of log2(Fold Change) > 1 genes (pos. FC):  
0  
  
Total number of log2(Fold Change) > 1 genes (neg. FC):  
0  
  
  
******  
Total number of p < 0.05 & log2(Fold Change) > 1 genes:  
0  
  
Total number of p < 0.05 & log2(Fold Change) > 1 genes (pos. FC):  
0  
  
Total number of p < 0.05 & log2(Fold Change) > 1 genes (neg. FC):  
0  
  
  
  
******  
  
******  

```r
sample_file <- sample_file_list[3]
mycat(paste0("## ", names(sample_file), ' {.tabset}\n'))
```

## Sample5_Control {.tabset}  

```r
diff_df <- read.delim(file = sample_file,header = TRUE,sep = ',')
DiffBind_volcano_plotly_top_protein_coding_promoters(diff_df = Diff_process_data(diff_df))
```

### Volcano Plot: Plot.ly  
<!--html_preserve--><div id="htmlwidget-fe2e9458b6f25482c932" style="width:768px;height:768px;" class="plotly html-widget"></div>
<script type="application/json" data-for="htmlwidget-fe2e9458b6f25482c932">{"x":{"data":[{"type":"scatter","inherit":false,"x":[-6.28,-6.18,-6.17,-6,-5.62,-5.41,-5.39,-5.37,-5.35,-5.32,5.37,5.41,5.43,5.43,5.5,5.5,5.52,5.58,5.62,5.78],"y":[14.8601209135988,14.3685562309868,14.3645162531851,13.3990271043133,11.7423214251308,10.7619538968712,10.6439741428069,10.6556077263149,10.4259687322723,10.4067139329795,13.4921441283042,10.7544873321858,10.7931741239682,10.7851561519523,11.181774106386,11.1203307943679,17.0496351456239,11.4881166390211,14.4078232426041,12.3223930472795],"text":["UQCC1","PMEPA1","CHD1L","PDP1","C8orf46","MRPL46","FAM78B","FCRLA","LSAMP","DPT","LAP3","RRP8","MTSS1","TRPS1","PITPNC1","DDX3Y","SGK223","RNF180","CROCC","HEPACAM"],"mode":"markers","name":"top_signif_fc","marker":{"color":"#00FF00"}},{"type":"scatter","inherit":false,"x":[-5.28,-5.27,-5.11,-5.1,-5.02,-5.02,-5.02,-4.97,-4.95,-4.94,-4.92,-4.91,-4.9,-4.89,-4.86,-4.86,-4.84,-4.78,-4.77,-4.77,-4.75,-4.75,-4.73,-4.72,-4.7,-4.64,-4.63,-4.63,-4.56,-4.56,-4.55,-4.52,-4.52,-4.52,-4.5,-4.45,-4.42,-4.4,-4.37,-4.36,-4.35,-4.35,-4.34,-4.33,-4.33,-4.29,-4.24,-4.23,-4.23,-4.21,-4.2,-4.16,-4.16,-4.16,-4.14,-4.13,-4.12,-4.09,-4.04,-4.04,-4.01,-3.97,-3.94,-3.91,-3.9,-3.9,-3.87,-3.87,-3.86,-3.85,-3.85,-3.84,-3.84,-3.79,-3.75,-3.75,-3.72,-3.7,-3.68,-3.67,-3.67,-3.65,-3.64,-3.64,-3.59,-3.56,-3.53,-3.51,-3.48,-3.46,-3.46,-3.4,-3.4,-3.35,-3.31,-3.31,-3.31,-3.29,-3.29,-3.27,-3.27,-3.26,-3.26,-3.24,-3.2,-3.2,-3.19,-3.16,-3.15,-3.15,-3.13,-3.12,-3.12,-3.11,-3.1,-3.07,-3.03,-3.02,-2.99,-2.95,-2.94,-2.94,-2.93,-2.87,-2.85,-2.84,-2.82,-2.82,-2.77,-2.74,-2.73,-2.67,-2.65,-2.63,-2.61,-2.59,-2.59,-2.53,-2.48,-2.46,-2.46,-2.46,-2.42,-2.42,-2.4,-2.36,-2.28,-2.25,-2.22,-2.22,-2.19,-2.11,-2.06,-1.93,-1.86,-1.83,-1.83,-1.79,-1.74,-1.68,-1.6,-1.41,-1.36,-1.21,-1.19,-1.13,-1.13,1.02,1.02,1.02,1.03,1.04,1.05,1.05,1.05,1.06,1.06,1.07,1.09,1.09,1.11,1.11,1.11,1.14,1.17,1.19,1.19,1.19,1.2,1.2,1.23,1.23,1.26,1.26,1.28,1.31,1.31,1.34,1.34,1.34,1.35,1.35,1.35,1.35,1.36,1.39,1.39,1.4,1.4,1.4,1.41,1.42,1.43,1.43,1.44,1.45,1.45,1.45,1.47,1.48,1.48,1.48,1.48,1.5,1.5,1.51,1.52,1.53,1.54,1.55,1.55,1.55,1.56,1.56,1.57,1.57,1.57,1.58,1.58,1.59,1.62,1.64,1.65,1.65,1.66,1.67,1.67,1.67,1.67,1.67,1.68,1.69,1.72,1.72,1.74,1.75,1.77,1.78,1.79,1.79,1.81,1.81,1.83,1.84,1.85,1.86,1.87,1.88,1.9,1.9,1.91,1.91,1.91,1.92,1.93,1.94,1.94,1.95,1.97,1.98,1.98,1.99,2,2,2,2.01,2.01,2.02,2.03,2.04,2.05,2.05,2.05,2.05,2.06,2.07,2.07,2.08,2.08,2.09,2.09,2.1,2.11,2.11,2.12,2.12,2.14,2.14,2.14,2.14,2.15,2.15,2.16,2.16,2.17,2.18,2.18,2.18,2.18,2.19,2.19,2.19,2.19,2.19,2.2,2.2,2.2,2.2,2.2,2.2,2.21,2.21,2.21,2.22,2.22,2.25,2.25,2.25,2.27,2.27,2.28,2.29,2.31,2.31,2.32,2.32,2.32,2.33,2.33,2.34,2.34,2.34,2.34,2.35,2.35,2.35,2.35,2.35,2.36,2.36,2.37,2.38,2.39,2.39,2.39,2.4,2.41,2.41,2.42,2.43,2.43,2.43,2.44,2.44,2.44,2.45,2.45,2.45,2.46,2.47,2.47,2.47,2.47,2.48,2.49,2.49,2.49,2.49,2.5,2.5,2.51,2.51,2.51,2.52,2.52,2.54,2.54,2.54,2.55,2.55,2.56,2.56,2.57,2.58,2.58,2.59,2.59,2.59,2.59,2.59,2.59,2.59,2.59,2.59,2.6,2.61,2.62,2.62,2.62,2.63,2.64,2.64,2.65,2.65,2.65,2.65,2.66,2.67,2.67,2.67,2.67,2.67,2.67,2.68,2.68,2.68,2.68,2.69,2.69,2.69,2.69,2.69,2.71,2.71,2.71,2.71,2.72,2.72,2.72,2.73,2.73,2.74,2.74,2.74,2.74,2.74,2.75,2.75,2.75,2.75,2.75,2.76,2.76,2.76,2.76,2.77,2.78,2.78,2.78,2.79,2.79,2.79,2.8,2.81,2.81,2.81,2.81,2.81,2.82,2.82,2.82,2.83,2.83,2.83,2.84,2.85,2.85,2.85,2.86,2.86,2.87,2.87,2.88,2.88,2.89,2.89,2.89,2.89,2.9,2.9,2.9,2.91,2.91,2.92,2.93,2.93,2.93,2.93,2.94,2.94,2.94,2.94,2.94,2.94,2.95,2.95,2.95,2.95,2.95,2.96,2.96,2.96,2.96,2.97,2.97,2.97,2.97,2.97,2.97,2.98,2.98,2.98,2.98,2.99,2.99,3,3,3,3,3.01,3.02,3.02,3.02,3.03,3.03,3.03,3.04,3.04,3.04,3.04,3.05,3.07,3.07,3.08,3.08,3.08,3.09,3.09,3.09,3.11,3.11,3.11,3.11,3.12,3.12,3.12,3.14,3.14,3.14,3.15,3.16,3.16,3.17,3.17,3.18,3.18,3.18,3.18,3.19,3.19,3.19,3.19,3.19,3.19,3.2,3.2,3.21,3.22,3.22,3.22,3.22,3.23,3.24,3.24,3.24,3.25,3.25,3.25,3.25,3.26,3.26,3.27,3.28,3.29,3.3,3.3,3.3,3.3,3.3,3.31,3.31,3.31,3.32,3.32,3.33,3.33,3.33,3.33,3.34,3.34,3.34,3.34,3.34,3.35,3.35,3.35,3.36,3.36,3.36,3.36,3.37,3.37,3.38,3.39,3.4,3.4,3.4,3.41,3.41,3.42,3.42,3.42,3.43,3.43,3.43,3.44,3.44,3.44,3.44,3.44,3.45,3.45,3.45,3.45,3.46,3.46,3.46,3.47,3.47,3.48,3.48,3.48,3.49,3.49,3.5,3.5,3.5,3.51,3.51,3.51,3.51,3.51,3.51,3.52,3.52,3.53,3.53,3.54,3.54,3.55,3.55,3.55,3.56,3.56,3.56,3.56,3.57,3.58,3.58,3.58,3.59,3.6,3.6,3.6,3.61,3.61,3.62,3.62,3.62,3.62,3.63,3.63,3.64,3.65,3.66,3.66,3.67,3.67,3.67,3.68,3.68,3.68,3.69,3.69,3.69,3.69,3.69,3.7,3.7,3.7,3.7,3.7,3.71,3.71,3.71,3.71,3.71,3.72,3.72,3.73,3.74,3.74,3.74,3.74,3.75,3.75,3.75,3.75,3.75,3.75,3.76,3.76,3.77,3.78,3.78,3.79,3.79,3.79,3.8,3.8,3.8,3.81,3.81,3.81,3.81,3.81,3.81,3.82,3.82,3.82,3.82,3.83,3.83,3.83,3.84,3.84,3.84,3.84,3.84,3.86,3.86,3.86,3.86,3.86,3.86,3.87,3.87,3.87,3.87,3.87,3.87,3.88,3.88,3.88,3.89,3.89,3.89,3.89,3.9,3.9,3.9,3.9,3.9,3.9,3.91,3.91,3.91,3.91,3.91,3.91,3.92,3.92,3.92,3.92,3.93,3.93,3.93,3.93,3.96,3.96,3.96,3.96,3.97,3.98,3.98,3.98,3.99,3.99,3.99,3.99,3.99,4,4,4,4,4.02,4.02,4.02,4.02,4.03,4.03,4.03,4.03,4.03,4.03,4.04,4.04,4.05,4.05,4.05,4.05,4.05,4.06,4.06,4.07,4.07,4.08,4.08,4.08,4.09,4.09,4.09,4.1,4.1,4.1,4.11,4.12,4.12,4.12,4.12,4.13,4.14,4.14,4.14,4.14,4.14,4.15,4.15,4.16,4.16,4.17,4.17,4.17,4.18,4.18,4.18,4.19,4.19,4.19,4.19,4.2,4.2,4.2,4.21,4.21,4.21,4.22,4.22,4.22,4.22,4.22,4.22,4.23,4.24,4.24,4.25,4.25,4.25,4.25,4.26,4.26,4.27,4.28,4.28,4.28,4.29,4.3,4.3,4.3,4.31,4.31,4.32,4.32,4.32,4.33,4.33,4.33,4.33,4.34,4.34,4.34,4.35,4.35,4.35,4.35,4.36,4.36,4.36,4.37,4.38,4.38,4.38,4.38,4.39,4.41,4.41,4.42,4.42,4.43,4.43,4.43,4.44,4.44,4.47,4.47,4.47,4.47,4.47,4.47,4.48,4.48,4.49,4.5,4.51,4.52,4.52,4.53,4.54,4.54,4.55,4.55,4.57,4.57,4.57,4.58,4.58,4.59,4.59,4.61,4.61,4.64,4.65,4.66,4.66,4.66,4.68,4.68,4.68,4.68,4.68,4.69,4.7,4.7,4.72,4.74,4.74,4.75,4.76,4.76,4.76,4.79,4.79,4.81,4.81,4.82,4.82,4.82,4.83,4.87,4.87,4.87,4.88,4.88,4.89,4.89,4.89,4.91,4.91,4.91,4.94,4.95,4.96,4.96,4.97,5.04,5.05,5.05,5.06,5.06,5.07,5.09,5.12,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.15,5.26,5.3,5.37,-6.47,-6.01,-5.84,-5.69,-5.47,-5.44,-5.43,-5.36,-5.29,-5.27,-5.25,-5.22,-5.17,-5.03,-5.02,-5.01,-4.96,-4.95,-4.94,-4.87,-4.86,-4.85,-4.83,-4.8,-4.8,-4.8,-4.72,-4.71,-4.68,-4.67,-4.63,-4.62,-4.6,-4.56,-4.56,-4.55,-4.52,-4.51,-4.51,-4.5,-4.47,-4.47,-4.46,-4.45,-4.44,-4.44,-4.43,-4.4,-4.38,-4.3,-4.29,-4.26,-4.25,-4.23,-4.23,-4.21,-4.21,-4.19,-4.16,-4.16,-4.15,-4.15,-4.14,-4.13,-4.09,-4.08,-4.08,-4.07,-4.07,-4.04,-4.02,-3.98,-3.98,-3.95,-3.95,-3.92,-3.9,-3.89,-3.89,-3.89,-3.87,-3.77,-3.76,-3.75,-3.73,-3.73,-3.71,-3.7,-3.65,-3.64,-3.61,-3.6,-3.59,-3.56,-3.49,-3.33,-3.29,-3.29,-3.28,-3.26,-3.19,-3.14,-3.13,-3.12,-3.11,-3.11,-3.09,-3.08,-3.07,-3.07,-3.06,-3.03,-3.03,-3.02,-2.94,-2.88,-2.86,-2.79,-2.73,-2.7,-2.65,-2.64,-2.61,-2.57,-2.55,-2.53,-2.4,-2.39,-2.28,-2.24,-2.16,-2.09,-2.06,-2.04,-1.92,-1.91,-1.82,-1.81,-1.76,-1.55,-1.44,-1.36,-1.36,-1.35,-1.34,-1.26,-1.24,-1.22,-1.01,1.04,1.09,1.13,1.13,1.17,1.23,1.24,1.29,1.29,1.29,1.31,1.31,1.33,1.38,1.4,1.43,1.44,1.47,1.48,1.5,1.5,1.53,1.55,1.56,1.65,1.66,1.69,1.69,1.7,1.71,1.71,1.73,1.74,1.74,1.77,1.8,1.82,1.83,1.84,1.85,1.85,1.87,1.87,1.88,1.88,1.9,1.9,1.91,1.92,1.93,1.96,1.97,1.99,2,2,2.01,2.02,2.05,2.05,2.05,2.07,2.07,2.14,2.15,2.15,2.16,2.17,2.19,2.19,2.2,2.22,2.22,2.24,2.24,2.25,2.25,2.27,2.28,2.28,2.29,2.32,2.32,2.32,2.33,2.33,2.35,2.38,2.39,2.39,2.39,2.39,2.39,2.4,2.41,2.41,2.41,2.42,2.42,2.43,2.44,2.45,2.46,2.46,2.47,2.48,2.49,2.52,2.54,2.54,2.54,2.55,2.58,2.58,2.58,2.6,2.61,2.64,2.65,2.65,2.65,2.66,2.67,2.69,2.69,2.71,2.72,2.74,2.74,2.76,2.78,2.8,2.81,2.81,2.82,2.85,2.85,2.86,2.86,2.88,2.88,2.88,2.88,2.89,2.9,2.91,2.92,2.93,2.93,2.94,2.94,2.95,2.96,2.97,2.98,2.98,2.99,3,3,3.01,3.01,3.02,3.02,3.02,3.03,3.04,3.04,3.04,3.04,3.04,3.06,3.09,3.1,3.12,3.12,3.12,3.13,3.14,3.14,3.15,3.16,3.17,3.17,3.19,3.2,3.2,3.22,3.24,3.25,3.25,3.26,3.26,3.27,3.28,3.28,3.29,3.3,3.3,3.3,3.31,3.32,3.33,3.35,3.36,3.36,3.36,3.37,3.38,3.4,3.4,3.4,3.4,3.4,3.4,3.41,3.41,3.41,3.41,3.42,3.42,3.42,3.42,3.44,3.44,3.44,3.44,3.45,3.45,3.45,3.46,3.46,3.48,3.49,3.49,3.5,3.51,3.51,3.51,3.51,3.52,3.52,3.52,3.53,3.53,3.53,3.55,3.56,3.56,3.56,3.57,3.57,3.57,3.57,3.58,3.59,3.6,3.6,3.6,3.61,3.61,3.62,3.62,3.62,3.63,3.63,3.63,3.64,3.64,3.65,3.65,3.65,3.65,3.66,3.66,3.67,3.67,3.68,3.7,3.7,3.7,3.71,3.73,3.74,3.74,3.76,3.76,3.76,3.76,3.76,3.77,3.77,3.77,3.77,3.78,3.78,3.78,3.79,3.79,3.79,3.79,3.8,3.8,3.8,3.8,3.8,3.81,3.81,3.81,3.81,3.83,3.84,3.85,3.85,3.85,3.85,3.85,3.85,3.86,3.86,3.86,3.87,3.87,3.88,3.88,3.88,3.88,3.88,3.88,3.89,3.89,3.89,3.89,3.89,3.89,3.89,3.9,3.9,3.92,3.93,3.93,3.93,3.93,3.93,3.93,3.94,3.94,3.95,3.96,3.97,3.97,3.97,3.98,3.98,3.98,3.98,3.98,3.99,3.99,3.99,3.99,4,4,4,4,4.01,4.02,4.03,4.03,4.04,4.04,4.04,4.05,4.05,4.05,4.06,4.06,4.06,4.06,4.07,4.07,4.07,4.07,4.07,4.08,4.09,4.1,4.1,4.11,4.11,4.12,4.13,4.14,4.15,4.15,4.15,4.17,4.17,4.17,4.18,4.18,4.18,4.19,4.19,4.2,4.21,4.21,4.21,4.21,4.21,4.22,4.22,4.23,4.23,4.23,4.23,4.23,4.24,4.25,4.25,4.26,4.26,4.26,4.26,4.28,4.28,4.29,4.29,4.3,4.3,4.3,4.31,4.31,4.32,4.33,4.34,4.34,4.34,4.34,4.35,4.36,4.36,4.37,4.38,4.38,4.39,4.41,4.43,4.43,4.43,4.43,4.43,4.43,4.44,4.45,4.45,4.45,4.46,4.46,4.46,4.47,4.48,4.48,4.49,4.5,4.5,4.5,4.5,4.51,4.54,4.55,4.56,4.56,4.56,4.57,4.58,4.59,4.59,4.6,4.6,4.62,4.62,4.63,4.63,4.64,4.66,4.67,4.68,4.68,4.68,4.68,4.69,4.69,4.71,4.74,4.76,4.77,4.79,4.79,4.8,4.8,4.81,4.82,4.83,4.87,4.87,4.87,4.89,4.9,4.92,4.93,4.94,4.96,4.97,5,5.03,5.03,5.11,5.12,5.12,5.13,5.14,5.17,5.18,5.18,5.19,5.21,5.24,5.25,5.25,5.29,5.29,5.44,5.49,5.49,5.49,5.6,5.92],"y":[10.0501222959631,10.1095789811991,9.40671393297954,9.00568284733036,16.2549252084179,9.09205147838773,9.0515870342214,8.92081875395237,8.7851561519523,13.7189666327523,8.48811663902113,12.7471469690201,11.0579919469777,12.7144426909922,10.8124792791635,8.34872198600186,8.32790214206428,7.87289520163519,10.033858267261,8.04143611677803,7.96657624451305,7.83863199776502,7.92081875395237,7.78251605578609,7.80966830182971,7.34486156518862,15.1487416512809,7.47886191629596,10.8068754016455,7.24872089601666,7.22841251911874,7.11013827874181,7.11013827874181,7.04914854111145,7.03905380426617,8.50445566245355,8.52432881167557,6.52432881167557,6.52578373592374,11.3098039199715,19.6363880201079,19.6363880201079,6.4907974776689,6.42829116819131,6.16241156176449,11.1260984021355,7.60380065290426,9.00261361560269,6.09854167860389,6.03432802877989,7.70996538863748,12.0467236633327,12.0467236633327,7.50584540598156,13.5451551399915,10.4571745730408,7.3936186348894,8.76447155309245,10.6757175447023,8.58670023591875,12.5654310959658,9.70333480973847,9.11407366019857,11.9546770212133,7.17263072694618,5.10790539730952,5.93181413825384,4.99567862621736,7.40340290437354,7.10846254232744,7.10846254232744,9.73048705578208,7.58004425151024,12.0969100130081,10.9393021596464,6.76700388960785,5.55752023093555,14.6345120151091,12.2773660774662,8.82681373158773,7.35261702988538,10.115204636051,9.03338901331807,7.09258863922541,6.30627305107635,14.345823458122,5.89279003035213,5.79588001734407,8.62160209905186,11.2749054789185,7.23882418684427,8.53313237964589,7.45469288353418,7.1605219526258,9.0467236633327,6.36251027048749,4.82681373158773,8.69464863055338,7.07987667370928,6.97061622231479,6.86966623150499,9.43062609038495,9.18309616062434,7.35654732351381,16.430626090385,8.53461714855158,9.88272870434424,7.05799194697769,7.25492520841794,5.3829996588791,11.3946949538589,8.7619538968712,6.32422165832592,5.04287180232319,5.57839607313017,5.82973828460504,7.06550154875643,12.430626090385,8.03857890593355,5.7619538968712,8.28066871301627,6.62708799702989,10.7166987712964,7.36251027048749,5.73518217699046,5.02456819149074,9.17783192063198,9.17783192063198,6.79860287567955,11.1438755557577,4.55595520408192,4.56066730616974,6.58838029403677,5.39147396642281,5.15428198203334,8.99139982823808,8.70114692359029,7.69250396208679,4.26600071346161,7.44977164694491,5.65955588515988,5.48280410205003,5.53910215724345,4.97061622231479,7.5016894462104,5.97881070093006,5.40011692792631,4.75696195131371,5.12033079436795,3.74472749489669,5.53165266958784,7.6458915608526,6.82973828460504,10.1592667653882,2.91721462968355,6.27245874297144,6.27245874297144,9.73048705578208,5.08196966321512,6.41566877563247,4.10568393731556,3.43889861635094,4.46852108295774,6.12784372725171,3.28316227670048,3.16621562534352,1.76447155309245,3.09097914578884,2.44009337496389,1.99139982823808,2.1771783546969,2.5543957967264,3.70774392864352,1.93930215964639,1.70996538863748,3.89962945488244,2.74714696902011,4.59176003468815,2.96257350205938,1.65364702554936,3.55595520408192,2.89619627904404,1.52143350440616,2.19111413264019,3.36451625318509,3.63078414258986,2.44611697335613,2.35457773065091,4.07779372256098,1.87942606879415,3.36552272983927,2.42596873227228,4.04914854111145,2.09044397075882,3.33629907461035,2.71669877129645,2.44369749923271,5.33818731446274,2.61261017366127,2.49620931694282,4.78781239559604,4.37468754903833,2.2839966563652,1.84771165561694,3.84771165561694,3.06449273417529,2.28650945690606,5.74714696902011,3.57024771999759,3.3840499483436,4.46597389394386,3.91364016932525,4.23136189875239,3.73048705578208,1.87614835903291,4.06803388527183,3.10679324694015,2.79048498545737,5.12551818230053,4.08407278830288,4.08407278830288,2.53017798402184,2.44611697335613,5.46980030179692,3.39147396642281,4.25026368443094,4.33161408331,3.74472749489669,5.75696195131371,7.93554201077308,3.98716277529483,3.35951856302958,4.59345981956604,4.11747546204512,3.87289520163519,3.60906489289662,3.60906489289662,3.45593195564972,3.45593195564972,4.24412514432751,3.30627305107635,5.50723961097316,6.38510278396687,4.74472749489669,3.23507701535011,7.36451625318509,4.34103515733556,3.16621562534352,2.9100948885606,2.9100948885606,2.41793663708829,2.06348625752111,6.97061622231479,3.2335871528876,3.58502665202918,4.3585258894959,5.10182351650232,4.55909091793478,5.22112552799726,2.21395878975745,4.80966830182971,4.04914854111145,3.73048705578208,4.09420411963213,3.27408836770495,3.1505805862031,4.66154350639539,4.99567862621736,4.87942606879415,3.62160209905186,7.05502409158795,4.31515463835559,4.18976748200492,4.69680394257951,3.76700388960785,6.96657624451305,3.93930215964639,3.19654288435159,4.15989390554324,4.44733178388781,4.07058107428571,9.79860287567955,4.83564714421556,4.79860287567955,4.29413628771608,5.15490195998574,3.36151074304536,4.26121944151563,5.1505805862031,4.88941028970075,5.18243463044022,4.86327943284359,4.51144928349956,3.93181413825384,4.01233373507373,4.59687947882418,3.83268266525182,11.7212463990472,3.53017798402184,8.15801519540989,4.1001794975729,7.32975414692588,8.87289520163519,4.51144928349956,4.21824462534753,3.85078088734462,10.7423214251308,5.52724355068279,4.58004425151024,3.53760200210104,6.65955588515988,4.51144928349956,7.5185573714977,5.30980391997149,8.09205147838773,6.44369749923271,5.85078088734462,4.89962945488244,4.22767829327708,7.41566877563247,6.77728352885242,3.78251605578609,3.56703070912559,3.04095860767891,10.1079053973095,8.57675412606319,8.36754270781528,7.5543957967264,7.44611697335613,3.69464863055338,8.7619538968712,6.25336580106242,6.25336580106242,5.56863623584101,3.12901118623942,7.92445303860747,5.47237009912866,5.3269790928711,8.63264407897398,4.68613277963085,8.97061622231479,4.59006687666871,9.31605286924849,4.07007043991541,9.60032627851896,6.45842075605342,5.70114692359029,6.97061622231479,5.53017798402184,12.2448877336049,9.50584540598156,7.53610701101409,5.44009337496389,8.51855737149769,6.41228903498109,6.19859628998265,5.98296666070122,5.24108810760203,6.82102305270683,4.94692155651658,9.28819277095881,6.03105031901866,6.75448733218585,6.63078414258986,5.31336373073771,3.93554201077308,12.7619538968712,7.09745322068601,9.80687540164554,7.48678239993206,7.06651271215129,7.06651271215129,6.26921772433361,5.5654310959658,5.27490547891853,10.0501222959631,4.94692155651658,4.85387196432176,8.57511836336893,11.0629838925352,8.03905380426617,6.67162039656126,5.06752623532285,5.00480370840282,11.3297541469259,11.3297541469259,6.49894073778225,5.83863199776502,8.48017200622428,4.30803489723264,5.40120949323688,4.19997064075587,4.19997064075587,8.55752023093555,7.99567862621736,9.46597389394386,7.94309514866353,4.64206515299955,14.1739251972992,6.02594909720712,4.74714696902011,3.95860731484178,6.95467702121334,5.83268266525182,4.00261361560269,14.6439741428069,10.6615435063954,5.16494389827988,5.0433514207948,4.85698519974591,4.57186520597121,3.96257350205938,3.96257350205938,3.66554624884907,5.79588001734407,6.40120949323688,6.99139982823808,5.53313237964589,5.26760624017703,7.49757288001557,10.0218194830626,8.22914798835786,12.0772745420067,10.1426675035687,7.57675412606319,6.65955588515988,6.57186520597121,12.9430951486635,11.2006594505464,10.2823294969977,5.2518119729938,5.2518119729938,4.95078197732982,13.9586073148418,11.5391021572435,8.22767829327708,6.8153085691824,16.3726341434073,8.90657831483776,5.91364016932525,4.97469413473523,4.56383735295924,7.68402965454308,7.11633856484638,6.70553377383841,6.30980391997149,10.2628073572953,4.51427857351842,4.44611697335613,11.5272435506828,4.88605664769316,12.0629838925352,8.55752023093555,7.08884239126002,6.89619627904404,6.29756946355447,9.51144928349956,8.40671393297954,5.23957751657679,4.00788851221305,3.99139982823808,12.7594507517174,7.10347378251045,6.41680122603138,4.88272870434424,14.1771783546969,11.2182446253475,10.4179366370883,5.64016451766011,16.2967086218813,9.96257350205938,4.79048498545737,11.9507819773298,11.5751183633689,10.279840696594,5.7281583934635,5.13548891894161,4.97881070093006,9.26841123481326,6.34198860334289,4.66354026615147,7.63264407897398,7.0195421077239,4.16241156176449,7.36051351073141,18.6382721639824,7.20273245916928,5.82681373158773,6.73992861201492,4.72124639904717,11.6882461389442,4.39794000867204,7.42596873227228,6.48545224733971,6.29929628285498,6.17783192063198,6.11069829749369,5.00744648216786,11.7798919119599,9.6903698325741,8.79588001734407,12.0867160982396,9.54515513999149,9.75696195131371,7.92445303860747,6.65560772631489,5.67366413907125,4.21253952548158,8.44855000202712,8.35163998901907,6.6903698325741,4.70996538863748,4.62342304294349,4.35753547975788,7.90308998699194,7.67366413907125,6.48017200622428,5.73992861201492,5.14569395819892,6.87614835903291,6.02227639471115,5.09799710864927,4.79317412396815,10.8210230527068,9.34775365899668,9.33535802444387,8.42250820016277,4.74714696902011,4.69897000433602,7.97469413473523,7.87614835903291,7.51427857351842,5.16494389827988,8.1073489661227,7.63078414258986,9.02548830726267,6.90657831483777,6.57186520597121,5.2335871528876,11.3840499483436,11.6497519816658,9.02733440773389,4.75202673363819,8.81247927916354,8.28149831113273,5.98716277529483,9.87942606879415,6.1232050237993,5.77211329538633,5.37986394502624,5.12147820449879,10.8996294548824,5.34008379993015,9.4672456210075,6.50863830616573,4.64016451766011,10.170696227169,8.1511952989482,4.90308998699194,6.84466396253494,6.18243463044022,4.58004425151024,4.32330639037513,7.95078197732982,6.38827669199266,5.31966448658544,8.79588001734407,7.03385826726097,6.11182050608168,7.79317412396815,17.5512936800949,5.19928292171762,7.86012091359876,7.31515463835559,11.9393021596464,7.39794000867204,5.34008379993015,4.63078414258986,12.301029995664,11.1090204030103,9.27408836770495,6.42365864979421,5.1001794975729,4.99139982823808,7.64975198166584,6.25414480482627,9.12901118623942,10.6439741428069,6.6345120151091,5.97061622231479,5.60554831917378,6.48811663902113,11.2433638917542,7.90308998699194,7.68613277963085,6.63638802010786,5.59006687666871,5.59006687666871,5.08884239126002,16.2139587897574,5.08039897621589,7.7851561519523,7.47886191629596,10.3904055907748,11.6458915608526,7.28735029837279,7.28735029837279,6.08354605145007,6.08354605145007,11.4659738939439,9.95078197732982,4.47755576649368,7.06752623532285,7.06752623532285,13.2898826348882,10.0644927341753,6.16621562534352,6.02273378757271,16.4989407377822,7.40011692792631,7.31605286924849,5.97061622231479,4.46218090492673,18.2146701649892,11.077793722561,6.32882715728492,7.42481215507234,7.09312646527793,6.59345981956604,4.40671393297954,10.8894102897008,7.54515513999149,6.80966830182971,6.55595520408192,7.69250396208679,5.2839966563652,5.2839966563652,10.3545777306509,6.24033215531037,6.86966623150499,4.73754891026957,4.6252516539899,8.09044397075882,7.50584540598156,3.65560772631489,12.5257837359237,5.72124639904717,5.63264407897398,3.80966830182971,3.7281583934635,8.88605664769316,8.76447155309245,5.91364016932525,4.80410034759077,8.01412464269161,6.48412615628832,4.99139982823808,8.67571754470231,3.95860731484178,10.6055483191738,10.3439017979872,6.87942606879415,15.2006594505464,5.66154350639539,8.21609642072726,7.01233373507373,4.9100948885606,11.3381873144627,10.6777807052661,7.90308998699194,6.04431224968649,5.03668448861389,5.03668448861389,7.64975198166584,3.64975198166584,5.01592296609717,4.06449273417529,6.83564714421556,6.08301995267962,11.8068754016455,6.16430942850757,5.22694530663574,16.5816987086803,8.1001794975729,6.13135556160517,4.17005330405836,8.99567862621736,9.85078088734462,7.34198860334289,6.21968268785985,12.9706162223148,15.5638373529592,6.50863830616573,4.24795155218056,10.2494916051487,7.2839966563652,15.5800442515102,12.2487208960167,12.2487208960167,5.29073003902417,7.9100948885606,5.24488773360493,7.77989191195995,5.44009337496389,14.3555614105322,14.3555614105322,7.88605664769316,6.95078197732982,6.66554624884907,6.56383735295924,6.54821356447571,6.5185573714977,12.2588484011482,10.1951793212788,6.97061622231479,6.95467702121334,5.39902710431325,7.7619538968712,7.73992861201492,6.82681373158773,6.49757288001557,4.44733178388781,11.0560111249262,6.4672456210075,5.7619538968712,4.5003129173816,4.35359627377693,5.73282827159699,4.56863623584101,8.97881070093006,8.45842075605342,6.63638802010786,5.85078088734462,5.7619538968712,13.8096683018297,9.56383735295924,5.72124639904717,5.72124639904717,5.72124639904717,4.52432881167557,9.36251027048749,7.16241156176449,10.2076083105017,5.85387196432176,4.55129368009492,10.8538719643218,5.98716277529483,4.58335949266172,8.20342566678957,4.74957999769111,4.54668165995296,8.68402965454308,7.25884840114821,6.96257350205938,6.08039897621589,6.00921730819686,5.91364016932525,6.30715308072277,6.29413628771608,4.82681373158773,4.82681373158773,9.18442225167573,7,6.03763066432998,7.22548303427145,7.11069829749369,6.02594909720712,6.02594909720712,4.91721462968355,9.25570701687732,7.55284196865778,6.47625353318844,5.00480370840282,4.98296666070122,4.97061622231479,6.34198860334289,6.20065945054642,5.05354773498693,5.04624030826677,4.97881070093006,4.9100948885606,10.5331323796459,8.86966623150499,6.04095860767891,8.40450377817443,7.16494389827988,6.32790214206428,4.88605664769316,10.1266793981846,7.77989191195995,6.37882371822496,5.11463877996849,5.07058107428571,5.0670191780768,14.0883098412461,7.66354026615147,6.65955588515988,6.43179827593301,6.34582345812204,5.12784372725171,11.5718652059712,10.698970004336,6.68613277963085,6.66154350639539,12.829738284605,5.23433144524099,5.22329881601159,5.16621562534352,12.546681659953,7.67366413907125,6.7851561519523,5.24108810760203,5.33348201944512,7.98296666070122,5.36351210364663,5.29929628285498,13.0996328713435,8.50863830616573,7.89962945488244,7.6675615400844,6.51286162452281,9.77728352885242,7.97469413473523,6.62342304294349,5.27408836770495,8.27408836770495,7.89619627904404,6.78781239559604,5.21538270736712,10.8013429130456,5.53313237964589,5.5185573714977,5.49349496759513,5.42136079003193,5.42021640338319,9.86646109162978,7.00043451177402,13.2572748686953,10.1444808443322,9.86012091359876,8.59006687666871,5.58838029403677,7.08407278830288,5.48017200622428,8.62893213772826,5.44733178388781,12.6861327796308,8.82390874094432,7.31605286924849,14.3535962737769,14.3535962737769,7.29157909986529,20.3506651412879,10.4522252946122,7.02594909720712,8.22914798835786,7.17522353752445,7.11747546204512,7.09205147838773,5.79317412396815,5.70774392864352,8.68824613894425,8.56703070912559,7.11182050608168,5.76447155309245,5.7594507517174,11.9507819773298,5.88272870434424,10.7099653886375,5.83564714421556,9.85387196432176,8.67778070526608,5.84466396253494,6.00261361560269,6.00261361560269,5.95860731484177,9.12959609472097,6.03432802877989,6.02181948306259,5.81247927916354,7.55909091793478,5.79860287567955,5.79860287567955,6.11069829749369,6.07468790850035,6.0096611452124,9.07520400420209,7.64397414280688,7.64397414280688,6.11633856484638,6.02965312376991,6.02594909720712,9.06550154875643,7.62342304294349,6.12262865413023,11.0209070993617,6.21609642072726,6.20273245916928,6.06499684854635,10.9507819773298,6.19791074211827,9.3429441471429,7.93181413825384,6.28988263488818,6.17069622716897,6.33629907461035,12.6655462488491,11.6458915608526,6.21041928783557,6.43179827593301,6.37263414340727,7.83564714421556,6.32422165832592,6.31425826139774,8.34872198600186,8.03526907894637,6.48280410205003,6.30715308072277,6.51999305704285,6.50863830616573,6.46470587995723,18.5451551399915,13.5030703519268,11.9100948885606,10.3746875490383,25.3260580013659,12.0352690789464,12.0048037084028,6.64397414280688,13.7931741239682,8.59345981956604,6.65169513695184,6.38510278396687,11.7304870557821,10.4271283977995,6.63827216398241,6.71669877129645,6.70553377383841,12.2240256688706,8.30364361126667,6.79048498545737,6.80134291304558,6.52870828894106,10.430626090385,8.57348873863542,6.96257350205938,6.92811799269387,6.86012091359876,6.75696195131371,17.9281179926939,6.91364016932525,10.5528419686578,8.8153085691824,7.03198428600636,7.16749108729376,7.09636748391576,9.35457773065091,7.14752000636314,7.03763066432998,9.40011692792631,7.16685288808721,11.6615435063954,7.28483264215154,7.25103713874384,14.6925039620868,7.28988263488818,9.0670191780768,7.38195190328791,18.4485500020271,7.39469495385889,7.51570016065321,7.45842075605342,17.1524273408579,13.4023048140745,9.76700388960785,11.6345120151091,11.3098039199715,10.0752040042021,7.66354026615147,7.65169513695184,9.90308998699194,9.82102305270683,9.80966830182971,13.6179829574251,9.97469413473523,7.97061622231479,10,8.04143611677803,7.98716277529483,7.97061622231479,10.4749551929632,7.94692155651658,8.21538270736712,8.21538270736712,12.5590909179348,8.25884840114822,8.21609642072726,8.31966448658544,14.3124710387854,8.5003129173816,8.35359627377693,14.7055337738384,8.45593195564972,11.0680338852718,8.57839607313017,8.57839607313017,8.68193666503724,8.64781748188864,8.62708799702989,8.74957999769111,11.0899094544059,8.80687540164554,8.79588001734407,8.87614835903291,9.16304326294045,11.896196279044,11.6635402661515,14.0814454694497,9.2580609222708,9.28232949699774,9.31336373073771,9.53610701101409,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,9.55595520408192,10.1402614338029,10.0250280057019,10.6458915608526,15.8601209135988,13.4522252946122,15.7931741239682,12,13.9244530386075,10.9393021596464,10.7986028756795,12.9281179926939,10.2549252084179,10.1197582241045,10.071092309756,9.97469413473523,9.73518217699046,9.12609840213554,9.05109823902979,9.07987667370928,8.55752023093555,8.66554624884907,8.74232142513082,15.3914739664228,10.8124792791635,8.38615817812393,8.31336373073771,12.7258421507363,8.08039897621589,8.08039897621589,7.78251605578609,7.73754891026957,7.69680394257951,7.60554831917378,11.0385789059336,7.19382002601611,9.51286162452281,10.8068754016455,7.03245202378114,7.24641694110709,7.15864052954514,8.79860287567955,6.98296666070122,6.87942606879415,6.79860287567955,6.58670023591875,8.06298389253519,8.50445566245355,8.66958622665081,8.55752023093555,6.76955107862173,13.8927900303521,6.63827216398241,7.85078088734462,11.1260984021355,11.5951662833801,6.14630178822383,9.65169513695184,6.05453141486818,6.0762380391713,6.02045162529591,5.81247927916354,10.698970004336,5.79860287567955,5.81815641205523,5.81815641205523,5.80410034759077,7.11125903931711,5.71444269099223,5.64397414280688,5.59006687666871,10.1958605676646,5.6458915608526,6.79588001734407,5.50307035192678,11.345823458122,10.829738284605,9.71444269099223,7.44009337496389,10.1555228242543,6.30980391997149,7.3840499483436,7.33254704711005,6.39469495385889,16.2048154103176,7.09799710864927,7.92081875395237,6.76700388960785,7.11238269966426,5.68613277963085,9.0423927129399,5.58838029403677,11.5867002359187,7.09258863922541,7.74957999769111,6.51004152057517,4.08039897621589,6.07935499859321,9.75696195131371,7.81247927916354,7.07987667370928,5.42021640338319,6.93930215964639,5.99139982823808,4.7619538968712,4.03198428600636,16.4089353929735,4.71219827006977,14.8569851997459,4.95078197732982,12.9546770212133,7.79860287567955,8.12262865413023,5.82973828460504,6.44249279809434,12.1924649719311,7.06550154875643,10.7011469235903,9.80966830182971,7.38615817812393,6.83863199776502,5.90308998699194,4.55595520408192,4.41566877563247,9.36957212497498,4.70774392864352,7.82681373158773,5.44369749923271,9.70774392864352,7.69250396208679,4.65560772631489,6.71669877129645,5.40011692792631,8.92811799269387,7.11182050608168,4.56703070912559,10.5185573714977,3.55909091793478,6.11975822410452,3.39361863488939,6.21324857785444,2.7851561519523,8.00392634551472,7.39902710431325,3.6903698325741,4.46852108295774,4.46852108295774,4.17134010346468,2.80687540164554,3.08618614761628,1.73048705578208,2.81815641205523,2.12551818230053,2.39577394691553,2.47237009912866,1.75696195131371,1.61083391563547,2.65364702554936,3.82973828460504,4.30189945437661,2.69464863055338,2.3840499483436,2.30980391997149,3.01999662841625,2.57024771999759,3.37263414340727,8.44490555142168,2.59176003468815,3.20901152491118,5.8153085691824,3.87289520163519,3.87942606879415,3.34872198600186,3.03105031901866,3.7281583934635,5.21968268785985,4.11747546204512,3.12551818230053,5.83268266525182,3.91364016932525,3.91364016932525,3.7851561519523,4.72124639904717,4.12147820449879,5.23136189875239,4.33818731446274,2.26440110030182,2.65955588515988,5.21609642072726,6.30189945437661,3.91364016932525,7.67366413907125,7.67985371388895,6.1232050237993,4.04769199033788,3.96657624451305,9.19246497193115,4.60032627851896,5.82390874094432,4.32790214206428,5.6252516539899,7.88272870434424,9.27984069659404,5.87942606879415,8.6345120151091,8.64397414280688,6.03857890593355,4.61798295742513,7.64016451766011,3.07314329105031,5.34198860334289,3.54668165995296,3.28483264215154,9.62160209905186,8.79048498545737,5.04191415147891,6.82973828460504,4.52870828894106,4.84466396253494,8.5016894462104,6.16941133131486,3.78251605578609,4.46218090492673,6.83863199776502,4.41005039867429,8.82973828460504,7.13667713987954,5.3269790928711,5.15242734085789,4.31069114087638,8.97061622231479,4.82681373158773,5.65364702554936,7.23807216157947,4.22621355501881,3.32605800136591,8.29328221766324,5.25414480482627,6.62708799702989,6.47237009912866,6.80410034759077,6.80410034759077,5.31336373073771,4.55595520408192,4.55595520408192,6.04527520902094,12.7619538968712,10.2487208960167,5.14146280243036,4.69897000433602,4.09908693226233,7.07883394936226,6.26921772433361,14.2097148359668,10.1985962899826,5.98296666070122,3.20134935455473,5.36151074304536,8.95467702121334,4.58335949266172,7.94309514866353,7.61439372640169,3.99139982823808,6.45842075605342,11.2254830342715,5.55909091793478,3.95860731484178,5.47755576649368,5.58169870868025,3.55752023093555,10.1426675035687,4.30364361126667,3.64206515299955,4.81247927916354,12.9430951486635,5.6458915608526,4.21896306137887,4.87942606879415,8.42596873227228,7.08884239126002,6.89619627904404,12.7594507517174,9.03715731879876,8.85698519974591,10.6819366650372,4.97881070093006,5.93930215964639,6.68402965454308,5.61083391563547,7.7619538968712,5.53760200210104,13.9586073148418,9.99567862621736,6.44369749923271,5.73754891026957,7.82973828460504,4.68402965454308,12.0867160982396,9.75696195131371,6.65560772631489,4.73048705578208,6.80410034759077,6.60554831917378,7.90308998699194,6.02227639471115,9.33535802444387,9.86012091359876,5.76700388960785,10.3062730510764,9.05109823902979,8.39254497678533,6.29756946355447,5.42829116819131,13.3746875490383,5.87614835903291,5.77728352885242,10.5867002359187,12.6161846340196,8.44855000202712,6.1232050237993,5.34486156518862,5.22040350874218,4.90657831483777,6.01547268665621,9.42596873227228,6.38827669199266,5.80687540164554,5.41116827440579,7.22914798835786,14.593459819566,8.79588001734407,5.72584215073632,3.84466396253494,12.4749551929632,6.16430942850757,9.22694530663574,12.9829666607012,7.64975198166584,5.61798295742513,9.24336389175415,6.63638802010786,5.59006687666871,8.58004425151024,8.58004425151024,6.58335949266172,10.3746875490383,7.47886191629596,12.7166987712964,15.2013493545547,7.75696195131371,6.45593195564972,11.5072396109732,10.1924649719311,10.869666231505,9.87289520163519,8.66154350639539,8.32790214206428,7.22257317761069,5.51427857351842,7.70553377383841,13.2588484011482,8.32605800136591,8.0301183562535,6.22329881601159,5.47755576649368,4.84466396253494,10.1739251972992,7.40671393297954,6.64975198166584,4.88605664769316,12.7695510786217,10.3062730510764,8.89279003035213,7.53910215724345,11.3545777306509,9.91721462968355,5.63264407897398,3.82102305270683,9.44249279809434,4.71896663275227,3.75202673363819,8.01412464269161,3.85698519974591,4.92445303860747,6.50723961097316,5.82102305270683,6.01055018233331,6.24565166428898,6.02410886359821,5.80687540164554,4.96257350205938,7.06752623532285,7.03715731879876,6.75202673363819,9.00656376950239,4.94692155651658,4.87289520163519,9.36151074304536,14.3115801779973,8.1001794975729,4.17005330405836,7.39902710431325,6.09854167860389,5.1791420105603,5.16941133131486,9.82390874094432,6.1157712303674,11.7904849854574,8.92811799269387,6.67778070526608,6.50723961097316,5.26360349772336,8.53760200210104,5.19722627470802,4.16877030613294,9.25570701687732,5.5185573714977,5.37468754903833,7.77989191195995,4.34775365899668,9.60906489289662,9.60906489289662,4.35951856302958,4.23957751657679,7.49757288001557,5.36051351073141,5.65560772631489,4.47886191629596,9.45469288353418,23.9956786262174,7.7619538968712,4.44733178388781,17.334419008982,4.58169870868025,7.05256627811295,5.89962945488244,11.7931741239682,10.0947439512515,9.36251027048749,4.73992861201492,4.71896663275227,10.4571745730408,7.4672456210075,4.7281583934635,4.63827216398241,8.66554624884907,5.94692155651658,5.94692155651658,10.8538719643218,7.03857890593355,6.9100948885606,4.58335949266172,10.1040252676409,9.48945498979339,7.20760831050175,6.25336580106242,6.11350927482752,17.4436974992327,7.33913452199613,7.1605219526258,6.08039897621589,4.89279003035213,8.33441900898205,12.8477116556169,11.5900668766687,7.27002571430044,6.13430394008393,4.94692155651658,4.83863199776502,8.47755576649368,6.34775365899668,6.10292299679058,11.5512936800949,5.0301183562535,16.0793549985932,10.3675427078153,10.2358238676097,10.2358238676097,7.93554201077308,5.04431224968649,21.8416375079047,16.7166987712964,10.0599818449923,8.60032627851896,6.12033079436795,5.06499684854635,4.98716277529483,8.67162039656126,5.03385826726097,6.46470587995723,8.79048498545737,7.57675412606319,5.22257317761069,5.21112488422458,5.21112488422458,5.02826040911222,7.42481215507234,5.25026368443094,5.05551732784983,5.29073003902417,8.39147396642281,8.36855623098683,5.31247103878537,16.7258421507363,7.60205999132796,6.59345981956604,5.32975414692588,5.32422165832592,16.4534573365219,13.5883802940368,5.42829116819131,5.32057210338788,7.97469413473523,6.78251605578609,5.39685562737982,5.32975414692588,6.94692155651658,5.21538270736712,6.84163750790475,5.53313237964589,6.82973828460504,5.53313237964589,5.30627305107635,9.86012091359876,7.04095860767891,5.51286162452281,10.9281179926939,8.52143350440616,5.48017200622428,5.48017200622428,25.9136401693253,10.1605219526258,5.6675615400844,5.66354026615147,5.6252516539899,5.64206515299955,8.56066730616974,14.7931741239682,5.73754891026957,7.08830984124614,5.69897000433602,9.20273245916928,11.7375489102696,5.84771165561694,7.37365963262496,7.27572413039921,5.88272870434424,7.35654732351381,7.35556141053216,7.33161408331,8.88272870434424,7.39902710431325,5.96257350205938,14.6736641390712,5.92081875395237,8.70996538863748,12.3429441471429,9.22112552799726,6.10127481841051,6.10127481841051,6.07468790850035,9.23507701535011,6.13076828026902,9.12033079436795,7.61083391563547,6.14752000636314,6.11350927482752,6.10790539730952,6.09854167860389,11.0209070993617,6.06499684854635,8.6252516539899,7.7851561519523,6.25570701687732,6.18375870000822,11.0675262353228,7.75696195131371,6.35457773065091,6.16749108729376,6.40671393297954,6.36754270781528,6.30102999566398,6.37059040089728,6.37059040089728,6.41680122603138,8.34872198600186,8.43889861635094,6.5185573714977,6.50863830616573,6.43297363384094,12.829738284605,16.5718652059712,14.5142785735184,6.56863623584101,6.58838029403677,6.40120949323688,8.22841251911874,8.41005039867429,12.2240256688706,8.80410034759077,8.47755576649368,6.82102305270683,6.7594507517174,6.72353819582676,6.87942606879415,8.54363396687096,6.87942606879415,6.80410034759077,14.5451551399915,6.93554201077308,6.91364016932525,6.91721462968355,12.6497519816658,6.85387196432176,9.15926676538819,10.5346171485516,8.86327943284359,8.78251605578609,7.07262963696098,7.11125903931711,9.28316227670047,12.8761483590329,18.1045774539606,7.27245874297144,7.26760624017703,7.09963287134353,9.21824462534753,11.2549252084179,7.33161408331,7.41680122603138,7.3429441471429,7.50445566245355,7.48811663902113,7.55284196865778,7.44490555142168,7.53910215724345,7.45593195564972,7.65364702554936,19.6716203965613,11.3098039199715,7.72124639904717,7.51999305704285,9.82102305270683,7.70114692359029,7.81247927916354,7.91721462968355,15.4571745730408,11.9507819773298,14.4365189146056,8.11918640771921,16.308918507877,8.04624030826677,10.3115801779973,8.25884840114822,10.7619538968712,29.2013493545547,14.3124710387854,8.45717457304082,8.55129368009492,8.42481215507234,10.7471469690201,8.63638802010786,8.75202673363819,8.85698519974591,8.83863199776503,8.97881070093006,9.00436480540245,8.91721462968355,9.41680122603138,9.41907502432438,9.41907502432438,12.2433638917542,9.59176003468815,9.65560772631489,9.82102305270683,9.74714696902011,9.80410034759077,9.80134291304558,12.4400933749639,12.4881166390211,10.0264103765727,12.6615435063954,10.0154726866562,10.8096683018297,11.156767221902,11.1414628024304,11.0649968485463,11.6289321377283,13.012780770092],"text":["TG","GALNT15","FRMD4A","C10orf85","ENSA","PARP6","PRKAB2","AMPH","AOAH","PI4KB","ADAM12","C15orf26","TNIK","HIPK2","KIAA1199","WFDC3","DGKI","CLIC5","TMC2","IL1R1","MB21D2","EFTUD1","RCL1","ZNF474","PDCD1LG2","AGBL1","METTL13","KHDRBS3","LEPREL1","CYP7B1","SLC39A12","DDX47","APOLD1","TES","NSMAF","NREP","SEMA6D","CPQ","ST8SIA6","WDR72","AP3S2","C15orf38-AP3S2","PHF8","LYSMD4","PAPSS2","ZBTB38","C1orf110","SERINC3","NTRK3","FBN1","CD58","PLCXD2","PHLDB2","DMRT3","ADPGK","C10orf107","PHC2","PALM2","S100A10","CUBN","PRUNE","ATP13A5","COMMD7","DLEU1","RUNX1","ADAMTSL3","EFEMP1","CD47","SPTSSB","PALM2-AKAP2","AKAP2","MTMR3","TPRG1","VTCN1","EFR3A","PRUNE2","IPMK","HIBADH","CLDN11","PKIG","SSPN","LPP","UBE2H","CAV1","SLC4A1AP","PEAK1","RUNX2","MMP20","OTUD7B","HUS1","C20orf187","CNIH3","WHAMM","NOL10","PPM1L","GPC6","RAB27A","MMP7","BACH1","PDE8A","PLEKHA5","PACS1","RPRD1B","GDAP2","CYP24A1","MCTP2","RIN2","CD101","LRRC28","DIEXF","ASAP1","SVIL","TJP1","PPFIBP1","DIRC1","ME3","RALY","TSC22D2","MYC","MS4A4E","FAM180A","FAM49B","CHD6","PAK2","KCNAB1","TMOD3","P2RY14","MED12L","IFI16","ITGB5","SORBS2","BCAT1","SRGAP2","MME","CADM1","ANXA2","PTGFRN","CFLAR","LIPA","ILDR2","DEC1","FAM214A","CP","BNC2","DCBLD2","ZNF639","NPSR1","RECK","NF2","YAP1","PRCC","CRTC3","GUCY1A2","IQGAP1","GLIS3","FILIP1L","CMSS1","AKAP13","RNF115","AL590452.1","RGS20","NYX","MSTO1","POLR3GL","IGF1R","PIGV","TOX","DUSP10","ABL1","HIST1H1C","SIK3","SCMH1","DUX4L2","CDK6","CPNE4","CLCN3","SPAG17","AKNAD1","TM4SF20","BRD2","STOM","RTN3","WWTR1","TSHZ2","ISPD","ADAMTS6","SPRY1","MAML2","TNN","THADA","DIAPH1","STIM1","RUSC2","HIST1H3E","GPR110","SH3GL2","PKN2","ASAP2","WDR17","NUDT2","CALM1","CDHR3","METTL7A","C6orf100","VTI1A","SCRN1","ACSL3","POMT2","SATB2","WRNIP1","MYCBP2","EXT1","CACNA2D3","ZBTB2","TCP11L2","HMMR","PARK2","IGF2BP3","FBXO42","CCDC103","FAM187A","ZNRF2","IFI44L","SREBF2","TRAK1","RP11-458D21.5","TEAD1","MACF1","COX20","SAP30BP","TYRO3","TRAM2","MCCC2","RP11-159D12.5","LIMD1","ASF1A","MCM9","COA1","STK17A","VEPH1","ANLN","TENM2","SPTBN1","ARRDC3","TLE1","ATP5G2","ABLIM2","TBC1D22B","TRAK2","STRADB","SLC28A3","CAP2","GAS8","ZNF521","ZC3H7A","MED8","ETV5","THNSL1","ARID4B","USH2A","UNC5CL","CRTAP","MEIS2","CDC27","RAP1GAP","SNX9","SUMF2","DCLK1","IL33","RRAS2","ARNTL","BCAR3","DBT","CSRP1","FNBP1","B3GALTL","SNAP23","ADORA3","TTC28","GRB10","SPECC1","DCP2","FAM134B","DFNA5","RHOJ","AL162389.1","ETF1","SYBU","MTHFD2L","UGP2","STMND1","SLC10A7","LDLRAD3","TCF7L2","ENC1","CHN1","PRPF40B","SYT11","SACS","TRIM66","JAKMIP2","FAM212B","SEPT7","SH3PXD2B","ARHGAP22","APCDD1","CHMP4B","SPRED2","ATP2B4","FAM111A","MTHFD1L","LRRC23","CLIP2","FGGY","PRELP","TMEM178A","LMO2","SEC14L1","GOLM1","OPTC","SDK2","ACTR6","PRKCE","TCEB3","ID3","MRPL48","USP36","DIRC3","HDAC9","MARCH6","TMEM108","RP11-691N7.6","TMX2-CTNND1","SSH1","TSNAX-DISC1","GDF11","SOX9","KCNQ1","RNF216","AKAP1","SPAG9","WASF2","RP11-770J1.4","PRICKLE2","RPN2","ARHGAP35","CELF1","ANKRD40","BEND3","EIF4G3","NOTCH2","PACRG","ABTB2","RAD51B","RALGAPA2","TARBP1","ABCD2","AC022431.2","CEP164","FAM64A","NIM1K","SOGA1","NCMAP","EGFR","KCNJ10","POU3F2","LIMA1","MAP2K6","MICALCL","YPEL2","IQCJ-SCHIP1","SCHIP1","IFT74","TSPAN5","CDC14B","SEC31A","GAS1","BCHE","MOXD1","NEGR1","METAP1D","ALG8","PLXNA2","POLR1A","SYCE2","GCDH","SORT1","TRAF3IP2","NCALD","ALKBH3","PLCB4","SPATA13","RP11-307N16.6","CARS","CSGALNACT1","AUTS2","TTC7B","DRAXIN","PPM1D","DIP2B","ABL2","DKK3","BICD1","CUX1","FARP1","CPVL","SLC38A9","RAD23A","SAMD13","CCDC80","AGAP1","C8orf76","ZHX1-C8ORF76","SOBP","FIBIN","ARHGAP31","BANF2","LRRFIP1","GPD1L","SLC44A3","PTPRJ","FNBP1L","SCARB2","SRGAP1","FOXJ3","CHD7","C20orf26","ITPKB","TMCC1","KLF9","PALLD","DDX60L","GMPR","GFAP","IGFBP2","UBE4B","CSRP2","ACER3","MAP1B","PSMA1","MFSD6","MAP6","ZNF703","CALN1","EXTL3","COL28A1","PNMAL1","FAM117A","GABRB1","LRP4","C5","NAV3","DPYSL5","KIRREL3","NFIB","ADAMTS3","SNX31","VOPP1","TRIO","KIF26B","OOSP1","C12orf65","GLI3","SHQ1","ZZZ3","ATXN7L1","CCDC41","HP1BP3","CDKAL1","BBOX1","POU2F1","ARHGAP29","TRIT1","CUEDC1","CHIT1","FOXK2","TRIM9","NEDD9","C5orf64","DISC1","ANKRD50","SLC15A2","AC017081.1","LIMCH1","PBX3","ZCCHC3","FAT1","TCF4","UBALD2","SPARCL1","MICAL2","CACHD1","CPNE5","COL4A1","PRKCA","LITAF","FAM107B","SLC6A11","ZMYND8","SWAP70","FOXN3","MYO18B","FHAD1","DST","TRIM2","NAV1","MYO5B","KIAA0355","CTNNBL1","CBFA2T2","RNF182","FYN","CLASP2","FOXO3","TOP1","FMO6P","UBIAD1","ZBTB16","STX6","USP10","KIAA0226L","NDUFAF5","TGFBR2","YIPF1","ATXN1","SLC16A4","BOC","MAGI2","ZDHHC13","FARS2","PSD3","C1orf143","DOCK4","NUFIP1","EMC8","NFIX","CCDC15","FOXO6","RGS12","ADCY8","CPSF4L","NOTCH2NL","BMPER","PC","GAB2","SCML4","APBA1","SP2","ZFAT","BPTF","SPATA5","EPHA7","CKB","RP11-210M15.2","CNTN5","SPTAN1","UBXN2B","MATN2","SLC1A2","NRG1","MKL1","CDKL1","ADAMTSL1","AMBRA1","LRP1B","RGL1","AVL9","SMNDC1","ADRB1","BTG2","SPATA6","AFF1","ZNF804A","CDKN1A","CDH4","ANKRD44","CACNG5","C12orf75","AP000708.1","SPSB4","LARP1B","C8orf56","HEXB","DARS","ZBTB20","KCNJ6","WNT5B","PLCH1","MERTK","LRRN2","DENND1A","APC","PTPRZ1","ZSWIM6","PBX1","DPYD","ZNF321P","ZNF816","NGDN","PAMR1","PDZRN3","SMG7","ADORA1","PCDH7","TMC1","LRRN3","IMMP2L","RP11-1084J3.4","C1QTNF3","KCNE4","PXDNL","THSD7A","FIP1L1","LNX1","ADAM22","MGAT5","SARS","SLX4IP","C1orf61","RASA3","CACNA1E","SH3RF3","DOCK9","ENOX1","EBAG9","SLC17A8","MRPS18B","ALK","PTCHD2","SND1","SEMA5A","PPFIBP2","SLC1A3","GPM6A","KIF1A","CTD-2535L24.2","AXIN2","JAZF1","CYTH1","KAZN","FHL2","TRAF3","ASS1","UBASH3B","IL1R2","SORL1","MYO10","ZNRF3","ITGA2","CATSPERB","COLGALT2","FAM65B","ABAT","ELAVL4","MKRN3","ATP13A4","COG6","RAB3B","CELF2","GRID2","ID4","NKAIN3","MAST2","DPP4","GJC1","IQSEC3","KIAA1598","TULP4","LAMB3","CLHC1","NTSR2","RP11-47I22.3","RP11-47I22.4","RFX2","TMEM181","ELP4","WDR25","FAM182B","SHROOM3","ETV1","STARD13","KCTD6","NPTX2","SMARCC1","GRIA2","FOXP1","TCF7L1","LARS2","AC062017.1","SRGAP3","TAOK3","C11orf49","FAM184B","SNTB1","SEPT9","SERINC5","PRCP","LAMTOR1","LRTOMT","VRK1","ABRA","AC016757.3","A2ML1","EFCAB4B","CDC73","B3GALT2","ADD2","FGFR3","ZNF277","RNF157","NTRK2","ARHGEF28","SNX11","GPR156","SP8","EFNB2","GAS2","VWC2L","NDUFA10","ALCAM","C9orf3","FAM181A","RP11-286N22.8","SLITRK3","GNG2","ROBO1","TIAM2","DPYSL2","EDNRB","PTCH1","IGF2BP1","GPNMB","NUAK2","AGBL4","SIAE","JMJD1C","FPGT-TNNI3K","TNNI3K","LRRC53","XPR1","TANC2","OPCML","KDM3B","UBE2U","ICOS","SESN3","LRRC3B","OLFML1","PTPRS","PTCHD4","RGS6","ZNF880","CHD9","CHST11","WIPF1","KIAA1614","HERC3","PGBD5","FAM110B","ZNF112","CTC-512J12.6","LMAN2L","TMEM51","ICA1","SERPINI1","POLN","ZSCAN5A","AC006116.20","TOMM7","SASH1","CDK14","ST6GAL2","DNAH11","ANKRD6","TMEM161B","GPR126","NPAS3","C12orf79","SNTG1","LPAR1","TLE4","SLC15A3","RP11-144F15.1","RPS6KC1","ECE1","C1orf198","KIF21A","CSMD1","LEMD1","NR1D2","RBM24","UTY","ULK4","C11orf44","TCP10","TTLL4","MAPKAP1","MRPS27","RFX4","CPM","BAI2","ANKRD17","MRAP2","SCEL","AK4","STON2","NHSL1","PCDH9","SLC39A11","ASCL1","NCAM1","AKAP6","LPHN2","PTPRD","C14orf64","MGAT5B","TBC1D16","CTHRC1","SCD5","GRIK5","IQGAP2","KIF5C","LRP2BP","NIN","PREX2","INTS9","UTRN","YTHDC2","PPP2R2B","RDX","PDE4B","ZFHX4","GPR98","ANGPT1","WIPF3","TNR","TAMM41","NOS1AP","WWC1","BAALC","AGO4","LYZL1","C2orf80","KCNJ16","NCKAP5","AFAP1","RFTN2","GBAS","C16orf62","FXYD6-FXYD2","FXYD6","LRRC16A","SKAP2","NSF","PRKD1","RSPH3","NMT1","VPS13D","ESRRG","LPHN3","CSNK1A1L","SSBP3","MAPT","SHANK2","MEOX2","SOX6","NFIA","MAP2","PLCB1","PCDH18","IQCE","ABCA5","CDH1","LHFP","ISM1","DGKB","C14orf166B","KAT2B","LACE1","MAPK10","ENTPD2","LRRC8D","RP11-302M6.4","SH3RF1","ACTN2","RAB3GAP2","MOB1B","LIX1","CTD-2215E18.1","ARHGEF10L","PPM1H","ARPP21","RNF217","C8orf4","TMTC2","CCDC13","MARCH4","ARHGEF7","DSEL","SDCCAG8","MAGI1","FGFR1","ANKFN1","FHIT","VIT","PDE4D","CAPN9","FOXP2","PSRC1","DLG2","KCND3","SEPT11","GRIK3","FAT3","LHFPL3","ETS1","RAPGEF4","MFHAS1","HOPX","CHST9","DCLK2","PCNXL2","PTPRO","IGSF11","RPTOR","MSI2","RBMS3","SLC22A23","ODF1","NRCAM","RP11-65D24.2","NR1H4","FBXL17","ACSS1","CCND2","SDK1","ATG7","TMPRSS5","KIAA1549L","ARHGAP15","GATM","C1orf87","MOB3B","KAT7","AKAP7","NRXN1","NTM","FBXL7","PSEN1","FREM2","NXPH1","BCL2L14","FAM181B","C3orf55","PNMA2","HEPN1","DCDC2","ZNF184","ABCA8","FREM1","PRTFDC1","OLIG3","EBF2","DAB1","LRRC4C","PRRX1","BMP7","RPH3A","SLC9A1","GCNT2","OTOS","C1orf21","ADCYAP1R1","KCNA10","MPPED2","GRIA1","STX18","NAV2","DOK5","SLC35F1","HAPLN1","SDC3","DOCK10","ST3GAL4","ELMO1","CTNND2","SLC24A3","MSRA","AGMO","LINGO2","ROBO2","FGF2","LRRN1","SUMF1","JARID2","GAB1","GLI2","GPR75-ASB3","USP2","SMOC2","FTCDNL1","ANK2","TMSB4Y","PHF21A","RP11-463C8.4","TMEM63C","SLC4A4","EIF1AY","LRIG1","BTBD3","TMEM100","KCNA2","FABP7","PLEKHA2","THRB","PTPRK","CSPG5","PPAP2B","KLHDC8A","BOD1","DSCAML1","ATP9B","PCDHGB2","PCDHGA6","PCDHGB7","PCDHGA9","PCDHGB1","PCDHGA7","PCDHGC3","PCDHGB6","PCDHGA4","PCDHGA2","PCDHGA3","PCDHGA12","PCDHGC5","PCDHGA11","PCDHGA10","PCDHGA1","PCDHGA5","PCDHGA8","PCDHGC4","PCDHGB3","PCDHGB4","XKR6","XKR4","ARHGAP10","MIR1208","RP11-1069G10.2","RPL29P19","RP13-631K18.3","CCDC26","MIR4419B","RP11-9L18.2","RP11-296O14.3","RP11-770E5.1","RP11-296O14.1","RP13-653N12.2","RNU1-70P","RP5-860P4.2","AC083843.1","RP11-600K15.1","AF064858.6","AL079339.1","RNU6-413P","RP4-666F24.3","AP001607.1","RP11-351M8.2","RP11-572P18.2","RP11-143P4.2","RP11-479J7.1","RP11-283G6.4","RP11-283G6.5","CTC-441N14.4","CTD-2576F9.1","AP000797.4","LINC00649","RP11-358M11.4","AC079613.1","RP11-244F12.1","LEPREL1-AS1","EXTL2P1","RP11-550P17.5","RP11-224P11.1","hsa-mir-490","LINC01033","RP11-542A14.1","RP11-10O17.3","KRT18P3","SLC25A38P1","NREP-AS1","AC009518.3","RPL7AL2","RP11-191N8.2","AF124730.4","RP11-359H3.1","CTD-2005H7.2","RP11-438D8.2","RN7SKP148","RP11-680E19.2","LINC00507","RP11-115J16.1","PSMA2P2","RP11-431K24.1","RP11-230G5.2","AC005022.1","RP11-194G10.3","AC074391.1","AC012370.3","RP11-90B22.1","RP11-464C19.3","AC108105.1","RP11-284G10.1","RP11-127O4.3","RNU1-35P","CTD-2021J15.1","MTND1P24","GS1-410F4.4","TM4SF1-AS1","U8","LMCD1-AS1","SNORA63","TDGF1P2","CASC17","CDKN2B-AS1","RN7SKP206","RP11-33A14.1","PVT1","KRT18P13","PPP1R10P1","AC090945.1","AC002480.5","RP11-289F5.1","RP11-246K15.1","RP11-307O10.1","DLG1-AS1","AC006159.5","RP5-1069C8.2","AJ006998.2","RN7SKP93","LINC00113","RP13-631K18.2","RP11-1018N14.5","LINC00189","RP11-889D3.2","RP11-89M16.1","RP13-526J3.1","ANKRD18DP","RP11-120B7.1","RP1-15D23.2","AC005029.1","GBAP1","NPHP3-AS1","KB-1568E2.1","CYP1B1-AS1","RN7SKP191","RP11-317J19.1","RP11-25E2.1","RNU6-710P","RP5-1125A11.4","RP11-523L1.2","RP11-145A3.1","AC144449.1","RP11-359H3.4","AC013448.1","RP11-301L8.2","RP11-408N14.1","RNU6-369P","RP11-338H14.1","RP11-149I23.3","RP11-88H10.2","RP4-782G3.1","CFLAR-AS1","AC005019.3","RP3-388N13.3","NPSR1-AS1","RP5-1033K19.2","RP11-798K3.2","AC007091.1","RP11-3P17.4","CALR4P","AC006482.1","TEX41","RN7SKP226","RP11-103J8.1","AC003090.1","RN7SL44P","RP11-148B18.4","RP11-29H23.5","MSTO2P","RP11-570H19.2","AC006076.1","LINC00578","AC098617.1","SLC31A1P1","AC079248.1","CTC-497E21.5","LINC00382","RP11-317M11.1","AC017080.1","AC096559.1","CTD-2282P23.1","MALAT1","LEMD1-AS1","RP5-899E9.1","RNU6-793P","RP11-1007G5.2","RP11-732M18.3","CTD-2337A12.1","RP11-309G3.3","AC006227.1","RP11-385M4.1","snoU13","RP11-6L6.7","RP11-329A14.2","RP11-478K15.6","TSEN15P1","AC068535.2","TOB1-AS1","RP11-159D12.2","RPL23AP46","RP11-528I4.2","RP11-1080G15.1","RP11-1080G15.2","AC003092.2","RP5-1011O1.2","MPPE1P1","LINC00431","AC004520.1","RP11-630C16.1","AC106732.1","AC091729.9","RP11-443B7.3","RNY1P5","RP1-155D22.2","RP1-28H20.3","RPS11P1","RP11-731C17.1","FENDRR","RP11-6N17.10","PGAM1P1","AC004448.5","RP11-646I6.5","EXTL3-AS1","RP11-278H7.1","ANKRD26P1","RP11-279F6.1","TRAF3IP2-AS1","RPL7P8","SRGAP2B","RP11-692D12.1","FTLP3","RP11-417B4.2","RP11-361D14.2","SNORA40","TDPX2","RP11-191L17.1","SUMO1P1","RP11-612J15.3","LINC01059","AC004012.1","RP11-166D19.1","AC005740.4","CTD-2187J20.1","RP11-175P13.3","AC097724.3","LINC00533","LINC00478","AC073626.2","RP11-388C12.1","KCNQ1OT1","RP11-1082L8.4","NDUFS5P2","RP11-481C4.1","RP11-197K3.1","RP11-705O24.1","AC012457.2","AC007358.1","RP11-93B21.1","RP11-7F17.5","AC009302.4","RP11-1008C21.2","RNA5SP144","RP11-167H9.4","RP11-167H9.5","RP11-536C5.2","RP4-723E3.1","RP5-1022J11.2","RP4-734C18.1","RP3-405J10.3","HMGN2P19","CTB-178M22.1","RP1-249H1.2","RP11-469N6.3","B4GALT4-AS1","RP11-337A23.3","LARS2-AS1","LINC01066","ELMO1-AS1","RP11-639B1.1","AP000770.1","RP11-130C6.1","AC010148.1","RP11-661G16.2","RP11-437L7.1","RP11-232A1.2","RP11-589C21.6","RP11-533E19.5","ACA59","RNU6-1246P","RP11-572M11.4","PCDH9-AS2","RPL5P20","RP11-196H14.2","NAP1L1P1","SMIM2-AS1","AC013410.1","ITPKB-IT1","CTD-2530H12.7","RP11-155G15.2","Y_RNA","RN7SL607P","KIRREL3-AS1","RP11-120J1.1","RP11-282O18.3","AC022182.3","AC093609.1","RP5-1061H20.5","RP3-510L9.1","AC114752.1","AC118653.2","CECR3","RP11-64D24.2","RP11-489E7.1","AC009313.2","RP11-81H14.2","RP11-644C3.1","RP11-159H3.1","IGFBP7-AS1","AL357519.1","CTA-125H2.1","RP11-472M19.2","IPO9-AS1","RP11-444D3.1","RP11-629G13.1","AC020606.1","RP1-1J6.2","PPP1R2P4","LAMTOR5-AS1","AC074011.2","LINC01105","RP11-698N11.2","RP11-82L7.4","RP11-511B23.2","RP11-222K16.1","RP11-625L16.1","RNU6-1096P","LOC124685","RP1-153P14.8","CTA-390C10.10","CASC18","RP11-167N24.5","AC003665.1","RP11-38P22.2","RP11-122C5.1","HNRNPA1P58","RP11-286N22.10","LIFR-AS1","AC078882.1","AC008753.6","SOX2-OT","RP5-1065J22.8","AC083864.3","DPY19L1P1","AC104088.1","RP11-531H8.1","RP11-75C10.9","RP11-698N11.4","RP11-10L7.1","RP11-84A19.2","ZBTB20-AS1","RP1-155D22.1","CTB-57H20.1","DPYD-AS1","CTD-2620I22.1","AP001258.4","AP001258.5","AC005013.5","RP5-1092L12.2","RP11-335O13.8","CTD-2290P7.1","RP1-193H18.3","AP000797.3","RP11-666A8.8","RNU6-1051P","RN7SL178P","RP11-86H7.6","RP4-737E23.2","RP11-308N19.1","SOX9-AS1","RP11-179A10.1","RP1-111D6.3","RP11-102J14.1","RP11-344L21.1","RNU6-1114P","PLK1S1","CASC15","AC004538.3","RN7SL542P","RP5-827O9.1","RP11-472N13.3","ZNF123P","AC069154.4","RP11-503E24.3","RN7SKP2","RP5-1180C18.1","RP11-118E18.4","AC008694.2","RP11-527N22.2","ZNRF3-IT1","RP11-541G9.1","RPL7L1P8","RP11-51G5.1","RP11-991C1.1","RP11-73C9.1","RP5-936J12.1","RP11-476O21.1","NAV2-AS4","RP11-187O7.3","RP11-20B7.1","RNY3P9","SNORA64","RP11-118E18.2","RP11-718B12.2","SNORA73","GRIFIN","RP11-160O5.1","RP11-452H21.1","Z83001.1","CTB-47B8.1","RP11-699L21.1","FAM58BP","RP11-717D12.1","FOXP1-AS1","GAPDHP53","BMPR1APS2","AL442639.1","RP11-718L23.1","RP11-65M17.3","RP5-945F2.2","RP11-536O18.1","AC079586.1","RNU7-165P","RP11-159K7.2","RP11-90K6.1","CTD-2245E15.3","RP11-613M5.2","NPM1P4","RP11-848P1.9","RP11-413N13.1","RP11-404J23.1","A2ML1-AS1","RNU7-62P","AC004158.3","AC004158.2","RP11-198M11.2","RP11-56I23.1","RNU6-256P","AC003051.1","LINC01135","RP11-303G3.6","RP11-134G8.7","RP1-40E16.9","AC107218.3","FAM181A-AS1","RN7SL366P","RP11-506B6.6","RP11-222N13.1","HMGN1P17","AC087269.2","RP11-408A13.2","AC037445.1","RP11-41O4.2","RP4-668G5.1","RP11-344F13.1","RP11-315F22.1","LINC00404","RP11-141M1.3","RP11-820L6.1","CTC-297N7.9","CTC-297N7.5","RP11-712B9.2","RNU6-753P","RNF5P1","CTD-2516F10.2","RP11-486M23.1","LINC00882","RP11-1078H9.6","AL162419.1","OACYLP","LINC01137","RP11-6F6.1","SPATA42","AC018890.6","AL391538.1","RP11-201M22.1","RP11-706J10.2","RP1-305G21.1","RP11-406A20.1","PSPHP1","RP11-564P9.1","RP11-618P13.1","RP3-495O10.1","AC073255.1","CTC-462L7.1","RP3-471C18.2","CTC-550M4.1","RNA5SP181","MTND5P21","AE000661.37","TRDD3","RP11-255G12.3","RP11-1141N12.1","AC018647.3","RNU6-27P","RP11-60A8.1","SLC7A15P","RNU7-190P","RP1-309H15.2","RP11-434D9.1","RP11-214L13.1","RP11-428C19.5","RP11-11L12.2","RP11-323I1.1","RNA5SP75","RP11-8L2.1","RP11-456O19.2","CTD-2008L17.2","AC003985.1","RPL21P41","ARHGEF26-AS1","TTTY19","ALDH7A1P2","RP11-367G18.2","KCND3-IT1","RP11-184M15.2","LINC00856","AC010145.4","RP11-146I2.1","RP11-99J16__A.2","RP11-394O9.1","AP003900.6","LINC00222","RP11-89M20.1","CHCHD4P2","RP11-714G18.1","RP11-437J19.1","CTD-2316B1.2","AC013406.1","CTD-2008L17.1","CTB-99A3.1","RP11-1018N14.2","GUCY1B2","CTC-340D7.1","AC007128.1","RP11-90C4.2","RP11-318M2.2","HMGB3P11","OFD1P3Y","MIR4739","CTD-2572N17.1","LINC01028","AC002539.1","RP11-343K8.3","RP11-664D1.1","CTC-498M16.4","EEF1A1P9","RP11-300M24.1","AC092646.1","AC005592.2","PNPT1P2","RP11-456O19.4","NR2F1-AS1","RP5-837I24.2","RP11-492M23.2","CRYBB2P1","RP11-423J7.1","AC009410.1","RP5-1010E17.1","SNORD112","AC004875.1","AC097372.1","RP11-814M22.2","RP3-453D15.1","RP11-163N6.2","RP11-293B20.2","RP11-620J15.2","RP11-291C6.1","RP11-776H12.1","LINC01038","RP11-347L18.1","EEF1B2P4","RP11-1003J3.1","RP11-396O20.1","RP11-788A4.1","NPM1P14","RP11-394I13.2","CAHM","RP11-94H18.2","HOXC13-AS","RP11-724M22.1","RP11-331K15.1","CCDC13-AS1","CTD-2541J13.2","RP11-114H23.1","MIR663A","TPT1P9","RNU6ATAC19P","LINC01162","CHIAP2","TMEM161B-AS1","RP11-630C16.2","CASC6","CASC8","RNA5SP350","RP5-1177M21.1","RP11-89K10.1","RP11-438C19.2","LHFPL3-AS1","RP11-351J23.1","RP11-109E10.2","AQP4-AS1","RP11-745C15.2","EIF4BP3","AC099778.1","GS1-122H1.2","RP11-33N16.3","CTBP2P1","RP11-436K8.1","AC090133.1","RP11-118B18.2","RP4-541C22.5","RP11-654C22.2","AC016712.2","CTC-458G6.2","AC133633.1","RP11-669M16.1","RP11-708B6.2","RP11-66N24.4","RP11-563J2.2","RNA5SP293","LINC00474","AC011747.6","SEPT7P3","RP11-254A17.1","RP11-1325J9.1","AC009498.1","LINC-ROR","RP5-1027O11.1","AC079135.1","RP11-977P2.1","AC005152.3","RP11-414B7.1","RP11-32B5.1","AC022909.1","RNU6-487P","RP11-121E16.1","LINC01036","LINC01158","RP11-317N12.1","KCND3-AS1","RP11-398G24.2","RP3-359N14.2","EDNRB-AS1","AHCYP3","RP11-40G16.1","RP11-109P6.2","AC112693.1","AC007193.6","RP11-147G16.1","RP11-1058G23.1","RP5-978I12.1","STX18-IT1","U3","RP11-715C4.1","AC068057.1","AC012451.1","RP1-177I10.1","LINC01091","CYCSP26","LINC00511","RPL21P67","RP11-429O1.1","RP11-3L10.3","RP11-588F10.1","RP11-292D4.3","MIR3139","RP11-329N22.1","RP11-448P19.1","RP11-334E6.3","RNU6-356P","TXLNG2P","AF212831.2","RN7SKP279","RP11-222A5.1","RP11-540K16.1","RNA5SP349","RP11-434H14.1","TTTY18","RP11-541G9.2","LINC00466","HMGB3P3","RP11-693J15.4","RP11-693J15.5","RP11-863K10.2","CTD-2008N3.1","RN7SL318P","PARP4P1","RMST","RP11-67P15.1","AC091969.1","AC007682.1","RP11-14O19.2","GS1-122H1.1","LINC00461","RP11-179A16.1","RP11-388N2.1","RP11-174J11.1","RP11-365H23.1","RP11-395D3.1","RP11-462G22.1","CTB-70G10.1"],"mode":"markers","name":"signif_fc","marker":{"color":"#A93EBC"}},{"type":"scatter","inherit":false,"x":[-0.91,-0.87,-0.84,-0.76,0.57,0.63,0.69,0.7,0.73,0.74,0.75,0.78,0.79,0.87,0.9,0.91,0.94,0.96,0.96,0.53,0.64,0.67,0.74,0.8,0.82,0.85,0.85,0.9,0.99],"y":[2.26921772433361,2.0762380391713,1.61261017366127,1.88941028970075,1.49349496759513,1.41341269532825,1.56224943717961,1.5003129173816,1.38721614328026,1.72353819582676,2.47495519296315,1.83564714421556,1.77728352885242,1.97881070093006,1.92081875395238,2.44129142946683,1.99567862621736,2.54060751224077,1.38721614328026,1.36251027048749,1.53760200210104,1.40782324260413,1.85078088734462,1.74957999769111,3.30277065724028,1.61439372640169,1.37365963262496,1.92081875395238,3.49894073778225],"text":["DNM3","IL1RAP","MYPN","ARHGAP20","GFPT1","VGLL4","TNFRSF19","IFI6","FMNL3","CORO1C","ARHGAP42","RAI14","CREB5","CD59","NR6A1","MTNR1B","C1orf122","IRF2BP2","ABLIM1","CTB-164N12.1","AC068570.1","RSL24D1P1","AL357519.2","NALCN-AS1","PRICKLE2-AS3","RPS4XP2","AL672294.1","MIR181A2HG","RP11-470E16.1"],"mode":"markers","name":"signif","marker":{"color":"#DCB68C"}},{"type":"scatter","inherit":false,"x":[-0.35,-0.32,-0.3,-0.23,-0.18,-0.03,0.05,0.12,0.39,0.39,0.47,0.48,0.53,0.54,0.55,0.55,0.55,0.56,0.57,0.59,0.59,0.6,0.61,0.62,0.7,0.8,1,1,-0.47,-0.2,-0.1,0.2,0.23,0.26,0.34,0.35,0.35,0.39,0.41,0.48,0.6,0.72,0.82,0.82],"y":[0.549750891680639,0.275724130399211,0.571865205971211,0.3767507096021,0.231361898752386,0.0190880622231565,0.0245681914907371,0.175223537524454,0.978810700930062,0.732828271596986,0.787812395596042,0.54515513999149,0.749579997691106,0.939302159646388,1.22112552799726,0.782516055786094,0.657577319177794,0.931814138253838,0.913640169325252,0.838631997765025,0.838631997765025,0.935542010773082,1.27002571430044,1.28819277095881,1.27002571430044,1.10292299679058,4.56863623584101,1.97061622231479,0.627087997029893,0.27083521030723,0.0883098412461389,0.337242168318426,0.224025668870631,0.346787486224656,0.436518914605589,0.958607314841775,0.397940008672038,0.844663962534938,0.838631997765025,0.804100347590766,0.935542010773082,1.20273245916928,1.21041928783557,1.10568393731556],"text":["FRMD3","PPP1R3C","BCAS1","NAALADL2","SETD5","CREM","MECOM","ANKRD28","PDE4DIP","WNK1","HMGCS1","ATPAF1","CTNND1","CD300LB","NFE2L3","CNKSR3","PI3","PAPPA2","PLEKHA8","CTNNA1","LRRTM2","PPP2R5C","NDUFC1","FSTL1","RP11-362K2.2","PDE1C","JAM3","PTK2","AC013410.2","RP11-644L4.1","MIR607","LINC00352","LINC00886","RP11-753H16.3","RP11-1L12.3","RPL26P31","RP5-855F14.1","RP11-340M11.1","LINC01132","RP1-167F1.2","CTD-2017C7.2","RP11-420A23.1","RP11-167H9.6","RP11-180P8.1"],"mode":"markers","name":"not_signif","marker":{"color":"#FF0000"}}],"layout":{"xaxis":{"title":"Fold"},"yaxis":{"title":"-log10(p.value)"},"hovermode":"closest","margin":{"b":40,"l":60,"t":25,"r":10}},"url":null,"width":null,"height":null,"source":"A","config":{"modeBarButtonsToRemove":["sendDataToCloud"]},"base_url":"https://plot.ly"},"evals":[],"jsHooks":[]}</script><!--/html_preserve-->

```r
# DiffBind_volcano_plot_top_protein_coding_promoters(diff_df = diff_df)
Diff_stats(diff_df = diff_df)
```

  
  
  
******  
### Overall DiffBind Stats  
Total number of DiffBind peaks:  
3192  
  
Total number of DiffBind genes:  
1824  
  
Total number positive fold change genes:  
1485  
  
Total number negative fold change genes:  
339  
  
  
******  
Total number of p <0.05 genes:  
1782  
  
Total number of p <0.05 genes (pos. FC):  
1452  
  
Total number of p <0.05 genes (neg. FC):  
330  
  
  
******  
Total number of log2(Fold Change) > 1 genes:  
1751  
  
Total number of log2(Fold Change) > 1 genes (pos. FC):  
1425  
  
Total number of log2(Fold Change) > 1 genes (neg. FC):  
326  
  
  
******  
Total number of p < 0.05 & log2(Fold Change) > 1 genes:  
1751  
  
Total number of p < 0.05 & log2(Fold Change) > 1 genes (pos. FC):  
1425  
  
Total number of p < 0.05 & log2(Fold Change) > 1 genes (neg. FC):  
326  
  
  
  
  
******  
### Protein Coding Gene Stats  
Total number of DiffBind peaks:  
2030  
  
Total number of DiffBind genes:  
1113  
  
Total number positive fold change genes:  
926  
  
Total number negative fold change genes:  
187  
  
  
******  
Total number of p <0.05 genes:  
1087  
  
Total number of p <0.05 genes (pos. FC):  
906  
  
Total number of p <0.05 genes (neg. FC):  
181  
  
  
******  
Total number of log2(Fold Change) > 1 genes:  
1066  
  
Total number of log2(Fold Change) > 1 genes (pos. FC):  
889  
  
Total number of log2(Fold Change) > 1 genes (neg. FC):  
177  
  
  
******  
Total number of p < 0.05 & log2(Fold Change) > 1 genes:  
1066  
  
Total number of p < 0.05 & log2(Fold Change) > 1 genes (pos. FC):  
889  
  
Total number of p < 0.05 & log2(Fold Change) > 1 genes (neg. FC):  
177  
  
  
  
******  
  
******  

```r
# LOOP DOES NOT WORK, FIGURE THIS OUT LATER
# DiffBind_volcano_plotly_top_protein_coding_promoters(diff_df = read.delim(file = sample_file_list[1], header = TRUE,sep = ','))

# for(i in seq_along(sample_file_list)){
#     sample_file <- sample_file_list[i]
#     mycat(paste0("## ", names(sample_file), ' {.tabset}\n'))
#     # mycat(paste0("File: ", sample_file, '\n'))
#     diff_df <- read.delim(file = sample_file,header = TRUE,sep = ',')
#     diff_df_processed <- Diff_process_data(diff_df)
#     DiffBind_volcano_plotly_top_protein_coding_promoters(diff_df = diff_df_processed)
#     Diff_stats(diff_df = diff_df)
# }

# this also does not work... 

# Map(function(x){
#     mycat(paste0("## ", names(x), ' {.tabset}\n'))
#     mycat('### Volcano Plot: Plot.ly\n')
#     print(plot_ly(data = Diff_process_data(read.delim(file = x, header = TRUE,sep = ',')), 
#             x = Fold, y = -log10(p.value), text = external_gene_name, mode = "markers", color = as.ordered(group), colors = plot_colors))
#     # DiffBind_volcano_plotly_top_protein_coding_promoters(diff_df = Diff_process_data(read.delim(file = x, header = TRUE,sep = ',')))
#     Diff_stats(diff_df = read.delim(file = x, header = TRUE,sep = ','))
# }, sample_file_list)
```

# System Information


```r
system('uname -srv',intern=T)
```

```
## [1] "Darwin 15.6.0 Darwin Kernel Version 15.6.0: Mon Aug 29 20:21:34 PDT 2016; root:xnu-3248.60.11~1/RELEASE_X86_64"
```

```r
sessionInfo()
```

```
## R version 3.3.0 (2016-05-03)
## Platform: x86_64-apple-darwin13.4.0 (64-bit)
## Running under: OS X 10.11.6 (El Capitan)
## 
## locale:
## [1] en_US.UTF-8/en_US.UTF-8/en_US.UTF-8/C/en_US.UTF-8/en_US.UTF-8
## 
## attached base packages:
## [1] stats     graphics  grDevices utils     datasets  methods   base     
## 
## other attached packages:
## [1] dplyr_0.5.0      data.table_1.9.6 plotly_3.6.0     ggplot2_2.1.0   
## 
## loaded via a namespace (and not attached):
##  [1] Rcpp_0.12.5      knitr_1.13       magrittr_1.5     munsell_0.4.3   
##  [5] colorspace_1.2-6 R6_2.1.2         stringr_1.0.0    httr_1.2.0      
##  [9] plyr_1.8.4       tools_3.3.0      grid_3.3.0       gtable_0.2.0    
## [13] DBI_0.4-1        htmltools_0.3.5  lazyeval_0.2.0   assertthat_0.1  
## [17] yaml_2.1.13      digest_0.6.9     tibble_1.0       gridExtra_2.2.1 
## [21] formatR_1.4      tidyr_0.5.1      viridis_0.3.4    base64enc_0.1-3 
## [25] htmlwidgets_0.7  evaluate_0.9     rmarkdown_0.9.6  stringi_1.1.1   
## [29] scales_0.4.0     jsonlite_0.9.22  chron_2.3-47
```

```r
# save.image(compress = TRUE, )
```

# References
