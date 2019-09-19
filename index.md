---
title: "Implementation of the Global Spatial Indicator Based on Variogram (G-SIVAR) in R"
output: html_document
---

### (0) Preprocessing 
#### (0a) Loading packages:

```r
library(lctools); library(scales); library(sp); library(gstat); library(reshape2);
library(dplyr); library(bbmle); library(ggplot2); library(tinytex)
```

#### (0b) Reading data:
*Obs: specify accordingly to match file directory.*

```r
setwd("C:/Users/Jorge/Documents/g-sivar") 
```

*Obs: Choose either Centro SP (0b1) or Cidade Ficticia (0b2) datasets to calculate the G-SIVAR.*

#### (0b1) Centro SP data:


```r
dados <- read.table("2.CentroSP.txt",h=T)
```

####  (0b2) Cidade Fict?cia data:

```r
#dados <- read.table("1.CidadeFicticiaBin.txt",h=T)
#nomes <- names(dados); nomes[2:3] <- c('x','y'); colnames(dados) <- nomes
```

### (1) Creating Spatial Data
* Scaling coordinates (0-1):

```r
x_nor = rescale(dados$x)
y_nor = rescale(dados$y)
```

#### (1a) Spatial points dataframe for Centro SP (non-scaled and scaled versions):

```r
#gd <- SpatialPointsDataFrame(coords=cbind(dados$x,dados$y),
                            # data = as.data.frame(dados$Avg_Tot_auto))
gd <- SpatialPointsDataFrame(coords=cbind(x_nor,y_nor),
                             data = as.data.frame(dados$Avg_Tot_auto))
```

#### (1b) Spatial points dataframe for Cidade Fict?cia (non-scaled and scaled versions):

```r
#gd <- SpatialPointsDataFrame(coords=cbind(dados$x,dados$y),
                             #data = as.data.frame(dados$ESCOLHA))
#gd <- SpatialPointsDataFrame(coords=cbind(x_nor,y_nor),
                             #data = as.data.frame(dados$ESCOLHA))
```

### (2) Empirical and Theoretical Semivariograms
* Tests exponential, Spherical and Gaussian models and returns the best fit for anisotropical empirical semivariograms

```r
ev <- variogram(unlist(gd@data) ~ 1, data = gd, alpha=c(0,30,60,90,120,150),tol.hor = 22) 
tv <- fit.variogram(ev, vgm(c("Exp", "Sph", "Gau"))) 
```

#### (2a) Calculating anisotropy parameters (direction of maximum continuity and anisotropy ratio):

* **Direction of maximum continuity**: direction with lowest semivariance at range

```r
max_dir = as.numeric(names(which.min(tapply(ev$gamma,ev$dir.hor,max)))) 
```
* **Orthogonal direction**: with respect to the direction of maximum continuity

```r
ort_dir = abs(ifelse(90 + max_dir >= 180, max_dir - 90, max_dir + 90)) 
```

#### (2b) Fitting anisotropy models for `max_dir` and `ort_dir` to calculate anisotropy ratio:


```r
ev_max_dir <- variogram(unlist(gd@data) ~ 1, data = gd, alpha = max_dir,
                            tol.hor = 22)
ev_ort_dir <- variogram(unlist(gd@data) ~ 1, data = gd, alpha = ort_dir,
                            tol.hor = 22)
tv_max_dir <- fit.variogram(ev_max_dir, vgm(c("Exp", "Sph", "Gau")))
tv_ort_dir <- fit.variogram(ev_ort_dir, vgm(c("Exp", "Sph", "Gau")))
```

* Anisotropy ratio: 

```r
anis_ratio = tv_ort_dir$range[2]/tv_max_dir$range[2]
```

### (3) Simulation of uncorrelated spatial data using the real data coordinates

```r
semivar_simulation <- function(var,coords){
  # Checking variable type (continuous/binary):
  var = unlist(var)
  type = ifelse(any(as.integer(var) != var) || var > 2,"con","bin")
  
  # Computing Empirical and Theoretical Semivariograms for uncorrelated spatial data:
  n = 100
  sill_sim <<- vector("numeric", n)
  
  for (i in 1:n){
    # Generating random sample:
    if (type == 'con') {var_sim = runif(length(var),min(var),max(var))
    } else {var_sim = sample(0:1, length(var), replace = T)}
    
    # Creating SpatialPointsDataFrame for uncorrelated spatial data:
    g <- SpatialPointsDataFrame(coords = coords, data = as.data.frame(var_sim))
    
    # Storing simulated variogram sills in sill_sim:
    ev_s <<- variogram(var_sim ~ 1, data = g)
    tv_s <<- fit.variogram(ev_s, vgm(c("Exp", "Sph", "Gau")))
    sill_sim[i] <<- sum(tv_s$psill) # sum of nugget and partial sill = sill
  }
  # Tossing outliers out to avoid a skewed sill_sim distribution:
  Q1 = quantile(sill_sim, .25); Q3 = quantile(sill_sim, .75)
  cond = (sill_sim > Q1 - 1.5*(Q3 - Q1)) & (sill_sim < Q3 + 1.5*(Q3 - Q1))
  # Generating a bootstrap sample without outliers (n = 100):
  sill_sim <<- sample(sill_sim[cond], n, replace = T)
}
```

