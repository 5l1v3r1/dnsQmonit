## dnsQmonit

###[+] Summary [+]

dnsQmonit.py was written to monitor A/AAAA/PTR DNS queries from a socket without additional third-party Python libraries; however, if the Python Scapy module is available, it will use this over the Python Socket module as it can provide more functionality, such as listening on promiscous interfaces. It's end goal is to identify domains of interest, provide some visibility into DNS queries for network forensics, host-profiling, and general DNS analysis.

You have the ability to write query information to a CSV, SQLite3 database, a syslog server, or a dnsQmonit server. Currently these items are recorded: date, time, source IP, source port, destination IP, destination port, query record type (A/AAAA/PTR), FQDN requested, the TLD, the SLD, the number of subdomain levels, the domain score, and the domain flags. It is recommend to write the data to a SQLite3 database as it makes extracting information a bit easier - plus you can always dump it to a CSV later. If you have the storage for it, a SIEM via syslog can make querying the data a breeze. At the bottom of this read me are some Splunk extraction parsers for syslog.

Scoring is fairly arbitrary based on the lists loaded into the program at execution. These will provide a decent place to start reviewing domains though.

dnsQmonit was written to work with, and tested on, Python 2.7. It supports IPv4 traffic via Socket/Scapy and IPv6 traffic with Scapy.

###[+] Flags [+]

Current flags are listed below:

* "u" - Uncommon domain. The SLD is not contained in the common domain list loaded into the program.
* "b" - Watchlist domain. The FQDN is contained in the watch list loaded into the program.
* "s" - The FQDN was marked as being short.
* "l" - The FQDN was marked as being long.
* "t" - The TLD was scored based on a weighted average in the program.
* "f" - The domain was first seen since program execution.
* "d" - The query came from a known DNS server.

###[+] Usage [+]
```
usage: dnsQmonit.py [-h] [-i <interface>] [-w <csv|sql>] [-r <pcap file>]
                  [-f <file name>] [-t <seconds>] [-d] [-n <host name>]
                  [--watchlist <file name>] [--commonlist <file name>]
                  [--serverlist <file name>] [--client <server ip>]
                  [--server <listening ip>] [-p <port number>]
                  [--syslog <syslog server IP>]

optional arguments:
  -h, --help            show this help message and exit
  -i <interface>, --interface <interface>
                        Specify the interface name to monitor (e.g. eth0) or
                        specify 'any' to monitor all interfaces. If the Python
                        Scapy module is not insalled, an IP address must be
                        bound to the monitored interface, even if it's just
                        listening.
  -w <csv|sql>, --write <csv|sql>
                        Write output to a CSV file or a SQLite3 DB.
  -r <pcap file>, --read <pcap file>
                        Read in a PCAP file for processing instead of
                        listening to network traffic. This option requires the
                        Python Scapy module to be installed.
  -f <file name>, --filename <file name>
                        File name for the CSV or SQLite3 DB.
  -t <seconds>, --time <seconds>
                        Specify a time interval for writing data to a file.
                        Default is 15 seconds but higher DNS traffic volumes
                        should be adjusted up.
  -d, --display         Display domain queries. The default is off as high-
                        volume networks can flood the screen.
  -n <host name>, --name <host name>
                        Set a name for this node to be stored with the logs.
                        Default is hostname but can be changed to be more
                        specific.
  --watchlist <file name>
                        Add a list of domains which you feel are suspicious,
                        malicious, or of interest for watch - used for scoring
                        purposes only. Place each domain on a newline.
  --commonlist <file name>
                        Add a list of domains which you feel are common,
                        benign, or safe - used for scoring purposes only.
                        Place each domain on a newline.
  --serverlist <file name>
                        Add a list of DNS server IP addresses you expect
                        queries from. These will be flagged in the written
                        data. Place each IP on a newline.
  --client <server ip>  Make the program run as a client and send data to
                        another system running the program as a server.
  --server <listening ip>
                        Make the program run as a server and listen for data
                        from clients running the program. Should be used in
                        conjunction with -w or -d.
  -p <port number>, --port <port number>
                        UDP port to listen on if running as a server or UDP
                        port to communicate over if running as a client.
                        Default is UDP/54321.
  --syslog <syslog server IP>
                        Send queries as Syslog to a remote server.
```
###[+] Execution [+]

