### 微信删除好友恢复

​        微信的聊天、好友数据主要存放在本地加密数据库EnMicroMsg.db中的message、rcontact数据表中，数据库文件存储在/data/data/com.tencent.mm目录下，/data/data目录属于安卓私有目录，对用户以及其他应用程序不可见，需要手机获取root权限才能查看（这个操作十分麻烦，可操作性不大）。

#### 一、  如何拿到私有目录下的数据库文件，需要根据手机型号进行不同的操作：

​		(1) 小米手机：设置->更多设置-> 备份与恢复， 进行备份应用的数据，在外部存储(/sdcard/下)/MIUI/backup/AllBackUp目录下保存了备份文件 微信(com.tencent.mm).bak； 将文件拷贝至电脑，用7z压缩包管理软件解压出来。

​       (2) 其他型号手机：未测试可行性（手机只有迁移聊天记录的功能，不能找回已删除好友）

#### 二、 拿到加密数据库文件如何解密：

   (1) 微信采用sqlite的加强版sqlcipher([windows-sqlcipher工具](https://pan.baidu.com/s/1IqhgbUGZCsLh4QmSD669vA ) / 网盘提取码 **yjem**)进行sqlite数据库操作，sqlcipher对数据库进行了加密。

   (2) 数据库解密-密码的生成：

   ​    手机MIEI码+uni 拼接后字符串的md5码的前7位；

   ​     (a) 手机MIEI码获取：拨号键盘上输入“*#06#”； 一般现在手机是双卡，有两个MIEI码；

   ​     (b) uni (userinformation)  用户唯一性ID, 存储在 shared_prefs/auth_info_key_prefs.xml (备份文件的sp下)的_auth_uin中。

   ​     (c) 生成md5码(python环境下)：
   

   ```python
   # Linux环境下：echo -n "$miei_uni" | md5sum | cut -c 1-7
   def generate_passwd(miei_uni):
       import hashlib
       return hashlib.md5(miei_uni.encode()).hexdigest()[:7]
   ```
		
   
   ​    (d) 如果两个MIEI码和uni的拼接都不成功的话，直接使用"1234567890ABCDEF"和uni码拼接。
   
#### 三、  找回已删除好友

   ##### 3.1  找回删除好友：
    

   ​      直接在sqlcipher GUI中执行SQL语句：

   ```sql
   SELECT * FROM rcontact WHERE nickname like '%记忆的关键字%'
   /*
   (username, alias, conRemark, nickname)
   (wxid标识符， 自定义微信ID, 备注， 昵称)
   rcontact 记录所有在该设备添加用的联系人，已经通讯录好友，删除的好友只是设置了标志位不可见
   */
   ```

##### 3.2  sqlcipher Linux下编译:
    

   &nbsp;&nbsp;&nbsp;&nbsp; sqlcipher 使用了OpenSSL提供的加密算法对数据库进行表级加密，github给出的编译命令是在OpenSSL安装的前提下进行的([sqlcipher官网](https://www.zetetic.net/sqlcipher/introduction/)，[sqlcipher源码](https://pan.baidu.com/s/11_1njKpnkm1eP7pkjokHAA)-网盘提取码 **thej**)，对于一般有图形界面的Linux系统，OpenSSL是安装了的，但是OpenSSL的开发库文件不一定装了， 对于自己编译安装的OpenSSL(下载的tar包)，是两个都有的。CentOS下OpenSSL库文件的安装：

   ```bash
   # 确认是否安装了openssl-devel
   yum install -y openssl-devel
   # Ubuntu下, openssl-devel名为libssl-dev
   # sudo apt-get install -y libssl-dev
   # 若下载超时，请更新Ubutu软件源， 见文章https://blog.csdn.net/CAU_Ayao/article/details/83507338
   
   # openssl-库文件安装后，动态链接编译sqlcihper
   # 自己选择下载目录
   # 使用git下载sqlcipher源码包， 也可以自己下载解压(外网慢，见上面链接)
   git clone https://github.com/sqlcipher/sqlcipher.git
   # 进入源码根目录
   cd sqlcihper
   ./configure --enable-tempstore=yes CFLAGS="-DSQLITE_HAS_CODEC" LDFLAGS="-lcrypto"
   make && make install 
   ```
##### 3.3 sqlcipher 命令行操作：
   
   
  ```bash
  # 进入数据库文件目录，或用绝对路径
  $ cd /home/apple/shared_nfs
  $ sqlcihper EnMicroMsg.db
  # 设置数据库密码（建议第一次进入设置，随便）
  sqlite> PRAGMA key = 'yourkey';
  sqlite> SELECT COUNT(1) FROM sqlite_master;
  ```
sqlcipher 提示：Error: file is not a database；现在命令行操作在此停滞，问题的解决请关注下一篇。
一种可能是密码错误（假设我们在GUI图形界面已经验证得到了正确的密码，排除密码错误的可能）。那究竟是什么原因呢？
        
