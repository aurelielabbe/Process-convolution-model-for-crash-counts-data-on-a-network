---
title: "A Process Convolution Model for Crash Count Data on a Network"
author: "Hassan Rezaee"
date: "8/27/2021"
output: html_document
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE)
```

## Introduction

The following is an application of the Network Process Convolution (NPC) model to a simulated crash data set over a small road network extracted from Ottawa, Canada. The NPC model incorporates the road network structure on which crashes are observed through a kernel convolution approach. The kernel functions are evaluated between sets of spatial locations representing the crash and knots/support points. The kernel functions are evaluated based on the path distance (rather than the Euclidean distance) between these locations. We have tried to include as much details in the paper and in the following repository, but feel free to contact us at hassan_rezaee65@yahoo.com should you have any questions or ideas to improve the code. 


We assume the following are available prior to the implementation:

1. Crash data: matrix of size nx3 including the coordinates of n crash data locations and the corresponding crash counts. The coordinates can be in UTM or Lat-Long format as long as they match the coordinate system of the road network. A UTM coordinate system is preferred as it would be easier to select the kernel width in meters.

2. Covariates: matrix of size nxp where p is the number of covariates available. Here, these covariates are generated from N(0,1). The data simulation process is explained in detail in the paper; hence, interested readers are referred to the article for more information. Here we focus solely on the implementation of the model only.

3. Road network shape files: road shape files of class SpatialLines. These shape files can be imported from the OpenStreetMap databases; however, in this study, the road network shape files were provided by the project industrial partner.

4. Data adjacency matrix: matrix of size nxn specifying the neighborhood structure of the road network at observed data locations. This is not required for the NPC model and is used in the pCAR model solely for comparison purposes.

5. Set of custom functions written in R (available in this repository) to prepare the road network, compute distances and kernel densities.





## Implementation
Lets start first by loading the required libraries.

```{r}
remove(list=ls())
while (!is.null(dev.list()))  dev.off()
lib_list = c("profvis","igraph","sp","shp2graph", "gMOIP")
```
```{r include=FALSE}
lapply(lib_list, require, character.only = TRUE)
```



Next, we load some pre-defined custom functions including:

- make_graph: used to create a graph (igraph object) from given road network shape files
- lines_midpoints: used to split all the lines of the road network shape files. We assume the crash data represent the links (and no intersection); however, the crash data are given usually by their point-reference locations. The lines midpoints are then used to relocate the crash data onto the nearest line midpoint on the road network.
- compute_D: used to compute the path distance and the corresponding weights between sets of points corresponding to the data and knot locations
- compute_K: used to compute kernel function values between sets of points based on their distances computed from compute_D



```{r}
source("lines_midpoints.R")
source("compute_K.R")
source("compute_D.R")
source("make_graph.R")
```




As mentioned, we have prepared a sample simulated data set. Data have simulated over a small road network taken from downtown Ottawa. The data simulation process is described in detail in the article.

```{r}
load("sample_crash_data_simulated.RData")
n = nrow(crash)
y = crash[,3]
coords = crash[,1:2]
p = ncol(covars)
```



## Create a graph from road network shape files
Next we need to load the road network shape files and create an igraph graph object. This is accomplished using the function make_graph.R

```{r}
graph_data = make_graph(roads_shp);
igr = graph_data[[1]]
nodes_coords = graph_data[[2]]
degrees = as.matrix(igraph::degree(igr, v = igraph::V(igr), mode = "total", loops = F, normalized = FALSE))
```


In a road crash modeling process we are given a set of observed crash locations and their corresponding frequencies. These locations must be relocated onto the graph created from the road network shape files. Here we use a simple nearest neighbor approach.
```{r}
knns = FNN::get.knnx(query=coords, data=nodes_coords, k=1, algo="kd_tree")
data_nodes = as.matrix(knns[["nn.index"]][,1])
```



## Knot locations
The Gaussian process that captures the spatial correlation among crash data is formed based on a kernel convolution approach. These kernels are evaluated based set of observed crash data locations, and the knots or support points. The common practice is to scatter these points on a regular grid. Since distances are computed between locations only on the road network, the knots must be located on the road network too. For that, we start by forming a regular mesh over the continuous space encompassing the road network and then relocate these points to the nearest location on the road network. In the end, we remove the knots that are outside the convex hull of the crash data locations.

```{r}
mx = n*1.5 # must be tuned in way that the resulting m (number of knots) is smaller than n (number of data)
xg = seq(from=min(nodes_coords[,1]), to=max(nodes_coords[,1]), length.out=round(sqrt(mx)))
yg = seq(from=min(nodes_coords[,2]), to=max(nodes_coords[,2]), length.out=round(sqrt(mx)))
xy = plot3D::mesh(xg,yg);xy=cbind(c(xy$x),c(xy$y))
m = nrow(xy)
knns = FNN::get.knnx(query=xy, data=nodes_coords, k=5, algo="kd_tree")
knot_ids = unique(as.matrix(knns[["nn.index"]][,1]))
# knots that are within the convex hull of data locations
pl = as.matrix(nodes_coords[data_nodes,])
point_in_ids = gMOIP::inHull(nodes_coords[knot_ids,], pl)
point_in_ids = which(point_in_ids==1);
knot_ids = knot_ids[point_in_ids]; 
m=length(knot_ids)
```
 
 
 
Display data and knots on the road network
```{r, fig.width = 4, fig.height = 4}
w=5; par(mfrow=c(1,1), mar = c(w, w, w, w));
plot(roads_shp, axes=T, main="Data (green), knots (blue)", xlab="Easting (m)", ylab="Northing (m)")
points(nodes_coords[knot_ids,], pch=19, col="blue", cex=1);
points(nodes_coords[data_nodes,], pch=19, col="green", cex=1);
```


## Compute path distances
Here, we compute the path distance (D matrix) and the corresponding weights (W matrix) between sets of data and knot locations on the road network. The weights are used in the kernel function evaluations to ensure that the resulting covariance matrix between data locations is stationary (about the variance) over the entire road network. Function compute_D performs the distance calculations. We have used the "get.shortest.paths" function available in the igraph package to find the trajectory between sets of source and end points. We then manually compute the path distance by simply accumulating all the Euclidean distances between successive points along the shortest path.
```{r}
DW = compute_D(igr, data_nodes, knot_ids, nodes_coords); D = DW[[1]]; W = DW[[2]]
```


## Compute weighted densities
Function compute_K is used to evaluate the kernel function based on the distances computed in the previous step. We have considered three kernel types: 1. Gaussian, 2. Exponential, 3. Epanechnikov. As noted in the paper, the kernel type does not exert a large influence on the covariance function as well as the modeling results.
```{r}
kernel_width = 1500 # The common practice is to choose kernel width equal or larger than the smaller distance between knot locations.
K = compute_K(3, kernel_width, D, W)
```


## Prepare INLA input data frame
Next, we prepare input data file for the INLA algorithm. As it shows we have three covariates indicated as covar1 through covar3 which are imported at first along with the road network and the observed crash data.
```{r}
inla_data = data.frame("id" = 1:n, "crash" = y, "covar1" = covars[,1], "covar2" = covars[,2], "covar3" = covars[,3])
```




## Run the selected models
Here we have considered the proposed model (NPC), a non-spatial model (Poisson regression; PR) and a spatial model (proper Conditional Autoregressive; pCAR) for comparison purposes. All these models are implemented in INLA.


We start by running the PR model on the simulated data. We have enabled INLA to compute the DIC and WAIC as well. After the model is fit to the data, we extract the fitted values and the 2.5th and 97.5th percentiles for the fitted crash values as well as the random effects (for NPC and pCAR only).
```{r}
formula_pr <- crash ~ 1 + covar1 + covar2 + covar3
inla_fit_pr <- INLA::inla(formula_pr, family = "poisson", data = inla_data, control.compute=list(config=T,dic=T,waic=T),
                      control.inla = list(h = 1),
                      verbose=F)
