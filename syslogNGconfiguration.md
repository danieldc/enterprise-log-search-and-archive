ELSA uses Syslog-ng to read in logs, parse some of them, and send them through the system.  Here's a snippet showing how to configure Syslog-ng >= 3.1 to use the pattern-db parser.

```
source s_network {
	tcp();
	udp();
};

parser p_db {
	db-parser(file("/usr/local/elsa/node/conf/patterndb.xml"));
};

filter f_rewrite_cisco_program { match('^(%[A-Z]+\-\d\-[0-9A-Z]+): ([^\n]+)' value("MSGONLY") type("pcre") flags("store-matches" "nobackref")); };
filter f_rewrite_cisco_program_2 { match('^[\*\.]?(?:Jan|Feb|Mar|Apr|May|Jun|Jul|Aug|Sep|Oct|Nov|Dec)\s+\d{1,2}\s\d{1,2}:\d{1,2}:\d{1,2}(?:\.\d+)?(?: [A-Z]{3})?: (%[^:]+): ([^\n]+)' value("MSGONLY") type("pcre") flags("store-matches" "nobackref")); };
filter f_rewrite_cisco_program_3 { match('^\d+[ywdh]\d+[ywdh]: (%[^:]+): ([^\n]+)' value("MSGONLY") type("pcre") flags("store-matches" "nobackref")); };

rewrite r_cisco_program {
        set("$1", value("PROGRAM") condition(filter(f_rewrite_cisco_program) or filter(f_rewrite_cisco_program_2) or filter(f_rewrite_cisco_program_3)));
        set("$2", value("MESSAGE") condition(filter(f_rewrite_cisco_program) or filter(f_rewrite_cisco_program_2) or filter(f_rewrite_cisco_program_3)));
};

rewrite r_snare { subst("MSWinEventLog.+(Security|Application|System).+", "$1", value("PROGRAM") flags(global)); };
rewrite r_pipes { subst("\t", "|", value("MESSAGE") flags(global)); };
rewrite r_host { set("$SOURCEIP", value("HOST")); };
rewrite r_extracted_host { set("$pdb_extracted_sourceip", value("HOST") condition("$pdb_extracted_sourceip" != "")); };

template t_db_parsed { template("$R_UNIXTIME\t$HOST\t$PROGRAM\t${.classifier.class}\t$MSGONLY\t${i0}\t${i1}\t${i2}\t${i3}\t${i4}\t${i5}\t${s0}\t${s1}\t${s2}\t${s3}\t${s4}\t${s5}\n"); };

destination d_elsa { program("perl /usr/local/elsa/node/elsa.pl -c /etc/elsa_node.conf" template(t_db_parsed)); };

log { 
	source(s_network);
	rewrite(r_host);
	rewrite(r_cisco_program);
	rewrite(r_snare);
	rewrite(r_pipes);
	parser(p_db);
	rewrite(r_extracted_host); 
	destination(d_elsa);
};
```