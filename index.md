---
layout: default
title: Stata Coding Guide
description: Julian Reif, University of Illinois
---

This guide describes how to put together a "push-button" publication-quality analysis in Stata. Following this guide will help:
1. Minimize coding errors during analysis
1. Automate the creation of tables and figures
1. Provide seamless integration with supporting *R* analyses
1. Produce a replication folder suitable for publication

As part of this guide, I created a comprehensive template that includes an example paper along with an accompanying replication package, available [here](https://github.com/reifjulian/my-project). Try it out and see how easy (or not!) it is for you to reproduce my example analysis. If you encounter problems let me know.


The rest of this guide explains the logic behind the organization of this template and provides instructions for how to set up a robust environment for your Stata projects.

<!-- Table of contents
1. toc1
{:toc}
-->


# Setting up your environment
-----------

I generally work on many projects at the same time and access them from different computers (laptop, home, work, etc.). A working project needs to be synced across my computers, and the analysis code must allow the project folder to have a different path on the different computers.  

## Dropbox

I use Dropbox to sync my projects across different computers. Dropbox has several appealing features. It creates backups of my projects across multiple computers and the Dropbox server, and in my experience has fewer bugs than alternatives such as Box. Dropbox makes it easy to share files with coauthors. Finally, all files inside of Dropbox have the same relative paths, which is helpful when writing scripts (more on this below).

## Stata profile

Stata automatically runs **profile.do** upon launch.

<img src="/assets/guide/stata_profile.PNG" width="100%" title="Stata profile">

**profile.do** must be stored in one of the paths searched by Stata. Type `adopath` at the Stata prompt to view a list of the paths for your particular computer.

Here is my Stata profile, stored in **C:/ado/personal/profile.do**:
```stata
* Settings specific to local environment
global DROPBOX "C:/Users/jreif/Dropbox"
global RSCRIPT_PATH "C:/Program Files/R/R-3.6.2/bin/x64/Rscript.exe"

* Run file containing settings common to all environments
run "$DROPBOX/stata_profile.do"
```

This file contains settings specific to my computer, such as the location of Dropbox and my *R* installation. I could define my project directories here. But instead, I store those in **$DROPBOX/stata_profile.do**, along with any other settings that are common across my computers:

```stata
set varabbrev off
global MyProject "$DROPBOX/my-project/MyProject"
```

In this example I have defined the location of only one project, `MyProject`. In practice I have a large number of globals defined here, one for every project I am working on. Whenever I start a new project, I define a new global for it and add it to **$DROPBOX/stata_profile.do**. Because all my computers are synced to Dropbox, I only have to do this once.

## *R* profile

I write more than 99% of my code Stata, including C++ plugins such as [strgroup](https://github.com/reifjulian/strgroup). On occasion, I will make use of an *R* function that is not available in Stata (e.g., [XGBoost](https://xgboost.readthedocs.io/en/latest/)). It is therefore convenient to setup an *R* environment that is consistent with the Stata environment.

*R* automatically runs **Rprofile.site** upon launch. On Windows, this file is located in the **C:/Program Files/R/R-n.n.n/etc** directory.  Alternatively you can store these settings in .Rprofile, which is run after Rprofile.site. Type `.libPaths()` at the R prompt to view a list of the paths for your particular computer.

```R
# Settings specific to local environment
Sys.setenv(DROPBOX = "C:/Users/jreif/Dropbox")

# Run file containing settings common to all environments
source(file.path(Sys.getenv("DROPBOX"), "R_profile.R"))
```

Store your general *R* settings on Dropbox. Store this file, **R_profile.R**, at the top level of your Dropbox folder. Define the paths for all your projects here. Here is an example that defines the location for one project, `MyProject`:

```R
Sys.setenv(MyProject = file.path(Sys.getenv("DROPBOX"), "my-project/MyProject"))
```

# Organizing the project
-----------

## Folder structure

An analysis starts with raw data (e.g., a dataset downloaded from the web). Scripts process these data, run analyses, and create tables.

```text
.
└── analysis
    └── data
        └── raw
    └── scripts
        ├── 0_run_all.do
        └── 1_...
```

The master script, `0_run_all.do`, executes the entire analysis. Running this script creates all necessary additional folders, intermediate files, and results:

```text
.
└── analysis
    └── data
        ├── proc
        └── raw
    └── results
        ├── figures
        └── tables
    └── scripts
        ├── 0_run_all.do
        └── 1_...
```

At any time, you can delete all these extra folders, keeping only `data/raw` and `scripts`, and then rerun your analysis from scratch. When the project is complete, a copy of `analysis` serves as a standalone replication package.

The analysis folder contains three subfolders. `scripts` stores all scripts and libraries required to run the analysis. `data` includes raw and processed data. `data/raw` is read-only. Scripts write only to `data/proc` or `results`

`results` contains all final output, including tables and figures. These can be linked to a LaTeX document on Overleaf or stored in an adjacent folder:


```text
.
└── analysis
    └── data
        ├── proc
        └── raw
    └── results
        ├── figures
        └── tables
    └── scripts
        ├── 0_run_all.do
        └── 1_...
└── paper
    ├── manuscript.tex
    ├── figures
    └── tables
```

When you are ready to update the paper, copy `analysis/results/figures` and `analysis/results/tables` to `paper`. `paper` contains manuscript files. Additional documents such as literature references can be stored there or in a separate folder at the top of project directory.

## Functions

Functions are pieces of code that are called repeatedly by your scripts. In Stata these are are called ADO-files. An introduction is available [here](https://blog.stata.com/2015/11/10/programming-an-estimation-command-in-stata-a-first-ado-command). Because these subroutines are not called directly by the master script, `0_run_all.do`, they should be stored in the subdirectory `scripts/functions`.

## Libraries

My code frequently employs user-written Stata commands, such as [regsave](https://github.com/reifjulian/regsave) or [reghdfe](http://scorreia.com/software/reghdfe/install.html). To ensure replication, it is **very important** to include copies of these programs with your code:
1. Unless a user has a local copy of the program, she won't be able to run your code if you don't supply this program.
1. These commands are updated over time and newer versions may not work with older code implementations.

Many people do not appreciate how code updates can inhibit replication. Here is an example. You perform a Stata analysis using a new, user-written estimation command called, say, `regols`. You publish your paper, along with your replication code, but do not include the code for `regols`. 10 years later a researcher tries to replicate your analysis. The code breaks because she has not installed `regols`. She opens Stata and type `ssc install regols`, which installs the newest version of that command. But, in the intervening 10 years the author of `regols` fixed a bug in how the standard errors are calculated. When the researcher runs your code she finds your estimates are no longer significant. Is this because you included the wrong dataset with your replication, because there is mistake in the analysis code, or because you failed to correctly copy/paste your output into your publication? The researcher does not know. She cannot replicate your published results and must now decide what to do.

When I start a new project, I include a script called `_install_stata_packages.do` that installs a copy of all required add-ons into a subdirectory of the project folder. Rerunning this script will install updated versions of these add-on's (if available). I delete the script when my project is ready to be published, which locks down the code for these packages and ensures I can replicate my analysis forever.

```text
.
└── analysis
    └── data
    	└── raw
    └── scripts
        ├── functions
        └── libraries
    	    └── stata
```

In theory, one can also install copies of add-on packages for *R* into `scripts/libraries`. In practice, I run into difficulties. Standard add-ons such as `tidyverse` take up hundreds of megabytes of space. Duplicating these large files for every new project is unappealing. Installing *R* libraries locally also frequently generates installation errors, or result in only partial installations. Packages such as [packrat](https://rstudio.github.io/packrat/) may provide better solutions. In my example, I include a script called `_install_R_packages.R` that installs these packages for the user. This solution requires an internet connection, and is vulnerable to the two replication concerns mentioned above.


# Publishing your code
-----------

Follow these steps before publishing your code to ensure replication.

1. Add a README file to the `analysis` folder. It should include the following information:
  1. Title and authors of the paper
  1. Required software, including version numbers
  1. **Clear** instructions for how to run the analysis. If the analysis cannot be run--because the data are proprietary, for example--this should be noted.
  1. Description of whether the output is stored

1. Remove `_install_stata_packages.do` from the `scripts` folder.

1. Disable all locally installed Stata programs not located in your Stata folder. (This will ensure that your analysis is actually using programs installed in your project subdirectory, rather than somewhere else on your machine.) On Windows, this can usually be done by renaming `c:/ado` to `c:/_ado`. You can test whether you succeeded as follows. Suppose you have a copy of `regsave` somewhere on your machine and also in your local project directory. Open up a new instance of Stata and type `which regsave`. Stata should report "command regsave not found". If not, Stata will tell you where the command is located, and you can then rename that folder by adding an underscore.

1. Delete the `/data/proc` and `/results` folders.

1. Run `0_run_all.do`, which should rerun the entire analysis and regenerate all tables and figures.

1. Copy `/results/figures` and `/results/tables` to the `paper` folder.

1. Recompile the paper and check the numbers.

1. Rename the `analysis` folder to something more descriptive, and zip it.

Checking numbers can be difficult and tedious. Include lots of asserts in your code when writing up your results to reduce errors. (See an example of an `assert` in `4_make_tables_figures.do`.)

# Good coding practice
-----------

Use forward slashes for pathnames (`$DROPBOX/project` not `$DROPBOX\project`). Backslashes are an escape character in Stata and can cause issues depending on what operating system you are running. Use forward slashes to ensure cross-platform compatibility.

Never use hard-coded paths like `C:/Users/jreif/Dropbox/MyProject`. All pathnames should reference a global variable defined in your Stata profile. I should be able to run your entire analysis from my personal computer without having to edit any of your scripts.

Include `set varabbrev off` in your Stata profile.  Most professional Stata programmers I know do this in order to avoid unexpected behaviors such as [this](https://www.ifs.org.uk/docs/stata_gotchasJan2014.pdf).

Sometimes an analysis will produce different results each time you run it. Here are two common reasons why this happens:
1. One of your commands requires random numbers and you forgot to use `set seed #`
1. You have a nonunique sort. Add `isid` checks to your code prior to sorting to ensure uniqueness. (Another option is to add the `unique` option to your sorts.) Nonunique sorts can be hard to predict:

```stata
* The random variable r here is not unique, because Stata's default type (float) does not have enough precision when N=100,000. (isid will generate an error, unless you have changed Stata's default type to double)
clear
set seed 100
set obs 100000
gen r = uniform()
isid r

* Cast r as a double to avoid this problem. (isid no longer generates an error)
clear
set seed 100
set obs 100000
gen double r = uniform()
isid r
```


# Other helpful links
-----------

[Dan Sullivan's best practices for coding](http://www.danielmsullivan.com/pages/tutorial_workflow_3bestpractice.html)

[Gentzkow and Shapiro coding guide](https://web.stanford.edu/~gentzkow/research/CodeAndData.pdf)

[Grant McDermott's data science lectures](https://github.com/uo-ec607/lectures)

[Roger Koenker's guide on reproducibility](http://www.econ.uiuc.edu/~roger/research/repro)




# Acknowledgments
-----------

The coding practices outlined in this guide have been developed and improved over many years. I would especially like to thank my frequent collaborators Tatyana Deryugina and David Molitor for providing many helpful suggestions that have improved my project organization over the years. I also thank Grant McDermott for helpful conversations regarding version control and reproducibility in *R*.
