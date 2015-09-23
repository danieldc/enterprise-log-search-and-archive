via Peter C. from Balabit:

Here's how to setup ELSA and syslog-ng PE Tested on Ubuntu 10.04 with syslog-ng PE 4.4.2a.

  1. Install ELSA as usual, as described at http://code.google.com/p/enterprise-log-search-and-archive/wiki/Quickstart
  1. Download the .run installer for syslog-ng PE from http://www.balabit.com/network-security/syslog-ng/central-syslog-server/download/syslog-ng-pe/4.2.2a/linux as the native .deb package removes the installed syslog package (ELSA leaves the local syslog alone and installs syslog-ng to /usr/local/syslog-ng to collect logs only from the network).
  1. Before installation remove the syslog-ng init script, which is installed by ELSA (rm /etc/init.d/syslog-ng, or save it somewhere).
  1. Install syslog-ng PE by "sh syslog-ng-premium-edition-4.2.2a-linux-glibc2.3.6-amd64.run" and choose "Don't register" when it's asked (so it does not deactivate rsyslog, which handles the local logs).
  1. Edit /etc/init.d/syslog-ng, and add the following line to the end of the "syslogng\_stop()" function of the init script, to make sure, that elsa database script is also stopped with syslog-ng:
> > ` kill -15 $(pgrep -f elsa.pl) `
  1. Copy /usr/local/etc/syslog-ng/etc/syslog-ng.conf to /opt/syslog-ng/etc and edit the version information at the top of the file. If you also want to store your logs encrypted in syslog-ng PE's logstore, create a destination based on information at http://www.balabit.com/sites/default/files/documents/syslog-ng-pe-4.0-guides/syslog-ng-pe-v4.0-guide-admin-en.html/index.html-single.html#configuring_destinations_logstore and add the destination to the log statement at the end of the configuration file.
  1. Start syslog-ng and you should see logs in ELSA in a minute.