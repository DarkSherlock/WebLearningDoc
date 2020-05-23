## TCP ##
三次握手的过程：


多么清晰的一张图，当然了，也不是我画的，我也只是引用过来说明问题了。

第一次握手：建立连接。客户端发送连接请求报文段，将SYN位置为1，Sequence Number为x；然后，客户端进入SYN_SEND状态，等待服务器的确认；
第二次握手：服务器收到SYN报文段。服务器收到客户端的SYN报文段，需要对这个SYN报文段进行确认，设置Acknowledgment Number为x+1(Sequence Number+1)；同时，自己自己还要发送SYN请求信息，将SYN位置为1，Sequence Number为y；服务器端将上述所有信息放到一个报文段（即SYN+ACK报文段）中，一并发送给客户端，此时服务器进入SYN_RECV状态；
第三次握手：客户端收到服务器的SYN+ACK报文段。然后将Acknowledgment Number设置为y+1，向服务器发送ACK报文段，这个报文段发送完毕以后，客户端和服务器端都进入ESTABLISHED状态，完成TCP三次握手。
完成了三次握手，客户端和服务器端就可以开始传送数据。以上就是TCP三次握手的总体介绍。
那四次分手呢？

第一次分手：主机1（可以使客户端，也可以是服务器端），设置Sequence Number和Acknowledgment Number，向主机2发送一个FIN报文段；此时，主机1进入FIN_WAIT_1状态；这表示主机1没有数据要发送给主机2了；
第二次分手：主机2收到了主机1发送的FIN报文段，向主机1回一个ACK报文段，Acknowledgment Number为Sequence Number加1；主机1进入FIN_WAIT_2状态；主机2告诉主机1，我“同意”你的关闭请求；
第三次分手：主机2向主机1发送FIN报文段，请求关闭连接，同时主机2进入LAST_ACK状态；
第四次分手：主机1收到主机2发送的FIN报文段，向主机2发送ACK报文段，然后主机1进入TIME_WAIT状态；主机2收到主机1的ACK报文段以后，就关闭连接；此时，主机1等待2MSL后依然没有收到回复，则证明Server端已正常关闭，那好，主机1也可以关闭连接了。

IP 192.168.1.116.3337 > 192.168.1.123.7788: S 3626544836:3626544836   
IP 192.168.1.123.7788 > 192.168.1.116.3337: S 1739326486:1739326486 ack 3626544837   
IP 192.168.1.116.3337 > 192.168.1.123.7788: ack 1739326487,ack 1   

第一次握手：192.168.1.116发送位码syn＝1,随机产生seq number=3626544836的数据包到192.168.1.123,192.168.1.123由SYN=1知道192.168.1.116要求建立联机;

第二次握手：192.168.1.123收到请求后要确认联机信息，向192.168.1.116发送ack number=3626544837,syn=1,ack=1,随机产生seq=1739326486的包;

第三次握手：192.168.1.116收到后检查ack number是否正确，即第一次发送的seq number+1,以及位码ack是否为1，若正确，192.168.1.116会再发送ack number=1739326487,ack=1，192.168.1.123收到后确认seq=seq+1,ack=1则连接建立成功。

![http](https://github.com/DarkSherlock/note/blob/master/images/http.webp)

参考：

+ [https://mp.weixin.qq.com/s/hkAS6kBY-mtN0WSNytA2gg](https://mp.weixin.qq.com/s/hkAS6kBY-mtN0WSNytA2gg)

+ [https://github.com/jawil/blog/issues/14](https://github.com/jawil/blog/issues/14)


##  HTTPS ##

![https请求过程](https://upload-images.jianshu.io/upload_images/449687-2c0fe3a7fbd6b3ad.png?imageMogr2/auto-orient/strip|imageView2/2/format/webp)
客户端发送 client_hello，包含一个随机数 random1。 

服务端回复 server_hello，包含一个随机数 random2，携带了证书公钥 P。 

客户端接收到 random2 之后就能够生成 premaster_secrect （对称加密的密钥）以及 master_secrect（用premaster_secret加密后的数据）。 

客户端使用证书公钥 P 将 premaster_secrect 加密后发送给服务器 (用公钥P对premaster_secret加密)。 

服务端使用私钥解密得到 premaster_secrect。又由于服务端之前就收到了随机数 1，所以服务端根据相同的生成算法，在相同的输入参数下，求出了相同的 master secrect。   
[https://mp.weixin.qq.com/s/E75toyRukUHEtt34-snEgQ](https://mp.weixin.qq.com/s/E75toyRukUHEtt34-snEgQ)   
[https://www.jianshu.com/p/80d6f39fd40b](https://www.jianshu.com/p/80d6f39fd40b)   
[https://www.jianshu.com/p/ca7df01a9041](https://www.jianshu.com/p/ca7df01a9041)   

服务器向CA申请证书，CA用自己的私钥加密服务器的公钥然后再用hash算法生成一个签名。客户端发起请求的时候，服务器下发CA签发的证书，证书包含加密过后的服务器公钥、域名、证书签名、证书签名使用的hash算法。
客户端拿到证书后，使用客户端内置的CA证书的公钥去解密证书，然后获取到签名和hash算法，然后客户端也用这个hash算法去计算下证书内容是否和证书的签名一致，不一致的话代表被中间人篡改过，通信就结束;一致的话就代表证书里的公钥是服务器下发的公钥,然后客户端会生成一个随机数（对称加密算法的key)并且携带本地支持的对称加密算法有哪一些，然后用服务器公钥加密发送给服务器，服务器接收到后用自己的私钥解密，然后选择一套对称加密算法后然后告知客户端，后面我们就用这个随机数和这套对称加密算法来加密数据吧。

中间人攻击：

- 中间人可以拦截服务端下发的证书，它如果自己生成一对公钥和私钥，用自己的公钥替换服务器的公钥，这样他需要重新加密证书，但是没有CA的私钥所以无法重新加密，这就行不通了。它也可以向CA申请证书，然后替换掉服务端的证书下发给客户端，但是客户端拿到证书后再去比较证书里的域名是不是自己要请求的域名，如果不是也会报出警告，所以客户端也能感知到证书被替换了。


