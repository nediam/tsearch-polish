Wersja polska artykułu: [Polski słownik dla modułu TSearch w PostgreSQL](PolskiSlownikTsearchPostgreSQL.md)

# Polish dictionary for PostgreSQL TSearch #

Polish dictionary for PostgreSQL full text search TSearch module, includes synonyms for words without tails ("zlobek" and "zlobkiem" finds "żłobek").

The charset used in dictionary files is UTF-8, so your database must also be in UTF-8 for it to work. You can convert dictionary files easily to other formats with "iconv" command, for example from utf-8 to iso-8859-2:

```
for f in polish.affix polish.dict polish.stop polish.syn; do cat $f | iconv -f utf-8 -t iso8859-2 > $f; done
```

## Download ##

[tsearch\_data\_polish\_20120730.zip](http://tsearch-polish.googlecode.com/files/tsearch_data_polish_20120730.zip) (6.2 MB)

Requirements: PostgreSQL 8.3 or higher. Tested mostly on PostgreSQL 9.1.

<a href='https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=RLZT2JFRL6E88'><img src='https://www.paypalobjects.com/en_US/GB/i/btn/btn_donateCC_LG.gif' /></a>

## Install ##

1. Extract zip, you will find a folder called "tsearch\_data\_polish" and following files inside it:

  * polish.affix
  * polish.dict (4.6 MB, main dictionary)
  * polish.stop (stop words, includes words without tails)
  * polish.syn (45 MB, synonyms without tails, includes all variants of the word)
  * polish.ths (synonyms with tails)

2. Copy all these files to "PostgreSQL/9.1/share/tsearch\_data/".

3. Create text search configuration and dictionaries:

```
CREATE TEXT SEARCH CONFIGURATION public.polish ( COPY = pg_catalog.english );
CREATE TEXT SEARCH DICTIONARY polish_ispell (
	TEMPLATE = ispell,
	DictFile = polish, -- tsearch_data/polish.dict
	AffFile = polish, -- tsearch_data/polish.affix
	StopWords = polish -- tsearch_data/polish.stop
);

CREATE TEXT SEARCH DICTIONARY polish_synonym (
	TEMPLATE = synonym,
	SYNONYMS = polish -- tsearch_data/polish.syn
);

CREATE TEXT SEARCH DICTIONARY polish_thesaurus (
	TEMPLATE = thesaurus,
	DictFile = polish, -- tsearch_data/polish.ths
	Dictionary = polish_ispell
);

ALTER TEXT SEARCH CONFIGURATION polish
	ALTER MAPPING FOR asciiword, asciihword, hword_asciipart, word, hword, hword_part
WITH polish_thesaurus, polish_synonym, polish_ispell, simple;
```

4. Test if it works.
```
-- Run each of queries below separately, each should return 't' (true) for success, 'f' (false) means it failed.

SELECT to_tsvector('public.polish', 'żłobek') @@ plainto_tsquery('public.polish', 'zlobek');
SELECT to_tsvector('public.polish', 'żłobek') @@ plainto_tsquery('public.polish', 'zlobkiem');
SELECT to_tsvector('public.polish', 'żłobek') @@ plainto_tsquery('public.polish', 'zlobkami');
SELECT to_tsvector('public.polish', 'zlobkami') @@ plainto_tsquery('public.polish', 'żłobek');
SELECT to_tsvector('public.polish', 'zlobkami') @@ plainto_tsquery('public.polish', 'zlobek');
SELECT to_tsvector('public.polish', 'abramów') @@ plainto_tsquery('public.polish', 'abramow');
SELECT to_tsvector('public.polish', 'abram') @@ plainto_tsquery('public.polish', 'abramow');
SELECT to_tsvector('public.polish', 'żłóbek') @@ plainto_tsquery('public.polish', 'zlobek');
SELECT to_tsvector('public.polish', 'zlobek') @@ plainto_tsquery('public.polish', 'żłobek');
SELECT to_tsvector('public.polish', 'zlobek') @@ plainto_tsquery('public.polish', 'żłóbek');

-- Some debugging.

SELECT * FROM ts_debug('polish', 'żłobek');
SELECT * FROM ts_debug('polish', 'żłóbek');
SELECT * FROM ts_debug('polish', 'zlobek');
SELECT * FROM ts_debug('polish', 'zlobkiem');
SELECT * FROM ts_debug('polish', 'abramowowi');

```

## Example: table, data, trigger, index, query ##

Create table and insert some data:

```
CREATE TABLE articles (
    body text NOT NULL,
    body_tsvector TSVECTOR
);
INSERT INTO articles (body) VALUES ('Żłobek nr 4');
INSERT INTO articles (body) VALUES ('Trafił żłobkiem w przedszkole');
INSERT INTO articles (body) VALUES ('Bylem sobie zlobkiem');
```

Update the TSearch vector:

```
UPDATE articles SET body_tsvector = to_tsvector('public.polish', body);
```

Create a trigger so that index is updated automatically on INSERT/UPDATE.

```
CREATE TRIGGER articles_body_tsvector_trigger
BEFORE INSERT OR UPDATE ON articles
	FOR EACH ROW EXECUTE PROCEDURE tsvector_update_trigger(body_tsvector, 'public.polish', body);
```

Speed up the search by indexing the tsearch vector:

```
CREATE INDEX articles_body_tsvector_idx ON articles USING GIN( body_tsvector );
```

Now, let's run some simple search:

```
SELECT body FROM articles
WHERE body_tsvector @@ plainto_tsquery('public.polish', 'zlobkiem');
```

Now, with rank:

```
SELECT body, ts_rank('{1,0.9,0.8,0.7}', body_tsvector, q) as rank
FROM articles, plainto_tsquery('public.polish', 'zlobek') as q
WHERE body_tsvector @@@ q
ORDER BY rank DESC
```

## Performance ##

There is one caveat, PostgreSQL does not cache the dictionary in memory, every new database connection reads an entire dictionary again, for a 50 MB dictionary that can take a few seconds, also it eats about 300 MB of memory.

The solution is to use connection pooling, have a look at [pgBouncer](http://pgfoundry.org/projects/pgbouncer/).

pgBouncer instructions for Windows users:

1. Download binary: http://winpg.jp/~saito/pgbouncer/ (link found here: http://wiki.postgresql.org/wiki/PgBouncer)

2. Have a look at manual: http://pgbouncer.projects.postgresql.org/doc/usage.html

3. Create a pgbouncer.ini file:

```
[databases]
test = host=127.0.0.1 port=5432 dbname=test

[pgbouncer]
listen_port = 6543
listen_addr = 127.0.0.1
auth_type = md5
auth_file = C:\czarek\pgbouncer-1.5.2\users.txt
logfile = C:\czarek\pgbouncer-1.5.2\pgbouncer.log
pidfile = C:\czarek\pgbouncer-1.5.2\pgbouncer.pid
admin_users = testuser
```

4. Create users.txt:

"testuser" "password"

5. Test if it works:

```
pgbouncer.exe pgbouncer.ini
```

6. Now, to use pgbouncer pooling just change the port when connecting to database, use 6543 instead of 5432.


7. Register service so it starts automatically:

```
pgbouncer.exe -regservice pgbouncer.ini
net start pgbouncer
```

## Note ##

The "polish\_thesaurus" dictionary resolves some of the cases for synonyms, but it also breaks some other cases, for example 'liść @@ lisc'. You might want to drop this dictionary. Unfortunately you cannot handle all the cases for synonyms without tails, it is due to that postgresql allows to use only ISPELL type of dictionary, and word variations are defined using an algorithm, you can't write down all the cases for synonyms. If you need 100% effectiveness you can use [Sphinx search](http://sphinxsearch.com/), it allows to handle all the cases of variations and its synonyms, it supports mysql and postgresql databases.