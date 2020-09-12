-   [Introduction](#introduction)

-   [Introduction](Introduction)
-   [Data Acquisition &
    Pre-processing](Data%20Acquisition%20&%20Pre-processing)
-   [Partitioning Occurrences for
    Evaluation](Partitioning%20Occurrences%20for%20Evaluation)
-   [Running ENMeval](Running%20ENMeval)
-   [Plotting results](Plotting%20results)
-   [Downstream Analyses](Downstream%20Analyses)
-   [Resources](Resources)

Introduction
------------

<!-- [`ENMeval`](https://cran.r-project.org/web/packages/ENMeval/index.html) is an R package that performs automated runs and evaluations of ecological niche models, and currently only implements [Maxent](https://www.cs.princeton.edu/~schapire/maxent/). `ENMeval` was made for those who want to "tune" their models to maximize predictive ability and avoid overfitting, or in other words, optimize model complexity to balance goodness-of-fit and predictive ability. The primary function, `ENMevaluate`, does all the heavy lifting and returns several items including a table of evaluation statistics and, for each setting combination (here, colloquially: *runs*), a model object and a raster layer showing the model prediction across the study extent. There are also options for calculating niche overlap between predictions, running in parallel to speed up computation, and more. For a more detailed description of the package, check out the open-access publication: -->
<!-- [Muscarella, R., Galante, P. J., Soley-Guardia, M., Boria, R. A., Kass, J. M., Uriarte, M. and Anderson, R. P. (2014), ENMeval: An R package for conducting spatially independent evaluations and estimating optimal model complexity for Maxent ecological niche models. Methods Ecol Evol, 5: 1198–1205.](https://onlinelibrary.wiley.com/doi/10.1111/2041-210X.12261/full) -->
<!-- ## Data Acquisition & Pre-processing -->
<!-- In this vignette, we briefly demonstrate acquisition and pre-processing of input data for `ENMeval`. There are a number of other excellent tutorials on these steps, some of which we compiled in the [Resources](Resources) section. -->
<!-- We'll start by downloading an occurrence dataset for [*Bradypus variegatus*] (https://en.wikipedia.org/wiki/Brown-throated_sloth), the Brown-throated sloth.  We'll go ahead and load the `ENMeval` and [`spocc`](https://cran.r-project.org/web/packages/spocc/index.html) packages.  (We are using `spocc` to download occurrence records). -->
<!-- ```{r occDownload} -->
<!-- if (!require('spocc')) install.packages('spocc', repos = "https://cran.us.r-project.org")  -->
<!-- if (!require('ENMeval')) install.packages('ENMeval', repos = "https://cran.us.r-project.org") -->
<!-- library(spocc) -->
<!-- library(ENMeval) -->
<!-- # Search GBIF for occurrence data. -->
<!-- bv <- occ('Bradypus variegatus', 'gbif', limit=300, has_coords=TRUE)   -->
<!-- # Get the latitude/coordinates for each locality. -->
<!-- occs <- bv$gbif$data$Bradypus_variegatus[,2:3]   -->
<!-- # Remove duplicate rows. -->
<!-- occs <- occs[!duplicated(occs),]   -->
<!-- ``` -->
<!-- We are going to model the climatic niche suitability for our focal species using climate data from [WorldClim](https://www.worldclim.org/). WorldClim has a range of variables available at various resolutions; for simplicity, here we'll use the 9 bioclimatic variables at 10 arcmin resolution (about 20 km across at the equator) included in the `dismo` package. These climatic data are based on 50-year averages from 1950-2000. Now's also a good time to load the package, as it includes all the downstream dependencies (`raster`, `dismo`, etc.). -->
<!-- ```{r envDownload, warning=FALSE, message=FALSE, fig.width=6, fig.height=8} -->
<!-- # First, load some predictor rasters from the dismo folder: -->
<!-- files <- list.files(path=paste(system.file(package='dismo'), '/ex', sep=''), pattern='grd', full.names=TRUE) -->
<!-- # Put the rasters into a RasterStack, kind of like a list for rasters: -->
<!-- envs <- stack(files) -->
<!-- # Plot first raster in the stack: bio1 (Annual Mean Temperature) -->
<!-- plot(envs[[1]]) -->
<!-- # Plot all the occurrence points on the raster -->
<!-- points(occs) -->
<!-- # There are some points all the way to the south-east, far from all others. Let's say we know that this represents a subpopulation that we don't want to include, and want to remove these points from the analysis. We can find them by first sorting the occs table by latitude. -->
<!-- occs[order(occs$latitude),] -->
<!-- # We see there are two such points, and we can find them by specifying a logical statement that says to find all records with latitude less than -20. -->
<!-- index <- which(occs$latitude < -20) -->
<!-- # Next, let's subset our dataset to remove them by using the negative assignment on the index vector. -->
<!-- occs <- occs[-index,] -->
<!-- # Let's plot our new points over the old ones to see what a good job we did. -->
<!-- points(occs, col='red') -->
<!-- ``` -->
<!-- Next, we will specify the background extent by cropping (or "clipping" in ArcGIS terms) our global predictor variable rasters to a smaller region. Since our models will compare the environment at occurrence (or, presence) localities to the environment at background localities, we need to sample random points from a background extent. To help ensure we don't include areas that are suitable for our species but are unoccupied due to limitations like dispersal constraints, we will conservatively define the background extent as an area surrounding our occurrence localities. We will do this by buffering a bounding box that includes all occurrence localities. Some other methods of background extent delineation (e.g., minimum convex hulls) are more conservative because they better characterize the geographic space holding the points. In any case, this is one of the many things that you will need to carefully consider for your own study. -->
<!-- ```{r backgExt, message=FALSE} -->
<!-- # Make a SpatialPoints object -->
<!-- occs.sp <- SpatialPoints(occs) -->
<!-- # Get the bounding box of the points -->
<!-- bb <- bbox(occs.sp) -->
<!-- # Add 5 degrees to each bound by stretching each bound by 10, as the resolution is 0.5 degree. -->
<!-- bb.buf <- extent(bb[1]-10, bb[3]+10, bb[2]-10, bb[4]+10) -->
<!-- # Crop environmental layers to match the study extent -->
<!-- envs.backg <- crop(envs, bb.buf) -->
<!-- ``` -->
<!-- We may also, however, want to remove the Caribbean islands (for example) from our background extent. For this, we can use tools from the [`maptools`](https://cran.r-project.org/web/packages/maptools/index.html) package, which is not automatically loaded with `ENMeval`. -->
<!-- ```{r removeCaribbean, message=FALSE} -->
<!-- if (!require('maptools')) install.packages('maptools', repos = "https://cran.us.r-project.org") -->
<!-- if (!require('rgeos')) install.packages('rgeos', repos = "https://cran.us.r-project.org") -->
<!-- library(maptools) -->
<!-- library(rgeos) -->
<!-- # Get a simple world countries polygon -->
<!-- data(wrld_simpl) -->
<!-- # Get polygons for Central and South America -->
<!-- ca.sa <- wrld_simpl[wrld_simpl@data$SUBREGION == 5 | wrld_simpl@data$SUBREGION == 13,] -->
<!-- # Both spatial objects have the same geographic coordinate system with slightly different specifications, so just name the coordinate reference system (crs) for ca.sa with that of  -->
<!-- # envs.backg to ensure smooth geoprocessing. -->
<!-- crs(envs.backg) <- crs(ca.sa) -->
<!-- # Mask envs by this polygon after buffering a bit to make sure not to lose coastline. -->
<!-- ca.sa <- gBuffer(ca.sa, width = 1) -->
<!-- envs.backg <- mask(envs.backg, ca.sa)   -->
<!-- # Let's check our work. We should see Central and South America without the Carribbean. -->
<!-- plot(envs.backg[[1]]) -->
<!-- points(occs) -->
<!-- ``` -->
<!-- In the next step, we'll sample 10,000 random points from the background (note that the number of background points is also a consideration you should make with respect to your own study). -->
<!-- ```{r backgPts, fig.width=3, fig.height=4} -->
<!-- # Randomly sample 10,000 background points from one background extent raster (only one per cell without replacement) -->
<!-- # (Note: Since the raster has <10,000 pixels, you'll get a warning and all pixels will be used for background.) -->
<!-- bg <- randomPoints(envs.backg[[1]], n=10000) -->
<!-- # Notice how we have pretty good coverage (every cell). -->
<!-- plot(envs.backg[[1]]) -->
<!-- points(bg, col='red')  -->
<!-- ``` -->
<!-- ## Partitioning Occurrences for Evaluation -->
<!-- A run of ENMevaluate begins by using one of six methods to partition occurrence localities into testing and training bins (folds) for k-fold cross-validation (Fielding and Bell 1997; Peterson et al. 2011). Generally, the data partitioning step is done within the main 'ENMevaluate' function call.  In this section, we illustrate the different options.  -->
<!-- 1. [Block](1. Block) -->
<!-- 2. [Checkerboard1](2. Checkerboard1) -->
<!-- 3. [Checkerboard2](3. Checkerboard2) -->
<!-- 4. [k-1 Jackknife](4. k-1 Jackknife) -->
<!-- 5. [Random k-fold](5. Random k-fold) -->
<!-- 6. [User-defined](6 User-defined) -->
<!-- The first three partitioning methods are variations of what Radosavljevic and Anderson (2014) referred to as 'masked geographically structured' data partitioning. Basically, these methods partition both occurrence records and background points into evaluation bins based on some spatial rules. The intention is to reduce spatial-autocorrelation between points that are included in the testing and training bins, which can overinflate model performance, at least for data sets that result from biased sampling (Veloz 2009; Hijmans 2012; Wenger and Olden 2012).  -->
<!-- #### 1. Block -->
<!-- First, the 'block' method partitions data according to the latitude and longitude lines that divide the occurrence localities into four bins of (insofar as possible) equal numbers. Both occurrence and background localities are assigned to each of the four bins based on their position with respect to these lines. The resulting object is a list of two vectors that supply the bin designation for each occurrence and background point. -->
<!-- ``` {r part.block} -->
<!-- blocks <- get.block(occs, bg) -->
<!-- str(blocks) -->
<!-- plot(envs.backg[[1]], col='gray', legend=FALSE) -->
<!-- points(occs, pch=21, bg=blocks$occ.grp) -->
<!-- ``` -->
<!-- #### 2. Checkerboard1 -->
<!-- The next two partitioning methods are variants of a 'checkerboard' approach to partition occurrence localities. These generate checkerboard grids across the study extent and partition the localities into bins based on where they fall in the checkerboard. In contrast to the block method, both checkerboard methods subdivide geographic space equally but do not ensure a balanced number of occurrence localities in each bin. For these methods, the user needs to provide a raster layer on which to base the underlying checkerboard pattern. Here we simply use the predictor variable RasterStack. Additionally, the user needs to define an *aggregation.factor*. This value tells the number of grids cells to aggregate when making the underlying checkerboard pattern. -->
<!-- The Checkerboard1 method partitions the points into k=2 bins using a simple checkerboard pattern. -->
<!-- ``` {r part.ck1} -->
<!-- check1 <- get.checkerboard1(occs, envs, bg, aggregation.factor=5) -->
<!-- plot(envs.backg[[1]], col='gray', legend=FALSE) -->
<!-- points(occs, pch=21, bg=check1$occ.grp) -->
<!-- # The partitioning method is more clearly illustrated by looking at the background points: -->
<!-- points(bg, pch=21, bg=check1$bg.grp) -->
<!-- # We can change the aggregation factor to better illustrate how this partitioning method works: -->
<!-- check1.large <- get.checkerboard1(occs, envs, bg, aggregation.factor=30) -->
<!-- plot(envs.backg[[1]], col='gray', legend=FALSE) -->
<!-- points(bg, pch=21, bg=check1.large$bg.grp) -->
<!-- points(occs, pch=21, bg=check1.large$occ.grp, col='white', cex=1.5) -->
<!-- ``` -->
<!-- #### 3. Checkerboard2 -->
<!-- The Checkerboard2 method partitions the data into k=4 bins. This is done by aggregating the input raster at two scales. Presence and background points are assigned to a bin with respect to where they fall in checkerboards of both scales. -->
<!-- ``` {r part.ck2} -->
<!-- check2 <- get.checkerboard2(occs, envs, bg, aggregation.factor=c(5,5)) -->
<!-- plot(envs.backg[[1]], col='gray', legend=FALSE) -->
<!-- points(bg, pch=21, bg=check2$bg.grp) -->
<!-- points(occs, pch=21, bg=check2$occ.grp, col='white', cex=1.5) -->
<!-- ``` -->
<!-- #### 4. k-1 Jackknife -->
<!-- The next two methods differ from the first three in that (i) they do not partition the background points into different groups, and (ii) they do not account for spatial autocorrelation between testing and training localities. Primarily when working with relatively small data sets (e.g. < ca. 25 presence localities), users may choose a special case of k-fold cross-validation where the number of bins (k) is equal to the number of occurrence localities (n) in the data set (Pearson et al. 2007; Shcheglovitova and Anderson 2013). This is referred to as the k-1 jackknife.  This method will take prohibitively long times for computation when the number of presence localities is medium to large. -->
<!-- ``` {r part.jk} -->
<!-- jack <- get.jackknife(occs, bg) -->
<!-- plot(envs.backg[[1]], col='gray', legend=FALSE) -->
<!-- points(occs, pch=21, bg=jack$occ.grp)  # note that colors are repeated here -->
<!-- ``` -->
<!-- #### 5. Random k-fold -->
<!-- The 'random k-fold' method partitions occurrence localities randomly into a userspecified number of (k) bins. This method is equivalent to the 'cross-validate' partitioning scheme available in the current version of the Maxent software GUI. -->
<!-- ``` {r part.rand} -->
<!-- # For instance, let's partition the data into five evaluation bins: -->
<!-- random <- get.randomkfold(occs, bg, k=5) -->
<!-- plot(envs.backg[[1]], col='gray', legend=FALSE) -->
<!-- points(occs, pch=21, bg=random$occ.grp) -->
<!-- ``` -->
<!-- #### 6. User-defined -->
<!-- For maximum flexibility, the last partitioning method is designed so that users can define *a priori* partitions. This provides a flexible way to conduct spatially-independent cross-validation with background masking. For example, perhaps we would like to partition points based on a k-means clustering routine. -->
<!-- ``` {r part.user1} -->
<!-- ngrps <- 10 -->
<!-- kmeans <- kmeans(occs, ngrps) -->
<!-- occ.grp <- kmeans$clusterc -->
<!-- plot(envs.backg[[1]], col='gray', legend=FALSE) -->
<!-- points(occs, pch=21, bg=occ.grp) -->
<!-- ``` -->
<!-- When using the user-defined partitioning method, we need to supply ENMevaluate with group identifiers for both occurrence points AND background points. If we want to use all background points for each group, we can set the background to zero. -->
<!-- ``` {r part.user2} -->
<!-- bg.grp <- rep(0, nrow(bg)) -->
<!-- plot(envs.backg[[1]], col='gray', legend=FALSE) -->
<!-- points(bg, pch=16, bg=bg.grp) -->
<!-- ``` -->
<!-- Alternatively, we may think of various ways to partition background data. This depends on the goals of the study but we might, for example, find it reasonable to partition background by clustering around the centroids of the occurrence clusters. -->
<!-- ``` {r part.user3} -->
<!-- centers <- kmeans$center -->
<!-- d <- pointDistance(bg, centers, lonlat=T) -->
<!-- bg.grp <- apply(d, 1, function(x) which(x == min(x))) -->
<!-- plot(envs.backg[[1]], col='gray', legend=FALSE) -->
<!-- points(bg, pch=21, bg=bg.grp) -->
<!-- ``` -->
<!-- Choosing among these data partitioning methods depends on the research objectives and the characteristics of the study system. Refer to the [Resources](Resources) section for additional considerations on appropriate partitioning for evaluation. -->
<!-- ## Running ENMeval -->
<!-- Once you decide which method of data partitioning you would like to use, you are ready to start building models. We now move on to the main function in ENMeval: `ENMevaluate`. -->
<!-- - [Initial considerations](Initial considerations) -->
<!-- - [Exploring the results](Exploring the results (the ENMevaluate object)) -->
<!-- #### Initial considerations -->
<!-- The two main parameters to define when calling `ENMevaluate` are (1) the range of regularization multiplier values and (2) the combinations of feature class to consider. The ***regularization multiplier*** (RM) determines the penalty for adding parameters to the model. Higher RM values impose a stronger penalty on model complexity and thus result in simpler (*flatter*) model predictions. The ***feature classes*** determine the potential shape of the response curves. A model that is only allowed to include linear feature classes will most likely be simpler than a model that is allowed to include all possible feature classes. Much more description of these parameters is available in the [Resources](Resources) section. For the purposes of this vignette, we demonstrate simply how to adjust these parameters. The following section deals with comparing the outputs of each model. -->
<!-- Unless you supply the function with background points (which is recommended in many cases), you will need to define how many background points should be used with the 'n.bg' argument. If any of your predictor variables are categorical (e.g., biomes), you will need to define which layer(s) these are using the 'categoricals' argument. -->
<!-- ENMevaluate builds a separate model for each unique combination of RM values and feature class combinations. For example, the following call will build and evaluate 2 models. One with RM=1 and one with RM=2, both allowing only linear features. -->
<!-- ``` {r enmeval1, hide=c(-1,-5)} -->
<!-- #eval1 <- ENMevaluate(occs, envs, bg, method='checkerboard2', RMvalues=c(1,2), fc=c('L')) -->
<!-- #eval1 -->
<!-- # We may, however, want to compare a wider range of models that can use a wider variety of feature classes: -->
<!-- eval2 <- ENMevaluate(occs, envs, bg, method='checkerboard2', RMvalues=c(1,2), fc=c('L','LQ','LQP')) -->
<!-- eval2 -->
<!-- ``` -->
<!-- When building many models, the command may take a long time to run. Of course this depends on the size of your dataset and the computer you are using. When working on big projects, running the command in parallel can be faster. -->
<!-- ``` {r enmeval2, hide=c(-1,-5,-11,-16), eval=FALSE} -->
<!-- eval2.par <- ENMevaluate(occs, envs, bg, method='checkerboard2', RMvalues=c(1,2), fc=c('L','LQ','LQP'), parallel=TRUE) -->
<!-- eval2.par -->
<!-- ``` -->
<!-- Another way to save time at this stage is to turn off the option that generates model predictions across the full study extent (rasterPreds). Note, however, that these are needed for calculating AICc values so those are returned as NA when the `rasterPreds` argument is set to FALSE. -->
<!-- ``` {r enmeval3, hide=c(-1,-5,-11,-16), eval=FALSE} -->
<!-- eval3 <- ENMevaluate(occs, envs, bg, method='checkerboard2', RMvalues=c(1,2), fc=c('L','LQ','LQP'), rasterPreds=FALSE) -->
<!-- eval3 -->
<!-- # Note that no predictions were generated: -->
<!-- eval3@predictions -->
<!-- # And no AICc values calculated: -->
<!-- eval3@results$aicc -->
<!-- ``` -->
<!-- We can also calculate one of two niche overlap statistics while running `ENMevaluate` by setting the `niche.overlap` argument, which supports Moran's I or Schoener's D. Note that you can also calculate this value at a later stage using the separate `calc.niche.overlap` function. -->
<!-- ``` {r enmeval4, hide=c(-1,-5,-11,-16), eval=FALSE} -->
<!-- overlap <- calc.niche.overlap(eval2@predictions, stat='D') -->
<!-- overlap -->
<!-- ``` -->
<!-- The `bin.output` argument determines if separate evaluation statistics for each testing bin are included in the results file.  If `bin.output=FALSE`, only the mean and variance of evaluation statistics across k bins is returned. -->
<!-- #### Exploring the results (the ENMevaluate object) -->
<!-- We'll use the `eval2` object as our example ENMeval results object for the following demonstrations.  The results of a call to `ENMevaluate` are stored as an object of class ENMevaluate.  This is a specialized class that holds the following items: -->
<!-- - A data.frame holding the model evaluation statistics -->
<!-- - A RasterStack of the model predictions -->
<!-- - A list of maxent model objects -->
<!-- - A data.frame of the original occurrence coordinates -->
<!-- - A vector of the evaluation bins used for the occurrence points -->
<!-- - A data.frame of the background coordinates -->
<!-- - A vector of the evaluation bins used for the background points -->
<!-- - (if `overlap=T`) A matrix of the pairwise niche overlap metric -->
<!-- ``` {r stuff} -->
<!-- str(eval2, max.level=3) -->
<!-- # The first thing to examine is the table of evaluation metrics. -->
<!-- eval2@results  -->
<!-- # Use this to, for example, find the model settings that resulted in delta.AICc of 0: -->
<!-- eval2@results[eval2@results$delta.AICc==0,]   -->
<!-- # Access a RasterStack of the model predictions: -->
<!-- # (Note that these predictions are in the 'raw' output format) -->
<!-- eval2@predictions -->
<!-- # Now let's plot the model with delta.AICc equal == 0: -->
<!-- plot(eval2@predictions[[which(eval2@results$delta.AICc==0)]]) -->
<!-- ``` -->
<!-- We can also access a list of Maxent model objects, which (as all lists) can be subset with double brackets (e.g. `results@eval2[[1]]`). The Maxent model objects provide access to various elements of the model (including the lambda file). The model objects can also be used for predicting models into other time periods or geographic areas. Note that the html file that is created when Maxent is run is **not**  kept. -->
<!-- ```{r mod.obj} -->
<!-- # Let's look at the model object for our "AICc optimal" model: -->
<!-- opt <- eval2@models[[which(eval2@results$delta.AICc==0)]] -->
<!-- opt -->
<!-- # The "lambdas" file can be used to see which variables were used: -->
<!-- opt@lambdas -->
<!-- # The "results" shows the Maxent model statistics: -->
<!-- opt@results -->
<!-- # The ENMevaluate object also remembers which occurrence partitioning method you used: -->
<!-- eval2@partition.method   -->
<!-- ``` -->
<!-- ## Plotting results -->
<!-- Plotting options in R are extremely flexible and here we demonstrate some key tools to explore the results of an ENMevaluate object graphically.   -->
<!-- - [Plotting model predictions](Plotting model predictions) -->
<!-- - [Plotting response curves](Plotting response curves) -->
<!-- ENMeval has a built-in plotting function (`eval.plot`) to visualize the results of different models.  It requires the results table of the ENMevaluation object.  By default, it plots delta.AICc values. -->
<!-- ``` {r plot.res} -->
<!-- eval.plot(eval2@results) -->
<!-- # You can  choose which evaluation metric to plot, and you can include error bars if relevant: -->
<!-- eval.plot(eval2@results, 'Mean.AUC', var='Var.AUC') -->
<!-- eval.plot(eval2@results, 'Mean.ORmin', var='Var.ORmin') -->
<!-- ``` -->
<!-- #### Plotting model predictions -->
<!-- If you generated raster predictions of the models (i.e., `rasterpreds=T`), you can easily plot them. For example, let's look at the first two models included in our analysis. Remember that the output values are in Maxent's 'raw' units. -->
<!-- ``` {r plot.pred1} -->
<!-- plot(eval2@predictions[[1]]) -->
<!-- # We can easily add the occurrence and background points, colored by evaluation bins: -->
<!-- points(eval2@bg.pts, pch=3, col=eval2@bg.grp, cex=0.5) -->
<!-- points(eval2@occ.pts, pch=21, bg=eval2@occ.grp) -->
<!-- ``` -->
<!-- Let's see how model complexity changes the predictions in our example.  We'll compare the model predictions of the model with only linear feature classes and with the highest regularization multiplier value we used (i.e., fc='L', RM=2) versus the model with all feature class combination and the lowest regularization multiplier value we used (i.e., fc='LQP',  RM=1). -->
<!-- ``` {r plot.pred2} -->
<!-- # bisect the plotting area to make two columns -->
<!-- par(mfrow=c(1,2)) -->
<!-- # L2 prediction -->
<!-- plot(eval2@predictions[['L_2']]) -->
<!-- # LQP1 prediction -->
<!-- plot(eval2@predictions[['LQP_1']]) -->
<!-- dev.off() -->
<!-- ``` -->
<!-- #### Plotting response curves -->
<!-- We can also plot the response curves of our model to see how different input variables influence our model predictions. -->
<!-- ``` {r plot.pred3} -->
<!-- #response(eval2@models[[1]]) -->
<!-- ``` -->
<!-- ## Downstream Analyses -->
<!-- ###**UNDER CONSTRUCTION** -->
<!-- - Extracting model results from object (various threshold) -->
<!-- - Use model object to make a new prediction if you want a logistic prediction -->
<!-- - Make a projection to a new extent -->
<!-- - Do MESS map (Use mess() is dismo) -->
<!-- ## Resources -->
<!-- ###**UNDER CONSTRUCTION** -->
<!-- - [Web resources](Web resources) -->
<!-- - [References](References) -->
<!-- #### Web Resources -->
<!-- [Hijmans, R. and Elith, J. (2016) Species distribution modeling with R. dismo vignette.](https://cran.r-project.org/web/package=dismo) -->
<!-- [Yoder, J. (2013) Species distribution models in R. The Molecular Ecologist.](https://www.molecularecologist.com/2013/04/species-distribution-models-in-r/) -->
<!-- [Maxent Google Group](https://groups.google.com/forum/embed/#!forum/maxent) -->
<!-- #### References -->
<!-- ###### General guides -->
<!-- [Merow, C., Smith, M., and Silander, J.A. (2013) A practical guide to Maxent: what it does, and why inputs and settings matter. Ecography 36, 1-12.](https://onlinelibrary.wiley.com/doi/10.1111/j.1600-0587.2013.07872.x/abstract) -->
<!-- [Peterson, A.T., Soberón, J., Pearson, R.G., Anderson, R.P., Martínez-Meyer, E., Nakamura, M., and Araújo, M.B. (2011) Ecological Niches and Geographic Distributions. Monographs in Population Biology, 49. Princeton University Press.](https://press.princeton.edu/titles/9641.html) -->
<!-- [Renner, I.W., Elith, J., Baddeley, A., Fithian, W., Hastie, T., Phillips, S.J., . . . Warton, D.I. (2015) Point process models for presence-only analysis. Methods in Ecology and Evolution 6, 366-379.](https://onlinelibrary.wiley.com/doi/10.1111/2041-210X.12352/abstract) -->
<!-- ###### Model Evaluation -->
<!-- [Aiello-Lammens, M.E., Boria, R.A., Radosavljevic, A., Vilela, B., and Anderson, R.P. (2015) spThin: an R package for spatial thinning of species occurrence records for use in ecological niche models. Ecography 38, 541-545.](https://onlinelibrary.wiley.com/doi/10.1111/ecog.01132/abstract) -->
<!-- [Fielding, A.H. and Bell, J.F. (1997) A review of methods for the assessment of prediction errors in conservation presence-absence models. Environmental Conservation 24, 38-49.](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.463.359&rep=rep1&type=pdf) -->
<!-- [Hijmans, R.J. (2012) Cross-validation of species distribution models: removing spatial sorting bias and calibration with a null model. Ecology 93, 679-688.](https://onlinelibrary.wiley.com/doi/10.1890/11-0826.1/abstract) -->
<!-- [Muscarella, R., Galante, P. J., Soley-Guardia, M., Boria, R. A., Kass, J. M., Uriarte, M. and Anderson, R. P. (2014), ENMeval: An R package for conducting spatially independent evaluations and estimating optimal model complexity for Maxent ecological niche models. Methods Ecol Evol, 5: 1198–1205.](https://onlinelibrary.wiley.com/doi/10.1111/2041-210X.12261/full) -->
<!-- [Radosavljevic, A. and Anderson, R.P. (2014) Making better Maxent models of species distributions: complexity, overfitting and evaluation. Journal of Biogeography 41, 629-643.](https://onlinelibrary.wiley.com/doi/10.1111/jbi.12227/abstract) -->
<!-- [Shcheglovitova, M. and Anderson, R.P. (2013) Estimating optimal complexity for ecological niche models: A jackknife approach for species with small sample sizes. Ecol. Model. 269, 9-17.](https://www.sciencedirect.com/science/article/pii/S0304380013004043) -->
<!-- [Veloz, S.D. (2009) Spatially autocorrelated sampling falsely inflates measures of accuracy for presence-only niche models. Journal of Biogeography 36, 2290-2299.](https://onlinelibrary.wiley.com/doi/10.1111/j.1365-2699.2009.02174.x/abstract) -->
<!-- [Wenger, S.J. and Olden, J.D. (2012) Assessing transferability of ecological models: an underappreciated aspect of statistical validation. Methods in Ecology and Evolution 3, 260-267.](https://onlinelibrary.wiley.com/doi/10.1111/j.2041-210X.2011.00170.x/abstract) -->
<!-- ###### Some examples -->
<!-- [Pearson, R.G., Raxworthy, C.J., Nakamura, M., and Peterson, A.T. (2007) Predicting species distributions from small numbers of occurrence records: a test case using cryptic geckos in Madagascar. Journal of Biogeography 34, 102-117.](https://onlinelibrary.wiley.com/doi/10.1111/j.1365-2699.2006.01594.x/abstract) -->
