# statInsightR

Easy statistical insights for R: simple functions to make statistical analysis easier.

## Installation

```r
# install.packages("remotes")
remotes::install_github("yudha17-dev/statInsightR")
```

## Usage

```r
library(statInsightR)

model <- lm(mpg ~ wt + hp, data = mtcars)

summary_clean(model)    # tidy model summary
check_model(model)      # assumption checks
auto_conclusion(model)  # plain-language conclusion
report_model(model)     # HTML report (model_report.html)
```

## License

MIT
