log-synth
=========

The basic idea here is to have a random log generator build fairly realistic log files for analysis. The analyses specified here are fairly typical use cases for trying to figure out where the load on a web-site is coming from.

The second utility provided here is to generate data based on a specified schema.  Skip down to the section on schema-driven generation.

How to Run the Log Generator
========================

Install Java 7, maven and get this software using git.

On a mac, this can help get the right version of Java

    export JAVA_HOME=$(/usr/libexec/java_home)

Then do this to build a jar file with all dependencies included

    mvn package

Then use this to write one million log lines into the file "log" and to write the associated user database into the file "users".

    java -cp target/log-synth-0.1-SNAPSHOT-jar-with-dependencies.jar com.mapr.synth.Main -count 1M log users

This program will produce a line of output on the standard output for each 10,000 lines of log produced.  Each line will contain the number of log lines produced so far and the number of unique users in the user profile database.


The Data Source
==============
The data source here is a set of heavily biased random numbers to generate traffic sources, response times and queries. In order to give a realistic long-tail experience the data are generated using special random number generators available in the Mahout library.

There are three basic entities involved in the random process that generates these logs that are IP addresses, users and queries. Users have a basic traffic rate and a variable number of users sit behind each IP address. Queries are composed of words which are generated somewhat differently by each user. The response time for each query is determined based on the terms in the queries with a very few terms causing much longer queries than others. Each log line contains an IP address, a user cookie, a query and a response time.

Logs of various sizes can be generated using the generator tools.

The Queries
==============
The general goal of the queries is to find out what and/or who is causing long query times and where lots of traffic is coming from.

The questions we would like to answer include:

* What are the top IP addresses by request count?
* What are the top IP addresses by unique user?
* What are the most common search terms?
* What are the most common search terms in the slowest 5% of the queries?
* What is the daily number of searches, (approximate) number of unique users, (approximate) number of unique IP addresses and distribution of response times (average, min, max, 25, 50 and 75%-iles).

Methods
========
The general process for generating log lines is to select a user, possibly one we have not seen before. If the user is new, then we need to select an IP address for the user. Otherwise, we remember the IP address for each user.

Queries have an overall frequency distribution that is long-tailed, but each user has a variation on that distribution. In order to model this, we sample each user's queries from a per-user Pittman-Yor process. In order to make users have similar query term distributions, each user's query term distribution is initialized from a Pittman-Yor process that has already been sampled a number of times.

We also need to maintain an average response time per term. The response time for each query is exponentially distributed with a mean equal to the sum of the average response times for the terms. Response times for words are sampled either from an exponential distribution, from a log-gamma distribution or from a gamma distribution with a moderately low shape parameter so that we can have interestingly long tails for response time.

Users are assigned to IP addresses using a Pittman-Yor process with a discount of 0.9. This gives long-tailed distribution to the number of users per IP address. This results in 90% of all IP addresses having only a single user.

Schema-driven Data Generation
=====================

You can also generate data based on a rough schema.  Schema-driven generation supports the generation of addresses, dates, foreign key references, unique id numbers, random integers, sort of realistic personal names and fanciful street names.

In addition to these primitive generators of strings and numbers, nested structures of arrays and objects can also be generated.  In a future release, it is anticipated that the generator will execute arbitrary Javascript in order to allow arbitrary dependencies and transformations of data as it is generated.

To generate data, follow the compilation directions above, but use this main program instead:

    java -cp target/log-synth-0.1-SNAPSHOT-jar-with-dependencies.jar com.mapr.synth.Synth -count 1M -schema schema

The allowable arguments include:

 `-count n`    Defines how many lines of data to emit.  Default value is 1000.  Suffixes including k, M, and G have customary meanings.

 `-schema file` Defines where to get the schema definition from.  The schema is in JSON format and consists of a list of field specifications.  Each field specification is a JSON object and is required to have the following values


 * `class` - Defines the distribution that is used to sample values for this field.  Possible values include `address`, `date`, `foreign-key`, `id`, `int`, and `street-name`.  Additional values that may be allowed or required for specific generators are detailed below.

 * `name` - This is the name of the field.  The output will consist of fields ordered as in the schema definition and any header file will contain the names for each field as defined by this value.

