# statInsightR

Easy statistical insights for R: simple functions to make statistical analysis easier.

`statInsightR` takes a fitted linear regression model (`lm()`) and turns it into
plain-language output. It can write a readable summary of the coefficients, run
the usual assumption checks, write a short conclusion, and produce a full HTML
report, each with one function call.

## Features

| Function | What it does | Returns |
|---|---|---|
| `summary_clean(model)` | Prints a plain-language description of each coefficient: direction, size, significance and 95% confidence interval | A data frame of coefficient details (invisibly) |
| `check_model(model)` | Runs diagnostic checks: multicollinearity, normality of residuals, heteroskedasticity, influential points and leverage | A list of diagnostic results (invisibly) |
| `auto_conclusion(model)` | Writes a short conclusion covering significant predictors, model fit and diagnostics | A character string |
| `report_model(model, file)` | Renders all of the above, plus diagnostic plots, into an HTML report | The path to the HTML file |

## Installation

`statInsightR` is not on CRAN. Install the development version from GitHub:

```r
# install.packages("remotes")
remotes::install_github("yudha17-dev/statInsightR")
```

### Requirements

- R 3.5 or newer
- Package dependencies (installed automatically): `car`, `lmtest`, `rmarkdown`, `kableExtra`
- [Pandoc](https://pandoc.org/), used by `report_model()` to render HTML. RStudio
  ships with Pandoc, so if you use RStudio it is already installed.

## Quick start

```r
library(statInsightR)

# Fit an ordinary linear regression
model <- lm(mpg ~ wt + hp, data = mtcars)

summary_clean(model)          # coefficients in plain language
check_model(model)            # assumption checks
cat(auto_conclusion(model))   # written conclusion
report_model(model)           # full HTML report
```

## Usage

All functions take a model fitted with `lm()`. Other model types, such as
`glm()` or mixed models, are not supported yet.

### 1. Coefficient summary with `summary_clean()`

```r
model <- lm(mpg ~ wt + hp, data = mtcars)
coefs <- summary_clean(model)
```

This prints:

- The share of variance in the response that the model explains (R² and adjusted R²).
- One entry for each predictor (the intercept is skipped). Each entry says how a
  one-unit increase changes the response on average, holding the other variables
  constant, and includes:
  - An effect-size label based on the absolute estimate: very small (< 0.1),
    small (< 0.5), moderate (< 1) or strong (≥ 1). The label depends on the scale
    of your variables, so treat it as a rough guide.
  - The t value, standard error and 95% confidence interval.
  - A significance statement: highly significant (p < 0.001), significant
    (p < 0.05) or not significant.

The function also returns a data frame invisibly, so you can store the results and
work with them:

```r
coefs
#> columns: Variable, Estimate, Std_Error, t_value, p_value, CI_low, CI_high

subset(coefs, p_value < 0.05)
```

### 2. Diagnostic checks with `check_model()`

```r
diag <- check_model(model)
```

The function runs these checks and prints a short explanation and suggestion
for each problem it finds:

| Check | Method | Flagged when |
|---|---|---|
| Multicollinearity | Variance inflation factor (`car::vif`) | Any VIF > 10 |
| Normality of residuals | Shapiro–Wilk test | p < 0.05 |
| Heteroskedasticity | Breusch–Pagan test (`lmtest::bptest`) | p < 0.05 |
| Influential observations | Cook's distance | Distance > 4 / n |
| High leverage | Hat values | Hat value > 2p / n |

The VIF check is skipped, with the reason printed, when the model has fewer than
two predictors, has aliased (perfectly collinear) coefficients, or VIF can't be
computed.

The returned list contains:

```r
names(diag)
#> "model" "warnings" "vif" "shapiro" "bptest" "cooks_distance" "hat_values"

diag$warnings          # only the checks that flagged a problem
diag$shapiro$p.value   # the full test objects are kept
which(diag$cooks_distance > 4 / nrow(mtcars))
```

To look at the influential points it reports, plot Cook's distance:

```r
plot(model, which = 4)
```

### 3. Written conclusion with `auto_conclusion()`

```r
conclusion <- auto_conclusion(model)
cat(conclusion)
```

The conclusion covers:

1. Which predictors are significant at the 5% level, with the direction and size
   of each effect, and which predictors are not.
2. Overall model fit, described from R² as weak (< 0.2), moderate (< 0.5),
   strong (< 0.8) or very strong.
3. Whether any predictor shows multicollinearity (VIF > 10).
4. Whether the residuals are normally distributed (Shapiro–Wilk).
5. Whether there is heteroskedasticity (Breusch–Pagan).
6. Which observations are influential (Cook's distance > 4 / n).

The text is formatted as Markdown (for example, **bold** predictor names), so you
can paste it straight into an R Markdown or Quarto document. In a document chunk,
use `results = "asis"` so the formatting renders:

````markdown
```{r, results = "asis"}
cat(statInsightR::auto_conclusion(model))
```
````

### 4. HTML report with `report_model()`

```r
report_model(model)                             # writes model_report.html
report_model(model, file = "mpg_model.html")    # choose the file name
```

The report is always saved in the current working directory (`getwd()`), and
only the file name part of `file` is used. The function returns the full path of
the saved file.

The report includes:

- **Overview**: the model formula
- **Clean summary**: the output of `summary_clean()`
- **Statistical diagnostics**: tables for VIF, the Shapiro–Wilk test, the
  Breusch–Pagan test and Cook's distance
- **Diagnostic plots**: residuals vs fitted, a normal Q–Q plot and a Cook's
  distance plot
- **Conclusion**: the output of `auto_conclusion()`

## Example workflow

```r
library(statInsightR)

# 1. Fit the model
model <- lm(Sepal.Length ~ Sepal.Width + Petal.Length + Petal.Width, data = iris)

# 2. Read the coefficients
summary_clean(model)

# 3. Check the assumptions
diag <- check_model(model)

# 4. If there are problems, adjust the model (for example, transform the response)
if ("normality" %in% names(diag$warnings)) {
  model <- lm(log(Sepal.Length) ~ Sepal.Width + Petal.Length + Petal.Width, data = iris)
}

# 5. Write up the result
cat(auto_conclusion(model))
report_model(model, file = "iris_report.html")
```

## Limitations

- Only `lm()` models are supported. `summary_clean()`, `check_model()` and
  `report_model()` stop with an error for other model types.
- All checks use a fixed 5% significance level.
- R's `shapiro.test()` accepts between 3 and 5000 residuals. For larger data sets,
  `check_model()` stops with an error, and `auto_conclusion()` says that the
  normality test could not be computed.
- The thresholds (VIF > 10, Cook's distance > 4 / n, leverage > 2p / n) are
  common rules of thumb, not strict cutoffs. Use your own judgement about your data.

## License

MIT. See [LICENSE](LICENSE) and [LICENSE.md](LICENSE.md).