Typical data collection could be launched as follows:
```
python dnsQmonit.py -i eth0 -w sql -f dnsqueries.db --watchlist watchdomain.txt --commonlist commondomain.txt --serverlist serverdomain.txt -d -t 30
```
Additionally, you can read in packets from a PCAP file and process them as if you were listening on a network:
```
        dnsQmonit# dnsQmonit.py -d -r ../packet_test.pcap --client 10.18.22.12
        [+] Reading packets from PCAP file.
        [-] DNS Packet: 1 - 01/01/2014 15:38:51 - 10.18.23.35:60678 > 10.18.22.12:53 - PTR Record - Score: 5 Flags: f - 12.22.18.10.in-addr.arpa
        [-] DNS Packet: 2 - 01/01/2014 15:38:51 - 10.18.23.35:60679 > 10.18.22.12:53 - A Record - Score: 5 Flags: f - qmontest.com.roonet.com
        [-] DNS Packet: 3 - 01/01/2014 15:38:51 - 10.18.23.35:60680 > 10.18.22.12:53 - AAAA Record - Score: 0 Flags: None - qmontest.com.roonet.com
        [-] DNS Packet: 4 - 01/01/2014 15:38:51 - 10.18.23.35:60681 > 10.18.22.12:53 - A Record - Score: 5 Flags: f - qmontest.com
        [-] DNS Packet: 5 - 01/01/2014 15:38:51 - 10.18.23.35:60682 > 10.18.22.12:53 - AAAA Record - Score: 0 Flags: None - qmontest.com
        [+] Total packets read: 48 - Inspected DNS Queries: 5 (10.4%).
```
###[+] Sample SQL Queries [+]

Below are a handful of database queries that demonstrate extracting the information around DNS queries for further review. For display purposes when querying the DB from a Bash Shell, I append 'column -t -s "|"'.

