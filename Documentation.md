# Introduction #
Enterprise Log Search and Archive is a solution to achieve the following:
  * Normalize, store, and index logs at unlimited volumes and rates
  * Provide a simple and clean search interface and API
  * Provide an infrastructure for alerting, reporting and sharing logs
  * Control user actions with local or LDAP/AD-based permissions
  * Plugin system for taking actions with logs
  * Exist as a completely free and open-source project

ELSA accomplishes these goals by harnessing the highly-specialized strengths of other open-source projects:  Perl provides the glue to asynchronously tie the log receiver (Syslog-NG) together with storage (MySQL) and indexing (Sphinx Search) and serves this over a web interface provided either by Apache or any other web server, including a standalone pure-Perl server for a lighter footprint.

# Table of Contents #



# Why ELSA? #
I wrote ELSA because commercial tools were both lacking and cost prohibitive.  The only tool that provided the features I needed was Splunk.  Unfortunately, it was cost prohibitive and was too slow to receive the log volume I wanted on the hardware I had available.  ELSA is inspired by Splunk but is focused on speed versus dashboards and presentation.

In designing ELSA, I tried the following components but found them too slow.  Here they are ordered from fastest to slowest for indexing speeds (non-scientifically tested):
  1. Tokyo Cabinet
  1. MongoDB
  1. TokuDB MySQL plugin
  1. Elastic Search (Lucene)
  1. Splunk
  1. HBase
  1. CouchDB
  1. MySQL Fulltext

# Capabilities #

ELSA achieves _n_ node scalability by allowing every log receiving node to operate completely independently of the others.  Queries from a client through the API against the nodes are sent in parallel so the query will take only the amount of time the of the longest response.  Query results are aggregated by the API before being sent to the client as a response.  Response times vary depending on the number of query terms and their selectivity, but a given node on modest hardware takes about one half second per billion log entries.

Log reception rates greater than 50,000 events per second per node are achieved through the use of a fast pattern parser in Syslog-NG called PatternDB.  The pattern parser allows Syslog-NG to normalize logs without resorting to computationally expensive regular expressions.  This allows for sustained high log reception rates in Syslog-NG which are piped directly to a Perl program which further normalizes the logs and prepares large text files for batch inserting into MySQL.  MySQL is capable of inserting over 100,000 rows per second when batch loading like this.  After each batch is loaded, Sphinx indexes the newly inserted rows in temporary indexes, then again in larger batches every few hours in permanent indexes.

Sphinx can create temporary indexes at a rate of 50,000 logs per second consolidate these temporary indexes at around 35,000 logs per second, which becomes the terminal sustained rate for a given node.  The effective bursting rate is around 100,000 logs per second, which is the upper bound of Syslog-NG on most platforms.  If indexing cannot keep up, a backlog of raw text files will accumulate.  In this way, peaks of several hours or more can be endured without log loss but with an indexing delay.

The overall flow diagram looks like this:

Live, continuously:

Network → Syslog-NG (PatternDB) → Raw text file

or

HTTP upload → Raw text file

Batch load (by default every minute):

Raw text file → MySQL → Sphinx

# Installation #
Installation is done by running the install.sh file obtained either by downloading from the sources online or grabbing from the install tarball featured on the ELSA Google Code home page.  When install.sh runs, it will check for the existence of /etc/elsa\_vars.sh to see if there are any local customizations, such as passwords, file locations, etc. to apply.  The install.sh script will update itself if it finds a newer version online, so be sure to store any changes in /etc/elsa\_vars.sh.  The install.sh script should be run separately for a node install and a web install.  You can install both like this: ` sh install.sh node && sh install.sh web`.  Installation will attempt to download and install all prerequisites and initialize databases and folders.  It does not require any interaction.

Currently, Linux and FreeBSD 8.x are supported, with Linux distros based on Debian (including Ubuntu), RedHat (including CentOS), and SuSE tested.  install.sh should run and succeed on these distributions, assuming that the defaults are chosen and that no existing configurations will conflict.

## Updates ##
Updating an installation is done via the install.sh file (assuming your ELSA directory is /usr/local/elsa): `sh /usr/local/elsa/contrib/install.sh node update && sh /usr/local/elsa/contrib/install.sh web update`.  This will check the web for any updates and apply them locally, taking into account local customizations in /etc/elsa\_vars.sh.

