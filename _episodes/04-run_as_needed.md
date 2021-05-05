---
title: "How snakemake chooses what jobs to run"
teaching: 0
exercises: 0
questions:
- "How do I visualise a Snakemake workflow?"
- "How does Snakemake avoid unecessary work?"
- "How do I control what steps will be run?"
objectives:
- "View the DAG for our pipeline"
- "Understand the logic Snakemake uses when running and re-running rules"
keypoints:
- "Snakemake is awesome"
---

(First, can I run Kallisto on the filtered reads because that makes the demo here easier? If not
I can introduce my re-pair script tho. OK, yes it works. Or at least it doesn't give an error and
the output looks like output, so I'll take it.)

Maybe I should introduce the DAG concept first? The concept of dependencies makes more sense then.
Let's do a brief DAG intro.

## The DAG

You may have noticed that one of the messages Snakemake always prints is:

~~~
Building DAG of jobs...
~~~

A DAG is a Directed Acyclic Graph and it can easily be pictured like so:

>>> Insert hot DAG pic here <<<

The above DAG is based on our existing rules, and shows all the jobs Snakemake would run to trim, count and
quantify the "ref1" sample. Note that:

* A rule can appear more than once, with different wildcards (a **rule** plus **wildcard values** defines a **job**)
* The arrows show data flow and also dependency ordering between jobs
* Snakemake can run the jobs in any order that doesn't break dependency - for example 'kallisto quant' cannot run before
  'kallisto index' or 'trimreads', but it may run before or after 'countreads', or at the same time if you enable concurrent
  processing
* This is not a flowchart - there are no if/else decisions or loops - Snakemake runs every job in the DAG once
* The DAG depends not only on the Snakefile but on the requested target outputs

> Some question about the DAG here...
>
>

## Snakemake is lazy and laziness is good

For the last few episodes, we've told you to run Snakemake like this:

~~~
$ snakemake -j1 -F -p desired_output_file
~~~
{: .language-bash}

As a reminder, the `-j1` flag tells Snakemake to run one job at a time, and `-p` is to print out
the shell commands before running them.

The `-F` flag turns on `forceall` mode, and in normal usage you probably don't want this.

At the end of the last chapter, we generated some kallisto results by running:

~~~
$ snakemake -j1 -F -p kallisto.ref1/abundance.h5
~~~
{: .language-bash}

Now try without the `-F` option. Assuming that the output files are already created, you'll see this:

~~~
$ snakemake -j1 -p kallisto.ref1/abundance.h5
Building DAG of jobs...
Nothing to be done.
Complete log: /home/zenmaster/data/yeast/.snakemake/log/2021-04-23T172441.519500.snakemake.log
~~~
{: .language-bash}

In normal operation, Snakemake only runs a job if:

1. An target file you explicitly requested to make is missing
1. An output file is missing and it is needed in the process of making a target file
1. Snakemake can see an input file which is newer than the output file

Let's demonstrate each of these in turn, by altering some files and re-running Snakemake without the
`-F` option.

~~~
$ rm kallisto.ref1/*
$ snakemake -j1 -p kallisto.ref1/abundance.h5
~~~
{: .language-bash}

This just re-runs the kallisto quantification - the final step.

~~~
rm trimmed/ref1_?.fq
$ snakemake -j1 -p kallisto.ref1/abundance.h5
~~~
{: .language-bash}

"Nothing to be done" - some intermediate output is missing but Snakemake already has the file you are telling it to
make, so it doesn't worry.

~~~
$ touch transcriptome/*.fa.gz
$ snakemake -j1 -p kallisto.ref1/abundance.h5
~~~

The `touch` command is a standard Linux command which sets the timestamp of the file, so now the transcriptome looks
to Snakemake as if it was just modified.

Snakemake sees that one of the input files used in the process of producing kallisto.ref1/abundance.h5 is newer than
the existing output file, so it needs to run the `kallisto index` and `kallisto quant` steps again. Of course, the
`kallisto quant` step needs the trimmed reads which we deleted earlier, so now the trimming step is re-run also.

This behaviour is really useful when you want to:

1. Change some inputs to an existing analysis without re-processing everything
1. Continue running a workflow that failed part-way

In this case, the default Snakemake behaviour will just do the right thing. There are a few gotchas:

### Changing the rules

If the rules in the Snakefile change, rather than the input, Snakemake won't see that the results are out of date.
For example, if we changed the quality cutoffs within the trimreads rule then this would not cause the step to be re-run,
because Snakemake just sees that the output file is newer than the input file.

The `-R` flag allows you to explicitly tell Snakemake that a rule has changed and that all outputs from that rule
need to be reevaluated.

```
$ snakemake -j1 -R trimreads -p kallisto.ref1/abundance.h5
```

### Removing input files

Snakemake can detect if you have added new input files, but not if you have removed input files. We'll look into this
more when we write rules that take lists of files as input.

### Incomplete jobs

Snakemake has a feature that it keeps a log of currently running jobs (this and other info is logged into the `.snakemake`
directory within your working directory). If Snakemake crashes or exits uncleanly then the next time
it runs it will refuse to use output files from incomplete jobs as the files may be partial output. We'll not look into
this during the course, but just be aware that this is what the Snakemake docs mean when they talk about incomplete jobs.
You tend to come across these more when working on a compute cluster, as opposed to a single machine.

>
> 
> TODO - now convert stuff from the existing slides. And see what we can do regarding the exercises. Or do we need more steps
> to make these useful?
>

{. :callout}
