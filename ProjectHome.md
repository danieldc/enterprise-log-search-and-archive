ELSA is a centralized syslog framework built on Syslog-NG, MySQL, and Sphinx full-text search.  It provides a fully asynchronous web-based query interface that normalizes logs and makes searching billions of them for arbitrary strings as easy as searching the web.  It also includes tools for assigning permissions for viewing the logs as well as email based alerts, scheduled queries, and graphing.

Features:
  * High-volume receiving/indexing (a single node can receive > 30k logs/sec, sustained)
  * Full Active Directory/LDAP integration for authentication, authorization, email settings
  * Instant ad-hoc reports/graphs on arbitrary queries even on enormous data sets
  * Dashboards using Google Visualizations
  * Email alerting, scheduled reports
  * Plugin architecture for web interface
  * Distributed architecture for clusters
  * Ships with normalization for some Cisco logs, Snort/Suricata, Bro, and Windows via Eventlog-to-Syslog or Snare

Screenshots and a more in-depth look at the need for it can be found at my blog at http://ossectools.blogspot.com/2011/03/fighting-apt-with-open-source-software.html .

Getting started is easy now with the [quickstart](http://code.google.com/p/enterprise-log-search-and-archive/wiki/Quickstart) script found in the wiki section.  The auto-installer has been tested with Ubuntu, openSUSE, CentOS, and FreeBSD.