---
title: 使用GitHub Action和Maven打包Spring应用并部署至VPS
summary: Use GitHub Action to package a Maven application and copy artifact to remote server
date: 2021-01-21
tags: ['Installation']
---

当你用maven打包了一个Spring应用后，可以选择通过ssh上传到远程VPS的目录下，再使用java命令直接启动。如果你的项目代码托管在GitHub上，那么每次push提交后，可以通过GitHub Action来自动进行maven打包-复制到VPS-VPS通过系统服务重启Spring应用，来实现简单的自动持续部署。

# 物料

- 你的Spring应用，代码托管在GitHub上
- VPS，这里我使用Digital Ocean，点击[我的链接](https://m.do.co/c/62cb7001ac9e)注册，注册后60天内可使用100$代金券

# 服务器端的监控脚本

通过scp或其他SSH工具，我们可以手动将本地maven打包好的`jar`复制到服务器的指定目录内或者GitHub Action自动复制。但是每次复制覆盖掉`jar`后，Spring应用不会自动启动，我们也无法获取每次部署的信息。为此我们来定义一个系统服务，监控文件的每次更改并重新启动应用。

假设有一个Spring应用命名为*chat*，我将这个应用的artifact包`chat-0.0.1-SNAPSHOT.jar`文件放在服务器的`/app/chat`文件夹内。

```bash
root@myvps:/app# tree
.
└── chat
    ├── application.log # Spring应用的日志
    ├── application.sh # 用于重启/启动/暂停java jar进程
    ├── chat-0.0.1-SNAPSHOT.jar # Spring应用
    ├── chat-deploy-watch.sh # 监测jar文件的修改，如果有修改则调用application.sh来启停jar进程
    └── watch.log # 每次jar文件的修改信息日志
```

首先来写application.sh脚本，这个脚本应该要提供start/stop/status命令来方便我们手动管理Spring应用进程。

```shell
#!/bin/bash

# application jar's name
appDir=/app/chat/
application=$(find $appDir -type f -name "*.jar") 


#if [ -z $application ];then
#	application=`ls -t |grep .jar$ |head -n1`
#fi

#Java runtime option
#JAVA_OPTIONS="-XX:+UseG1GC -XX:+HeapDumpOnOutOfMemoryError -Xms512M -Xmx4G"


## determine existence of processus of application
isExist(){
	## check pid
    pid=`ps -ef | grep ${application} | grep -v grep | awk '{print $2}'`
    ## if pid not exist return 0 else 1
    if [ -z "${pid}" ]; then
    	return 0
    else
    	return 1
    fi
}


log=$appDir/application.log

## start application in background and generate appDir/application.log
function start()
{
	## deermine existence of processus
    isExist
    ## if not running
    if [ $? -eq "0" ]; then
    	echo "Your ${application} is running, please check it..."
    	#nohup java -jar $JAVA_OPTIONS ./$application  > $log 2>&1 &
        nohup java -jar $application  > $log 2>&1 &
    	echo "${application} startup success"
    else
    	echo "${application} is running, pid=${pid} "
    fi
}

function startx()
{
    isExist
    if [ $? -eq "0" ]; then
    	echo "Your ${application} is running, please check it..."
    	 java -jar  $JAVA_OPTIONS ./$application
    	echo "${application} startup success"
    else
    	echo "${application} is running, pid=${pid} "
    fi
}


## stop the processus application
function stop()
{
    isExist
    ## if not exist - ok
    if [ $? -eq "0" ]; then
    	echo "${application} is not running"
    else
    	echo "${application} is running, pid=${pid}, prepare kill it "
    	# if exist - kill the processus
    	kill -9 ${pid}
    	echo "${application} has been successfully killed"
    fi
}

## check status
function status()
{
	appPid=`ps -ef |grep java|grep $application |awk '{print $2}'`
	if [ -z $appPid ] 
	then
		echo -e "Not running " 
	else
		echo -e "Running [$appPid] " 
	fi
}

## restart
function restart(){
    stop
    start
}

## arg
case "$1" in
	"start")
		start
		;;
	"startx")
		startx
		;;
	"stop")
		stop
		;;
	"restart")
		restart
		;;
	"status")
		status
		;;	
	*)
		echo "please enter the correct commands: "
		echo "such as : sh application.sh [ start | stop | restart |status ]"
		;;
esac
```

在`start`函数里，使用`nohup`命令来运行`java jar`并且将Spring产生的日志从终端改写输出到application.log文件里。有了这个脚本后，就可以手动通过`./application.sh start/stop/restart/status`启停jar进程或查看状态了。为了使其自动化，再写`chat-deploy-watch.sh`脚本来监测jar文件的变化以便自动启停jar进程。

```shell
#!/bin/bash
# directory of app
appDir=/app/chat
# the name of file
#fileName=find $appDir -name *.jar
#log
log=/app/chat/watch.log
#file type
fileType="jar"

/usr/bin/inotifywait -mr --timefmt '%d/%m/%y %H:%M' --format '%T %w %f' -e close_write $appDir/ | while read DATE TIME DIR FILE; do
 
FILECHANGE=${DIR}${FILE}
 
if [[ $FILECHANGE =~ $fileType ]]
then
        echo "At ${TIME} on ${DATE}, file $FILECHANGE was changed." >> $log
        $appDir/application.sh restart
fi
done

```

注意系统需要安装`inotifywait`命令来做随时监测：` apt install inotify-tools`。  

用`chmod u+x`来给application.sh和chat-deploy.sh赋予权限。

每次jar文件的修改信息都会被写入watch.log日志文件中，大概是这样的：

```
At 14:47 on 18/01/21, file /app/chat/chat-0.0.1-SNAPSHOT.jar was changed.
At 15:17 on 18/01/21, file /app/chat/chat-0.0.1-SNAPSHOT.jar was changed.
At 15:43 on 18/01/21, file /app/chat/chat-0.0.1-SNAPSHOT.jar was changed.
...
```

`chat-deploy-watch.sh`文件用于随时监测，那么这个进程就要永远保持启动状态，为了更好地管理这个进程，使其随时启动/暂停/查看状态，我们来写一个service服务封装这个进程，用`service`来管理这个后台服务。

`etc/systemd/system/chat.service`

```sh
[Unit]
Description=Watch for chat artifacts
After=syslog.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
ExecStart=/app/chat/chat-deploy-watch.sh

[Install]
WantedBy=multi-user.target
```

可以看到ExecStart指定启动`/app/chat/chat-deploy-watch.sh`。我们将这个文件放到`etc/systemd/system`，使用`systemctl daemon-reload`刷新守护进程状态，再`systemctl start chat.service`启动这个Unit，并将这个Unit设置为开机自动`systemctl enable chat.service`。可以通过`service chat start/stop/status`来管理这个服务。

用`service chat status`看看我们服务的状态：

```
root@myvps:/app/chat# service chat status
● chat.service - Watch for chat artifacts#
   Loaded: loaded (/etc/systemd/system/chat.service; enabled; vendor preset: enabled)
   Active: active (running) since Mon 2021-01-18 18:34:31 UTC; 2 days ago
 Main PID: 29480 (chat-deploy-wat)
    Tasks: 45 (limit: 1152)
   CGroup: /system.slice/chat.service
           ├─ 1911 java -jar /app/chat/chat-0.0.1-SNAPSHOT.jar
           ├─29480 /bin/bash /app/chat/chat-deploy-watch.sh
           ├─29481 /usr/bin/inotifywait -mr --timefmt %d/%m/%y %H:%M --format %T %w %f -e close_write /app/chat/
           └─29482 /bin/bash /app/chat/chat-deploy-watch.sh

Jan 18 18:45:24 myvps chat-deploy-watch.sh[29480]: /app/chat/chat-0.0.1-SNAPSHOT.jar is running, pid=30512, prepare kill it
Jan 18 18:45:24 myvps chat-deploy-watch.sh[29480]: /app/chat/chat-0.0.1-SNAPSHOT.jar has been successfully killed
Jan 18 18:45:24 myvps chat-deploy-watch.sh[29480]: Your /app/chat/chat-0.0.1-SNAPSHOT.jar is running, please check it...
Jan 18 18:45:24 myvps chat-deploy-watch.sh[29480]: /app/chat/chat-0.0.1-SNAPSHOT.jar startup success
```

现在试试手动maven打包将jar文件放到服务器上吧，看看相应的log文件是否正确。

# 从GitHub打包部署

我们可以用GitHub Action ([GitHub Actions 入门教程](http://www.ruanyifeng.com/blog/2019/09/getting-started-with-github-actions.html)) 来在线使用mvn命令打包，并连接ssh将jar包复制到vps上。

## 可选：加密解密部分文件

如果你的Spring项目内带有一些隐私信息，比如数据库的用户名，密码，建议将文件加密后再上传到GitHub，可以看这篇教程[使用git-crypt来加密你的敏感数据](https://juejin.cn/post/6844904073477505037)。导出git-crypt.key，我们将在下一步用到它。

## 将密钥放入GitHub Secrets

要想让GitHub Action能通过ssh连接vps，那么就得告诉它ssh的端口，地址，用户名和密码等信息或是ssh key。打开GitHub项目，点击Settings，左侧找到Secrets，新建New repository secret，在这里存放我们的隐私信息。

![image-20210121190736485](https://dev.azure.com/zslyvain/9285f0e6-8055-4a5c-aec3-50d9555ac078/_apis/git/repositories/4eb461c6-bb1f-489f-978b-686e8c32decf/items?path=%2F1625856607773_6465.png&versionDescriptor%5BversionOptions%5D=0&versionDescriptor%5BversionType%5D=0&versionDescriptor%5Bversion%5D=master&resolveLfs=true&%24format=octetStream&api-version=5.0)

可选：如果你根据上一步使用了git-crypt加密了某些文件，将刚刚导出的git-crypt.key也一并加入到Secrets中。

## 编写GitHub Action文件

我们希望每次push后GitHub自动执行maven打包-复制到VPS的过程，那么就需要一个workflow来定义我们的过程步骤。

在GitHub网站上中找到Action，可以选择在网页上新建一个Action (set up workflow yourself)或者在项目内新建文件夹*.github/workflow/*然后新建yml文件，比如*maven.yml*。

```yml
# This workflow will build a Java project with Maven
# For more information see: https://help.github.com/actions/language-and-framework-guides/building-and-testing-java-with-maven

name: Java CI with Maven & Deploy

on:
  push:
    branches: [ master ]
  pull_request:
    branches: [ master ]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - name: 'Checkout GitHub Action'
      uses: actions/checkout@master

    # 可选：如果你使用git crypt加密文件，那么你就得解密文件
    - name: GitHub Action to unlock git-crypt secrets
      uses: sliteteam/github-action-git-crypt-unlock@1.1.0
      env:
        GIT_CRYPT_KEY: ${{ secrets.GIT_CRYPT_KEY }}
	
	# 可选：如果你使用git crypt加密文件，解密文件后需要将目录删除掉确保隐私
    - name: Delete git crypt directory
      run: sudo rm -rf .git/git-crypt

    - name: Set up JDK 1.8
      uses: actions/setup-java@v1
      with:
        java-version: 1.8

    - name: Build with Maven
      run: mvn -B package --file pom.xml

    - name: Deploy to server
      uses: appleboy/scp-action@master
      with:
        host: ${{ secrets.VPS_HOST }}
        username: ${{ secrets.VPS_USERNAME }}
        port: ${{ secrets.VPS_PORT }}
        password: ${{ secrets.VPS_PASSWORD }}
        source: "./target/chat-0.0.1-SNAPSHOT.jar"
        target: "/app/chat/"
        strip_components: 2
```

可选：如果你git crypt加密了文件，比如application.properties，那在maven打包前肯定就需要解密这个文件。  

workflow的最后一步 Deploy to server，这里使用了[scp-action](https://github.com/appleboy/scp-action)，原理即是使用你放在secrets中的密码用户名等信息，连接ssh，再将上一步Build with Maven打包好的`chat-0.0.1-SNAPSHOT.jar`拷贝到服务器的target文件夹内。注意`strip_components`用于将source指定的文件的路径前缀消除掉，否则复制到服务器上的就会是`target/chat-0.0.1-SNAPSHOT.jar`而不是`chat-0.0.1-SNAPSHOT.jar`。