* Calling `semivar_simulation` function:

```r
semivar_simulation(gd@data,gd@coords)
```

### (4) Semivariance Scaling
#### (4a) Centering the simulated data on 1:

```r
sill_sim_c <- sill_sim/mean(sill_sim)
```

#### (4b) Estimating mean and standard deviation of the distribution of `sill_sim_c` via Maximum Likelihood Estimation (MLE):

```r
m <- mle2(sill_sim_c ~ dnorm(mean=mu,sd=sd),start=list(mu=0.1,sd=0.1),
          data=data.frame(sill_sim_c))
```


#### (4c) Computing scaled theoretical semivariogram model (with anisotropy):
* Model parameters (coercing sill to 1 and adjusting partiall sill and nugget accordingly):

```r
tv_sc <-  vgm(psill = tv$psill[2]/sum(tv$psill), model = as.vector(tv$model[2]),
              range = tv$range[2], nugget = tv$psill[1]/sum(tv$psill),
              anis = c(max_dir,anis_ratio))
```

* Calculating semivariance values for each distance:

```r
var_func <- variogramLine(tv_sc, dist_vector = ev$dist[ev$dir.hor==max_dir])
```

### (5) Computing Moran's I 
* For each empirical semivariogram distance measure:

```r
moran <- moransI.v(gd@coords, ev$dist[ev$dir.hor==max_dir],
                   unlist(gd@data), family='fixed')
```

![](index_files/figure-html/unnamed-chunk-19-1.png)<!-- -->

```r
moran <- as.data.frame(moran)
names(moran)[3] <- "Morans_I"
```

### (6) Hipothesis Test and Comparison to Moran's I
#### (6a) Comparison table:

```r
df = data.frame(dist = ev$dist[ev$dir.hor==max_dir], Morans_I = moran$Morans_I,
                pv_Morans_I = moran$`P-value res.`, G_SIVAR = var_func$gamma, 
                pv_G_SIVAR = pnorm(var_func$gamma, m@coef[1], m@coef[2]), 
                alpha_0.05 = rep(qnorm(0.05, m@coef[1], m@coef[2]),nrow(moran)))

df
```

```
##          dist   Morans_I   pv_Morans_I   G_SIVAR   pv_G_SIVAR alpha_0.05
## 1  0.02997025 0.40159142  3.774803e-37 0.6631591 8.279057e-20  0.9386693
## 2  0.05982504 0.29709011 1.020651e-116 0.7408862 1.835049e-12  0.9386693
## 3  0.08861750 0.23367910 4.382844e-150 0.7988081 3.409271e-08  0.9386693
## 4  0.10740838 0.19573417 6.097565e-168 0.8294313 2.385855e-06  0.9386693
## 5  0.14428686 0.15559866 1.002898e-176 0.8766439 4.692270e-04  0.9386693
## 6  0.17698592 0.12312978 1.208985e-161 0.9074510 6.530168e-03  0.9386693
## 7  0.20751158 0.09676341 2.054673e-144 0.9292254 2.883906e-02  0.9386693
## 8  0.23636311 0.07887251 1.540709e-125 0.9450748 7.036727e-02  0.9386693
## 9  0.26701349 0.06289219 1.095970e-105 0.9580433 1.302417e-01  0.9386693
## 10 0.29989497 0.04830749  1.496657e-82 0.9685720 1.996499e-01  0.9386693
## 11 0.33305294 0.03842298  2.152394e-67 0.9765158 2.644051e-01  0.9386693
## 12 0.36369590 0.03277455  2.746327e-62 0.9820595 3.152078e-01  0.9386693
## 13 0.39427808 0.02816293  1.144748e-57 0.9862873 3.565280e-01  0.9386693
## 14 0.42563245 0.02374234  5.061804e-52 0.9895896 3.900507e-01  0.9386693
## 15 0.45755535 0.01803952  1.290005e-38 0.9921361 4.164858e-01  0.9386693
```

#### (6b) Plotting results:

```r
df_melt <- melt(df %>% select(dist,Morans_I,G_SIVAR,alpha_0.05), id = "dist")

ggplot(data = df_melt, aes(x = dist, y = value, colour = variable, linetype = variable)) +
  geom_line(aes(linetype=variable)) + geom_point() +
  scale_linetype_manual(values=c("solid","solid","dotted")) +
  scale_color_manual(values=c('black','blue','red')) +
  theme(legend.title = element_blank())
```

![](index_files/figure-html/unnamed-chunk-21-1.png)<!-- -->

