# Take Home Data Homework

For this project, pretend that you've been given the included jupyter notebook
from a junior analyst and they'd like you to review the code, look for bugs,
potential performance improvements and code structure.

## Setup

### spark

The notebook requires spark, which can be downloaded https://spark.apache.org/downloads.html.
Download v3.0.3 with hadoop 2.7.  Installation involves unpacking the downloaded file
somewhere and setting the SPARK_HOME environment variable to point to that folder. Spark
will be run in local mode.

### python

The requirements.txt provides a lists of the python packages that needed to be installed.
I used python 3.7 for my virtual environment but later versions will probably work too.

You'll also want to install jupyter notebook.

## Data

There is a `code_groups.csv` file that contains a classification of some ICD codes.
(The data is randomly generated, there is no medical rationale for the groupings).

There is also a `demographics.csv` file and an `encounters.csv` file.
The `encounters.csv` file contains one ICD code per row. The codes are listed in priority
order; so the first ICD code is the primary diagnosis for that visit.

The objective of the provided notebook is to find the most common ICD group
for a set of age ranges. In other words, for patients that are 40-50 years old
what is the most common ICD group, same for 50-60, 60-70, etc.

There are 4 provided datasets: `data_1`, `data_2`, `data_3` and `data_4`.
Starting with `data_1`: it is only 1000 patients and the notebook will currently run with this dataset.
The next datasets, `data_2` and `data_3` introduce data quality issues and highlight some bugs in the code.
Finally, `data_4` has data for 1 million patients and can be used to test the performance of the code.

## Deliverable

Edit the current notebook to show what you would fix and improve in the current code,
add any data quality checks you might want to put in, and show your debugging process
as much as you can.

The main goal of this assignment is to see how well you can read and understand existing code, and
how well you communicate your thought process while examining the pipeline.



