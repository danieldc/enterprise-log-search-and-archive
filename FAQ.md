### Why am I not getting results for class:<some class> ###
Searches that only look for attributes, like class, require indexes that use RAM.  Since RAM is much more limited than disk, RAM-based indexes, called temporary indexes, are only available for a short period of time, usually around an hour.  In order to search the entire result set, you need to provide a keyword to search on, such as the host that sent the log, an IP address, port, anything that isn't "meta" information, such as the log class or program.
### How do I get more than 100 results returned? ###
Generally speaking you should filter your result to avoid having so many results returned.  This can be done either with the hyphen-term syntax `-term1 -term2` or adding in "must" conditionals `+mustbe1 +mustbe2`.  This will allow you to spend less time scrolling through your result set looking for something of interest.  If you can't filter your search like this, you can use the `limit` keyword to return an arbitrary amount of records with `limit:1000`.  Up to 9999 results can be returned this way into the browser.  More than 9999 will go into "bulk" mode in which the results are stored on the web server as a flat file and must be manually downloaded.  An email is sent with a link to that download file upon completion of the search.
### How do I drop all logs for a clean start? ###
```
#!/bin/sh
echo "" > /data/elsa/log/node.log
mysqladmin -f drop syslog
mysqladmin -f drop syslog_data
sh /usr/local/elsa/contrib/install.sh node set_node_mysql
/usr/local/sphinx/bin/indexer --config /usr/local/etc/sphinx.conf --rotate --all
```