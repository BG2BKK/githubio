+++
date = '2016-04-13T11:37:24+08:00'
draft = false
title = 'DNS with golang'

+++

* [DNS消息格式](http://www.cnblogs.com/cobbliu/archive/2013/04/02/2996333.html)
* [EDNS详解](http://www.cnblogs.com/cobbliu/p/3188632.html)
* [rfc6891](https://tools.ietf.org/html/rfc6891)
* [edns_client_subnet draft](https://www.ietf.org/id/draft-ietf-dnsop-edns-client-subnet-08.txt)
* [小米的ends实践](http://noops.me/?p=653)
	
```bash
6. respond时也需要增加一个Additional RRs区域，直接把请求的Additional内容发过去就可以(如果支持source netmask，将请求中的source netmask复制到scope netmask中，OpenDNS要求必须支持scope netmask)
```
	* 意思是非OpenDNS就可以不支持scope netmask吗？目前新浪的仍然不支持

* [miekg/dns: golang lib](https://github.com/miekg/dns)
* dig with edns
	* https://www.gsic.uva.es/~jnisigl/dig-edns-client-subnet.html
	* http://xmodulo.com/geographic-location-ip-address-command-line.html
	* 在线查询：curl ipinfo.io/23.66.166.151


* DNS报文格式
	* IP packet
		* IP Header 20 bytes
		* IP Data: UDP
			* UDP Header 8 bytes
			* UDP Data: DNS
				* DNS Header 12 bytes
				* DNS Data: RR
					* RR: Question[]
					* RR: Answer[]
					* RR: Authority[]
					* RR: Additional[]
					* RR
						* QName
						* QType
						* QClass
						* RDLENGTH
						* RDATA
					* RR OPT
						* QName null
						* QType OPT=41
						* QClass = UDP payload 2bytes
						* TTL = Extended-RCODE 1byte: extend + VERSION 1byte: 0 + Z 2bytes: 0
						* RDLen	len of data(OPT)
						* OPT
							* Option-Code 2bytes: EDNS0_SUBNET
							* Option-Length 2bytes
							* Option-Data	
								* Family 1byte: IPV4(1)
								* Source Netmask 1byte: 32
								* Scope Netmask 1byte: 0
								* Client Subnet 4bytes: 65.135.152.203
		

* 我们总是在追一些时髦的技术，而不顾基础还不牢靠

* 我们总是看见新的框架，然而框架本质上仍然是那些东西，mvc，cs


<!-- http://www.tcpipguide.com/free/index.htm -->
<style type="text/css">
div.t-h:hover > div {display:block;}
.t-m-s {font-size:80%;border: 2px solid #ccc;border-spacing:0;border-collapse:collapse;}
.g-h-s {background:#9bf;color:#339;padding:4px; font-size:80%;font-weight:normal;border:1px solid #ccc;}
</style>

<table class="t-m-s">
<div align="center"><table border="3" cellpadding="4" cellspacing="2"><caption align="top"><p align="center"><font face="Arial"><b>TCP Message Format </b></font></p></caption>
<tbody><tr class="g-h-s">
<td>Octet</td>
<td>Bits</td>
<td>Len</td>
<td>Name</td>
<td>Notes</td>
</tr>
<tr>
<td>0-1</td>
<td>-</td>
<td>2</td>
<td>Source port</td>
<td>-</td>
</tr>
<tr>
<td>2-3</td>
<td>-</td>
<td>2</td>
<td>Destination Port</td>
<td>-</td>
</tr>
<tr>
<td>4-7</td>
<td>-</td>
<td>4</td>
<td>Sequence number</td>
<td>position of last octet we sent.</td>
</tr>
<tr>
<td>8-11</td>
<td>-</td>
<td>4</td>
<td>Acknowledge Number</td>
<td>Next octet number we expect from the peer.</td>
</tr>
<tr>
<td>12</td>
<td>0-3</td>
<td>-</td>
<td>HLEN</td>
<td>4 bits. The number of 32 bit multiples (4 octets) in the TCP header including any 'options' fields.</td>
</tr>
<tr>
<td>12</td>
<td>4-7</td>
<td>-</td>
<td>Reserved</td>
<td>should be zero</td>
</tr>
<tr>
<td>13</td>
<td>-</td>
<td>1</td>
<td>Code bits</td>
<td>8 bits (6 used) valid if 1<br>
bit 0 (URG) Urgent<br>
bit 1 (ACK) Acknowledgement<br>
bit 2 (PSH) Requests PUSH<br>
bit 3 (RST) Reset connection<br>
bit 4 (SYN) Sync sequence numbers<br>
bit 5 (FIN) sender finished</td>
</tr>
<tr>
<td>14-15</td>
<td>-</td>
<td>2</td>
<td>Window</td>
<td>Specifies the amount of data we can accept.</td>
</tr>
<tr>
<td>16-17</td>
<td>-</td>
<td>2</td>
<td>Checksum</td>
<td>Standard IP checksum. Includes a <a href="#tcp-check" class="t-db">TCP pseudo header</a>.</td>
</tr>
<tr>
<td>18-19</td>
<td>-</td>
<td>2</td>
<td>Urgent pointer</td>
<td>Points to end of urgent data.</td>
</tr>
<tr class="t-c">
<td colspan="5"><a href="#tcp_opts" class="t-db">TCP Options</a></td>
</tr>
<tr class="t-c">
<td colspan="5">TCP data</td>
</tr>
</tbody>
</div>
</table>