# Plugins #
ELSA ships with several plugins:
  * Windows logs from [Eventlog-to-Syslog](http://eventlog-to-syslog.googlecode.com)
  * Snort/Suricata logs
  * Bro logs
  * Url logs from [httpry\_logger](http://enterprise-log-search-and-archive.googlecode.com/files/httpry_logger.pl)
These plugins tell the web server what to do when a user clicks the "Info" link next to each log.  It can do anything, but it is designed for returning useful information in a dialog panel in ELSA with an actions menu.  An example that ships with ELSA is that if a [StreamDB](http://streamdb.googlecode.com) URL is configured (or OpenFPC) any log that has an IP address in it will have a "getPcap" option which will autofill pcap request parameters for one-click access to the traffic related to the log being viewed.

New plugins can be added easily by subclassing the "Info" Perl class and editing the elsa\_web.conf file to include them.  Contributions are welcome!

# File Locations #

The main ELSA configuration files are /etc/elsa\_node.conf and /etc/elsa\_web.conf.  All configuration is controlled through these files, except for query permissions which are stored in the database and administrated through the web interface.  Nodes read in the elsa\_node.conf file every batch load, so changes may be made to it without having to restart Syslog-NG.

Most Linux distributions do not ship recent versions of Syslog-NG.  Therefore, the install compiles it from source and installs it to $BASE\_DIR/syslog-ng with the configuration file in $BASE\_DIR/syslog-ng/etc/, where it will be read by default.  By default, $BASE\_DIR is /usr/local and $DATA\_DIR is /data.  Syslog-NG writes raw files to $DATA\_DIR/elsa/tmp/buffers/<random file name> and loads them into the index and archive tables at an interval configured in the elsa\_node.conf file, which is 60 seconds by default.  The files are deleted upon successful load.  When the logs are bulk inserted into the database, Sphinx is called to index the new rows.  When indexing is complete, the loader notes the new index in the database which will make it available to the next query.  Indexes are stored in $DATA\_DIR/sphinx and comprise about as much space as the raw data stored in MySQL.

Archive tables typically compress at a 10:1 ratio, and therefore use only about 5% of the total space allocated to logs compared with the index tables and indexes themselves.  The index tables are necessary because Sphinx searches return only the ID's of the matching logs, not the logs themselves, therefore a primary key lookup is required to retrieve the raw log for display.  For this reason, archive tables alone are insufficient because they do not contain a primary key.

If desired, MySQL database files can be stored in a specified directory by adding the "mysql\_dir" directive to elsa\_node.conf and pointing it to a folder created which has proper permissions and SELinux/apparmor security settings.

## Hosting all files locally ##
If your ELSA web server will not have Internet access, you will need to host the Javascript for the web pages locally.  To do this, after installing:
```
cd /usr/local/elsa/web/inc
wget "http://yuilibrary.com/downloads/yui2/yui_2.9.0.zip"
unzip yui_2.9.0.zip
```
Edit the elsa\_web.conf file and set yui/local to be "inc" and comment out "version" and "modifier."

### Caveats for Local File Hosting ###
If Internet access is not available, some plugins will not function correctly.  In particular the whois plugin uses an external web service to do lookups, and these will not be possible without Internet connectivity.  In addition, dashboards will not work if the client's browser does not have connectivity to Google to pull down their graphing library.

# Web Server #

The web frontend is typically served with Apache, but the Plack Perl module allows for any web server to be used, including a standalone server called Starman which can be downloaded from CPAN.  Any implementation will still have all authentication features available because they are implemented in the underlying Perl.

The server is backended on the ELSA web database, (elsa\_web by default), which stores user information including permissions, query log, stored results, and query schedules for alerting.

Admins are designated by configuration variables in the elsa\_web.conf file, either by system group when using local auth, or by LDAP/AD group when using LDAP auth.  To designate a group as an admin, add the group to the array in the configuration.  Under the “none” auth mode, all users are admins because they are all logged in under a single pseudo-username.

The web server is required for both log collectors and log searchers (node and web) because searches query nodes (peers) using a web services API.

# Configuration #

Most settings in the elsa\_web.conf and elsa\_node.conf files should be fine with the defaults, but there are a few important settings which need to be changed depending on the environment.
## elsa\_web.conf: ##
  * Nodes: Contains the connection information to the log node databases which hold the actual data.
  * Auth\_method: Controls how authentication and authorization occurs.  For LDAP, the ldap settings must also be filled out.
  * Link\_key: should be changed to something other than the default.  It is used to salt the auth hashes for permalinks.
  * Email: For alerts and archive query notifications, you need to setup the email server to use.  If you wish to get the actual results from an alert, in addition to a link to the results, add the following config to the email section:
```
 "email": {
    "include_data": 1
  }
```
  * Meta\_db: Should point to the database which stores the web management information.  This can reside on a node, but probably shouldn't.  The performance won't be much of a factor, so running this locally on the web server should be fine.
  * Excluded\_classes: If you want to remove some classes from the menus and searches altogether, configure the config entry for excluded\_classes like this:
```
  "excluded_classes": {
    "BRO_SSL": 1
  },
```
  * APIKeys: The "apikeys" hash holds all known username/apikey combinations, such as:
```
"apikeys": { "elsa": "abc" }
```
  * Peers: Configuration for how this ELSA node will talk to other ELSA nodes.  Note that a configuration for itself (127.0.0.1) is required for any query to complete.  An example configuration is:
```
"peers": {
  "127.0.0.1": {
    "url": "http://127.0.0.1/",
    "user": "elsa",
    "apikey": "abc"
  }
}
```
  * Default OR: By default, all search terms are required to be found in the event to constitute a match (AND).  If you wish, you can set the config value "default\_or" to a true value to change the default behavior to making the search match if any of the given values are true:
```
"default_or": 1
```

## elsa\_node.conf: ##
  * Database: Edit the connection settings for the local database, if non-default.
  * Log\_size\_limit: Total size in bytes allowed for all logs and indexes.
  * Sphinx/perm\_index\_size: This setting must be tweaked so that perm\_index\_size number of logs come into the system before (num\_indexes `*` sphinx/index\_interval) seconds pass.
  * Archive/percentage: Percentage of log\_size\_limit reserved for archive.
  * Archive/days: Max number of days to retain logs for in the archive
  * Sphinx/days: Max number of days to retain logs for in the indexes
  * forwarding/forward\_only: This node will only forward logs and not index them.
  * forwarding/destinations: An array of hashes of forwarders, as detailed in the Forwarding section.

## Forwarding Logs ##
ELSA can be setup to forward (replicate) logs to an unlimited number of destinations in several ways:
| Method | Config Directive |
|:-------|:-----------------|
| File Copy | `cp`             |
| SSH    | `scp`            |
| HTTP/S | `url`            |

### File Copy ###
Configuration options:
| Option | Meaning | Required |
|:-------|:--------|:---------|
| `dir`  | Directory to copy the file to.  This can be a destination where backup agent reads from or an NFS mount. | Yes      |

### SSH ###
Configuration options:
| Option | Meaning | Required |
|:-------|:--------|:---------|
| `user` | Username for SSH | Yes      |
| `password` | Password for the user | If no `key_path` |
|  `key_path` | Path for RSA/DSA keypair files (.pub) | If no `password` |
| `host` | IP or DNS name of host to forward to | Yes      |
| `dir`  | Remote directory to copy to | Yes      |

### URL ###
Configuration items:
| Option | Meaning | Required |
|:-------|:--------|:---------|
| `url`  | Full URL, (including https://), of where to send logs | Yes      |
| `verify_mode` | Boolean indicating whether strict SSL certificate checking is to be enforced.  Use zero for certificates that don't have a trusted certificate authority on the forwarder (default self-signed, for instance) | No       |
| `timeout` | Number of seconds to issue a timeout on. Defaults to zero (no timeout) | No       |
| `ca_file` | SSL certificate authority file to use to verify the remote server's certificate | No       |
| `cert_file` | Client-side SSL certificate the server may require to verify the client's identity | No       |
| `key_file` | Key corresponding with `cert_file` | No       |

An example forwarding configuration may look like this:
```
"forwarding": {
  "forward_only": "1",
  "destinations": [
    { "method": "url", "url": "http://example.com/API/upload" },
    { "method": "url", "url": "https://secure.example.com/API/upload", "ca_file": "/etc/mycafile.pem" }
  ]
}
```

## Low volume configuration tuning ##
If your ELSA node isn't receiving many logs (less than a few hundred per minute), you may need to tune your setup so that permanent indexes aren't underutilized.  There are at most `num_indexes` number of permanent indexes, and if there isn't a free one available, the oldest one will be overwritten.  If this happens before the `log_size_limit` has been reached, then it means that you rolled logs before you wanted to.  This means you need to tweak some settings in `elsa_node.conf`:

  * Increase num\_indexes to something larger like 400
  * Increase `allowed_temp_percent` to 80

This should give you .8 x 400 x 60 seconds of time before temp indexes get rolled into a perm index, and should give you more perm indexes before they get rolled.  With 400 perm indexes, that should be more than 88 days of possible index time.  If that's still not enough, move index\_interval up from 60 seconds to something larger (this will extend the "lifetime" of a temp index).

If you set `num_indexes` to be larger than 200, you should increase the open files limit for searchd (Sphinx).  You can do this on Linux by editing `/etc/security/limits.conf` and adding:
```
root soft nofile 100000
root hard nofile 200000
```

Then logout, login, and restart searchd.

### Changing num\_indexes ###
If you change the ` num_indexes ` setting in /etc/elsa\_node.conf, you will need to regenerate the ` /usr/local/etc/sphinx.conf ` file.  To do so, either delete or move the existing sphinx.conf file and then run:
```
echo "" | perl /usr/local/elsa/node/elsa.pl -on
pkill searchd
/usr/local/sphinx/bin/searchd --config /usr/local/etc/sphinx.conf
```

This will regenerate the config file using the new ` num_indexes ` value.  There is one last step that needs to be taken, and that is to instantiate the actual Sphinx files by running indexer on these previously non-existent files.  This step depends on what the new value of ` num_indexes ` is.  In this example, we have changed ` num_indexes ` from 200 to 400, so we need to instantiate indexes 201 through 400.  We do this thusly:

```
for COUNTER in `seq 201 400`; do /usr/local/sphinx/bin/indexer --config /usr/local/etc/sphinx.conf temp_$COUNTER perm_$COUNTER; done
```

Now, restart searchd and the new indexes should be available.

## Making changes to syslog-ng.conf ##
install.sh will use /usr/local/elsa/node/conf/syslog-ng.conf as a template, using /etc/elsa\_syslog-ng.conf (if it exists) as a reference for any persistent changes, and write the combination to /usr/local/syslog-ng/etc/syslog-ng.conf which is what is actually run.  So, put any local changes in /etc/elsa\_syslog-ng.conf to make sure they survive an update.  Keep in mind that the file is included before the log {} statements, so you can redefine sources and destinations there, or put in additional log {} statements.

## Firewall Settings ##
|Source|Destination|Port|
|:-----|:----------|:---|
|Web Clients|Web Node   |TCP 80/443|
|Web Node|LDAP/AD Server|TCP 389/636|
|Web Node|Log Node   |TCP 3306 **_deprecated_**|
|Web Node|Log Node   |TCP 9306 (formerly 3307) **_deprecated_**|
|Web Node|Log Node   |TCP 80/443|
|Log Clients|Log Node   |TCP/UDP 514|

## API Keys ##
The literal structure of an APIKey as it is transmitted is in the form of an HTTP Authorization header.  The format is this:
` Authorization: ApiKey <username>:<current epoch timestamp>:<SHA512 hex digest of timestamp concatenated with configured API key> `

As an example, if the API key were "abc," then the request would look like this for a user of "myuser" and a timestamp of 1364322947 would be:

<pre>Authorization: ApiKey myuser:1364322947:05e84771a03cf3aaf88e947e915f73b4ef3685a382f8ca603b787168eb464a06eb178a908b868832af6ff913ca9b096880c4f4089bc4e0585fe6ac40e29f061d</pre>

To revoke an API key, simply remove that username from the list of "apikeys" in elsa\_web.conf or change the key for that username to reset it.

# Permissions #

Log access is permitted by allowing certain groups either universal access (admins) or a whitelist of attributes.  The attributes can be log hosts (the hosts that initially generate the logs), ranges of hosts (by IP), log classes, or log nodes (the nodes that store the logs).  Groups can be either AD groups or local system groups, as per the configuration.  Those in the admins group have the "Admin" drop-down menu next to the "ELSA" drop down menu in the web interface which has a "Manage Permissions" item which opens a new window for administrating group privileges.

# Queries #
## Syntax ##
Query syntax is loosely based on Google search syntax.  Terms are searched as whole keywords (no wildcards).  Searches may contain boolean operations specifying that a term is required using the plus sign, negating using the minus sign, or no sign indicating that it is an “OR.”  Parenthesis may be used to group terms.  Numeric fields, including hosts, may have greater than or less than (and equal to) operators combined with the boolean operators.

### Boolean Operators ###

| Operator | Meaning |
|:---------|:--------|
| keyword  | Query MUST include the keyword |
| -keyword | Query MUST NOT include the keyword |
| OR keyword | Query MAY include the keyword |

### Range Operators ###
Range operators can only be used to filter search results, not provide the results to be filtered.  That is, you must include a keyword in addition to the range operator.  You can provide a single range operator; they do not need to be in pairs.

| Operator | Meaning |
|:---------|:--------|
| attribute>value | Attribute MAY be greater than value |
| attribute<value | Attribute MAY be less than value |
| attribute>=value | Attribute MAY be greater than or equal to value |
| attribute<=value | Attribute MAY be less than or equal to value |
| +attribute>value | Attribute MUST be greater than value |
| +attribute<value | Attribute MUST be less than value |
| +attribute>=value | Attribute MUST be greater than or equal to value |
| +attribute<=value | Attribute MUST be less than or equal to value |
| -attribute>value | Attribute MUST NOT be greater than value |
| -attribute<value | Attribute MUST NOT be less than value |
| -attribute>=value | Attribute MUST NOT be greater than or equal to value |
| -attribute<=value | Attribute MUST NOT be less than or equal to value |

### Transforms ###
Queries can have transforms applied to them.  Transforms are covered later in the documentation.  The syntax for using transforms is represented below.
| Term | Meaning|
|:-----|:-------|
| search clause | Any combination of keywords and filters, as defined above |
| transform name | Name of the transform as defined in the transform plugin |
| param | Parameter supplied to the transform to direct its behavior |

` <search clause> [ | <transform name>([param1,param2,paramN]) ] [ | <transform name>([param1,param2,paramN]) ]  `

### Directives ###
Queries have a number of modifiers in the form of directives which instruct ELSA how to query.
| Term | Meaning | Default Value | Query can Batch | Example |
|:-----|:--------|:--------------|:----------------|:--------|
| limit | Return this number of results.  A limit of zero means return an unlimited number, which constitutes a bulk query and forces the query to run in batch mode, with results delivered via a link in an email. | `100`         | Batch can occur when limit set to 0 or > Max matches (default is 1000) | `limit:1000` |
| cutoff | Like limit, except it tells ELSA to stop searching after finding this many records, which is valuable when searching a common term and the total number of hits (as opposed to total returned) is irrelevant. | undefined     | No              | `cutoff:100` |
| offset | Partners with limit to indicate how far into a result set to go before returning results.  Meaningless unless a limit larger than the default 100 is used. | `0`           | No              | `offset:900` |
| orderby | Order results by this attribute. | Technically, undefined, but effectively timestamp, ascending in most scenarios. | No              | `orderby:host` |
| orderby\_dir | Direction to order results.  Must be used in conjunction with orderby. | `asc`         | No              | `orderby_dir:desc` |
| start | Quoted value representing the earliest timestamp to return.  Valid values are almost any date representation.  See details for the complete documentation [here](http://search.cpan.org/~sbeck/Date-Manip-6.39/lib/Date/Manip/Date.pod#VALID_DATE_FORMATS). | undefined     | No              | `end:"2013-01-01 00:00:00"` |
| end  | Quoted value representing the latest timestamp to return.  Valid values are as with start. | undefined     | No              | `end:"2013-01-01 00:00:00"` |
| groupby | Similar to SQL GROUP BY, returns the unique values for a given attribute and the count of the distinct values. | undefined     | No              | `groupby:host` |
| node | Apply a filter for results only from this node (subject to boolean representations as detailed above. | undefined     | No              | `node:192.168.1.1` |
| datasource | Query the given datasource as configured in the elsa\_web.conf file. | undefined     | No              | `datasource:hr` |
| timeout | Stop querying and return any results found after this number of seconds. | `300`         | No              |  `timeout:10` |
| archive | If set to a true value, query will be run on archived data instead of indexed data, batching if the estimated query time exceeds the configured value (with a default of 30 seconds). | `0`           | Yes, if estimated time is > query\_time\_batch\_threshold (30 seconds by default) | `archive:1` |
| analytics | If set to a true value, the query will automatically be batched and have no limit set.  Results will be saved to a bulk result file, with a link to that file emailed. | `0`           | Yes, always     | `analytics:1` |
| nobatch | Run the query in the foreground, regardless of the estimated time it will take. | `0`           | No, never       | `nobatch:1` |
| livetail | _Deprecated_ |               |                 |

### Query examples ###
Queries can be very simple, like looking for any mention of an IP address:
```
10.0.20.1
```
Or a website
```
site:www.google.com
```
Here is an example query for finding Symantec Anti-Virus alerts on Windows logs on ten hosts that does not contain the keyword “TrackingCookie”
```
eventid:51 host>10.0.0.10 host<10.0.0.20 -TrackingCookie
```
One could also look for account lockouts that do not come from certain hosts:
```
class:windows locked -host>10.0.0.10 -host<10.0.0.20
```
To see what hosts have had lockout events, one could run:
```
class:windows ”locked out”
```
and choose the ANY.host field from the “Report On” menu.
Here's an example showing hits from website example.com or website bad.com:
```
site:example.com OR site.bad.com
```


## Ordering ##
You can change the column used to order your query as well as the direction using the `orderby` and `orderby_dir` keywords.  For instance, to order a query by host in reverse order, use: `orderby:host orderby_dir:desc`.  The default is `orderby:timestamp orderby_dir:ASC`.

## Keywords ##
Keywords are the words indexed and available for searching.  Note that you cannot search for a partial keyword, it must be complete.  Also note that keywords are comprised of not only alpha-numeric words, but also hyphens, dots, and at-signs.  So, these are all complete keywords:
```
1.1.1.1
this-example.com
me@example.com
mal.form.ed-.ip.addr
```
Searches for 1.1 or example.com or ip.addr would all fail to find these terms.  If you need to perform searches on partial keywords, you need to switch from an index query to an archive query by clicking the "Index" pull-down menu and choosing archive.  Keep in mind that archive searches are slow, so narrowing down a time period will help significantly.

## Search data flow ##
When the API issues the query, it is parsed and sent to Sphinx.  It then receives the log ID's that match and the API queries MySQL for those ID's:

Query → Parse → Authorize → Log → Sphinx → MySQL → Aggregation → Presentation

# Archive Queries #

Queries for logs in the archive tables take much longer than indexed queries.  For this reason, they are run in the background and the requester is notified via email when the query results are ready.  The results are viewed through the link in the email or through the web interface menu for “Saved Results.”  Archive queries are run exactly like normal queries except that the “Index” toggle button is changed to “Archive.”  They may be performed on the same time range available in the indexed logs as a way of performing wildcard searches not restricted to a keyword.  For example, if it was necessary to find a log matching a partial word, one could run an archive search with a narrow time selection.  A user may only run a single archive query at a time to prevent system overload.  In addition, there is a configuration variable specifying how many concurrent users may run an archive query (the default is four).  Most systems can search about 10 million logs per minute per node from the archive.  The overall flow looks like this:

Archive Query → Parse → Authorize → Log → Batch message to user
> (then in background)	→ MySQL → Store in web MySQL → Email

# Web Services API #
Web services currently expose three URL's for use in external apps or between nodes:
  * /API/query
  * /API/stats
  * /API/upload

/API/query parameters:
  * q: The literal query JSON object comprised of query\_string and query\_meta\_params
  * permissions: The literal JSON object containing ELSA permissions
  * peer\_label: The optional label to assign results from this peer (so they aren't "127.0.0.1")

/API/stats parameters:
  * start: Start date of stats
  * end: End date of stats

/API/upload parameters:
  * filename: Name of the file being uploaded.  Must have the header "Content-disposition" with "attachment; filename=" in the client request.
  * start: Client declared start of the log data
  * end: Client declared end of the log data
  * md5: Client calculated MD5 of the entire file.  This **must** match what the server calculates.
  * count: declared count of records by the client
  * total\_errors: (optional) error count
  * batch\_time: (optional) time period over which the records were collected, in seconds (e.g. 60)
Required headers in addition to Authorization:
  * Content-type: form-data

## Imports ##
You can also use /API/upload to invoke imports which allows you to POST arbitrary, unparsed files and have them imported properly.  To do so, you need to set the above parameters for /API/upload, plus:
  * name: (required) Arbitrary name to give this upload, filename works well, does not have to be unique.
  * description: Arbitrary description, can include original client IP, etc.
  * format: Specific format to use for parsing this file (optional in some cases)
  * program: Optional syslog program name to assign, necessary for certain parsers.

### Importer Plugins ###
You can add custom importer plugins for parsing log messages.  Plugins do any necessary data preparation as well as the actual loading or queueing of data.  They declare what formats they parse (to allow for an exact match with a provided "format" POST parameter), as well as an optional heuristic method which will assign a score for that parser as to what priority it should be given when ELSA determines the best parser to use for an imported log which doesn't declare its format.  See the plugins in `elsa/web/lib/Importer/` for examples.

# Alerts #

Any query that has been run may be turned into an alert by clicking the “Results...” menu button and choosing “alert.”  This will execute the exact same search after every new batch of logs is loaded, and will notify the user via email of new hits in a manner similar to the archive search results.

# Scheduled Queries #

Any query may be scheduled to run at a specified interval.  This can be useful for creating daily or hourly reports.  Creating the scheduled query is similar to creating the alert in that you choose the option from the “Results...” button after performing a search you wish to create a report from.

# Command-line Interface and API #

ELSA ships with a command-line interface, elsa/web/cli.pl, which can be run when logged in on the web frontend from the shell.  This can be helpful for testing or piping results to other programs.  However, the Perl API provides a much more comprehensive method for accessing ELSA in a scripted fashion.  You can use the included cli.pl as an example for using the API.

# Performance Tuning #
## Node ##
### Hardware ###
For very high logging levels, there are many factors which can affect overall throughput and search response times.  Hardware is a major factor, and disk speed is the biggest consideration.  ELSA should only ever use a maximum of three CPU's at a time (during index consolidation) and usually uses only one.  However, Sphinx is a threaded application, and will happily scale to as many CPU's are on the box for running queries in parallel.  For this reason, four CPU's is recommended for production systems with a high load.  RAM is also a factor, but less so.  2-4 GB should be enough.  Searchd, the Sphinx daemon, will consume most of the RAM with its temporary indexes.

### Filesystem Considerations ###
Ext4 is currently the default filesystem on most recent Linux distributions.  It is a happy medium between ReiserFS which is good for small files and XFS which excels at large files.  ELSA deals mostly with large files, so it is recommended to create a dedicated partition for your $DATA\_DIR and format it with XFS, but ext4 should not hinder performance much.

### MySQL ###
ELSA batch loads files using "LOAD DATA INFILE" and so it benefits from a few configuration file changes to enlarge the batch loading buffer:
```
[mysqld]
bulk_insert_buffer_size = 100M
```
In addition, MySQL 5.5 in general offers significant performance increases over previous versions in many areas, though those optimizations will be more apparent on the database serving the web interface versus the log node databases.

### VMware Considerations ###
If you are running ELSA as a VM, you should set the disk to the "high" setting to get the best performance.

## Web ##
The web server itself should not need any special performance tuning.  Web clients, however, are highly encouraged to use either Firefox or Chrome for browsers because of the heavy use of Javascript on the page.
# Monitoring #

You can use the "Stats" page under the "Admin" menu on the web interface to see what ELSA's usage looks like.  To diagnose problems, refer to the $DATA\_DIR/elsa/log directory, especially the node.log and web.log files, respectively.

You may also want to look for network problems on nodes, especially kernel drops.  You can errors like this with this command:
```
netstat -s | grep -i errors
```
Look at the whole output of "netstat -s" for context if you see errors.

It may also be a good idea to establish a log that you know should periodically occur.  Then do a query on the web interface and report on a time value, such as hour or day, and look for any fluctuations in that value that could indicate log loss.

# Adding Parsers #
In order to add parsers, you need to add patterns to the patterndb.xml file.  If you need to create new log classes and fields, it's not too hard, but right now there is no web interface (that's planned in the future).  You'll need to add classes to the "classes" table, fields to the "fields" table, then use the offsets listed under $Field\_order in web/lib/Fields.pm to create the right entries in "fields\_classes\_map."  Please note that the class name **MUST** be upper-case.  Other than those few database entries, adding the pattern and restarting syslog-ng and apache is all you have to do.  The new fields
will show up in the web interface, etc.  If you can, try to create patterns which re-use existing classes and fields, then just dropping them into the patterndb.xml file will instantly make them parse correctly-no DB work or restarts needed.  I plan on making a blog post on how to do this soon, but let me know if you run into any troubles.  Here's an example to get you started:

Example log
program:
`test_prog`
message:
`source_ip 1.1.1.1 sent 50 bytes to destination_ip 2.2.2.2 from user joe`
Pick a class\_id greater than 10000 for your own custom classes.  Let's
say this is the first one, so your new class\_id will be 10000.
What to insert into syslog database on log node:
```
INSERT INTO classes (id, class) VALUES (10000, "NEWCLASS");
```
Our fields will be conn\_bytes, srcip, and dstip, which already exist
in the "fields" table as well as "myuser" which we will create here
for demonstration purposes:
```
INSERT INTO fields (field, field_type, pattern_type) VALUES ("myuser",
"string", "QSTRING");

INSERT INTO fields_classes_map (class_id, field_id, field_order)
VALUES ((SELECT id FROM classes WHERE class="NEWCLASS"), (SELECT
id FROM fields WHERE field="srcip"), 5);
INSERT INTO fields_classes_map (class_id, field_id, field_order)
VALUES ((SELECT id FROM classes WHERE class="NEWCLASS"), (SELECT
id FROM fields WHERE field="conn_bytes"), 6);
INSERT INTO fields_classes_map (class_id, field_id, field_order)
VALUES ((SELECT id FROM classes WHERE class="NEWCLASS"), (SELECT
id FROM fields WHERE field="dstip"), 7);
```
Now the string field "myuser" at field\_order 11, which maps to the
first string column "s0":
```
INSERT INTO fields_classes_map (class_id, field_id, field_order)
VALUES ((SELECT id FROM classes WHERE class="NEWCLASS"), (SELECT
id FROM fields WHERE field="myuser"), 11);
```
5, 6, and 7 correspond to the first integer columns in the schema
"i0," "i1," and "i2."  In the pattern below, we're extracting the data
and calling it i0-i2 so that it goes into the log database correctly.
The above SQL maps the names of these fields in the context of this
class to those columns in the raw database when performing searches.

Example pattern:
```
<ruleset name="does_not_matter" id='does_not_matter_either'>
                <pattern>test_prog</pattern>
                <rules>
                        <rule provider="does_not_matter" class='21' id='21'>
                                <patterns>
                                        <pattern>source_ip @IPv4:i0:@ sent @ESTRING:i1: @bytes to destination_ip @IPv4:i2:@ from user @ANYSTRING:s0:@</pattern>
                                </patterns>
                                <examples>
                                  <example>
                                    <test_message program="test_prog">source_ip 1.1.1.1 sent 50 bytes to destination_ip 2.2.2.2 from user joe</test_message>
                                    <!-- srcip -->
                                    <test_value name="i0">1.1.1.1</test_value>
                                    <!-- conn_bytes -->
                                    <test_value name="i1">50</test_value>
                                    <!-- dstip -->
                                    <test_value name="i2">2.2.2.2</test_value>
                                    <!-- myuser -->
                                    <test_value name="s0">joe</test_value>
                                  </example>
                                </examples>
                        </rule>
                </rules>
        </ruleset>
```
Add this in the patterndb.xml between the 

&lt;patterndb&gt;



&lt;/patterndb&gt;


elements.  You can test this on a log node using the
/usr/local/syslog-ng/bin/pdbtool utility like so:
```
/usr/local/syslog-ng/bin/pdbtool test -p /usr/local/elsa/node/conf/patterndb.xml
```
This should print out all of the correct test values.  You can test it against example messages as well like this:
```
/usr/local/syslog-ng/bin/pdbtool match -p /usr/local/elsa/node/conf/patterndb.xml -P test_prog -M "source_ip 1.1.1.1 sent 50 bytes to destination_ip 2.2.2.2 from user joe"
```

After the patterndb.xml file and the database are updated, you will need to restart syslog-ng:
```
service syslog-ng restart
```
If you are already logged into ELSA, simply refreshing the page should make those new classes and fields available.

# Transforms #
ELSA has a powerful feature called transforms which allow you to pass the results of a query to a backend plugin.  The plugins that currently ship with ELSA include whois, dnsdb, and CIF (Collective Intelligence Framework).  There are also utility transforms filter, grep, and sum.
## Syntax ##
Transforms are modeled after UNIX-style command pipes, like this:
```
site:www.google.com | whois | sum(descr)
```
This command finds all URL requests for site www.google.com, passes those results to the whois plugin which attaches new fields like org and description, and then passes those results to the sum transform which takes the argument "descr" indicating which field to sum.  The result is a graph of the unique "descr" field as provided by the whois plugin.

Plugins take the syntactical form of:
```
query | plugin_1(arg1,arg2,argn) | plugin_n(arg1,arg2,argn)
```

## Current Plugins ##
The currently shipped plugins are:
|Name|Args|Description|Configuration|
|:---|:---|:----------|:------------|
|whois|    |ARIN and RIPE online databases to add network owner info|web: "transforms/whois/known\_subnets", "transforms/whois/known\_orgs"|
|dnsdb|    |isc.dnsdb.org's database (if an api key is provided)|web: "transforms/dnsdb/limit", "transforms/dnsdb/apikey"|
|cif |    |Queries a local Collective Intelligence Framework server|web: "transforms/whois/known\_subnets", "transforms/whois/known\_orgs", "transforms/cif/base\_url"|
|grep|regex on field, regex on value|Only passes results that match the test|             |
|filter|regex on field, regex on value|Only passes results that do not match the test|             |
|sum |field|Sums the total found for the given field|             |
|anonymize|    |Anonymizes any IP's found that match the configuration for "transforms/whois/known\_subnets"|web: "transforms/whois/known\_subnets"|
|database (example)|field to pass to database|Adds record found in database to displayed record after using the given field as a lookup in the database|web: "transforms/database/"|
|geoip|    |Uses the local GeoIP database to attach geo info to any IP's of hostnames found|             |
|has | value,operator (defaults to >),field|Defaults to returning only records that have more than the given count in a groupby result.  Args can change operator to less than, etc., and also specify a specific field in a non-groupby result.|             |
|interval|    |Calculates the number of seconds elapsed between records returned and adds that value as a transform field|             |
|local|    |Returns only records which have a value in the configured local subnets|web: "transforms/whois/knonw\_subnets"|
|remote|    |Returns only records which do not have a value in the configured local subnets|web: "transforms/whois/knonw\_subnets"|
|parse|pattern\_name|Given the name of a configured pattern, will use preconfigured regular expressions to extract fields from result messages.  It can be used as a way of post-search parsing.|web: "transforms/parse/(pattern\_name)"|
|scanmd5|    |Checks all configured URL sources for hits on any MD5's contained in a record.  By default, it will check Shadowserver, but can also check VirusTotal if an API key is configured.|web: "transforms/scanmd5/virustotal\_apikey"|


# Subsearches #
Subsearches are a special kind of transform that is built-in to ELSA.  They are used to take the results of a groupby (report) query and concatenate those results as an OR onto a second query.  For example:
```
dstip:1.1.1.1 groupby:srcip | subsearch(dstip:2.2.2.2)
```
This query will find all source IP's that talked to 1.1.1.1 and then find any of those IP's which also talked to 2.2.2.2.  You can mix in other transforms as well:
```
dstip:1.1.1.1 groupby:srcip | subsearch(dstip:2.2.2.2) | whois | filter(cc,us)
```
This will find IP's which talked to both 1.1.1.1, 2.2.2.2, and are not in the US.

Subsearches can be chained together arbitrarily:
```
dstip:1.1.1.1 groupby:srcip | subsearch(dstip:2.2.2.2 groupby:srcip) | subsearch(class:windows groupby:eventid)
```
This will find all unique Windows event ID's for hosts that talked to both 1.1.1.1 and 2.2.2.2.

To make a field from the source groupby become a specific field in the subsearch, you can pass a second argument:
```
dstip:1.1.1.1 groupby:srcip | subsearch(dstip:2.2.2.2,srcip)
```
This will mandate that the subsearch uses srcip:host for each host found in the first query.

# OSSEC Integration #
ELSA can read logs from OSSEC, here's how:
Edit /usr/local/syslog-ng/etc/syslog-ng.conf:
```
source s_ossec {
  file("/OSSEC_BASE_DIR/ossec/logs/archives/archives.log" program_override('ossec-archive') follow_freq(1) flags(no-parse));
}
log {
 source(s_ossec); destination(d_elsa);
};
```

To enable archive output in OSSEC, add to ossec.conf file 

&lt;global&gt;

 section:
```
<logall>yes</logall>
```

# Bro Integration #
For forwarding flat-file logs from Bro using syslog-ng, use the below configuration, which assumes that ELSA is the localhost (127.0.0.1).  Put the below configuration `/etc/elsa_syslog-ng.conf` which is the file that ELSA will include in the actually run syslog-ng configuration at /usr/local/syslog-ng/etc/syslog-ng.conf:
```
source s_bro_conn { file("/usr/local/bro/logs/current/conn.log" flags(no-parse) program_override("bro_conn")); };
source s_bro_http { file("/usr/local/bro/logs/current/http.log" flags(no-parse) program_override("bro_http")); };
source s_bro_dns { file("/usr/local/bro/logs/current/dns.log" flags(no-parse) program_override("bro_dns")); };
source s_bro_notice { file("/usr/local/bro/logs/current/notice.log" flags(no-parse) program_override("bro_notice")); };
source s_bro_smtp { file("/usr/local/bro/logs/current/smtp.log" flags(no-parse) program_override("bro_smtp")); };
source s_bro_smtp_entities { file("/usr/local/bro/logs/current/smtp_entities.log" flags(no-parse) program_override("bro_smtp_entities")); };
source s_bro_ssl { file("/usr/local/bro/logs/current/ssl.log"
flags(no-parse) program_override("bro_ssl")); };

filter f_bro_headers { message("^#") };
rewrite r_bro_host { set("10.12.1.57", value("HOST")); };

log { 
  source(s_bro_conn);
  source(s_bro_http);
  source(s_bro_dns);
  source(s_bro_notice);
  source(s_bro_smtp);
  source(s_bro_smtp_entities);
  source(s_bro_ssl);
  rewrite(r_bro_host);
  rewrite(r_cisco_program);
  rewrite(r_snare);
  rewrite(r_pipes);
  parser(p_db);
  rewrite(r_extracted_host);
  log { filter(f_bro_headers); flags(final); };
  #  destination(d_elsa); 
  log { destination(d_elsa); };
};

```

If ELSA is not localhost, define an ELSA destination:
```
destination d_elsa { tcp("remote.elsa.host.ip" port(514)); };

log { source(s_bro_conn); destination(d_elsa); };
log { source(s_bro_http); destination(d_elsa); };
log { source(s_bro_dns); destination(d_elsa); };
log { source(s_bro_notice); destination(d_elsa); };
log { source(s_bro_smtp); destination(d_elsa); };
log { source(s_bro_smtp_entities); destination(d_elsa); };
log { source(s_bro_ssl); destination(d_elsa); };
```

Finally, enable the new configuration either by running:

```
/usr/local/elsa/contrib/install.sh node set_syslogng_conf &&
service syslog-ng restart
```

or, you can just run the node update:

```
/usr/lcoal/elsa/contrib/install.sh node update
```

# Calculating Disk Requirements #
The basic rule of thumb is that ELSA will require about 50% more disk than flat log files.  This will provide archived and indexed logs.  Archive logs require about 10% of flat file logs, log indexes require 40-50% more disk than the flat files, so together, there is a roughly 50% overall penalty.

To specify how much disk to use, see the config file entry for log\_size\_limit, which is the total limit ELSA will use.  Within that limit, the archive section's config value for "percentage" dictates what percentage of the overall log\_size\_limit will be used for archive, and the rest will be used for indexed logs.  If you do not wish to archive, set the percentage to zero and all space will be for the index, or vice versa.
# GeoIP Support #
In addition to whois lookups, ELSA has a transform for GeoIP provided by MaxMind.com.  By default, ELSA will use the country database provided in the standard Perl module, but you can download the free city database from [here](http://geolite.maxmind.com/download/geoip/database/GeoLiteCity.dat.gz).  The transform works like any other transform, e.g.:
```
site:www.google.com | geoip
```
This will attach the location fields to results.  Results that have these fields can then be exported using the GoogleEarth export, which returns a .kml file suitable for opening in Google Earth or Google Maps.

# Configuring IDS to Forward Logs #
## Snort ##
There are two ways to configure Snort to send logs.  Either configure barnyard or Snort itself to send logs to local syslog.  Both configuration entries (in either snort.conf or barnyard.conf) will look like this:
```
output alert_syslog: LOG_LOCAL6 LOG_ALERT
```
## Suricata ##
To log to local syslog from Suricata, edit the "outputs" stanza to contain:
```
outputs:
 - syslog:
      enabled: yes
      identity: "snort"
      facility: local6
```
## Forwarding Local Logs to ELSA ##
You will then need to configure the local syslog on the box that is running Snort to forward logs to ELSA.
### rsyslog/Syslogd ###
If the box is running a simple syslogd, it would look like this to forward all logs to ELSA (which is usually a good idea):
```
*.* @ip.address.of.elsa
```
### Syslog-NG ###
If it's running syslog-ng, use this:
```
source src { unix-dgram("/dev/log"); };
filter f_local6 { facility(local6); };
destination d_elsa { udp("ip.address.of.elsa"); };
log { source(src); filter(f_local6); destination(d_elsa); };
```
# Eventlog-to-Syslog #
Sending logs from Windows servers is best achieved with the free, open-source program [Eventlog-to-Syslog](http://eventlog-to-syslog.googlecode.com).  It's incredibly easy to install:
  1. Login as an administrator or use runas
  1. Copy evtsys.exe and evtsys.dll to Windows machine in the system directory (eg.C:\Windows\System32).
  1. Install with: `evtsys.exe -i -h ip.of.elsa.node`
  1. Profit
The logs will be sent using the syslog protocol to your ELSA server where they will be parsed as the class "WINDOWS" and available for reporting, etc.
# Dashboards #
## Creating ##
To create a dashboard, click on the "ELSA" menu button in the upper-left-hand corner of the main web interface.  A dialog box will open showing a grid of current dashboards you've created as well as a link for "Create/import."  Click the link to open another dialog which will ask for params:
  * Description: What the title of the dashboard page will show.
  * Alias:  The final part of the URL used for accessing, e.g. http://elsa/dashboard/alias
  * Auth required:  The level of authorization, can be none, authentication, or a specific group.
  * Import:  You can paste import configuration (as generated by a dashboard export) here to auto-create all of these parameters, plus all of the underlying charts.
  * Groups:  This field shows up when you've selected "Specific Groups" as the auth.  You can paste in a groupname here, or use the drop down later.

Once created, a dashboard appears on the table of dashboards and the "Actions" button will allow you to view, edit, and delete the dashboard.
## Authorization ##
Dashboards present a way to provide access to charts which have underlying queries that some users would not normally have permissions to query on their own.  It is essentially a way to delegate query access for the purposes of making charts and is especially helpful for making reports that are customer-facing.  Queries are logged and authorized as if they were made by the creator of the chart.  A log is noted in the web.log file to record that the query was run on the behalf of another user.  As previously stated, access to the dashboard itself can be governed, so there is essentially a two-tiered access system: the first is access to the dashboard, the second is the access to the data.

Currently, only a single group can be permitted access if using group-specific authorization.  This restriction may be lifted in the future.
## Adding Charts ##
Charts can be added either from the main ELSA query interface using the "Results" button and "Add to dashboard" or you can do so from the "edit dashboard" interface if you've chosen the "edit" option from the "Actions" menu in the "Dashboards" dialog.  When adding a chart from the main query interface, you must choose a dashboard and a chart, which can be "New Chart" to create one.  The dashboard must exist beforehand, so you may need to create a dashboard first.
## Adding Queries ##
Queries are easiest to add using the above method in which a query you've just run in the standard ELSA query interface is added via the "Results" button.  If the query has a "Report On" or "groupby" value, that value will be used to create the chart.  Otherwise, the query will be plotted over time by count of occurrences.
## Editing Charts ##
Charts can be edited from the edit dashboard interface in two ways: the appearance and the queries.  The appearance will dictate what kind of chart it is, the title, and other formatting variables.  The queries dictate the underlying data series.  When editing charts, changes appear live as you edit.
### Advanced Chart Editing ###
In some cases, you may need to edit the actual JSON used to construct a dashboard to get exactly the right look and feel.  Here's an excerpt from the ELSA mailing list touching on how to do that:
<pre>
the width (fixed) is located into /opt/elsa/web/lib/Web/GoogleDashboard.pm<br>
I set<br>
our $Default_width = 1850;<br>
for fit 46" screen 1920pixel<br>
<br>
For the number of elements (charts?) I found something into /opt/elsa/web/lib/API/Charts.pm<br>
There is:<br>
# Sanity check<br>
if ($args->{x} > 2){<br>
die('Cannot have more than 3 charts on one line: ' . $args->{x});<br>
}<br>
<br>
So 3 charts per line, if you need more, increase the number 2.<br>
<br>
<br>
For the height there isn't a unique value to modify as width but as Martin suggested to me some post ago, you can export dashboard, modify values, then create new dashboard with new values.<br>
For example for a map I needed to set height:<br>
"y" : "6",<br>
"options" : {<br>
"width" : 940,<br>
"height": 500,<br>
<br>
In this case you can modify height (and width) for each chart.<br>
</pre>
## Chart Types ##
ELSA uses [Google Visualizations](https://developers.google.com/chart/) to draw charts.  See their documentation for what chart types are available.  Of particular note is the "table" type, which is hidden under the "More" menu of available char types.  It's a great way to display long text in a readable format.
## Viewing Dashboards ##
Dashboards are accessed via
`<elsa>/dashboard/<alias>?<time unit>=<number of units>&start=<ISO start>&end=<ISO end>`
All parameters after the alias are optional, and the default is to show the past seven days by hour.  To see, for instance, the last fifteen minutes by second, you'd use: ` <alias>?seconds=900 ` which would give you extremely granular data.  You could also view any time period at any granularity by providing a more specific start and/or end, such as ` <alias>?seconds=900&start=two days ago `
or ` <alias>?seconds=900&start=2012-08-27 00:00:00 `.  You can add ` &refresh=<seconds> ` to a dashboard URL to have it refresh every n seconds, where n is at least five.
## Performance Considerations ##
Take care when creating charts that the queries used do not tax the system too much.  This can happen when a query is not selective enough.  That is, there is more "grouping" than "searching" occurring.  For anything less than a billion records, this should not be much of an issue, but if your query returns more than a billion or so, you may notice that it can take seconds or minutes for the charts to load.
# Troubleshooting #
## Log Files ##
### /data/elsa/log/node.log ###
This is the main ELSA log on each log node.  It will contain any errors or information regarding the recording and indexing of logs.  If no new logs are coming in, this is the first log file to check.
### /var/log/apache2/error.log ###
This log can be named differently or be in /var/log/httpd.  It is the standard Apache log file which will be the first place to check if any "Query Failed" error messages appear on the web interface.  Errors only show up here if they are major enough to break the underlying ELSA code.  Typically, these kinds of errors are connectivity or permissions related.
### /data/elsa/log/web.log ###
This is the main ELSA log for the web interface.  It has information on any ELSA-specific actions initiated from the web interface.  If queries are not returning the results you expect, check this log.
### /data/elsa/log/syslog-ng.log ###
Syslog-NG's internal log file will give low-level debugging info like raw message rates.  It should generally only be needed when you're not sure that a node is receiving logs.
### /data/elsa/log/query.log ###
This file contains the query log generated by the Sphinx searchd daemon.  It should not normally be needed, but can be a good place to get a feel for what queries the system is running and how long they are taking.
### /data/elsa/log/searchd.log ###
This is the Sphinx searchd daemon log and will contain info on index rotation.

## Common Troubleshooting Symptoms ##
| **Symptom** | **Resolution** |
|:------------|:---------------|
| Chronic warnings in the web UI for "couldn't connect to MySQL" | This can be caused if a web frontend has issues and the MySQL server decides it no longer wishes to speak to the server because of too many dropped connections.  To fix it, you need to log into the node referred to in the message and issue: ` mysqladmin flush-hosts ` which will cause the MySQL daemon to once again accept connections from the "flaky" frontend. |
| "Query Failed" red error message | This is a low-level error indicating that there is a connectivity or permissions problem between the web server and the MySQL/Sphinx daemons on the node.  It will also show up in the node.log as "No nodes available."  You can verify database connectivity by manually running: `mysql -h<node IP> -uelsa -p syslog` and `mysql -h<node IP> -P9306`.  If both work, then the problem may be something more specific with either MySQL or Sphinx.  To troubleshoot that, run `tcpdump -i eth0 -n -s0 -X "port 3306 or port 9306"` and watch the traffic to see what's occurring when you run a query. |

# Datasources #
ELSA can be configured to query external datasources with the same framework as native ELSA data.  Datasources are defined by plugins.  The only plugin currently available is for databases.  Database datasources are added under the "datasource" configuration section, like this:
```
"datasources": {                 
  "database": { 
    "hr_database": { 
      "alias": "hr",
      "dsn": "dbi:Oracle:Oracle_HR_database", 
      "username": "scott", 
      "password": "tiger", 
      "query_template": "SELECT %s FROM (SELECT person AS name, dept AS department, email_address AS email) derived WHERE %s %s ORDER BY %s LIMIT %d,%d", 
      "fields": [ 
        { "name": "name" }, 
        { "name": "department" },
        { "name": "email" }
      ] 
```

The configuration items for a database datasource are as follows:
| **Item** | **Purpose** |
|:---------|:------------|
| alias    | What the datasource will be referred to when querying |
| dsn      | Connection string for Perl |
| username | User        |
| password | Password    |
| query\_template | sprintf formatted query with the placeholders listed below |
| fields   | A list of hashes containing name (required), type (optional, default is char), and alias which functions as both an alternative name for the field as well as the special aliases "count" to refer to the column to use for summation and "timestamp" which defines the column to use in time-based charts. |

Query\_template parameters (all are required):
  1. The columns for SELECT
  1. The expression for WHERE
  1. The column for GROUP BY
  1. The column for ORDER BY
  1. OFFSET
  1. LIMIT

# Livetail #
**_Livetail is deprecated until further notice due to stability issues._**
ELSA has the ability to allow each user to get a live feed of a given search delivered to a browser window.  Livetail allows you to use full PCRE to search incoming logs without impacting logging performance.  This is done by forking a separate process on each node that reads the text file being written by the main logging process, ensuring that no extra load is put on the main process and therefore avoiding log loss in high volume situations.

## Starting a Livetail ##
To start a livetail, simply choose the "Livetail" option from the "Index" button, which will open a new window.  Your search will begin immediately and results will be displayed from all nodes as they are available.  Your browser window will poll the server every five seconds for new results.  The window will scroll with new results.  If you keep your mouse pointer over the window, it will cease scrolling.

## Ending a Livetail ##
Livetails will automatically be cancelled when you close the browser window.  If the browser crashes and the livetail continues, it will be replaced by any livetail you start again, or will timeout after an hour.  An administrator can cancel all livetails by choosing "Cancel Livetails" from the "Admin" menu.

Livetail results are temporary and cannot be saved.  You can copy and paste data from the window, or run a normal ELSA search to save the data.
# Saved Searches (Macros) #
Any completed search can be saved by clicking the "Results" button and choosing "Save Search."  This will bring up a dialog box asking for a name to save the search as.  The name must be alphanumeric plus underscore.  You can view and edit all saved searches using the "Saved Searches" menu option in the "ELSA" menu at in the upper-left-hand part of the menu bar at the top of the ELSA page.
## Macros ##
Any saved search can be invoked inside of another query by using the dollar-sign-name convention.  For example, if there is a saved search named "trojan" which was saved with a query like this: ` +sig_msg:trojan `, then you can invoke that query within any other query like this: `srcip:1.1.1.1 $trojan`.  The query will be interpolated and fully evaluate to `srcip:1.1.1.1 +sig_msg:trojan`.
## Built-in Macros ##
The system will auto-populate some network-based macros for convenience if the whois transform configuration has been entered.  ELSA ships with default values of RFC1918 IP space:
```
"whois": {
  "known_subnets": {
    "10.0.0.0": {
      "end": "10.255.255.255",
      "org": "MyOrg"
  },
```
Edit the "known\_subnets" stanza to add your local org-specific values.  ELSA will use these values to create macros for srcip and dstip such that the macros ` $myorg`, `$src_myorg`, and `$dst_myorg` will be available automatically and will resolve to `srcip>10.0.0.0 srcip<10.255.255.255 dstip>10.0.0.0 dstip<10.255.255.255` for `$myorg`, and the src and dst versions for the `$src_myorg` and `$dst_myorg`, respectively.

Having these macros available can greatly aid searching for IDS events, HTTP events, among many others.  For instance, you can easily find HTTP POST's to your org by searching `+method:POST +$dst_myorg`.

These built-in macros will be overridden by user-created macros of the same name.

# Importing Logs #
Logs can be directly imported into ELSA using the named pipes `/data/elsa/tmp/import` and `/data/elsa/tmp/realtime` on the log nodes.  You cannot currently import logs via the web interface, though this may change in the future.  The import pipe will use the timestamp already included in the log message and will go into special import tables and indexes which will not be rotated with other logs.  Imported logs are rotated when they reach a percentage specified in the elsa\_node.conf for `import_log_size_limit` or 50% of `log_size_limit` if no `import_log_size_limit` is specified.  The realtime pipe will funnel any logs received into the main logging channel next to logs received on the wire.  The timestamp used will be the current time, just like logs on the wire.  The realtime pipe can be used for receiving logs with an external program, like a web service, and piping them directly into ELSA.

For example, to import the Apache web log into ELSA, you would issue this command on the log node:

`cat /var/log/apache2/access_log > /data/elsa/tmp/import`

# Host Checks #
It is often helpful to know if a host has not sent logs to ELSA within a given timeframe.  This is possible if you configure the `host_checks` config in `elsa_web.conf` like this:
```
  "host_checks": {
    "127.0.0.1": 60,
    "192.168.1.1": 900
  },
  "admin_email_address": "sysadmin_team@example.com",
```
The above will configure ELSA to send an email to "sysadmin\_team@example.com" if there are no logs within 60 seconds from 127.0.0.1 or 900 seconds (15 minutes) of 192.168.1.1.

Host checks are run via the cron.pl script which runs every minute, so an interval of less than 60 seconds will be nonsensical.  The interval is padded with 60 seconds to make sure that the log loading batch job is completed on each node, so specifying an interval of 60 seconds will really be 120 seconds, but 900 seconds would be 960 seconds.

There is currently no threshold for number of alerts sent, so you will get an email each minute until the logs start again or the configuration is changed.
# Pcap/Stream/Block Integration #
To integrate with a pcap server like OpenFPC or StreamDB, set these config params in elsa\_web.conf:

| streamdb\_url | StreamDB base URL |
|:--------------|:------------------|
| streamdb\_urls | Hash of hashes with StreamDB nodes and networks they cover |
| pcap\_url     | OpenFPC base URL  |
| block\_url    | Base URL for your custom blocking app |

This example config:
```
{
  #"streamdb_url": "http://streamdb",
  "streamdb_urls": {
    "192.168.1.1": { "start": "10.1.0.0", "end": "10.1.255.255" },
    "192.168.1.2": { "start": "10.2.0.0", "end": "10.2.255.255" }
  },
  "pcap_url": "http://myopenfpc",
  "block_url": "http://blackhole"
}
```
will set both getStream, getPcap, and blockIp options available as plugins after clicking "Info" for a log entry.

# Preferences #
You can set per-user preferences by navigating to the "Preferences" dialog under the "ELSA" menu in the upper-left-hand corner of the page.  Preference changes will take effect at the next page load.

|Type|Name|Value to Enable|Function|
|:---|:---|:--------------|:-------|
|default\_settings|reuse\_tab|0              |Overrides server setting for whether or not to reuse the current tab for each new query|
|default\_settings|grid\_display|1              |Defaults results to grid view|
|default\_settings|no\_column\_wrap|1              |Disables column wrapping in grid view|
|custom|openfpc\_username|<user name>    |User name for sending to OpenFPC if pcap\_url is set|
|custom|openfpc\_password|

&lt;password&gt;

|OpenFPC password|
|default\_settings|pcap\_offset|

&lt;seconds&gt;

|Number of seconds before/after to set get\_pcap retrieval time to|
|default\_settings|use\_utc|1              |Display all dates in UTC (GMT)|
|default\_settings|orderby\_dir|DESC           |Default to reverse sort (descending)|
|default\_settings|timeout|<natural number>|Override the system default for query timeout|
|default\_settings|default\_or|1              |Override the system default for making events match if any of the query terms match instead of if all query terms match|
|default\_settings|limit|100            |Default limit to use for number of results to return|
|default\_settings|rows\_per\_page|15             |Default for rows per page of results when displayed|

# Keyboard Shortcuts #
|Key|Action|
|:--|:-----|
|F8 |Closes all result tabs|
|F9 |Closes all result tabs before active|
|F10|Closes all tabs except active|

# External Documentation #
  * Adding additional disk to an ELSA VM http://opensecgeek.blogspot.com/2013/02/mysql-and-elsa-when-your-storage-runs.html
  * FreeBSD timezone issues: https://rt.cpan.org/Public/Bug/Display.html?id=55029#txn-740063