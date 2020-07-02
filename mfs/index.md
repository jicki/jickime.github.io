# MooseFS 部署



# MooseFS 分布式文件系统


## 环境说明

```
172.16.32.85 MFS-Controller(master)

172.16.32.86 MFS-Controller(metalogger)

172.16.32.57 Chunk Server1 

172.16.32.58 Chunk Server2

172.16.32.59 Chunk Server3
```

## MFS 说明

MFS优势

	1. 数据高可用性（数据可以存储在多个机器上的多个副本）
	2. 在线动态扩展存储
	3. 垃圾回收站
	4. 文件快照(本文不研究)

MFS问题

	1. 单点故障：即使V1.6增加了Metalogger，但不能达到故障自动切换
	2. 数据校验：为了提高存储效率，MFS的数据块不创建校验值(checksum)，降低了一定的数据安全性
	3. 规模试用：适用于小规模的分布式文件系统，除非进行相关的二次开发


	
# MFS 安装与配置

## 初始化环境

```
	# 同步系统时间
	
	# 创建用户
	
	groupadd -g 1010 mfs && useradd -u 1010 -g 1010 mfs
	groupadd -g 1001 upload && useradd -u 1001 -g 1001 upload
	
	# 安装依赖
	yum install -y zlib-devel gcc gcc-c++
	
	#创建目录
	mkdir /opt/{software,local} cd /opt/software
	
	# 下载软件
	cd /opt/software
	wget http://ppa.moosefs.com/src/moosefs-2.0.89-1.tar.gz  
```

## Master安装配置

```
	tar -xzvf moosefs-2.0.89-1.tar.gz
	cd moosefs-2.0.89
	./configure --prefix=/opt/local/mfs-2.0.89 --with-default-user=mfs --with-default-group=mfs --disable-mfsmount --disable-mfschunkserver
	make && make install
	cd /opt/local/
	ln -s mfs-2.0.89 mfs
	cd mfs/etc/mfs/
	cp mfsmaster.cfg.dist mfsmaster.cfg
	cp mfsexports.cfg.dist mfsexports.cfg
	cd /opt/local/mfs/var/mfs && cp metadata.mfs.empty metadata.mfs
	chown -R mfs:mfs /opt/local/mfs*
```

## 编译参数说明

```
	--disable-mfsmaster                  # 不创建master
	--disable-mfschunkserver             # 不创建chunkserver
	--disable-mfsmount                   # 不创建mfs客户端(mfsmount和mfstools)   
	--disable-mfsmetalogger              # 不创建mfsmetalogger(master和metalogger可能会故障切换，安装时master和metalogger都必须安装master和logger功能)
```

## 修改系统参数

修改/etc/security/limits.conf

```
	* - nofile 655350
```

## 配置 mfsmaster.cfg

```
	WORKING_USER = mfs     					                    # mfs服务的运行用户，单进程
	WORKING_GROUP = mfs    					                    # mfs服务的运行组
	SYSLOG_IDENT = mfsmaste					                    # master server在syslog中的标识，日志记录在/var/log/messages
	LOCK_MEMORY = 0        					                    # 是否执行mlockall()以避免mfsmaster 进程溢出（默认为0）
	NICE_LEVEL = -19       					                    # 运行的优先级(由于服务器上的应用进程只有mfs，此处无需额外设定)
	DATA_PATH = /opt/local/mfs/var/mfs                          # 元数据存放目录：changelog，sessions和stats等；
	EXPORTS_FILENAME = /opt/local/mfs/etc/mfs/mfsexports.cfg    # 被挂接目录及其权限控制文件
	BACK_LOGS = 10                       					    # metadata 的改变log 文件数目(默认是50);
	METADATA_SAVE_FREQ = 1										# 单位Hour，flush内存汇总metadata的频率
	BACK_META_KEEP_PREVIOUS = 1									# 本地历史metadata文件的保存数量
	MATOML_LISTEN_HOST = master-ip    							# metalogger连接的IP
	MATOML_LISTEN_PORT = 9419 									# metalogger连接的端口
	MATOCS_LISTEN_ HOST = master-ip  							# chunkserver连接的IP
	MATOCS_LISTEN_PORT = 9420 									# chunkserver连接的端口
	MATOCU_LISTEN_HOST = master-ip    							# Client连接的IP
	MATOCU_LISTEN_PORT = 9421 									# Client连接的端口地址
	REJECT_OLD_CLIENTS = 1 										# 弹出低于1.6.0的客户端挂接

```

## 配置mfsexports.cfg 

```
   192.168.0.0/24     /            rw,alldirs,maproot=0    		# /... = path in mfs structure
   192.168.0.0/24     .            rw                      		# .标识MFSMETA 文件系统
	[IP]        * | A.B.C.D | A.B.C.D/XX | A.B.C.D - A.B.C.G
	[path]      / | /dir | .
	[privil]
	   ro/rw/readonly/readwrite
	   alldirs = any subdirectory can be mounted
	   mapall=1000:1000     All users are mapped as users with uid:gid = 1000:1000 
	   maproot=nobody       Local roots are mapped as 'nobody' users
	   password=TEXT
```


## MFS CGI 服务

CGI服务操作：

```
/opt/local/mfs/sbin/mfscgiserv start|stop

#说明：最好在master的hosts中加入mfsmaster的解析，因为http访问CGI时，会自动通过主机名去定位master
```


