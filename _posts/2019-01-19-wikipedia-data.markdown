---
layout: post
title:  "Getting Wikipedia data"
date:   2019-01-19 00:08:00 +0700
categories: [dataset]
---

If you are thinking about crawling data from Wikipedia, **don't**. Wikipedia dedicates a page for reasons why we shouldn't poke it. Instead, this post shows how to extract data from Wikipedia database dump.

{% highlight python %}
db = MySQLdb.connect(
    host='localhost', 
    user='root', 
    password='', 
    db='wiki', 
    cursorclass=SSCursor)
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
{% endhighlight %}
