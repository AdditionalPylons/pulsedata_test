# Objective

For this project, pretend that you've been given the included jupyter notebook
from a junior analyst and they'd like you to review the code, look for bugs,
potential performance improvements and code structure.

The objective of the provided notebook is to find the most common ICD group
for a set of age ranges. In other words, for patients that are 40-50 years old
what is the most common ICD group, same for 50-60, 60-70, etc.

Edit the current notebook to show what you would fix and improve in the current code,
add any data quality checks you might want to put in, and show your debugging process
as much as you can.

The main goal of this assignment is to see how well you can read and understand existing code, and
how well you communicate your thought process while examining the pipeline.

## Approach

If this were a code review, I'd be able to leave in-line comments with suggested changes, 
but I don't have that option here, so I've done my best to keep notes of my process along 
the way. I started by reading the code top to bottom, tried running it with the different 
data sets, and then went line-by-line through function calls/transformations where something 
seemed off.

## Initial Impressions

The code is easy enough to reason about, but I saw a few problems right away:

### Code Organization

#### Imports aren't handled in a very uniform way

We import all of pyspark, but then import from 
pyspark.sql twice separately. The initial pyspark import is never used directly. It's not 
*too* bad here, but in general we want to consolidate this so that dependencies are easier 
to track and our namespace won't get cluttered.

#### Cells could benefit from some reordering

I can understand the flow here, but changing data sets means I need to scroll to 
halway through the code to update a variable. Also, spark isn't initialized until 
after we've defined all our supporting functions. This isn't *wrong*, but it makes sense  
to me to initialize it right away so we're not hoisting references to it in **read_csv()** or 
**transform_encounters()** when it doesn't exist yet.

### Obvious improvements

#### We can get rid of pandas and drop a dependency

I love pandas too, but it's not a one-size fits all solution. Running data sets 2-4 suggests 
that pandas may be running into some encoding issues, and we only use it on one line anyway.
Everything we want to do here with pandas can be done more effectively / cleanly using 
spark itself directly with a couple of tweaks.

#### We can use a columnar data store instead of CSV

I chose parquet because it's what I'm most familiar with and because it's an effective 
long-term solution, but there are other options. Having Pyarrow in requirements.txt implies 
that using that was the intended solution, but I'm new to it (and spark more broadly), so
I decided to stick to what I knew would work. Parquet isn't the only answer, and I'd be happy 
to learn and use the desired formats on the job. The downside to using parquet is that converting from CSV
on the first run takes some extra time, but we more than make up for it with faster aggregate queries and 
transforms later. Also, once we have convert to parquet once, there is no need to do it again 
and subsequent runs can be much faster.

#### We should add some data quality checks

I'm ok with no data cleaning happening here yet (maybe this is an intermediary transform 
after we've already done some cleaning in the pipeline), but no checks is asking for trouble.
Implementing even some basic checks shows there are some issues right away in some of the data sets.

#### We should dig deeper on some possible bugs

I couldn't put my finger on it right away when I first ran this notebook, but something
definitely felt off. For smaller data sets, I would have expected some collisions for most 
common code, but there was only ever one most common code for each age bucket. Larger 
data sets always seemed to converge around ICD_CODE 10 as the most common for each age. 
These could be idiosyncrasies of the data, but they suggest we should do some testing with a 
known outcome to verify our pipeline works as expected.

### Ambiguities

#### We're only using the primary ICD code for analysis
Per the directions, it seems like we want to consider *all* ICD codes in our analysis:
> The objective of the provided notebook is to find the most common ICD group
for a set of age ranges. In other words, for patients that are 40-50 years old
what is the most common ICD group, same for 50-60, 60-70, etc.

But then we're only actually counting the primary codes:
```
.join(code_groups_sdf, enc_sdf.ICD_CODE_1==code_groups_sdf.ICD_CODE)
```

I have updated this in my submission, but it's really a question of what we are 
actually looking to measure. There is nothing wrong technically with the way it was before.

#### We're hard coding ignoring ages < 40

In **age_bucket()** we're returning 0 for any ages < 40. If we decide in a future 
analysis that we want to track these values, it'll be harder to change than if we 
bucketed those values properly now and just filtered them from results. There may be 
some business reason for always ignoring these ages, so I haven't changed the code in my submission, 
but it's something worth calling out. Hard coding constraints is generally frowned upon.

## Debugging

I ran into a few bugs as I was working on this. Here, I'll detail how they were found and fixed.

### Possible pandas encoding issue

