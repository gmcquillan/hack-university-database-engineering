# Views, Functions, Stored Procedures, Oh My!

## Overview

These three feature sets are perhaps some of the most abstract that 
relational databases support. Because of this they are both powerful
and relatively complex.

### What do these functions have in common?

Or, why do I need these when I have data in tables and aggregation functions?

Well, these systems are all particularly useful for the *synthesis* of data, not
just writing, querying and grouping data.

For example, various problems require that you keep a running tally of some number
of data points as they go by. Perhaps the number is so large that it is impractical
to store each data point in its own row. Views, Functions and Stored Procedures
offer different ways to massage and normalize this data without undue load
on the storage systems of the database.

These systems take incoming data, apply some logic and transform the data.

### When are they useful?

Like mentioned above, these are useful when we want to keep our data processing
close to the data itself, either for speed purposes, or just for separation
of concerns.

These tools allow for higher-level abstractions in your database.

In part, it's a question of fashion. It used to be quite popular to keep
database logic with the data. But because it's not a widely solved problem
for dealing with revisions and updates to that logic, they've somewhat fallen
out of favor.

## Views

Views by themselves are really much like an extension of CTEs(Common Table Expression) covered before.
You can assign a query to a symbol and query from it as if it were a sub select.

This is useful when you have lots of variant queries that all share a base set of data.
You don't want to have to write out the same CTE over and over again.

There is a version of these Views called a Materialized View which instead of just acting as
a short hand for a CTE, it actually physically stores the results (like a cache). Logic
for refreshing this View is configurable.

### Sample View

All data is -- to varying degrees -- in need of some cleanup. Our crime dataset is pretty great actually.
Even so, there are a few places where it might be nice to know more about the Address. Most include
State and Zip code in addition to the street address, but others assume some local knowledge and 
only include a colloquial name for the place (usually a park). 

If we wanted a handy place holder for all the strange addresses in this dataset, we could make a view, 
rather than remembering the filtering parameters every time:

```sql

CREATE VIEW weird_addresses AS
	SELECT * FROM crimedataraw
		WHERE "Address" NOT LIKE '% OR %' AND "Address" != 'Unknown/Not Available';
```


## Functions

SQL is a language! It's easy to forget that it's not just an easy way of describing
how to store data and query it -- but it's a language that's Turing complete -- capable of anything.

Functions are especially popular for cases where you want to do an "Upsert". Until Postgres 9.5, there
was not built-in syntax for doing this. So people used functions to this end. That is,
When you said "I have this value, and I want to insert it if I don't have a record, or update
the existing record if it already exists" has traditionally been done using functions.

Here's how to create a simple function:

```sql
CREATE FUNCTION addition(in_a int, in_b int
	) RETURNS int
	LANGUAGE plpgsql
	AS $$
BEGIN
	return in_a + in_b;
END;
$$;
```

Now check that you can see it in the system:

```PSQL

vagrant=> \df
                            List of functions
 Schema |  Name    | Result data type |    Argument data types     |  Type  
--------+----------+------------------+----------------------------+--------
 public | addition | integer          | in_a integer, in_b integer | normal
(1 row)
```

And we can use it: 

```PSQL
vagrant=> select * from addition(4, 7);
addition 
--------
     11
(1 row)
```

### Notice the type information for the return and the arguments

Types are pretty strict, but some (like varchar and text) are automatically converted for you.

You can define your own composite types and use them as parameters or return types in functions,
just like with many other languages.


### What happens if we have two functions with the same name?


### How can we see which functions are loaded?

```PSQL
\df
```

Note: ``\?`` shows all of the commandline shortcuts available. It's really great.


### How do we update functions?

It's all about the create command. It takes an optional second argument in the following form: 

``CREATE OR REPLACE FUNCTION (...);``

This will update the function definition -- and new callers well get the new logic.

### Exception Handling

There are several cases of error that yield exceptions, often this means an assumption is no longer valid.
The network has timed out reaching a resource, a type is invalid, something is already inserted in
a table you're trying to insert data into, could be many things. Fortunately, you get the chance to
inspect the state of the system and retry if you choose.

Here's a common recipe for doing an "Upsert"

```PSQL 

CREATE OR REPLACE FUNCTION upsert_thing(
	in_column1 text, in_column2 text, in_thing int
    ) RETURNS int
    LANGUAGE plpgsql
    AS $$
BEGIN
    UPDATE stuff SET thing = in_thing
        WHERE column1=in_column1
        AND column2=in_column2
    IF FOUND THEN
        RETURN 0;
    END IF;
    BEGIN
        INSERT INTO stuff(column1, column2, thing)
          VALUES (in_column1, in_column2, in_thing);
    EXCEPTION WHEN OTHERS THEN
    	UPDATE stuff SET thing = in_thing
	    WHERE column1=in_column1
	    AND column2=in_column2
    END;
    RETURN 1;
END;
$$;
```

