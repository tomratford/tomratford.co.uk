---
title: "Porting SAS formats"
author: "Tom Ratford"
date: "2023-01-02"
output: hugodown::hugo_document
rmd_hash: 314948b083378332

---

As part of my fun winter break activities I decided to write a simple package porting SAS-like formats to help practice my "test driven development" as well as continue to improve my R skills. Whilst creating it I ended up going down two different rabbit holes which I hope to cover in this post. The package can be found and installed from github at [tomratford/SASformatR](https://github.com/tomratford/SASformatR/).

# A whistle-stop tour of the package

The package tries to port the idea of how SAS formats are created. There is very simple functionality for ranges of numeric values and all the standard `enum -> enum` formats.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='kr'><a href='https://rdrr.io/r/base/library.html'>library</a></span><span class='o'>(</span><span class='nv'>SASformatR</span><span class='o'>)</span></span>
<span></span>
<span><span class='nf'><a href='https://rdrr.io/pkg/SASformatR/man/proc_format.html'>proc_format</a></span><span class='o'>(</span></span>
<span>  PARAMCD <span class='o'>=</span> <span class='nf'><a href='https://rdrr.io/pkg/SASformatR/man/value.html'>value</a></span><span class='o'>(</span></span>
<span>    <span class='s'>"SYSBP"</span> <span class='o'>=</span> <span class='s'>"Systolic Blood Pressure"</span>,</span>
<span>    <span class='s'>"HR"</span> <span class='o'>=</span> <span class='s'>"Heart Rate"</span></span>
<span>  <span class='o'>)</span>,</span>
<span>  AGEGR1 <span class='o'>=</span> <span class='nf'><a href='https://rdrr.io/pkg/SASformatR/man/value.html'>value</a></span><span class='o'>(</span></span>
<span>    <span class='s'>"&lt;21"</span> <span class='o'>=</span> <span class='s'>"&lt;21"</span>,</span>
<span>    <span class='s'>"21 - 39"</span> <span class='o'>=</span> <span class='s'>"21 - 39"</span>,</span>
<span>    <span class='s'>"40&lt;="</span> <span class='o'>=</span> <span class='s'>"&gt;=40"</span></span>
<span>  <span class='o'>)</span></span>
<span><span class='o'>)</span></span></code></pre>

</div>

We can then use [`put()`](https://rdrr.io/pkg/SASformatR/man/put.html) to format these values straight to character, or use [`SASformat()`](https://rdrr.io/pkg/SASformatR/man/SASformat.html) to create a new S3 `SASformat` object.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='nv'>age</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/r/base/seq.html'>seq</a></span><span class='o'>(</span><span class='m'>10</span>,<span class='m'>50</span>,<span class='m'>10</span><span class='o'>)</span></span>
<span></span>
<span><span class='nf'><a href='https://rdrr.io/pkg/SASformatR/man/put.html'>put</a></span><span class='o'>(</span><span class='nv'>age</span>,<span class='s'>"AGEGR1"</span><span class='o'>)</span></span>
<span><span class='c'>#&gt; [1] "&lt;21"     "&lt;21"     "21 - 39" "&gt;=40"    "&gt;=40"</span></span>
<span></span><span></span>
<span><span class='nf'>dplyr</span><span class='nf'>::</span><span class='nf'><a href='https://dplyr.tidyverse.org/reference/bind.html'>bind_rows</a></span><span class='o'>(</span>age <span class='o'>=</span> <span class='nv'>age</span>, agegr1 <span class='o'>=</span> <span class='nf'><a href='https://rdrr.io/pkg/SASformatR/man/SASformat.html'>SASformat</a></span><span class='o'>(</span><span class='nv'>age</span>,<span class='s'>"AGEGR1"</span><span class='o'>)</span><span class='o'>)</span></span>
<span><span class='c'>#&gt; <span style='color: #555555;'># A tibble: 5 × 2</span></span></span>
<span><span class='c'>#&gt;     age agegr1    </span></span>
<span><span class='c'>#&gt;   <span style='color: #555555; font-style: italic;'>&lt;dbl&gt;</span> <span style='color: #555555; font-style: italic;'>&lt;SASformt&gt;</span></span></span>
<span><span class='c'>#&gt; <span style='color: #555555;'>1</span>    10 &lt;21       </span></span>
<span><span class='c'>#&gt; <span style='color: #555555;'>2</span>    20 &lt;21       </span></span>
<span><span class='c'>#&gt; <span style='color: #555555;'>3</span>    30 21 - 39   </span></span>
<span><span class='c'>#&gt; <span style='color: #555555;'>4</span>    40 &gt;=40      </span></span>
<span><span class='c'>#&gt; <span style='color: #555555;'>5</span>    50 &gt;=40</span></span>
<span></span></code></pre>

</div>

The `SASformat` object has the same underlying type, so regular arithmetic can be used.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='nv'>agegr1</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/pkg/SASformatR/man/SASformat.html'>SASformat</a></span><span class='o'>(</span><span class='nv'>age</span>, <span class='s'>"AGEGR1"</span><span class='o'>)</span></span>
<span></span>
<span><span class='nf'><a href='https://rdrr.io/r/base/class.html'>class</a></span><span class='o'>(</span><span class='nv'>agegr1</span><span class='o'>)</span></span>
<span><span class='c'>#&gt; [1] "SASformat"</span></span>
<span></span><span></span>
<span><span class='nf'><a href='https://rdrr.io/r/base/typeof.html'>typeof</a></span><span class='o'>(</span><span class='nv'>agegr1</span><span class='o'>)</span></span>
<span><span class='c'>#&gt; [1] "double"</span></span>
<span></span><span></span>
<span><span class='nv'>agegr1</span></span>
<span><span class='c'>#&gt; [1] "&lt;21"     "&lt;21"     "21 - 39" "&gt;=40"    "&gt;=40"</span></span>
<span></span><span><span class='nv'>agegr1</span> <span class='o'>+</span> <span class='m'>10</span></span>
<span><span class='c'>#&gt; [1] "&lt;21"     "21 - 39" "&gt;=40"    "&gt;=40"    "&gt;=40"</span></span>
<span></span></code></pre>

</div>

The key purpose was to try to mimic (as much as possible) SAS formats. [`proc_format()`](https://rdrr.io/pkg/SASformatR/man/proc_format.html) does not return anything, it instead creates a format catalogue as a side effect. Another key thing to note is that this catalog is not stored in the environment it is called in.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='nv'>my_new_env</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/r/base/environment.html'>new.env</a></span><span class='o'>(</span><span class='o'>)</span></span>
<span><span class='nf'><a href='https://rdrr.io/r/base/with.html'>with</a></span><span class='o'>(</span><span class='nv'>my_new_env</span>,</span>
<span>     <span class='o'>&#123;</span></span>
<span>       <span class='nv'>i_am_here</span> <span class='o'>&lt;-</span> <span class='m'>1</span></span>
<span>       <span class='nf'><a href='https://rdrr.io/pkg/SASformatR/man/proc_format.html'>proc_format</a></span><span class='o'>(</span><span class='o'>)</span></span>
<span>     <span class='o'>&#125;</span><span class='o'>)</span></span>
<span><span class='nf'><a href='https://rdrr.io/r/base/ls.html'>ls</a></span><span class='o'>(</span>envir <span class='o'>=</span> <span class='nv'>my_new_env</span><span class='o'>)</span></span>
<span><span class='c'>#&gt; [1] "i_am_here"</span></span>
<span></span></code></pre>

</div>

# Using enviroments as mutable package data

R has the ability to export data in a package. Typically this is used for exporting data frames for use in examples or vignettes. These are immutable structures, for example, if we try to assign using the `::` operator we will get an error back.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='nf'>dplyr</span><span class='nf'>::</span><span class='nv'><a href='https://dplyr.tidyverse.org/reference/starwars.html'>starwars</a></span> <span class='o'>&lt;-</span> <span class='nv'>mtcars</span></span>
<span><span class='c'>#&gt; Error in dplyr::starwars &lt;- mtcars: object 'dplyr' not found</span></span>
<span></span></code></pre>

</div>

Using the more involved `assign` will also not work, but will give us a more useful error message.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='nf'><a href='https://rdrr.io/r/base/assign.html'>assign</a></span><span class='o'>(</span><span class='s'>"starwars"</span>, <span class='kc'>NULL</span>, envir <span class='o'>=</span> <span class='nf'><a href='https://rdrr.io/r/base/as.environment.html'>as.environment</a></span><span class='o'>(</span><span class='s'>"package:dplyr"</span><span class='o'>)</span><span class='o'>)</span></span>
<span><span class='c'>#&gt; Error in assign("starwars", NULL, envir = as.environment("package:dplyr")): cannot change value of locked binding for 'starwars'</span></span>
<span></span></code></pre>

</div>

However, the exported 'data' can be of any type. In `SASformatR` I export a empty environment, this is where the format catalogues will live.

{{< code language="R" title="<a href=\"https://github.com/tomratford/SASformatR/blob/main/data-raw/ctls.R\">data-raw/ctls.R</a>">}}
## code to prepare `ctl` dataset goes here

ctls <- new.env(parent = emptyenv())

usethis::use_data(ctls, overwrite = TRUE)
{{< /code >}}

This can then be accessed from other functions within the package using `SASformat::ctls`. For example, within the function [`proc_format()`](https://rdrr.io/pkg/SASformatR/man/proc_format.html).

{{< code language="R" title="<a href=\"https://github.com/tomratford/SASformatR/blob/main/R/proc_format.R\">R/proc_format.R</a>">}}
proc_format <- function(..., catalog = "formats") {
  partial <- SASformatR::ctls[[catalog]]
  if (is.null(SASformatR::ctls[[catalog]])) {
    partial <- list()
  }

  newlist <- c(list2(...), partial)

  assign(catalog,
         newlist[!duplicated(names(newlist))],
         envir = SASformatR::ctls)
}
{{< /code >}}

Firstly we get the current values within the catalog, and then combine these with the new values provided to the function. The key code here is the last [`assign()`](https://rdrr.io/r/base/assign.html), where I assign this new list of formats to the `SASformatR::ctls` environment with the name of the desired catalog - `"formats"` by default.

This allows formats to remain hidden from the main environment, which means it is perfectly possible to reuse names without fear of conflicts. Equally having a consistent area for formats means that we do not need to worry about load order or name conflicts within other packages, as we can explicitly specify the location.

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='nf'><a href='https://rdrr.io/pkg/SASformatR/man/proc_format.html'>proc_format</a></span><span class='o'>(</span>catalog <span class='o'>=</span> <span class='s'>"EMA"</span>,</span>
<span>  agegr1 <span class='o'>=</span> <span class='nf'><a href='https://rdrr.io/pkg/SASformatR/man/value.html'>value</a></span><span class='o'>(</span></span>
<span>    <span class='s'>"&lt;21"</span> <span class='o'>=</span> <span class='s'>"&lt; 21"</span>,</span>
<span>    <span class='s'>"21 - 39"</span> <span class='o'>=</span> <span class='s'>"21 - 39"</span>,</span>
<span>    <span class='s'>"40&lt;="</span> <span class='o'>=</span> <span class='s'>"&gt;= 40"</span></span>
<span>  <span class='o'>)</span></span>
<span><span class='o'>)</span></span>
<span><span class='nf'><a href='https://rdrr.io/pkg/SASformatR/man/proc_format.html'>proc_format</a></span><span class='o'>(</span>catalog <span class='o'>=</span> <span class='s'>"FDA"</span>,</span>
<span>  agegr1 <span class='o'>=</span> <span class='nf'><a href='https://rdrr.io/pkg/SASformatR/man/value.html'>value</a></span><span class='o'>(</span></span>
<span>    <span class='s'>"&lt;18"</span> <span class='o'>=</span> <span class='s'>"&lt; 18"</span>,</span>
<span>    <span class='s'>"18 - 39"</span> <span class='o'>=</span> <span class='s'>"18 - 39"</span>,</span>
<span>    <span class='s'>"40 - 79"</span> <span class='o'>=</span> <span class='s'>"40 - 79"</span>,</span>
<span>    <span class='s'>"80&lt;="</span> <span class='o'>=</span> <span class='s'>"&gt;= 80"</span></span>
<span>  <span class='o'>)</span></span>
<span><span class='o'>)</span></span>
<span></span>
<span><span class='nv'>EMA</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/pkg/SASformatR/man/SASformat.html'>SASformat</a></span><span class='o'>(</span><span class='nv'>age</span>, <span class='s'>"agegr1"</span>, <span class='s'>"EMA"</span><span class='o'>)</span></span>
<span><span class='nv'>FDA</span> <span class='o'>&lt;-</span> <span class='nf'><a href='https://rdrr.io/pkg/SASformatR/man/SASformat.html'>SASformat</a></span><span class='o'>(</span><span class='nv'>age</span>, <span class='s'>"agegr1"</span>, <span class='s'>"FDA"</span><span class='o'>)</span></span>
<span><span class='nv'>agegr1</span> <span class='o'>&lt;-</span> <span class='nf'>dplyr</span><span class='nf'>::</span><span class='nf'><a href='https://dplyr.tidyverse.org/reference/bind.html'>bind_cols</a></span><span class='o'>(</span>AGEGR1 <span class='o'>=</span> <span class='nv'>EMA</span>, AGEGR2 <span class='o'>=</span> <span class='nv'>FDA</span><span class='o'>)</span></span>
<span><span class='nv'>agegr1</span></span>
<span><span class='c'>#&gt; <span style='color: #555555;'># A tibble: 5 × 2</span></span></span>
<span><span class='c'>#&gt;   AGEGR1     AGEGR2    </span></span>
<span><span class='c'>#&gt;   <span style='color: #555555; font-style: italic;'>&lt;SASformt&gt;</span> <span style='color: #555555; font-style: italic;'>&lt;SASformt&gt;</span></span></span>
<span><span class='c'>#&gt; <span style='color: #555555;'>1</span> &lt; 21       &lt; 18      </span></span>
<span><span class='c'>#&gt; <span style='color: #555555;'>2</span> &lt; 21       18 - 39   </span></span>
<span><span class='c'>#&gt; <span style='color: #555555;'>3</span> 21 - 39    18 - 39   </span></span>
<span><span class='c'>#&gt; <span style='color: #555555;'>4</span> &gt;= 40      40 - 79   </span></span>
<span><span class='c'>#&gt; <span style='color: #555555;'>5</span> &gt;= 40      40 - 79</span></span>
<span></span></code></pre>

</div>

> Aside: These are **not** the standard EMA and FDA age groupings but they serve the purpose of an real-world example

# Custom rendering of S3 classes within a `View()` command

Another thing which I desperately wanted to have was to be able to see the formatted values when using the [`View()`](https://rdrr.io/r/utils/View.html) command. This is actually very simple (but it was quite annoying to find out). Your S3 object needs a method for [`format()`](https://rdrr.io/r/base/format.html) and `[` (I imagine this will also work for S4 objects but I haven't tested this). For those unaware, <a href="https://rdrr.io/r/base/Extract.html"><code>\`\[\`()</code></a> is the function that is called when you do something like `x[1]`, you can actually call it like <code>\`\[\`(x,1)</code>. Here is the implementations within `SASformatR`.

{{< code language="R" title="<a href=\"https://github.com/tomratford/SASformatR/blob/main/R/extract.R\">R/extract.R</a>">}}
`[.SASformat` <- function(x, ...) {
  structure(NextMethod(),
            class = class(x),
            format = attr(x, "format"),
            catalog = attr(x, "catalog"))
}
{{< /code >}}

As we are not changing the values underneath, we can just call [`NextMethod()`](https://rdrr.io/r/base/UseMethod.html) to get the base implementation for vectors. We can then reapply the attributes of our class using the [`structure()`](https://rdrr.io/r/base/structure.html) command.

{{< code language="R" title="<a href=\"https://github.com/tomratford/SASformatR/blob/main/R/format.R\">R/format.R</a>">}}
format.SASformat <- function(x, ...) {
  check_sasformat(x) # in case of changes to the catalog
  put(unclass(x),
      attr(x,"format"),
      attr(x,"catalog"))
}
{{< /code >}}

This uses the [`put()`](https://rdrr.io/pkg/SASformatR/man/put.html) command that was created before.

With these two methods created we can now view our rendered values within Rstudio's [`View()`](https://rdrr.io/r/utils/View.html) function, for example

<div class="highlight">

<pre class='chroma'><code class='language-r' data-lang='r'><span><span class='nv'>my_df</span> <span class='o'>&lt;-</span> <span class='nf'>dplyr</span><span class='nf'>::</span><span class='nf'><a href='https://dplyr.tidyverse.org/reference/bind.html'>bind_rows</a></span><span class='o'>(</span>age <span class='o'>=</span> <span class='nv'>age</span>, agegr1 <span class='o'>=</span> <span class='nf'><a href='https://rdrr.io/pkg/SASformatR/man/SASformat.html'>SASformat</a></span><span class='o'>(</span><span class='nv'>age</span>,<span class='s'>"AGEGR1"</span><span class='o'>)</span><span class='o'>)</span></span>
<span><span class='nf'><a href='https://rdrr.io/r/utils/View.html'>View</a></span><span class='o'>(</span><span class='nv'>my_df</span><span class='o'>)</span></span></code></pre>

</div>

{{< image src="/porting-sas-formats/view.png" alt="A dataset of two columns: age, and the formatted agegr1." position="center">}}