To see the first appearance of all domains since program execution:
```
	dnsQmonit# sqlite3 --header queries.db 'SELECT * FROM Queries WHERE Flags like "%f%" ORDER BY Date ASC LIMIT 10' |column -t -s "|"
	Date        Time      Hostname  SrcAddr       SrcPort  DstAddr      DstPort  RecordType  Domain                    TLD  SLD               Levels  Score  Flags
	12/28/2013  02:21:17  roonet    10.18.24.123  10653    10.18.22.12  53       A           www.google.com            com  google.com        3       5      f
	12/28/2013  02:21:17  roonet    10.18.24.123  56353    10.18.22.12  53       A           www.reddit.com            com  reddit.com        3       5      f
	12/28/2013  02:21:18  roonet    10.18.24.123  49300    10.18.22.12  53       A           a.thumbs.redditmedia.com  com  redditmedia.com   4       10     uf
	12/28/2013  02:21:18  roonet    10.18.24.123  28827    10.18.22.12  53       A           static.adzerk.net         net  adzerk.net        3       5      f
	12/28/2013  02:21:18  roonet    10.18.24.123  46222    10.18.22.12  53       A           www.redditstatic.com      com  redditstatic.com  3       10     uf
	12/28/2013  02:21:19  roonet    10.18.24.123  30873    10.18.22.12  53       A           en.wikipedia.org          org  wikipedia.org     3       5      f
	12/28/2013  02:21:19  roonet    10.18.24.123  41991    10.18.22.12  53       A           imgur.com                 com  imgur.com         2       5      f
	12/28/2013  02:21:19  roonet    10.18.24.123  9430     10.18.22.12  53       A           ppcdn.500px.org           org  500px.org         3       5      f
	12/28/2013  02:21:19  roonet    10.18.24.123  6511     10.18.22.12  53       A           soundcloud.com            com  soundcloud.com    2       5      f
	12/28/2013  02:21:19  roonet    10.18.24.123  54498    10.18.22.12  53       A           tecidentity.com           com  tecidentity.com   2       10     uf
```
To see all queries that have a flag indicating they are on the watchlist or a score of 25+:
```
	dnsQmonit# sqlite3 --header queries.db 'SELECT * FROM Queries WHERE Score >= 20 OR Flags LIKE "%b%" LIMIT 10' |column -t -s "|"
	Date        Time      Hostname  SrcAddr       SrcPort  DstAddr      DstPort  RecordType  Domain                    TLD  SLD            Levels  Score  Flags
	12/28/2013  02:27:46  roonet    10.18.24.123  49659    10.18.22.12  53       A           wq1porm.s3.amazonaws.com  com  amazonaws.com  4       40     bf
	12/28/2013  03:25:46  roonet    10.18.24.123  3294     10.18.22.12  53       A           horadric.ru               ru   horadric.ru    2       45     uft
	12/28/2013  12:10:25  roonet    10.18.23.40   64786    10.18.22.12  53       A           horadric.ru               ru   horadric.ru    2       40     ut
	12/28/2013  13:33:45  roonet    10.18.24.101  62667    10.18.22.12  53       A           mc.yandex.ru              ru   yandex.ru      3       40     ft
	12/28/2013  19:53:42  roonet    10.18.24.66   62487    10.18.22.12  53       A           s3.amazonaws.com          com  amazonaws.com  3       35     b
	12/28/2013  20:25:12  roonet    10.18.23.40   58101    10.18.22.12  53       A           horadric.ru               ru   horadric.ru    2       40     ut
	12/28/2013  20:26:24  roonet    10.18.23.40   59262    10.18.22.12  53       A           s3.amazonaws.com          com  amazonaws.com  3       35     b
	12/28/2013  23:05:18  roonet    10.18.23.40   50717    10.18.22.12  53       A           horadric.ru               ru   horadric.ru    2       40     ut
	12/28/2013  23:55:46  roonet    10.18.24.101  52421    10.18.22.12  53       A           s3.amazonaws.com          com  amazonaws.com  3       35     b
	12/28/2013  23:59:22  roonet    10.18.24.101  59823    10.18.22.12  53       A           s3.amazonaws.com          com  amazonaws.com  3       35     b
```
To see all queries of a specific domain by keyword:
```
	dnsQmonit# sqlite3 --header queries.db 'SELECT * FROM Queries WHERE (Domain LIKE "%reddit%" OR Domain LIKE "%google%") AND Flags LIKE "%f%" LIMIT 10' |column -t -s "|"
	Date        Time      Hostname  SrcAddr       SrcPort  DstAddr      DstPort  RecordType  Domain                     TLD  SLD                    Levels  Score  Flags
	12/28/2013  02:21:17  roonet    10.18.24.123  10653    10.18.22.12  53       A           www.google.com             com  google.com             3       5      f
	12/28/2013  02:21:17  roonet    10.18.24.123  56353    10.18.22.12  53       A           www.reddit.com             com  reddit.com             3       5      f
	12/28/2013  02:21:18  roonet    10.18.24.123  49300    10.18.22.12  53       A           a.thumbs.redditmedia.com   com  redditmedia.com        4       10     uf
	12/28/2013  02:21:18  roonet    10.18.24.123  46222    10.18.22.12  53       A           www.redditstatic.com       com  redditstatic.com       3       10     uf
	12/28/2013  02:21:20  roonet    10.18.24.123  11657    10.18.22.12  53       A           radioreddit.com            com  radioreddit.com        2       10     uf
	12/28/2013  02:21:20  roonet    10.18.24.123  19023    10.18.22.12  53       A           reddit.tv                  tv   reddit.tv              2       11     uft
	12/28/2013  02:21:20  roonet    10.18.24.123  36195    10.18.22.12  53       A           redditgifts.com            com  redditgifts.com        2       5      f
	12/28/2013  02:21:56  roonet    10.18.24.66   64859    10.18.22.12  53       A           android.googleapis.com     com  googleapis.com         3       5      f
	12/28/2013  02:45:33  roonet    10.18.24.123  50810    10.18.22.12  53       A           lh5.googleusercontent.com  com  googleusercontent.com  3       5      f
	12/28/2013  02:52:03  roonet    10.18.24.123  48590    10.18.22.12  53       A           www.google-analytics.com   com  google-analytics.com   3       5      f
```
To see all queries between a date and time range where the score is over 10 and listed in descending order:
```
	dnsQmonit# sqlite3 --header queries.db 'SELECT * FROM Queries WHERE (Date BETWEEN "12/28/2013" AND "12/29/2013") AND (Time BETWEEN "11:00:00" AND "12:11:34") AND Score > 10 ORDER BY Score DESC LIMIT 10' |column -t -s "|"
	Date        Time      Hostname  SrcAddr       SrcPort  DstAddr      DstPort  RecordType  Domain       TLD  SLD          Levels  Score  Flags
	12/28/2013  12:10:25  roonet    10.18.23.40   64786    10.18.22.12  53       A           horadric.ru  ru   horadric.ru  2       40     ut
	12/28/2013  11:57:09  roonet    10.18.24.101  61722    10.18.22.12  53       A           204st.us     us   204st.us     2       11     uft
```
To see the Top 10 unique domains with a score over 25:
```
	dnsQmonit# sqlite3 --header queries.db 'SELECT * FROM Queries WHERE Score > 25 GROUP BY Domain ORDER BY Score DESC LIMIT 10' |column -t -s"|"
	Date        Time      Hostname  SrcAddr       SrcPort  DstAddr         DstPort  RecordType  Domain                                  TLD  SLD               Levels  Score  Flags
	12/31/2013  15:46:31  roonet    10.18.23.35   60775    10.18.22.12     53       AAAA        google.1234567890.123456890.1234590.ru  ru   1234590.ru        5       45     ult
	01/03/2014  11:02:39  roonet    10.18.22.12   35701    208.67.220.220  53       A           ge.tt                                   tt   ge.tt             2       41     bsdt
	01/07/2014  13:37:46  roonet    10.18.22.12   36369    208.67.220.220  53       A           helplinux.ru                            ru   helplinux.ru      2       40     udt
	12/31/2013  13:20:32  roonet    10.18.23.40   62364    10.18.22.12     53       A           horadric.ru                             ru   horadric.ru       2       40     ut
	01/06/2014  17:23:22  roonet    10.18.22.12   2035     208.67.220.220  53       A           mkln.ru                                 ru   mkln.ru           2       40     udt
	01/08/2014  23:50:06  roonet    10.18.22.12   37083    208.67.222.222  53       A           oflex.ru                                ru   oflex.ru          2       40     udt
	12/28/2013  02:27:46  roonet    10.18.24.123  49659    10.18.22.12     53       A           wq1porm.s3.amazonaws.com                com  amazonaws.com     4       40     bf
	01/07/2014  23:34:03  roonet    10.18.24.123  56865    10.18.22.12     53       A           www.babycenter.ru                       ru   babycenter.ru     3       40     ut
	01/07/2014  13:37:46  roonet    10.18.22.12   34881    208.67.220.220  53       A           www.helplinux.ru                        ru   helplinux.ru      3       40     udt
	01/09/2014  10:03:27  roonet    10.18.24.66   14883    10.18.22.12     53       A           ad.yieldmanager.com                     com  yieldmanager.com  3       35     b
```
Dump all queries matching your search to a CSV file for further analysis:
```
	dnsQmonit# sqlite3 --header -separator ',' queries.db 'SELECT * FROM Queries WHERE SrcAddr = "10.18.24.123" AND RecordType = "PTR"' > queries.csv
	dnsQmonit# cat queries.csv 
	Date,Time,Hostname,SrcAddr,SrcPort,DstAddr,DstPort,RecordType,Domain,TLD,SLD,Levels,Score,Flags
	12/28/2013,02:23:37,roonet,10.18.24.123,51383,10.18.22.12,53,PTR,lb._dns-sd._udp.roonet.com,com,roonet.com,5,10,uf
	12/28/2013,02:23:38,roonet,10.18.24.123,51388,10.18.22.12,53,PTR,b._dns-sd._udp.64.24.18.10.in-addr.arpa,arpa,in-addr.arpa,9,15,ufl
	12/28/2013,02:23:38,roonet,10.18.24.123,56279,10.18.22.12,53,PTR,db._dns-sd._udp.64.24.18.10.in-addr.arpa,arpa,in-addr.arpa,9,10,ul
	12/28/2013,02:23:38,roonet,10.18.24.123,52258,10.18.22.12,53,PTR,b._dns-sd._udp.roonet.com,com,roonet.com,5,5,u
```
###[+] Splunk Extraction Queries [+]
```
EXTRACT-LogDate = (?i)^(?P<LogDate>\w+\s+\d+)
EXTRACT-LogTime = (?i)^\w+\s+\d+\s+(?P<LogTime>[^ ]+)
EXTRACT-LogSender = (?i)^(?:[^ ]* ){3}(?P<LogSender>[^ ]+)
EXTRACT-QueryDate = (?i)^(?:[^ ]* ){5}(?P<QueryDate>[^,]+)
EXTRACT-QueryTime = (?i)^[^,]*,(?P<QueryTime>[^,]+)
EXTRACT-QueryHostName = (?i)^(?:[^,]*,){2}(?P<QueryHostName>[^,]+)
EXTRACT-QuerySrcAddr = (?i)^(?:[^,]*,){3}(?P<QuerySrcAddr>[^,]+)
EXTRACT-QuerySrcPort = (?i)^(?:[^,]*,){4}(?P<QuerySrcPort>[^,]+)
EXTRACT-QueryDstAddr = (?i)^(?:[^,]*,){5}(?P<QueryDstAddr>[^,]+)
EXTRACT-QueryDstPort = (?i)^(?:[^,]*,){6}(?P<QueryDstPort>[^,]+)
EXTRACT-QueryRecordType = (?i)^(?:[^,]*,){7}(?P<QueryRecordType>[^,]+)
EXTRACT-QueryDomain = (?i)^(?:[^,]*,){8}(?P<QueryDomain>[^,]+)
EXTRACT-QueryTLD = (?i)^(?:[^,]*,){9}(?P<QueryTLD>[^,]+)
EXTRACT-QuerySLD = (?i)^(?:[^,]*,){10}(?P<QuerySLD>[^,]+)
EXTRACT-QueryLevels = (?i)^(?:[^,]*,){11}(?P<QueryLevels>[^,]+)
EXTRACT-QueryScore = (?i)^(?:[^,]*,){12}(?P<QueryScore>[^,]+)
EXTRACT-QueryFlags = (?i)^(?:[^,]*,){13}(?P<QueryFlags>[^,]+)
```