### Functions Reference

- [Functions on Postgresql.org](http://www.postgresql.org/docs/current/static/sql-createfunction.html)


## Stored Procedures

But RDBMS, and Postgres in particular are not limited to function in SQL. They're able to run 
many other programming languages inside your database!

Since we're all using Python, let's have an example doing that.

### PLPython

In Postgres, each language that can be supported for stored procedures has its own dialect.
For python, this is called PLPython.


#### Which Libraries Are Available?

This is an important topic, especially considering that one of the major draws of python
is it's expansive and well-used sets of libraries.

It turns out that any library available globally to the python interpreter that 
Postgres is configured to use will be available. So, unlike a lot of applications
which use virtualenvs, you pretty much need to make sure that your system-wide dependencies
are up to date.

#### Setting up NLTK Example

This sets up our library for doing some simple Natural Language processing
(NLTK stands for Natural Language Tool Kit), and then downloads some 
sample classification data, so we don't have to train any models
to process new text.

```sh
sudo pip install nltk
sudo mkdir -p /usr/local/share/nltk_data
sudo python -m nltk.downloader -d /usr/local/share/nltk_data book
```

This sets us up with ntlk, and a bunch of data that can be used
to tokenize, or tag words we find elsewhere. We'll see how it works
down below.

Confirm that you got those setup correctly:

```python

from nltk.corpus import brown

brown.words()
[u'The', u'Fulton', u'County', u'Grand', u'Jury', ...]
```

##### Setting up python packages via Apt

Sometimes it's easiest just to you precompiled packages for python services.
These packages are super useful for text manipulation. Go ahead and install these, too.

```bash
sudo pip install requests
sudo aptitude install python-lxml
sudo aptitude install postgresql-plpython
```

##### Preparing Postgres to run Python

```psql
vagrant=# create extension plpythonu;
CREATE EXTENSION
vagrant=# \dL
                             List of languages
   Name    |  Owner   | Trusted |               Description                
-----------+----------+---------+------------------------------------------
 plpgsql   | postgres | t       | PL/pgSQL procedural language
 plpythonu | vagrant  | f       | PL/PythonU untrusted procedural language
(2 rows)

```

##### Python code to gather data.
Now we need a way of getting data. Let's take a look at nice set of tools for scraping web sites.

```python
import requests
import nltk
from lxml import html

# Find something on longform.org to get lots of words in one page.
resp = requests.get('https://www.propublica.org/article/how-david-rubenstein-helped-save-the-carried-interest-tax-loophole')
words = nltk.word_tokenize('\n'.join([item for item in html.fromstring(resp.text).itertext()])))
freq_of_words = nltk.FreqDist(words)
return freq_of_words
```

##### Running our data gathering code inside the database.

Now let's wrap up our python code in the stored procedure trappings: 

```sql

vagrant=# CREATE OR REPLACE FUNCTION get_word_freq(url text, OUT word text, OUT count integer) returns setof record as 
$$
import requests
import nltk
from lxml import html
from nltk.corpus import stopwords
import string

# Find something on longform.org to get lots of words in one page.
resp = requests.get(url)
stop = stopwords.words('english') + [punct for punct in string.punctuation]
word_freqs = nltk.FreqDist(nltk.word_tokenize('\n'.join([item for item in html.fromstring(resp.text).itertext()])))

return {key: value for key, value in word_freqs.iteritems() if key not in stop}.iteritems()

$$ LANGUAGE plpythonu;
```

Try it out!
```sql
vagrant=# select * from get_word_freq_for_website('https://www.propublica.org/article/how-david-rubenstein-helped-save-the-carried-interest-tax-loophole'::text) ORDER BY count desc;
```


##### Possible uses

- Take URLs from other tables, and annotate word frequencies.
- If we remove common words, punctuation, etc., we could easily estimate reading level of documents.
- Correlate documents based on word frequency.
- Try to identify author, or relate to similar author style based on word choice.

### Stored Procedure References

- [PostgreSQL Tutorial.com](http://www.postgresqltutorial.com/postgresql-stored-procedures/)
- [PostgreSQL.org PLPython Docs](http://www.postgresql.org/docs/current/static/plpython.html)
