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
halfway through the code to update a variable. Also, spark isn't initialized until 
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
transforms later. Also, once we have converted to parquet once, there is no need to do it again 
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

## Ambiguities

### We're only using the primary ICD code for analysis

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

### We're hard coding ignoring ages < 40

In **age_bucket()** we're returning 0 for any ages < 40. If we decide in a future 
analysis that we want to track these values, it'll be harder to change than if we 
bucketed those values properly now and just filtered them from results. There may be 
some business reason for always ignoring these ages, so I haven't changed the code in my submission, 
but it's something worth calling out. Hard coding constraints is generally frowned upon.

## Debugging

I ran into a few bugs as I was working on this. Here, I'll detail how they were found and fixed.

### Pandas encoding issue

Running data sets 2-4 resulted in the following error:

```
UnicodeDecodeError: 'utf-8' codec can't decode byte 0x93 in position 0: invalid start byte
```

Luckily switching to a pure spark solution was able to resolve this on its own. If not, 
I could have tried inferring the encoding using **chardet** or something similar, but 
I'm not aware of a foolproof way to figure out the proper encoding without some trial 
and error. Glad this one wasn't a major issue.

### Age bucketing bug

The **age_bucket()** function has a bug in it at this line:

```
    elif age > 80:
        return 90
```

I glossed over it at first, but started noticing that my output never included results 
for this age bucket, even with sufficiently large data sets. Once I noticed that, tracing it 
back to this line was pretty easy.

### Results not being aggregated by count

Initially, we have this line:

```
.withColumn(
        '_row',
        sf.row_number().over(Window().partitionBy(['AGE_BUCKET']).orderBy(sf.desc('count'))))
    .filter(sf.col('_row') == 1)
```

We're partitioning by age bucket, ordering by count, and then just picking the top count. 
But if we have multiple ICD codes with the same count, we're not capturing all the codes that are tied for top spot, 
we're just grabbing whichever one happens to sit at the top by its secondary ordering column. 
This will skew our results and not give us the answer we're really looking for. The line 
above hasn't changed, but I have aggregated codes with an identical count before we hit 
the partitioning with this line:

```
joined
    .groupBy('AGE_BUCKET', 'GROUP').count()
    .groupBy('AGE_BUCKET', 'count').agg(sf.collect_set('GROUP'))
```

I also split up the transformation some so I could see the intermediate results, and so 
it was easier to reason about.

### Cleaning bad rows in the data sets

A simple data quality check showed that there was at least one bad row in the demographics data:

```
+-------+------+--------------------+----------+---------+
|summary|CLINIC|                 MRN|FIRST_NAME|LAST_NAME|
+-------+------+--------------------+----------+---------+
|  count| 10000|               10000|     10000|    10000|
|   mean|  null|     5.07491028763E7|      null|     null|
| stddev|  null|2.8727825630081754E7|      null|     null|
|    min|     A|            01015674|     Aaron|   Abbott|
|    max|     C|            99998883|       Zoe|�the Kid�|
+-------+------+--------------------+----------+---------+
```

I implemented some simple data cleaning, but didn't want to be too aggressive with it--we 
don't want to drop any rows that could still be valid for analysis, even if some fields 
didn't quite match what we expect. Dropping rows is also usually a last resort, but 
fields like MRN and ICD_CODE are categorical variables and can't be imputed anyway.
Bearing all that in mind, I dropped only rows that couldn't have been used for analysis 
because of invalid/missing data, and duplicate rows, which can be safely dropped here.

## Further improvements

At this point, I would say the code was probably good enough to pass review. But 
I was able to make some more improvements to make the code more readable and more 
maintainable moving forward.

### Using enums and top-level constants for configuration

I had to run this notebook multiple times checking different things--sometimes 
I was checking output to test a bugfix, sometimes I was comparing performance to a 
previous benchmark. I added some configuration settings at the top to streamline this process.

### Abstracting read_csv() to read_file()

After converting to parquet, I originally had separate **read_csv()** and **read_parquet()** functions.
I decided to combine these into a single **read_file()** function that handles both appropriately.

### Declaring rather than inferring schemas

Spark allows us to define schemas including data type, and it will try to infer them 
if we don't. Defining them ahead of time for small tables like this lets us avoid an extra 
initial pass through the data for spark's schema inference, and it allows us to handle setting date 
fields as date data types all in one go.

### Using different variable names throughout the pipeline

In order to keep each cell atomic and idempotent, I updated variable names to use new ones 
after each transformation instead of overwriting the old references. This was mostly to help with 
my own testing and for maintainability--using the same variable names throughout is fine for quick analyses.