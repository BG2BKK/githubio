+++
date = "2016-05-05T20:13:45+08:00"
draft = false
title = "DNS with golang"

+++

* Record Tyeps
    * [Record Types](DNS Management: Record Types and When To Use Them)

    * A records:    A 记录

    * CNAME

    * MX Record

    
* miekg/dns

* dig with edns
	* https://www.gsic.uva.es/~jnisigl/dig-edns-client-subnet.html
	* http://xmodulo.com/geographic-location-ip-address-command-line.html
	* curl ipinfo.io/23.66.166.151
	* 

* 发现UDP和TCP发出的包，效果大不同
	* 急需DNS抓包工具

* DNS报文格式
	* IP packet
		* IP Header 20 bytes
		* IP Data: UDP
			* UDP Header 8 bytes
			* UDP Data: DNS
				* DNS Header 12 bytes
				* DNS Data: RR
					* RR: Question
					* RR: Answer
					* RR: Authority
					* RR: Additional
					* RR
						* QName
						* QType
						* QClass
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

<div align="center"><table border="3" cellpadding="4" cellspacing="2"><caption align="top"><p align="center"><font face="Arial"><b>Table 147: UDP Message Format </b></font></p></caption>
<tbody><tr>
<td bgcolor="#CCCCFF"><p align="center"><font face="Arial"><b>Field 
Name</b></font></p>
</td>
<td bgcolor="#CCCCFF"><p align="center"><font face="Arial"><b>Size (bytes)</b></font></p>
</td>
<td align="left" bgcolor="#CCCCFF"><p align="left"><font face="Arial"><b>Description</b></font></p>
</td>
</tr>
<tr>
<td align="center"><p align="center"><font face="Arial"><b><i>Source 
Port</i></b></font></p>
</td>
<td align="center"><p align="center"><font face="Arial">2</font></p>
</td>
<td><p align="left"><font face="Arial"><b><i>Source Port:</i></b> The 
16-bit port number of the process that originated the UDP message on 
the source device. This will normally be an ephemeral (client) port 
number for a request sent by a client to a server, or a well-known/registered 
(server) port number for a reply sent by a server to a client. </font><a href="t_TCPIPTransportLayerProtocolTCPandUDPAddressingPort.htm"><font face="Arial" color="#0101C0">See 
the section describing port numbers for details</font></a><font face="Arial">.</font></p>
</td>
</tr>
<tr>
<td align="center" bgcolor="#E1E1FF"><p align="center"><font face="Arial"><b><i>Destination 
Port</i></b></font></p>
</td>
<td align="center" bgcolor="#E1E1FF"><p align="center"><font face="Arial">2</font></p>
</td>
<td bgcolor="#E1E1FF"><p align="left"><font face="Arial"><b><i>Destination 
Port:</i></b> The 16-bit port number of the process that is the ultimate 
intended recipient of the message on the destination device. This will 
usually be a well-known/registered (server) port number for a client 
request, or an ephemeral (client) port number for a server reply. Again, 
</font><a href="t_TCPIPTransportLayerProtocolTCPandUDPAddressingPort.htm"><font face="Arial" color="#0101C0">see 
the section describing port numbers for details</font></a><font face="Arial">.</font></p>
</td>
</tr>
<tr>
<td align="center"><p align="center"><font face="Arial"><b><i>Length</i></b></font></p>
</td>
<td align="center"><p align="center"><font face="Arial">2</font></p>
</td>
<td><p align="left"><font face="Arial"><b><i>Length:</i></b> The length 
of the entire UDP datagram, including both header and <i>Data</i> fields.</font></p>
</td>
</tr>
<tr>
<td align="center" bgcolor="#E1E1FF"><p align="center"><font face="Arial"><b><i>Checksum</i></b></font></p>
</td>
<td align="center" bgcolor="#E1E1FF"><p align="center"><font face="Arial">2</font></p>
</td>
<td bgcolor="#E1E1FF"><p align="left"><font face="Arial"><b><i>Checksum:</i></b> 
An optional 16-bit checksum computed over the entire UDP datagram plus 
a special “pseudo header” of fields. See below for more information. 
</font></p>
</td>
</tr>
<tr>
<td align="center"><p align="center"><font face="Arial"><b><i>Data</i></b></font></p>
</td>
<td align="center"><p align="center"><font face="Arial">Variable</font></p>
</td>
<td><p align="left"><font face="Arial"><b><i>Data:</i></b> The encapsulated 
higher-layer message to be sent.</font></p>
</td>
</tr>
</tbody></table>

<br><a name="Figure_200"></a></div>

<style type="text/css">
div.l-f table{width:100%;padding:4px;}

table {margin: 0 auto;}
table.t-m-n > tbody > tr > td {border:1px solid #ccc;padding:4px;}
table.t-m-s > tbody > tr > td {border:1px solid #ccc;padding:4px;}
table.p-m-n > tbody > tr > td {padding:4px;border-collapse:collapse;}
table.p-m-s > tbody > tr > td {padding:4px;border-collapse:collapse;}
tr {vertical-align:top;}
/* end tag modifiers - Printer friendly */


div.t-h:hover > div {display:block;}
.t-m-s {font-size:80%;border: 2px solid #ccc;border-spacing:0;border-collapse:collapse;}
.g-h-s {background:#9bf;color:#339;padding:4px; font-size:80%;font-weight:normal;border:1px solid #ccc;}
</style>

<table class="t-m-s">
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
</tbody></table>
