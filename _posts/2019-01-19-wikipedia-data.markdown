---
layout: post
title:  "Getting Wikipedia data (with an example of extracting Wikipedia tables)."
date:   2019-01-19 00:08:00 +0700
categories: [dataset]
---


## Where do I get it?

**Firstly, DON'T crawl data from Wikipedia website.** It will take ages and if you're still not convinced, Wikipedia as a specific section on [why crawlers are bad](https://en.wikipedia.org/wiki/Wikipedia:Database_download#Why_not_just_retrieve_data_from_wikipedia.org_at_runtime?). Instead, Wikipedia lets you [download their database dump ](https://en.wikipedia.org/wiki/Wikipedia:Database_download#Where_do_I_get_it?). So, let's get your hand dirty and download the data from the [torent link](http://itorrents.org/torrent/3A8C87DE09C85193CFBCB10DC64B7A64C2CEE7FC.torrent) (get the first highlighted link if you're not sure). The XML database file is about 64GB after decompressed.

## Working with Wikipedia XML dumps 

The easiest way to read data is to import the XML file into a database using Wikipedia provided tools. Or, if you want to read the data directly from XML, check out [basic wikipedia parsing](https://www.heatonresearch.com/2017/03/03/python-basic-wikipedia-parsing.html) or [Attardi's wikiextractor](https://github.com/attardi/wikiextractor). 

Before we begin, let's create a database for storing the dump. In this post, I use MySQL but **it would be better to use PostgreSQL** since the database is quite large.

{% highlight shell %}
mysql -u <username> -p -e "CREATE DATABASE wiki;"
{% endhighlight %}

The SQL to initialize the database can be found [here](https://en.wikipedia.org/wiki/Wikipedia:Database_download#SQL_schema) (replace the raw [https://phab.wmfusercontent.org/.../tables.sql](https://phabricator.wikimedia.org/source/mediawiki/browse/master/maintenance/tables.sql) if you see error).

{% highlight shell %}
curl https://phab.wmfusercontent.org/file/data/dwg43cjkmehfovblakgl/PHID-FILE-bg5agl6teizmrfk5b6v2/tables.sql | mysql -u <username> -p wiki
{% endhighlight %}

Next, we're going to use the Java [MWDumpper](https://github.com/wikimedia/mediawiki-tools-mwdumper) to import the XML into the database.

{% highlight shell %}
git clone https://github.com/wikimedia/mediawiki-tools-mwdumper.git
cd mwdumper
mvn compile
mvn package
{% endhighlight %}

MWDumpper reads in the XML file and produces the query to insert data into the table in a pipeline manner. Therefore, you can filter these queries to get the data you want. You can also check out MWDumpper filter options.

{% highlight shell %}
cat enwiki-20181020-pages-articles-multistream.xml | java -jar mwdumper.jar --format=sql:1.25 | mysql -u <username> -p wiki
{% endhighlight %}

In my case, I only want to get the content of English articles. Thus, the full command I used to import the table **text** was

{% highlight shell %}
cat enwiki-20181020-pages-articles-multistream.xml | java -jar mwdumper.jar --format=sql:1.25 --filter=latest --filter=notalk --filter=namespace:0 | awk '/INSERT INTO text .*/{print}' | | mysql -u <username> -p wiki
{% endhighlight %}

## Parsing Wiki text

Now that we have the data at hand, we can start the fun part and parse the articles!!! 

Here is an example code to connect to the database and reading the articles. Since MySQL doesn't have good support for large data, we have to manually query partition of the data. **PostgreSQL would be a better choice**, also with better Python documentation.  

{% highlight python %}
import MySQLdb
from MySQLdb.cursors import SSCursor

db = MySQLdb.connect(host='localhost', user='root', password='', db='wiki', cursorclass=SSCursor)
cursor = db.cursor()

offset = 0
row_count = 1000
fetch_size = 100
while True:
    query = 'SELECT old_text FROM text LIMIT {}, {}'.format(offset, row_count)
    cursor.execute(query)

    for _ in range(row_count//fetch_size):
        rows = cursor.fetchmany(size=fetch_size)
        for record in rows:
            wikitext = str(record[0], 'utf8')
            # do your things...
    offset += row_count
{% endhighlight %}

Wikipedia use a special type of Markdown call WikiText which changes overtime and is not backward compatible. There are a bunch of parsers for it but [wikitextparser](https://github.com/5j9/wikitextparser) seems to me the best one for Python. It supports easy parsing and extracting of templates, sections, lists, etc. The example code shows how to extract tables and save them to CSV files.

{% highlight python %}
import wikitextparser as wtp

...
    wikitext = str(record[0], 'utf8')
    parsed = wtp.parse(wikitext)
    for table in parsed.tables:
        data = table.data() # data is a 2d list of table cells
        with open('out.csv', 'w') as out:
            writer = csv.writer(out)
            writer.writerows(data)
...
{% endhighlight %}

The full code will be available on my github soon. Happy training :-)
