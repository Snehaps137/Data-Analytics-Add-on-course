---
title: "Descriptive Analysis of Research Data"
output: flexdashboard::flex_dashboard
runtime: shiny
---

```{r setup, include=FALSE}
library(flexdashboard)
library(shiny)
library(dplyr)
library(ggplot2)
#library(googleVis)
```

```{r global, include=FALSE}
# load data in 'global' chunk so it can be shared by all users of the dashboard
df=read.csv("Arya_data_RCF.csv",header = TRUE)
library(dplyr)
library(ggplot2)
source("http://pcwww.liv.ac.uk/~william/R/crosstab.r")
df$Agegroup <- cut(df$X2..Age.completed.years., breaks = c(-Inf,35,45,Inf), labels = c("Less than 35 years","35-45 years","More than 45 years"))
df$X1..Name.of.the.Tribe=as.factor(df$X1..Name.of.the.Tribe)
```

Column {.sidebar}
-----------------------------------------------------------------------

**Fix the choices**

```{r}
selectInput("cat_var", label = "Category:",
            choices = colnames(df), selected = "X16..APL.and.BPL.based.on.Color.of.ration.card")

selectInput("gr_var", label = "Grouped by:",
            choices = colnames(df), selected = "X1..Name.of.the.Tribe")
sliderInput("rs_adjust", label = "Sampling Limit:",
            min = 5, max = nrow(df), value = 10, step = 1)
```

Column{data-width=400}
-----------------------------------------------------------------------

**Percentage of respondents in the subsample with middle age category**

```{r}
renderGauge({
    invalidateLater(1000, session)
    dane <- round(mean(sample(df$X2..Age.completed.years.,input$rs_adjust))/60*100,2)
    df <- data.frame(Label = "IRR", Value = as.numeric(dane))
    gauge(dane, min = 0, max = 100, symbol = '%', gaugeSectors(
  success = c(80, 100), warning = c(40, 79), danger = c(0, 39)
))
  })
```


**EDA of `r renderText(input$cat_var)` over the `r renderText(input$gr_var)`.**

```{r,fig.width=20, fig.height=11}
datasel=reactive({df %>%
  count(.data[[input$cat_var]],.data[[input$gr_var]]) %>%       
  group_by_at(input$gr_var) %>%
  mutate(pct= prop.table(n) * 100)})
renderPlot({
  ggplot(datasel()) +aes(get(input$gr_var), pct, fill=get(input$cat_var)) +
  geom_bar(stat="identity") +
  ylab("percentage") +
  geom_text(aes(label=paste0(sprintf("%1.1f", pct),"%")),
            position=position_stack(vjust=0.5)) +labs(fill=input$cat_var)+
  theme_bw()
})
```

Column{data-width=400}
-----------------------------------------------------------------------


**Cross Tabulation**

```{r}
renderTable({
  datasel()
})
```

**Result of $\chi^2$ test**

```{r}
datachisquare <- reactive({
    req(input$cat_var, input$gr_var)

    df %>%
      {
        table(.[[input$cat_var]], .[[input$gr_var]])
      }
  })


  output$results <- renderPrint({

    print(chisq.test(datachisquare()))
  })
tableOutput("results")
```


>*Note:* if p-value is less than 0.05, the null hypothesis that  no significance difference over grouping variable`r renderText(input$gr_var)`  is rejected. So statistically the difference in `r renderText(input$cat_var)` over `r renderText(input$gr_var)` is significant at 5% level. Otherwise the null hypothesis is accepted.
