+++
date = "2016-03-12T16:17:52+08:00"
draft = false
title = "HTTP2的实践过程"

+++

目录
--------------------
* 前言
* 基于openssl自建证书
	* CentOS
	* Ubuntu
* nginx的支持HTTP/2的patch
* nghttp2安装，配置，使用
	* nghttpd作为http2 server
	* nghttp作为http2 client
	* nghttpx作为proxy，转向nginx后端
	* h2load作为压测工具

前言
---------

* 在研究HTTP/2协议时，常常和https协议混在一起，而二者之间的关系是怎样的呢？
* 现有的http2 server中，nginx基于1.9.*有HTTP/2协议的patch，还有nghttp2 server，已经有人运行在个人博客做前端。

只说实践过程，作为记录。

关于涉及到的概念等，需要在别的文档中写。

基于openssl自建证书
------------------------------
在线上配置HTTPS时，需要从权威CA申请证书，在nginx中配置证书crt和私钥key。

        listen 8443 ssl;
        ssl_certificate  /usr/lib/ssl/nginx.crt;
        ssl_certificate_key /usr/lib/ssl/nginx.key;

而在线下调试时，如果需要配置https，则需要自建和签发证书。
[基于openssl自建证书](http://segmentfault.com/a/1190000002569859)

### 在CentOS系统上自建证书

1.自建CA，颁发证书

```bash
CA首先需要自建证书，作为颁发证书所用的根证书。CA的openssl配置文件/etc/pki/tls/openssl.cnf：

####################################################################
[ ca ]
default_ca	= CA_default		# The default ca section

####################################################################
[ CA_default ]

dir		= /etc/pki/CA		# Where everything is kept
certs		= $dir/certs		# Where the issued certs are kept
crl_dir		= $dir/crl		# Where the issued crl are kept
database	= $dir/index.txt	# database index file.
#unique_subject	= no			# Set to 'no' to allow creation of
					# several ctificates with same subject.
new_certs_dir	= $dir/newcerts		# default place for new certs.

certificate	= $dir/cacert.pem 	# The CA certificate
serial		= $dir/serial 		# The current serial number
crlnumber	= $dir/crlnumber	# the current crl number
					# must be commented out to leave a V1 CRL
crl		= $dir/crl.pem 		# The current CRL
private_key	= $dir/private/cakey.pem# The private key
RANDFILE	= $dir/private/.rand	# private random number file

x509_extensions	= usr_cert		# The extentions to add to the cert

# Comment out the following two lines for the "traditional"
# (and highly broken) format.
name_opt 	= ca_default		# Subject Name options
cert_opt 	= ca_default		# Certificate field options



default_days	= 365			# how long to certify for
default_crl_days= 30			# how long before next CRL
default_md	= default		# use public key default MD
preserve	= no			# keep passed DN ordering

policy		= policy_match

# For the CA policy
[ policy_match ]
countryName		= match
stateOrProvinceName	= match
organizationName	= match
organizationalUnitName	= optional
commonName		= supplied
emailAddress		= optional
...
```
其中我们可以看到根地址在/etc/pki/CA下，且指明了CA证书certificate是该目录下的cacert.pem，私钥在private/cakey.pem中，CA需要匹配countryName、stateOrProvinceName和organizationName，且commonName需要提供，这点比较重要。

```bash	
在/etc/pki/CA下创建初始文件

$ touch serial index.txt
$ echo 01 > serial

```

2.生成根密钥 

```bash
$ cd /etc/pki/CA
$ openssl genrsa -out private/cakey.pem 2048
```

3.生成根证书

```bash	
使用req指令，通过私钥，生成自签证书
$ openssl req -new -x509 -key private/cakey.pem -out cacert.pem
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:BeiJing
Locality Name (eg, city) [Default City]:
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:root
Email Address []:
```	

4.为nginx server生成密钥

```bash
$ mkdir /data/zhendong/nginx_ssl
$ openssl genrsa -out nginx.key 2048
```

5.为nginx生成 证书签署请求

```bash
$ openssl req -new -key nginx.key -out nginx.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:BeiJing
Locality Name (eg, city) [Default City]:
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:localhost
Email Address []:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:

Common Name填成nginx server的server_name用来访问。
在openssl.cnf中需要match的项目，一定要一样。
```

6.向CA请求证书

```bash
$ openssl ca -in nginx.csr -out nginx.crt

如果失败，可以尝试以下命令

$ openssl x509 -req -in nginx.csr -CA /etc/pki/CA/cacert.pem -CAkey /etc/pki/CA/private/cakey.pem -CAcreateserial -out nginx.crt
```

7.配置nginx

```bash
listen 8443 ssl ;
ssl_certificate  /data1/zhendong/nginx_ssl/nginx.crt;
ssl_certificate_key /data1/zhendong/nginx_ssl/nginx.key;
```

8.通过curl访问

```bash
$ curl --cacert /etc/pki/CA/cacert.pem https://localhost:8443/
this is abtesting server

$ curl --cacert /etc/pki/CA/cacert.pem https://127.0.0.1:8443/
curl: (51) SSL: certificate subject name 'localhost' does not match target host name '127.0.0.1'
```

###在Ubuntu系统上自建证书

Ubuntu系统与CentOS的不同之处在于软件包管理不同，当克服这部分不同后，就可以执行与CentOS系统一样的操作。

首先Ubuntu的openssl目录在**/usr/lib/ssl**下，而实际上这是一个软链接。

```bash
huang@ThinkPad-X220:/usr/lib/ssl$ ll /usr/lib/ssl/ total 52
drwxr-xr-x   3 root root  4096  8月 27 12:18 ./
drwxr-xr-x 238 root root 40960  8月 26 11:18 ../
lrwxrwxrwx   1 root root    14  2月  4  2015 certs -> /etc/ssl/certs/
drwxr-xr-x   2 root root  4096  7月  8 14:34 misc/
lrwxrwxrwx   1 root root    20  6月 11 23:35 openssl.cnf -> /etc/ssl/openssl.cnf
lrwxrwxrwx   1 root root    16  2月  4  2015 private -> /etc/ssl/private/
```
不管怎样，我们的工作将在**/usr/lib/ssl**进行。

```bash
/usr/lib/ssl/openssl.cnf内容如下：
####################################################################
[ ca ]
default_ca	= CA_default		# The default ca section

####################################################################
[ CA_default ]

dir		= ./demoCA		# Where everything is kept
certs		= $dir/certs		# Where the issued certs are kept
crl_dir		= $dir/crl		# Where the issued crl are kept
database	= $dir/index.txt	# database index file.
#unique_subject	= no			# Set to 'no' to allow creation of
					# several ctificates with same subject.
new_certs_dir	= $dir/newcerts		# default place for new certs.

certificate	= $dir/cacert.pem 	# The CA certificate
serial		= $dir/serial 		# The current serial number
crlnumber	= $dir/crlnumber	# the current crl number
					# must be commented out to leave a V1 CRL
crl		= $dir/crl.pem 		# The current CRL
private_key	= $dir/private/cakey.pem# The private key
RANDFILE	= $dir/private/.rand	# private random number file

x509_extensions	= usr_cert		# The extentions to add to the cert
```

从配置文件中看到，我们需要在**/usr/lib/ssl**下建立**demoCA**目录以及其他。

```bash
$ cd /usr/lib/ssl
$ mkdir demoCA
$ mkdir demoCA/newcerts
$ mkdir demoCA/private
```

余下的操作将与在CentOS系统上没有区别。最后执行情况为：

```bash
$ curl --cacert /usr/lib/ssl/demoCA/cacert.pem https://localhost:8443
this is abtesting server
```

nginx的支持HTTP/2的patch
------------------------------------
	
> nginx目前已正式支持HTTP/2，包括tengine-2.1.2+ 和 openresty

nginx在8月份的时候从1.9.3版本推出了支持HTTP/2的[patch](http://nginx.org/patches/http2/)，使用时与标准nginx并无区别。

```bash
$ cd nginx-1.9.4/
$ patch -p1 < patch.http2-v3_1.9.4.txt  
$ ./configure --with-http_v2_module --with-http_ssl_module
$ make
$ make install
```
支持http2的nginx关键配置为

```bash
	listen 8443 http2;
```

只需要在listen后加http2就可以了。目前curl在基于nghttp2提供的HTTP/2库后，可以支持访问http2的server，但是目前没有配置成功，所以对支持http2的nginx进行访问的工作，将在介绍完nghttp2后一并记录。

nghttp2安装，配置，使用
--------------------------------
[nghttp2](https://github.com/tatsuhiro-t/nghttp2)是由tatsuhiro开发的，之前的spdylay也是他开发的，一直走在http2.0的前列。nghttp2包括了HTTP/2.0的库，基于这个库tatsu实现了HTTP/2.0的server、client和压测工具h2load。

由于nghttp2所依赖的库太新了，目前只在Ubuntu系统成功安装。安装过程：

```bash
$ autoreconf -i
$ automake
$ autoconf
$ ./configure
$ make
```	

其中**./configure**的结果非常重要

```bash
    Version:        1.2.2-DEV shared 14:8:0
    Host type:      x86_64-unknown-linux-gnu
    Install prefix: /usr/local
    C compiler:     gcc
    CFLAGS:         -g -O2
    WARNCFLAGS:     
    LDFLAGS:        
    LIBS:           
    CPPFLAGS:       
    C preprocessor: gcc -E
    C++ compiler:   g++
    CXXFLAGS:       -g -O2 -std=c++11
    CXXCPP:         g++ -E
    Library types:  Shared=yes, Static=yes
    Python:
      Python:         /usr/bin/python
      PYTHON_VERSION: 2.7
      pyexecdir:      ${exec_prefix}/lib/python2.7/dist-packages
      Python-dev:     yes
      PYTHON_CPPFLAGS:-I/usr/include/python2.7
      PYTHON_LDFLAGS: -L/usr/lib -lpython2.7
      Cython:         cython
    Test:
      CUnit:          yes
      Failmalloc:     yes
    Libs:
      OpenSSL:        yes
      Libxml2:        yes
      Libev:          yes
      Libevent(SSL):  yes
      Spdylay:        yes
      Jansson:        yes
      Jemalloc:       yes
      Zlib:           yes
      Boost CPPFLAGS: 
      Boost LDFLAGS:  
      Boost::ASIO:    
      Boost::System:  
      Boost::Thread:  
    Features:
      Applications:   yes
      HPACK tools:    yes
      Libnghttp2_asio:no
      Examples:       yes
      Python bindings:yes
      Threading:      yes
      Third-party:    yes
```

编译帮助文档

```bash
make html
```

### nghttpd作为http2 server

http2-no-tls

	nghttpd -v 8080   -n 24 --no-tls -d ~/workspace/Nginx_ABTesting/utils/html/ 

http2-with-tls

	nghttpd -v 8080 -n 24 /usr/lib/ssl/nginx.key /usr/lib/ssl/nginx.crt -d ~/workspace/Nginx_ABTesting/utils/html/  

关于nghttpd的选项，其实可以与nginx配置做到一一对照的。目前对nghttpd的源码及实现了解的比较少，因其日本人的过于C++代码的风格实在晦涩难懂，所以很少做调优。

### nghttp作为http2 client

http2-client-no-tls

	nghttp http://127.0.0.1:8080

http2-client-with-tls

	nghttp --cert /usr/lib/ssl/demoCA/cacert.pem https://127.0.0.1:8080

	nghttp https://127.0.0.1:8080

经过使用，nghttp也可以对nginx发出http2请求并成功返回。

通过strace和阅读源码，nghttp（包括压测工具h2load）作为client时，会读取系统的证书/usr/lib/ssl/demoCA/cacert.pem，因此可以不用指定。
### nghttpx作为proxy，转向nginx后端

client ——> http2-proxy-no-tls ——> http1.1 upstream(nginx):
	
	# nghttpx -f127.0.0.1,8080 -b127.0.0.1,8022 --frontend-no-tls
	
	# curl 127.0.0.1:8022
	this is beta3 server

	# nghttp http://127.0.0.1:8080
	this is beta3 server

client ——> http2-proxy-with-tls ——> http1.1 upstream(nginx):
	
	# nghttpx -f127.0.0.1,8080 -b127.0.0.1,8022 /usr/lib/ssl/nginx.key /usr/lib/ssl/nginx.crt
	
	# nghttp https://127.0.0.1:8080
	this is beta3 server

采用nginx-http2作为proxy，使用方法与nginx-http1.1没有区别。

### h2load作为压测工具

```bash
$ h2load -n100 -c10 -t4  https://127.0.0.1:8080
starting benchmark...
spawning thread #0: 3 concurrent clients, 25 total requests
spawning thread #1: 3 concurrent clients, 25 total requests
spawning thread #2: 2 concurrent clients, 25 total requests
spawning thread #3: 2 concurrent clients, 25 total requests
Protocol: TLSv1.2
Cipher: ECDHE-RSA-AES128-GCM-SHA256
progress: 8% done
progress: 16% done
progress: 24% done
progress: 32% done
progress: 40% done
progress: 48% done
progress: 56% done
progress: 64% done
progress: 72% done
progress: 80% done
progress: 88% done
progress: 96% done

finished in 67.29ms, 1486 req/s, 111.02KB/s
requests: 100 total, 100 started, 100 done, 100 succeeded, 0 failed, 0 errored
status codes: 100 2xx, 0 3xx, 0 4xx, 0 5xx
traffic: 7650 bytes total, 3450 bytes headers, 2100 bytes data
                     min         max         mean         sd        +/- sd
time for request:      343us      8.76ms      1.40ms      1.49ms    91.00%
```
h2load的使用与wrk没有区别，参数都是一样的。

根据不同配置，我们有以下几种场景：

```bash
server/proxy			https

nginx-http/1.1			with-tls
nginx-http/2			no-tls
nghttpd
```
相结合，压测场景比较多。幸运的是，不论server是nginx还是nghttpd，其参数和调优都可以指定，比如线程数；不论wrk还是h2load，参数也可以指定，使用起来区别不大。
