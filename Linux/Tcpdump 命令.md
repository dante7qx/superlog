## Tcpdump

命令用法

```shell
tcpdump -h
tcpdump [ options ] [ expression ]
Usage: tcpdump [-aAbdDefhHIJKlLnNOpqStuUvxX#] [ -B size ] [ -c count ]
		[ -C file_size ] [ -E algo:secret ] [ -F file ] [ -G seconds ]
		[ -i interface ] [ -j tstamptype ] [ -M secret ] [ --number ]
		[ -Q in|out|inout ]
		[ -r file ] [ -s snaplen ] [ --time-stamp-precision precision ]
		[ --immediate-mode ] [ -T type ] [ --version ] [ -V file ]
		[ -w file ] [ -W filecount ] [ -y datalinktype ] [ -z postrotate-command ]
[ -g ] [ -k ] [ -o ] [ -P ] [ -Q met[ --time-zone-offset offset ]
		[ -Z user ] [ expression ]
```

**tcpdump  options  expression** 

- options只负责在哪（网口），怎么抓（出栈/入栈），以及怎么打印。
- expression则相当与filters，例如只抓tcp的包，还是只抓udp的包。

**options**

- -t — 不打印时间戳
- -n — 一个n代表不做地址转换，即打印IP地址，而不是对应的Hostname；两个n代表不做端口转换，例如打印80，而不是http。
- -e — 打印MAC地址。
- -i — 指定监听/抓包的网口。
- -P — 指定监听/抓包的方向，in代表从网口进来，out代表从网口出去。
- -A — 以Ascii码打印包的内容，多用于Http, MQ的明文数据包。

**expression**

- host（缺省）、net、port
  - host — host 210.27.48.2，指明 210.27.48.2是一台主机。
  - net   — net 202.0.0.0， 指明 202.0.0.0是一个网络地址。
  - port — port 8101，指明 8101 是端口号。
- src、dst、src or dst（缺省）、dst or src
  - src — src 210.27.48.2，指明ip包中源地址是210.27.48.2。
  - dst — dst net 202.0.0.0， 指明目的网络地址是202.0.0.0。
- ether、icmp、arp、ip、tcp、udp
- and、or、not、==

**demo**

```bash
## 获取 8101 端口的进/出包
sudo tcp port 8101

## 在en0上抓取来自192.168.0.192/26网段，访问3306端口的包
sudo tcpdump -tnei en0 src net 192.168.0.192/26 and dst port 3306

## 抓取并打印本地所有收到的HTTP的明文数据
sudo tcpdump -i any src port 80 -A -P in
```

```shell
## 在en0上抓取arp或icmp的包，不打印时间戳，打印MAC，不做hostname转化:
sudo tcpdump -tnei en0 arp or icmp

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on en0, link-type EN10MB (Ethernet), capture size 262144 bytes

f8:6c:e1:a2:bb:70 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.1.4 tell 192.168.1.1, length 46

a4:5e:60:bd:42:41 > f8:6c:e1:a2:bb:70, ethertype ARP (0x0806), length 42: Reply 192.168.1.4 is-at a4:5e:60:bd:42:41, length 28

f8:6c:e1:a2:bb:70 > ff:ff:ff:ff:ff:ff, ethertype ARP (0x0806), length 60: Request who-has 192.168.1.3 tell 192.168.1.1, length 46
```

```shell
## 在en0上抓取访问主机IP为 192.168.1.4，并且端口是 443 的 TCP 包
sudo tcpdump -nti en0 dst host 192.168.1.4 and  src port 443 and tcp

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on en0, link-type EN10MB (Ethernet), capture size 262144 bytes
IP 47.74.46.254.443 > 192.168.1.4.65291: Flags [.], seq 750255810:750255846, ack 815597361, win 260, options [nop,nop,TS val 124173824 ecr 1347398360], length 36

IP 47.74.46.254.443 > 192.168.1.4.65291: Flags [.], seq 36:72, ack 1, win 260, options [nop,nop,TS val 124174144 ecr 1347410012], length 36

IP 47.74.46.254.443 > 192.168.1.4.65291: Flags [.], seq 144:180, ack 1, win 260, options [nop,nop,TS val 124174174 ecr 1347411273], length 36

IP 47.74.46.254.443 > 192.168.1.4.65291: Flags [.], seq 72:108, ack 1, win 260, options [nop,nop,TS val 124174174 ecr 1347411273], length 36
```



参考资料**

- https://medoc.readthedocs.io/en/latest/docs/utils/tcpdump.html
- https://www.cnblogs.com/maifengqiang/p/3863168.html