y_hat_pr = inla_fit_pr$summary.fitted.values$mean
y_hat_pr_qts = matrix(NA, n, 2)
y_hat_pr_qts[,1] = inla_fit_pr[["summary.fitted.values"]][["0.025quant"]]
y_hat_pr_qts[,2] = inla_fit_pr[["summary.fitted.values"]][["0.975quant"]]
beta_hat_pr = inla_fit_pr[["summary.fixed"]][["mean"]]
beta_hat_pr_qts1 = inla_fit_pr[["summary.fixed"]][["0.025quant"]]
beta_hat_pr_qts2 = inla_fit_pr[["summary.fixed"]][["0.975quant"]]
```


Next we run the pCAR mode. Note that the pCAR method models the spatial correlation among crash data through adjacency matrices. These matrices are computed from the road network directly, i.e., a first degree neighborhood structure and loaded into the project along with other input data at first.
```{r}
g = INLA::inla.read.graph(adj_mat)
formula_car <- crash ~ 1 + covar1 + covar2 + covar3 + f(id, model = "besagproper", graph = g, constr= FALSE,
                                                          hyper = list(prec = list(
                                                            prior = "loggamma",
                                                            initial = 0,
                                                            fixed = F,
                                                            param = c(1.31, 0.33))))
inla_fit_car <- INLA::inla(formula_car, family = "poisson", data = inla_data, control.compute=list(config=T,dic=T,waic=T),
                       control.inla = list(h = 1),
                       verbose=F, control.fixed = list(prec.intercept = 0.3, prec=1))
