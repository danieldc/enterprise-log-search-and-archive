# Supported Operating Systems #
## Linux and FreeBSD only! ##
Tested on:
  * Ubuntu 10.04
  * openSUSE 12.1
  * CentOS 6.0 Final
  * FreeBSD 8.2
Will probably work with:
  * Debian
  * RHEL, Fedora
  * SLES
  * FreeBSD > 8.2

# Step 1 #
Grab the auto-installer:
```
wget "http://enterprise-log-search-and-archive.googlecode.com/svn/trunk/elsa/contrib/install.sh"
```
Edit the top of the script and change directory locations and passwords as necessary.  The defaults will work with the default settings that shipped with the OS.
# Step 2 #
Deploy either as a log node, a web frontend, or both
```
sudo sh -c "sh install.sh node && sh install.sh web"
```
# Step 3 #
Go get some coffee, check your email, install eventlog-to-syslog on servers, etc.  The full install may take up to 30 minutes, but can run unattended.
# Step 4 #
Enjoy!  ELSA should be up and running with the web interface (if you installed it) up and running via Apache on port 80.  A test query of "seq" should return results.
# Adding Nodes #
To add log nodes, repeat the "sudo install.sh node" step on each node you want, then edit the /etc/elsa\_web.conf file on the web frontend server to point to the new nodes under the "nodes" configuration directive and restart apache2.