## metalogger安装配置

```
	yum install fuse-devel fuse 
	modprobe fuse
	tar -xzvf moosefs-2.0.89-1.tar.gz
	cd moosefs-2.0.89
	./configure --prefix=/opt/local/mfs-2.0.89  --with-default-user=mfs --with-default-group=mfs --disable-mfschunkserver --disable-mfsmount && make && make install
	cd /opt/local/
	ln -s mfs-2.0.89 mfs
	chown -R mfs:mfs /opt/local/mfs*

```


## 配置mfsmetalogger.cfg

```
	WORKING_USER = mfs     					                   
	WORKING_GROUP = mfs    					                   
	SYSLOG_IDENT = mfslogger					               
	LOCK_MEMORY = 0        					                   
	NICE_LEVEL = -19       					                   
	DATA_PATH = /opt/local/mfs/var/mfs                          
	BACK_LOGS = 10                       					    # 与master保持一致
	META_DOWNLOAD_FREQ = 1										# 单位Hour，多久同步一次master的metadata.mfs.back; 默认24H
	BACK_META_KEEP_PREVIOUS = 3                                 # 保留3份历史metadata_ml.mfs.back
	MASTER_HOST = 10.6.0.233
	MASTER_PORT = 9419
	MASTER_TIMEOUT = 10
	MASTER_RECONNECTION_DELAY = 5
# 拷贝master的mfsmaster.cfg和mfsexports.cfg，以备metalogger切换为master
```
	
## ChunkServer 安装配置

```
	tar -xzvf moosefs-2.0.89-1.tar.gz
	cd moosefs-2.0.89
	./configure --prefix=/opt/local/mfs-2.0.89  --with-default-user=mfs --with-default-group=mfs --disable-mfsmaster --disable-mfsmount && make && make install
	cd /opt/local/
	ln -s mfs-2.0.89 mfs
	chown -R mfs:mfs /opt/local/mfs*
```	

## 配置 mfschunkserver.cfg

```
	WORKING_USER = mfs
	WORKING_GROUP = mfs
	SYSLOG_IDENT = mfschunkserver
	LOCK_MEMORY = 0
	NICE_LEVEL = -19
	DATA_PATH = /opt/local/mfs-2.0.89/var/mfs
	HDD_CONF_FILENAME = /opt/local/mfs-2.0.89/etc/mfs/mfshdd.cfg
	BIND_HOST = chunkserver-ip
	MASTER_HOST = master-ip
	MASTER_PORT = master-port
	MASTER_TIMEOUT = 10
	CSSERV_LISTEN_HOST = chunkserver-ip		 # IP address to listen for client (mount) connections
	CSSERV_LISTEN_PORT = 9422                # port to listen for client (mount) connections 
```


## 配置 mfshdd.cfg

```
	/opt/mfsdata                  # 建议划分单独的空间给 MooseFS 使用，chunkserver进程需要有权限操作该存储目录
```


## Client安装配置

```
	yum install fuse-devel fuse -y
	modprobe fuse
	tar -xzvf moosefs-2.0.89-1.tar.gz
	cd moosefs-2.0.89
	./configure --prefix=/opt/local/mfs-2.0.89  --with-default-user=mfs --with-default-group=mfs --enable-mfsmount --disable-mfsmaster --disable-mfschunkserver && make && make install
	cd /opt/local/
	ln -s mfs-2.0.89 mfs
	chown -R mfs:mfs /opt/local/mfs*
```	

## 配置mfsmount.cfg

```
	mfsmaster=10.6.0.233
	mfspassword=secret
```	
	
## fuse.conf设置

```
	mount_max = NNN     # Set the maximum number of FUSE mounts allowed to non-root users. The default is 1000.
	user_allow_other    # Allow non-root users to mount
```


## 挂载

```
	mfsmount -m /mfsdata -H 10.6.0.233 				 # 挂载metadata
	mfsmount /mfsdata -H 10.6.0.233    				 # 挂载mfs文件系统
	mfsmount /mfsdata -H 10.6.0.233 -S imagesdata	 # 挂载mfs文件系统的指定目录，挂载imagesdata后，/mfsdata对应10.6.0.233:/imagesdata
```


	1. fuse无需modprobe fuse，执行mfsmount时会自动加载
	2. 暂时没有找到办法进行自动挂载(rc.local没有测试)
	3. Client不管chunks断没断掉，都是可以挂载的而且有目录信息，目录信息存在Master的内存中。


## master 的 mfsexports.cfg

用户权限控制

```
	# Admin (only 10.6.0.192 can operate the / and metadata )
	10.6.0.192            .              rw
	10.6.0.192            /              rw,alldirs,mapall=upload:upload
	# users (Users operate the share sites)
	10.6.0.0/24           /imagesdata     rw,alldirs,mapall=upload:upload
```

	1. mapall=uid:gid      # mfs client上的所有用户，均以uid:gid 挂载mfs
	2. maproot=uid:gid     # mfs client上的root用户以uid:gid挂载mfs, 其他用户以当前系统用户和用户组挂载
	3. master更改map用户后，客户端必须remount才会生效，因为挂载用户身份的确认发生在mfsmount:"#mfsmount --> mfsmaster accepted connection with parameters: read-write,restricted_ip,map_all ; root mapped to mfs:mfs ; users mapped to mfs:mfs"

