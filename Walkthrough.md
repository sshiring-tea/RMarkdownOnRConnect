# Programmatically Generating RMarkdown Reports on RConnect

### Introduction

This is a demonstration of using RConnect to programmatically generate dynamic RMarkdown
reports from "base" R, not from within a Shiny app. As only Shiny apps or RMarkdown
documents can be published on Rconnect, this is achieved by treating an RMarkdown document
as a driver script to knit child RMarkdown documents. Both the driver script and the child 
RMarkdown documents can be personalized by passing user-definable runtime variables. By
including variables that can be set at runtime, the RMarkdown document is technically a
*parameterized* RMarkdown document.

There are several advantages in using RConnect over JAMS: 
- Easier to configure and maintain
- Native browser-based interface
- Accepts user input
- Versioning support
- Output files accessible via HTTP

### Workflow

1. A driver script is embedded into a RMarkdown document
    - Additional RMarkdown document(s) serve as template(s) for the actual report
2. Declare any user-definable runtime variables as parameters in the yaml header 
    - Only these can be set via RConnect
    - Are avaiable from within params$
3. Any variables passed to final documents must likewise be declared within templates’ header
4. Must declare to rmarkdown all expected output files that will be produced
5. To publish RConnect, all dependency files (report templates, data files, etc.) must be included in the package

A trivial example, with explaination, follows. What this generator does is to use a
template to generate a number *n* of output RMarkdown documents, passing *n* to each 
output as a variable. The output files are programmatically and dynamically generated
by a for loop. It accepts 3 user-defined variables, 2 of which control the start/stop
behavior of the for loop and the final one which is passed along to the child documents.
Please note that the driver script itself is knitted and thus avaible as an RMarkdown 
document on RConnect. Note user-defined variables are accessed via params$

### Example and Commentary

#### make_reports.RMD

**yaml header**
``` r
---
title: "make_reports.RMD"
author: "Stephen Shiring"
output:
  slidy_presentation:
  font_adjustment: -1
params:
  start: 1
  stop: 3
  runvar: "TEA"
---
```

- "title", "author", and "output" are standard flags. 
- "params" are variables that can be set at run tume, with their default values. 
    - There is a sliding panel on RMarkdown that will allow the user to change these values and regenerate the document
    - "start", "stop" control the for loop and numbering of the output files
    - "runvar" is a variable that is passed to the output documents

**Body**

``` r
## Today's date: `r Sys.Date()`<br>

<b>runvar:</b> `r params$runvar`<br><br>

Generating reports from <b>`r params$start`</b> to <b>`r params$stop`</b>.
```

- Provides some generic text since the driver is itself knitted and avaiable.

``` r 
\```{r echo = FALSE, message = FALSE, warning = FALSE}

# Set template filename. 
rmd_file <- "report_template.RMD"

# Generate list of expected output files
lstOutputs <- list()
for (i in seq(params$start, params$stop)) {
  tmp <- paste(sprintf("report_%s.html", i))
  lstOutputs <- append(lstOutputs, tmp)
}

# Tell rmarkdown what output files to expect
rmarkdown::output_metadata$set(rsc_output_files = lstOutputs)
```
- RConnect or the RMarkdown rendered needs to know ahead of time how many output files to expect to be produced
    - The variables that sets this is "rsc_output_files"
    - The rmarkdown renderer variables can be set using rmarkdown::output_metadata$set()
- We give it a list containing all the output filenames

``` r
# Generate each report
for (i in seq(params$start,params$stop)) {
  # Set report parameters
  rmd_params = list(docName=paste0("test"), iteration=i, runvar=params$runvar)
  
  # Render using RMD file, parameters, and output filename. Render in a clean environment.
  rmarkdown::render(rmd_file, params = rmd_params, output_file=lstOutputs[[i]], envir = new.env())
}
```

We can pass any variables we want to the output documents as long as we define 
them to the RMarkdown renderer prior to rendering the document. The output file
(in this case the template file "report_template.RMD") also needs to know to expect
these variables, which is defined in its yaml header using the "params" field.

For each document, the variables are set using the params field; this accepts a named
list ("rmd_params"). When calling "render", we specify the template file, the parameters, 
the output filename, and declare a new environment to render in ("envir = new.env()", 
this is required in order to prevent parameters from bleeding over.)

### RConnect
The driver script and any template files and dependencies are then published to RConnect;
the templates and any data files must be published to the same directory. The user-defined
variables are accessible from an "INPUT" tab located to the left in the document's pane.
Note that the you must save the report after modifying the parameters and rerun the report
in order to see the updated results. The output files are available using from the path
set to the report; this can be customly defined or, alternatively, by using the URL
from the "Open Solo" window. e.g.: if the custome path is
https://connect.teainc.org/TestReports/ and and output filename is "report_1.html", 
then the final document can be accessed from 
https://connect.teainc.org/TestReports/report_1.html
