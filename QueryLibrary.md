# Query Library #

Note: Replace 10.0.0.0 with your actual local subnet.

| Description | Query |
|:------------|:------|
| POST's to non-US sites | ` +method:POST domains:ru domains:in domains:co.cc domains:co.cz domains:cn -domains:yandex.ru -domains:odnoklassniki.ru -domains:oneindia.in -domains:sina.com.cn ` |
| POST's to non-US sites (GeoIP) | ` +method:POST country_code:CN country_code:RU country_code:RO country_code:NL ` |
| POST's (full xenophobia) | ` +method:POST -country_code:US ` |
| Vulnerable local web servers | ` +URL.status_code:500 dstip>10.0.0.0 dstip<10.255.255.255 ` |
| Blackhole second stage loads | ` +sig_msg:kit srcip>10.0.0.0 srcip<10.255.255.255 groupby:srcip | subsearch(class:snort +exe) ` |
| Java exe downloads | ` +sig_msg:vulnerable groupby:srcip | subsearch(+sig_msg:exe) ` |
| RFC1918 IP space | ` srcip>=10.0.0.0 srcip<=10.255.255.255 dstip>=10.0.0.0 dstip<=10.255.255.255 srcip>=172.16.0.0 srcip<=172.31.255.255 dstip>=172.16.0.0 dstip<=172.31.255.255 srcip>=192.168.0.0 srcip<=192.168.255.255 dstip>=192.168.0.0 dstip<=192.168.255.255 ` |