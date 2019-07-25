kerberos 认证 High Availability 搭建步骤

kerberos 存在单点故障问题，因此使用主备模式提高可用性

***

使用两台机器

|iedev07-2945107.slc07.dev.ebayc3.com|master|
|---|---|
|iedev08-2945122.slc07.dev.ebayc3.com|slave|
# 预先准备 安装kdc服务器 
`yum -y install  krb5-server krb5-auth-dialog krb5-workstation krb5-devel krb5-libs`

安装完毕之后需要对三个文件进行修改
```
/etc/krb5.conf
/var/kerberos/krb5kdc/kdc.conf
/var/kerberos/krb5kdc/kadm5.acl
```




## 第1步 配置两台机器的/etc/hosts文件 

## 第2步 配置/etc/krb5.conf文件(在master机器上）
cat /etc/krb5.conf 可以看到，其中realm的配置自己按照需求配，kdc写上master和slave机器的地址和端口，端口默认都是88，这个会在kdc.conf里面配置
```
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 default_realm = EBAY.COM
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 400
 renew_lifetime = 600
 forwardable = true
 udp_preference_limit = 1

[realms]
 EBAY.COM = {
   kdc = iedev07-2945107.slc07.dev.ebayc3.com:88
   kdc = iedev08-2945122.slc07.dev.ebayc3.com:88
   admin_server = iedev07-2945107.slc07.dev.ebayc3.com:749
   default_domain = EBAY.COM
 }


[domain_realm]
.ebay.com = EBAY.COM
ebay.com = EBAY.COM

[kdc]
 profile = /var/kerberos/krb5kdc/kdc.conf
```
## 第3步 配置/var/kerberos/krb5kdc/kdc.conf(在master机器上）
cat /var/kerberos/krb5kdc/kdc.conf 写上realms的名字，默认文件的路径不用动，端口也是默认开88
```
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 EBAY.COM = {
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }
```

## 第4步 配置/var/kerberos/krb5kdc/kadm5.acl(在master机器上）
cat /var/kerberos/krb5kdc/kadm5.acl
```*/admin@EBAY.COM  *```

## 第5步  生成数据库和凭证（在master机器上）
```kdb5_util create -r EBAY.COM -s ```
-r指定realm名字 会提示输入密码,成功之后/var/kerberos/krb5kdc/下会生成4个数据库的principal文件

再输入```kadmin.local```是免密登陆的
登陆之后 输入

               ank -randkey host/iedev07-2945107.slc07.dev.ebayc3.com
              
               ank -randkey host/iedev08-2945122.slc07.dev.ebayc3.com
   
               xst host/iedev07-2945107.slc07.dev.ebayc3.com@EBAY.COM
           
               xst host/iedev08-2945122.slc07.dev.ebayc3.com@EBAY.COM
               
正常执行到这一步在/etc/ 下生成了一个krb5.keytab文件
如果到这一步有问题，到/var/kerberos/krb5kdc/目录下删除4个principal的文件，检查/etc/krb5.conf、/var/kerberos/krb5kdc/kdc.conf、/var/kerberos/krb5kdc/kadm5.acl并且清除缓存```kdestroy```重新生成数据库和凭证


## 第6步 将配置文件发送到slave机器对应的目录文件夹下----直接scp命令（在master机器上）
```
/etc/krb5.conf
/var/kerberos/krb5kdc/kdc.conf
/var/kerberos/krb5kdc/kadm5.acl
/var/kerberos/krb5kdc/.k5.EBAY.COM  这个是隐藏文件  ll -a 显示
```

## 第7步 在slave机器上安装kerberos软件 （在slave机器上）
`yum -y install  krb5-server krb5-auth-dialog krb5-workstation krb5-devel krb5-libs`

## 第8步 slave机器配置主从（在slave机器上）

到/var/kerberos/krb5kdc/新建一个文件kpropd.acl内容为
```host/iedev07-2945107.slc07.dev.ebayc3.com@EBAY.COM
```host/iedev08-2945122.slc07.dev.ebayc3.com@EBAY.COM

