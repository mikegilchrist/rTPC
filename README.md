
<!-- README.md is generated from README.Rmd. Please edit that file -->

# rTPC

The aim of rTPC is to help fit thermal performance in R. These functions
act as wrappers that aid in the fitting of common functions for fitting
thermal performance curves.

## Installation

``` r
# install package from GitHub
devtools::install_github("padpadpadpad/rTPC")
```

To show each function, we shall fit the models using nls.multstart. To
do we shall install other packages that will help us.

``` r
# install nls.multstart
install.packages('nls.multstart')
install.packages('purrr')
install.packages('dplyr')
install.packages('tidyr')
install.packages('ggplot2')
install.packages('nls.multstart')
install.packages('broom')
```

We can now load in the data and the packages to fit the TPCs

``` r
# load in packages
library(purrr)
library(dplyr)
#> 
#> Attaching package: 'dplyr'
#> The following objects are masked from 'package:stats':
#> 
#>     filter, lag
#> The following objects are masked from 'package:base':
#> 
#>     intersect, setdiff, setequal, union
library(tidyr)
library(ggplot2)
library(nls.multstart)
library(broom)
library(rTPC)

# write function to label ggplot2 panels
label_facets <- function(string){
  len <- length(string)
  string = paste('(', letters[1:len], ') ', string, sep = '')
  return(string)
}

# load in data
data("Chlorella_TRC")

# change rate to be non-log transformed
d <- mutate(Chlorella_TRC, rate = exp(ln.rate))

# filter for just the first curve
d_1 <- filter(d, curve_id == 1)

# plot
ggplot(d_1, aes(temp, rate)) +
  geom_point()
```

<img src="man/figures/README-quicklook-1.png" width="100%" />

We can now run nls.multstart for each function

``` r
# run in purrr - going to be a huge long command this one
d_models <- group_by(d_1, curve_id, growth.temp, process, flux) %>%
  nest() %>%
  mutate(., lactin2 = map(data, ~nls_multstart(rate ~ lactin2_1995(temp = temp, p, c, tmax, tref),
                       data = .x,
                       iter = 500,
                       start_lower = c(p = 0, c = -2, tmax = 35, tref = 0),
                       start_upper = c(p = 3, c = 0, tmax = 55, tref = 15),
                       supp_errors = 'Y')),
                       sharpeschoolhigh = map(data, ~nls_multstart(rate ~ sharpeschoolhigh_1981(temp_k = K, b_tref, e, eh, th, tref = 15),
                                                     data = .x,
                                                     iter = 500,
                                                     start_lower = c(c = 0.01, e = 0, eh = 0, th = 270),
                                                     start_upper = c(c = 2, e = 3, eh = 10, th = 330),
                                                     supp_errors = 'Y')))
```

We can now make predictions of each model and plot them.

``` r
# stack models
d_stack <- gather(d_models, 'model', 'output', 6:ncol(d_models))

# preds
newdata <- tibble(temp = seq(min(d_1$temp), max(d_1$temp), length.out = 100),
                  K = seq(min(d_1$K), max(d_1$K), length.out = 100))
d_preds <- d_stack %>%
  unnest(., output %>% map(augment, newdata = newdata)) %>%
  mutate(., temp = ifelse(model == 'sharpeschoolhigh', K - 273.15, temp))

# plot
ggplot(d_preds, aes(temp, rate)) +
  geom_point(aes(temp, rate), d_1) +
  geom_line(aes(temp, .fitted, col = model)) +
  facet_wrap(~model, labeller = labeller(model = label_facets)) +
  theme_bw() +
  theme(legend.position = 'none',
        strip.text = element_text(hjust = 0),
        strip.background = element_blank())
```

<img src="man/figures/README-plot predictions-1.png" width="100%" />