`-format CSV | TSV | JSON` Defines what format the output should use.

The following classes of values are allowed:

`address` - This distribution generates fairly plausible, if somewhat fanciful street addresses.  There are no additional parameters allowed.

`date` - This distribution generates dates which are some time before an epoch date.  Dates shortly before the epoch are more common than those long before.  On average, the dates generated are 100 days before the epoch.  A format field is allowed which takes a format for the data in the style of Java's SimpleDateFormatter.

`foreign-key` - This distribution generates randomized references to an integer key from another table.  You must specify the size of the table being referenced using the size parameter.  You may optionally specify a skewness factor in the range [0,3].  A value of 0 gives uniform distribution.  A value of 1 gives a classic Zipf distribution.

`id` - This distribution returns consecutive integers starting at the value of the start parameter.

`int` - This distribution generates random integers that are greater than or equal to the min parameter and less than the max parameter.

`street-name` - This distribution generates fanciful three word street names.

`string` - This distribution generates a specified distribution of strings.  One parameter called `dist` is required.  This parameter should be a structure with string keys and numerical values.  The probability for each key is proportional to the value.

The following schema generates a typical fact table from a simulated star schema:

    [
        {"name":"id", "class":"id"},
        {"name":"user_id", "class": "foreign-key", "size": 10000},
        {"name":"item_id", "class": "foreign-key", "size": 2000}
    ]

Here we have an id and two foreign key references to dimension tables for user information and item information.  This definition assumes that we will generate 10,000 users and 2000 item records.

The users can be generated using this schema.

    [
        {"name":"id", "class":"id"},
        {"name":"name", "class":"name", "type":"first_last"},
        {"name":"gender", "class":"string", "dist":{"MALE":0.5, "FEMALE":0.5, "OTHER":0.02}},
        {"name":"address", "class":"address"},
        {"name":"first_visit", "class":"date", "format":"MM/dd/yyyy"}
    ]

For each user we generate an id, a name, an address and a date the user first visited the site.

Items are simpler and are generated using this schema

    [
        {"name":"id", "class":"id"},
        {"name":"size", "class":"int", "min":10, "max":99}
    ]

Each item has an id and a size which is just a random integer from 10 to 99.

You can use the sequence type to generate variable or fixed-length arrays of values which can themselves be complex.  If you use the JSON output format, this structure will be preserved.  If you want to flatten an array produced by sequence, you can use the flatten sampler.

For example, this produces users with names and variable length query strings

    [
        {"name":"user_name", "class":"name", "type": "last_first"},
        {"name":"query", "class":"flatten", "value": {
            "class": "sequence", "length":4, "base": {
                "class": "word"
            }
        }}
    ]

If you use the TSV format with this schema, the queries will be comma delimited unquoted strings.  If you omit the flatten step, you will get a list of strings surrounded by square brackets and each string will be quoted (i.e. an array in JSON format).

You can also generate arbitrarily nested data by using the map sampler.  For example, this schema will produce records with an id and a map named stuff that has two integers ("a" and "b") in it.

    [
        {"name": "id", "class": "id"},
        {
            "name": "stuff", "class": "map",
            "value": [
                {"name": "a", "class": "int", "min": 3, "max": 4},
                {"name": "b", "class": "int", "min": 4, "max": 5}
            ]
        }
    ]

Here is a list of all of the currently known kinds of samplers that you can use as the class argument in your schema

***address*** - Generates a complete street address

```json
  {"name":"address", "class":"address"},
```

***date*** - Generates dates.  By default, these are formatted like yyyy-MM-dd, but you can specify a different format using the format option.  Dates are selected by default to be before July 1, 2013.  The amount before is selected exponentially with mean of 100 days.  If you specify start or end dates, the dates will be sampled uniformly between your dates. The default start is January 1, 1970.  The default end is July 1, 2013.

```json
  {"name":"first_visit", "class":"date", "format":"MM/dd/yyyy"},
  {"name":"second_date", "class":"date", "start":"2014-01-31", "end":"2014-02-07"},
  {"name":"third_date", "class":"date", "format":"MM/dd/yyyy", "start":"01/31/1995", "end":"02/07/1999"}
```

