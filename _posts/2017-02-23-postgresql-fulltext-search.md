---
layout: post
status: publish
published: true
title: An idiot's guide to fulltext search in PostgreSQL.
author: Travis Mick
date: 2017-02-23
---

I love PostgreSQL. It's probably the most powerful open-source database system out there. Recent features to handle [JSON](https://www.postgresql.org/docs/9.6/static/datatype-json.html) and [geospatial](http://www.postgis.net/) data are allowing it to supplant specialized database systems and become closer to a one-DB-fits-all solution. One feature that I've recently been able to exploit is its [fulltext search](https://www.postgresql.org/docs/9.6/static/textsearch.html) engine. It allowed me to easily move from a terrible search implementation (using regular expressions) to one that actually meets users' expectations.

In this article, I will walk through a basic fulltext search configuration, as well as highlight a few potential improvements that can be made if you're so inclined.

Many of the features discussed in this post are only available as of PostgreSQL 9.6. Earlier versions have some rudimentary fulltext functionality, but a lot of the more powerful tools we'll be using are fairly new.

<!-- more -->

## The Wrong Way To Search

Let's start with a basic table:

```sql
create table test_table (id serial, title text, description text);
```

And populate it with some bogus data:

```sql
insert into test_table (title, description) values ('Hello world!', 'This is an example.'), ('Filler text', 'foo bar baz');
```

How might we go about doing a simple search on this table? The naive strategy, which I originally might have used, would go something like this:

```sql
select * from test_table where title ~* 'hello' or description ~* 'hello';
```

This is the result (which is correct, in this case):

```
id  | title        | description
----+--------------+---------------------
1   | Hello world! | This is an example.
(1 row)
```

However, consider that a human searching for something might not always know *exactly* what they're looking for. Let's put a new row in that table:

```sql
insert into test_table (title, description) values ('A helpful example', 'This is where things get complicated');
```

Let's say a user *kind of* knows what they're looking for: they enter the query "gets complicated":

```sql
select * from test_table where description ~* 'gets complicated';
```

There is no result. Why? Because we have no way to identify that "gets" and "get" are semantically equivalent.

Also consider that a regex-based search is *at least* linear in the length of each string we check (unless you enable trigram indices, apparently, but that's out of scope of this article). If you're searching a large table, you're going to be in big trouble.

Fulltext search solves both of these problems: It treats semantically equivalent words as matches, and can be drastically sped up by virtue of indices.

## A Better Solution

To handle fulltext searches, there are two main things we have to do: add a `tsvector` to the table to store keywords, and write a `tsquery` when we want to perform a search.

A `tsvector` is essentially a list of keywords. We can assign one to each row in the table. Postgres will automatically filter out common words and replace similar words with roots or synonyms. Here's an example query, to demonstrate how a human-written string is translated into an indexable `tsvector`:

```sql
select to_tsvector('This is where things get complicated');

to_tsvector
-------------------------------
'complic':6 'get':5 'thing':4
(1 row)
```

We can see that the input got filtered down into three keywords. One of them, "complicated," got trimmed down to the root "complic." The rules for how this translation takes place are defined by the `regconfig`.

Your default `regconfig` is probably either "english" or "simple." It can be retrieved by selecting `get_current_ts_config()`, or set with the `default_text_search_config` directive. If you need to change it per-query, you can stick it in any call to `to_tsvector` or its related functions, e.g. `to_tsvector('english', 'hello world')`. In most cases, you'll want "english" (or some other human language) rather than "simple" (which doesn't handle magic such as removing meaningless words and distilling words into lexemes). Read [the documentation](https://www.postgresql.org/docs/9.6/static/textsearch-configuration.html) if you think you'll need to do anything funky with regconfig.

Now that we understand what we're doing, let's add a tsvector column to the table:

```sql
alter table test_table add column tsv tsvector;
```

Note: It isn't always necessary to create a dedicated `tsvector` column (i.e., you can skip right ahead to creating a fulltext index on a concatenation of existing columns), but I've done it here because it's a more flexible option and it's easier to reason about.

How do we populate this column? Well, it's quite easy to add a trigger that does it for us. We just tell it the name of our `tsvector` column, the language to use for processing, and then list all of the fields that should be considered for keywords:

```sql
create trigger test_tsv before insert or update on test_table
for each row execute procedure tsvector_update_trigger(
  tsv, 'pg_catalog.english', title, description
);
```

Note that `'pg_catalog.english'` is actually our `regconfig` showing up again. It is mandatory to supply a `regconfig` to `tsvector_update_trigger`, and you must fully qualify it with the schema name (`pg_catalog`).

Now, any new or updated tuples will be given a `tsvector` containing the title and description. Let's see it in action:

```sql
insert into test_table (title, description) values ('We have tsvectors now!', 'Pretty cool, eh?') returning tsv;

tsv
-----------------------------------------
'cool':6 'eh':7 'pretti':5 'tsvector':3
(1 row)
```

We can do a quick hack to trigger the hook and create vectors for our pre-existing rows. Be careful doing this in production.

```sql
update test_table set id=id;
```

Creating the tsvectors was half the battle. Now, how do we query them? The quite intuitive answer: `tsquery`.

A `tsquery`, like a `tsvector`, is essentially a list of keywords. However, we can do some fun things, like apply boolean operations and specify that words have to be in a certain order. Here's a list of a few useful operators we can abuse:

| Syntax        | Description                                                                                                                      |
|---------------|----------------------------------------------------------------------------------------------------------------------------------|
| `foo <-> bar` | Matches "foo" followed by "bar"                                                                                                  |
| `foo <2> baz` | Matches "foo" with "baz" two words later (i.e., one word in between). Replace the 2 with another number to match a larger "gap." |
| `foo & bar`   | Matches a vector that contains both "foo" and "bar," in any order and with anything in between.                                  |
| `foo | bar`   | Matches a vector that contains either "foo" or "bar."                                                                            |
| `!baz`        | Matches a vector that doesn't contain "baz."                                                                                     |

Here's an example of a simple tsquery, to see how it is interpreted:

```sql
select to_tsquery('foo <-> baz');

to_tsquery
-----------------
'foo' <-> 'baz'
```

And here's how we would use it to search (say hello to the `@@` operator):

```sql
select id, title, description from test_table where tsv @@ to_tsquery('foo <-> bar');

id  | title       | description 
----+-------------+-------------
2   | Filler text | foo bar baz 
```

Of course, we're not going to get our users to type their queries in this format. Fortunately, Postgres gives us a few helper functions for creating queries. Let's take a look at both `plainto_tsquery` and `phraseto_tsquery`, and see how they differ:

```sql
select plainto_tsquery('foo baz'), phraseto_tsquery('foo baz');
 plainto_tsquery | phraseto_tsquery 
-----------------+------------------
 'foo' &amp; 'baz'   | 'foo' <-> 'baz'
(1 row)
```

Most noticeably, `plainto_tsquery` just matches for the boolean conjunction of all words in the input, while `phraseto_tsquery` matches the exact ordering. Choose the one most appropriate for your application. You can easily make queries using a condition like `where tsv @@ plainto_tsquery(user_input_here)`.

At this point, you're able to handle simple fulltext queries. Congratulations. If you're interested, read on for some interesting tweaks and optimizations.

## Even Better

The next thing you're going to want to add is an index. The Postgres documentation recommends using GIN indices for `tsvectors` (the reasons why are outside the scope of this post):

```sql
create index tsv_index on test_table using gin (tsv);
```

This will speed up your queries significantly, and is quite important if your tables are large.

There are several other features you might want to explore. For sake of brevity, I'll just list a couple interesting ones and provide links to the appropriate documentation:

* You can [rank results](https://www.postgresql.org/docs/9.6/static/textsearch-controls.html#TEXTSEARCH-RANKING) by relevance.
* You can [apply weights to the different columns](https://www.postgresql.org/docs/9.6/static/textsearch-controls.html#TEXTSEARCH-PARSING-DOCUMENTS) that go into your tsvector, for example to make the title more important in ranking than the description.
* You can [write custom triggers](https://www.postgresql.org/docs/9.6/static/textsearch-features.html#TEXTSEARCH-UPDATE-TRIGGERS) to handle more complicated tsvector constructions.
* You can [obtain highlighted results](https://www.postgresql.org/docs/9.6/static/textsearch-controls.html#TEXTSEARCH-HEADLINE) to indicate to the user which part of the document matched their query.
* You can [match on word prefixes](https://www.postgresql.org/docs/9.6/static/textsearch-controls.html#TEXTSEARCH-PARSING-QUERIES), which is useful for implementing a search-box typeahead feature.
* There's [all kinds of customization](https://www.postgresql.org/docs/9.6/static/textsearch-dictionaries.html) you can do with the dictionaries.

There are many more features I've omitted here, so don't be afraid to read through that entire chapter of the documentation to learn more.

## Conclusions

There are lots of reasons why we want to avoid substring-based or regex-based searching in our database. Perhaps most importantly, the results are often unintuitive to the user.

Postgres gives us an easy way to harness powerful fulltext searching using tsvectors and tsqueries. There's absolutely no reason not to use them -- you'll avoid needing to configure a separate system for searching, you'll easily be able to keep your keyword vectors and indices consistent with the latest database contents, and most importantly you'll be giving your users the search results they expect.

