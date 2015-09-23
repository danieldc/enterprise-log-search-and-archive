## Install ELSA ##
```
cd /tmp/
 wget http://enterprise-log-search-and-archive.googlecode.com/svn/trunk/elsa/contrib/install.sh
cp /tmp/install.sh /etc/elsa_vars.sh
```

## Remove everything except the credentials ##
`sudo sh -c "sh install.sh node && sh install.sh web"`

  1. Install will take 30 minutes or so.
  1. Install will ask for your password for the OpenSSH portion…must enter root’s password.
  1. After install, if “build\_web\_perl FAIL” shows up, do the following:
  1. `sudo cpanm -n Module::Install`
  1. sudo sh -c "sh install.sh web"
  1. Verify that the files are in place:
  * /etc/elsa\_node.conf
  * /etc/elsa\_web.conf
  * /etc/elsa/
  * /etc/elsa\_vars.sh: This file is more of a configuration file that should only have credentials if they are non-default.
  1. Stop IPTABLES or change the rules
  1. service iptables stop

# Installation for CentOS using OS's syslog-ng #
Post install, install the fedora epel repo, as the provided repos do not contain syslog-ng
`yum -d 0 -e 0 -y install http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm`

Install syslog-ng
`yum install syslog-ng`

elsa is going to look at /usr/local/syslog-ng, and while we could define the root install of syslog-ng with yum, I just made symbolic links
```
mkdir -p /usr/local/syslog-ng/bin
mkdir /usr/local/syslog-ng/sbin
ln -s /usr/bin/pdbtool /usr/local/syslog-ng/bin/
ln -s /sbin/syslog-ng /usr/local/syslog-ng/sbin/
ln -s /usr/bin/loggen /usr/local/syslog-ng/bin/
```

grab elsa & install

```
cd /tmp
wget "http://enterprise-log-search-and-archive.googlecode.com/svn/trunk/elsa/contrib/install.sh"
sh install.sh node && sh install.sh web
```

elsa will start syslog-ng, but it will be using the wrong config (it'll be using the config from /etc/syslog-ng). Symlink the elsa syslog-ng and try it again

```
rm /etc/syslog-ng/syslog-ng.conf
ln -s /usr/local/syslog-ng/etc/syslog-ng.conf /etc/syslog-ng/
rm /etc/elsa_node.conf
sh install.sh node
```

At this point, all should be working on individual nodes. I added the following to /usr/local/syslog-ng/etc/syslog-ng.conf to monitor /var/log/messages - post edit restart syslog-ng:

```
source s_messages { file("/var/log/messages"); };
log { source(s_messages);  destination(d_elsa);}; 
```

Edit /etc/elsa\_web.conf, add a section for the local node on both nodes, and on the 'main' node add an entry under peers for the other elsa node. Also, edit the apikey section:

```
  "nodes" : {
      "127.0.0.1" : {
         "db" : "syslog",
         "password" : "<pass>",
         "username" : "elsa",
      }
   },
   "peers" : {
      "127.0.0.1" : {
         "apikey" : "<key>",
         "url" : "http://127.0.0.1/",
         "username" : "elsa"
      },
      "172.16.202.10" : {
         "apikey" : "<key>",
         "url" : "http://172.16.202.10/",
         "username" : "elsa"
      }


   },
   "apikeys" : {
      "elsa" : "<key>"
   },
```

Restart httpd