***foreign-key*** - Selects values from 0 (inclusive) to size (exclusive).  The default value of size is 1000. Values are normally biased towards smaller values.  You can adjust the bias by setting skew.  Setting skew to 0 makes the selection uniform.  The default skew is 0.5.  This sampler uses space proportional to size so be slightly cautious.  

***id*** - Selects sequential integers.

```json
   {"name":"id", "class":"id"},
```


***int*** - Samples values from min (inclusive) to max (exclusive) with an adjustable skew toward small values.  If you set skew to a negative number, larger values will be preferred.

```json
  {"name":"size", "class":"int", "min":10, "max":99}
  {"name": "z", "class": "int", "min": 10, "max": 20, "skew": -1},
  {"name":"x", "class":"lookup", "resource":"data.json", "skew":1},
```

***event*** - Samples Poisson distributed event times with specified rates.

```json
  {"name":"foo1", "class":"event", "rate": "0.1/d"},
  {"name":"foo2", "class":"event", "start": "2014-01-01", "format":"yyyy-MM-dd HH:mm:ss", "rate": "10/s"},
  {"name":"foo3", "class":"event", "format": "MM/dd/yyyy HH:mm:ss", "start": "02/01/2014 00:00:00", "rate": "0.5/s"}
```

***name*** - Samples from (slightly) plausible names.

```json
  {"name":"name", "class":"name", "type":"first_last"},
```

***lookup*** - Samples from lines of a file.

***flatten*** - Turns an object into fields.

***join*** -

***map*** - Samples from complex objects, fields of which are sampled according to a recursive schema you specify.

```json
  {
    "name": "stuff",
    "class": "map",
    "value": [
      {"name": "a", "class": "int", "min": 3, "max": 4},
      {"name": "b","class": "int","min": 4,"max": 5}
    ]
  }
```
 
***street-name*** - Samples from (slightly) plausible street names.

***string*** - Samples from one of a set of strings you specify.

```json
   {"name":"foo", "class":"string", "dist":{"YES":0.95, "NO":0.05, "NA":1}}
```

***country*** - Samples from ISO country codes.

```json
  {"name":"co", "class":"country"},
```

***browser*** - Samples from browser types with kind of plausible frequency distribution.

```json
  {"name":"br", "class":"browser"},
```

***state*** - Samples from any of the 58 USPS state abbreviations.  Yes, 58.

```json
  {"name":"st", "class":"state"},
```

***language*** - Samples from ISO language codes according to prevalence on the web.

```json
  {"name":"la", "class":"language"},
```

***os*** - Samples from operating system codes.  My own bias will show here.

```json
  {"name":"os", "class":"os"}
```

***word*** - Samples words at random.  A seed file is given, but if more words are needed than seeded, they will be invented.

***sequence*** - Repeatedly samples from a single distribution and returns an array of the results.

```json
  {"name":"c", "class":"sequence", "base":{"class":"os"}},
  {"name":"d", "class":"sequence", "base":{"class":"int", "min":3, "max":9}, "length":10}
  {
    "name": "x",
    "class": "sequence",
    "array": [
      {
        "class": "int",
        "min": 3,
        "max": 4
      },
      {
        "class": "int",
        "min": 6,
        "max": 7
      },
      {
        "class": "int",
        "min": 8,
        "max": 9
      }
    ]
  }
```

Quoting of Strings
============

By default all strings in CSV or TSV formats are fully quoted.  This can confuse some software (sadly) so there are additional options to control how quoting is done.  

There are three basic strategies supported:

* *DOUBLE_QUOTE* This is the previous behavior where all strings are safely quoted according to JSON conventions.

* *BACK_SLASH* With this convention, all internal spaces, tabs, back slashes and commas are quoted by preceding them with backslash character. This convention is useful with Hive.

* *OPTIMISTIC* With this convention, no quoting of strings is done.  This should not normally be used since it is very easy to get unparseable data.

The default convention is DOUBLE_QUOTE.