z_hat_car = as.matrix(inla_fit_car[["summary.random"]][["id"]][["mean"]])
y_hat_car = as.matrix(inla_fit_car$summary.fitted.values$mean)
beta_hat_car = inla_fit_car[["summary.fixed"]][["mean"]]
beta_hat_car_qts1 = inla_fit_car[["summary.fixed"]][["0.025quant"]]
beta_hat_car_qts2 = inla_fit_car[["summary.fixed"]][["0.975quant"]]
z_hat_car_qts = matrix(NA, n, 2)
z_hat_car_qts[,1] = inla_fit_car[["summary.random"]][["id"]][["0.025quant"]]
z_hat_car_qts[,2] = inla_fit_car[["summary.random"]][["id"]][["0.975quant"]]
y_hat_car_qts = matrix(NA, n, 2)
y_hat_car_qts[,1] = inla_fit_car[["summary.fitted.values"]][["0.025quant"]]
y_hat_car_qts[,2] = inla_fit_car[["summary.fitted.values"]][["0.975quant"]]
print("CAR done")
```



Last we run the proposed NPC model. This model is specified as the "z" model in INLA.
```{r}
formula_npc <- crash ~ 1 + covar1 + covar2 + covar3 + f(id, model = "z", Z = K,
                                                          hyper = list(prec = list(
                                                            prior = "loggamma",
                                                            initial = 0,
                                                            fixed = F,
                                                            param = c(1.316611, 0.3361755))));
inla_fit_npc <- INLA::inla(formula_npc, family = "poisson", data = inla_data, control.compute=list(config=T,dic=T,waic=T),
                       control.inla = list(h=1),
                       verbose=F, control.fixed = list(prec.intercept = 0.3, prec=1))
y_hat_npc = inla_fit_npc[["summary.fitted.values"]][["0.5quant"]]
x_hat = inla_fit_npc[["summary.random"]][["id"]][["mean"]][-(1:n)]
z_hat_npc = K%*%x_hat
beta_hat_npc = inla_fit_npc[["summary.fixed"]][["mean"]]
beta_hat_npc_qts1 = inla_fit_npc[["summary.fixed"]][["0.025quant"]]
beta_hat_npc_qts2 = inla_fit_npc[["summary.fixed"]][["0.975quant"]]
x_hat_npc_qts = matrix(NA, m, 2)
x_hat_npc_qts[,1] = inla_fit_npc[["summary.random"]][["id"]][["0.025quant"]][-(1:n)]
x_hat_npc_qts[,2] = inla_fit_npc[["summary.random"]][["id"]][["0.975quant"]][-(1:n)]
z_hat_npc_qts = matrix(NA, n, 2)
z_hat_npc_qts[,1] = inla_fit_npc[["summary.random"]][["id"]][["0.025quant"]][1:n]
z_hat_npc_qts[,2] = inla_fit_npc[["summary.random"]][["id"]][["0.975quant"]][1:n]
y_hat_npc_qts = matrix(NA, n, 2)
y_hat_npc_qts[,1] = inla_fit_npc[["summary.fitted.values"]][["0.025quant"]]
y_hat_npc_qts[,2] = inla_fit_npc[["summary.fitted.values"]][["0.975quant"]]

```




To provide a preliminary comparison between the three models' fitting performance, here we display the scatter plots between true crash data and the fitted values along with their 95% credible intervals computed within the INLA algorithm.
```{r, fig.width = 9, fig.height = 3}
w=5; par(mfrow=c(1,3), mar = c(w, w, w, w));
m1 = c(0, max(y_hat_pr_qts[,2], y_hat_car_qts[,2], y_hat_npc_qts[,2]))
plot(y, y_hat_pr, xlim = m1, ylim=m1, ylab=expression(hat(y)),xlab=expression(y), main="PR");abline(0,1)
for (i in 1:n){
   segments(x0=y[i], y0=y_hat_pr_qts[i,1], x1 = y[i], y1 = y_hat_pr_qts[i,2], col = "gray", lwd = 1)
}
points(y, y_hat_pr, pch=19, cex=1, col="black")
plot(y, y_hat_car, xlim = m1, ylim=m1, ylab=expression(hat(y)),xlab=expression(y), main="pCAR");abline(0,1)
for (i in 1:n){
  segments(x0=y[i], y0=y_hat_car_qts[i,1], x1 = y[i], y1 = y_hat_car_qts[i,2], col = "gray", lwd = 1)
}
points(y, y_hat_car, pch=19, cex=1, col="black")
plot(y, y_hat_npc, xlim = m1, ylim=m1, ylab=expression(hat(y)),xlab=expression(y), main="NPC");abline(0,1)
for (i in 1:n){
  segments(x0=y[i], y0=y_hat_npc_qts[i,1], x1 = y[i], y1 = y_hat_npc_qts[i,2], col = "gray", lwd = 1)
}
points(y, y_hat_npc, pch=19, cex=1, col="black")
```



