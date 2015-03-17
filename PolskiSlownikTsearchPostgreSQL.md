# Polski słownik dla wyszukiwania pełnotekstowego przy użyciu modułu TSearch w PostgreSQL #

Tutaj pobierzesz polski słownik dla wyszukiwania pełnotekstowego w bazie PostgreSQL przy użyciu modułu TSearch. Słownik zawiera synonimy dla słów bez ogonków, także szukając "zlobek" czy "zlobkiem" znajdzie nam rekord "żłobek".

Wszystkie pliki i przykłady są gotowe do natychmiastowego działania, zainstalowanie, skonfigurowanie i przetestowanie TSearcha zajmie Ci nie więcej niż 5 minut.

Kodowanie w plikach słownika to UTF-8, także by działało to poprawnie kodowanie w bazie musi także być UTF-8. Można przekonwertować pliki do innych formatów komedną "iconv" pod linuksem, przykład konwertowania z utf-8 do iso-8859-2:

```
for f in polish.affix polish.dict polish.stop polish.syn; do cat $f | iconv -f utf-8 -t iso8859-2 > $f; done
```

## Pobierz ##

[tsearch\_data\_polish\_20120730.zip](http://tsearch-polish.googlecode.com/files/tsearch_data_polish_20120730.zip) (6.2 MB)

Wymagania: PostgreSQL 8.3 lub wyższy. Testowane głównie na PostgreSQL 9.1.

<a href='https://www.paypal.com/cgi-bin/webscr?cmd=_s-xclick&hosted_button_id=RLZT2JFRL6E88'><img src='https://www.paypalobjects.com/pl_PL/PL/i/btn/btn_donateCC_LG.gif' /></a>

## Instalacja ##

1. Rozpakuj zipa, znajdziesz katalog o nazwie "tsearch\_data\_polish", a wnim następujące pliki:

  * polish.affix
  * polish.dict (4.6 MB, główny słownik z regułami odmian słów)
  * polish.stop (słowa typu "i", "oraz", zawiera także wersje bez ogonków )
  * polish.syn (45 MB, synonimy słów bez ogonków, zawiera wszystkie odmiany słowa)
  * polish.ths (synonimy z ogonkami)

2. Skopiuj powyższe pliki do katalgu "PostgreSQL/9.1/share/tsearch\_data/".

3. Utwórz konfigurację i słowniki TSearch:

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

4. Sprawdź czy działa.

```
-- Uruchom każde z poniższych zapytań osobno, dla każdego powinna zostać 
-- zwrócona wartość 't' (od true) jako pomyślna, oraz 'f' (od false) jako porażka.

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

-- Zdebugujmy coś.

SELECT * FROM ts_debug('polish', 'żłobek');
SELECT * FROM ts_debug('polish', 'żłóbek');
SELECT * FROM ts_debug('polish', 'zlobek');
SELECT * FROM ts_debug('polish', 'zlobkiem');
SELECT * FROM ts_debug('polish', 'abramowowi');

```

## Przykład: tabela + dane + wyzwalacz + indeks + zapytanie ##

1. Utwórz tabelę i wstaw parę danych:

```
CREATE TABLE articles (
    body text NOT NULL,
    body_tsvector TSVECTOR
);
INSERT INTO articles (body) VALUES ('Żłobek nr 4');
INSERT INTO articles (body) VALUES ('Trafił żłobkiem w przedszkole');
INSERT INTO articles (body) VALUES ('Bylem sobie zlobkiem');
```

2. Uaktualnij wektor TSearcha dla danych:

```
UPDATE articles SET body_tsvector = to_tsvector('public.polish', body);
```

3. Stwórz wyzwalacz, zeby indeks był automatycznie uaktualniany przy operacjach typu INSERT/UPDATE.

```
CREATE TRIGGER articles_body_tsvector_trigger
BEFORE INSERT OR UPDATE ON articles
	FOR EACH ROW EXECUTE PROCEDURE tsvector_update_trigger(body_tsvector, 'public.polish', body);
```

4. Przyśpiesz wyszukiwanie tworząc indeks typu GIN na wektorze TSearch:

```
CREATE INDEX articles_body_tsvector_idx ON articles USING GIN( body_tsvector );
```

5. Teraz uruchomimy proste zapytanie:

```
SELECT body FROM articles
WHERE body_tsvector @@ plainto_tsquery('public.polish', 'zlobkiem');
```

6. Tu bardziej złożone zapytanie z rankingiem:

```
SELECT body, ts_rank('{1,0.9,0.8,0.7}', body_tsvector, q) as rank
FROM articles, plainto_tsquery('public.polish', 'zlobek') as q
WHERE body_tsvector @@@ q
ORDER BY rank DESC
```



## Wydajność ##

Jest jeden drobny problem ze słownikami TSearch, otóż nie są one cachowane w pamięci, każde nowe połączenie z bazą wczytuje pliki słownika od nowa, w przypadku naszego słownika te pliki mają aż 50 MB, może to potrwać kilka sekund i zajmuje znaczne ilości pamięci, około 300 MB.

Rozwiązaniem na to jest użycie puli połączeń ("connection pooling"), spójrz na projekt [pgBouncer](http://pgfoundry.org/projects/pgbouncer/).

Instrukcje do zainstalowania pgBouncer na Windows:

1. Zgraj binarkę: http://winpg.jp/~saito/pgbouncer/ (link znaleziony tutaj: http://wiki.postgresql.org/wiki/PgBouncer)

2. Zerknij na oficjalną instrukcję: http://pgbouncer.projects.postgresql.org/doc/usage.html

3. Utwórz plik "pgbouncer.ini" (zmień ścieżki etc):

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

4. Utwórz plik "users.txt" a w nim wpisz użytkownika i hasło:

"testuser" "password"

5. Sprawdź czy działa:

```
pgbouncer.exe pgbouncer.ini
```

6. Teraz by używać pgBouncer po prostu zmień port podczas łączenia się z bazą, z 5432 na 6543.

7. Zarejestruj usługę by startowała automatycznie:

```
pgbouncer.exe -regservice pgbouncer.ini
net start pgbouncer
```

## Uwaga ##

Słownik "polish\_thesaurus" rozwiązuje niektóre z przypadków dla wyszukiwania synonimów, ale przy okazji psuje działanie w innych przypadkach, dla przykładu 'liść @@ lisc'. Możesz chcieć usunąć ten słownik. Niestety, ale w postgresie nie da się zapisać wszystkich przypadków dla synonimów bez ogonków, jest to spowodowane tym, że postgres pozwala jedynie używać słownika typu ISPELL, a odmiany w nim zapisywane są algorytmem, nie ma wypisanych wszystkich przypadków odmiany, i przez to nie można zapisać wszystkich przypadków dla synonimów. Jeżeli potrzebujesz 100% skuteczności zostaje skorzystać ze [Sphinx search](http://sphinxsearch.com/), który pozwala zapisać wszystkie przypadki odmian i ich synonimów, wspiera bazy mysql i postgresql.