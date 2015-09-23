The steps are identical to the quickstart for installation except you add "update" after each install.sh command.

# Step 1 #
Grab the auto-installer to ensure you've got the latest one:
```
wget "http://enterprise-log-search-and-archive.googlecode.com/svn/trunk/elsa/contrib/install.sh"
```
Edit the top of the script and change directory locations and passwords as necessary.  The defaults will work with the default settings that shipped with the OS.
# Step 2 #
Deploy either as a log node, a web frontend, or both
```
sudo sh -c "sh install.sh node update  && sh install.sh web update"
```
# Step 3 #
Go get some coffee, check your email, install eventlog-to-syslog on servers, etc.  The full install may take up to 30 minutes, but can run unattended.
# Step 4 #
Enjoy!  ELSA should be up and running with the web interface (if you installed it) up and running via Apache on port 80.  A test query of "seq" should return results.