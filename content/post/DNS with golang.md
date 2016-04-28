+++
date = "2016-04-08T12:05:30+08:00"
draft = true
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

