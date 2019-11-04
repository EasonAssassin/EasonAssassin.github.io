----
[TOC]

----
#02--Jenkins相关

@(Python_HuaYun)

##1.环境安装

**方法一：**
- 获取jenkins.war包：http://mirrors.jenkins.io/war-stable/
- 安装java环境：
```
yum install java-1.8.0-openjdk  java-1.8.0-openjdk-devel
```
- 启动jenkins：`java -jar jenkins.war --httpPort=8080`（端口自定义）
- 启动jenkins（完整版&自定义版）：`/etc/alternatives/java -Djava.awt.headless=true -DJENKINS_HOME=/var/lib/jenkins -jar /usr/lib/jenkins/jenkins.war --logfile=/var/log/jenkins/jenkins.log --webroot=/var/cache/jenkins/war --httpPort=8080 --debug=7 --handlerCountMax=100 --handlerCountMaxIdle=20`
- 适用到24.33：`nohup /etc/alternatives/java -Djava.awt.headless=true -DJENKINS_HOME=/root/.jenkins -jar /root/.jenkins/jenkins.war --logfile=/var/log/jenkins/jenkins.log --webroot=/var/cache/jenkins/war --httpPort=8080 --debug=7 --handlerCountMax=100 --handlerCountMaxIdle=20 &`
- 此安装方法，jenkins的所有配置和业务文件都位于`~/.jenkins/`目录下。便于备份！

**方法X（不建议使用）**
- ubuntu环境
```powershell
# 安装Java
sudo apt-get install default-jre
sudo apt-get install openjdk-8-jdk
# 添加Jenkins仓库
$ wget -q -O - http://pkg.jenkins-ci.org/debian/jenkins-ci.org.key | sudo apt-key add -
# 在/etc/apt/sources.list末尾添加源
deb http://pkg.jenkins-ci.org/debian binary/
# 安装Jenkins
sudo apt-get install jenkins
# 启动jenkins
service jenkins start
```
- centos环境
> 如下添加jenkins源链接可能会变化，具体请参考：https://pkg.jenkins.io/redhat/
```
# 安装java环境
yum install java-1.8.0-openjdk  java-1.8.0-openjdk-devel
# 新增jenkins源
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat/jenkins.repo

# 添加jenkins源所需key
rpm --import https://pkg.jenkins.io/redhat/jenkins.io.key
# 安装jenkins
yum install jenkins


```

##2.打开网页并按照提示进行操作：
url: localhost:8080
- 输入密码
- 安装必备插件
	- Git Client & Git：自动创建Git仓库中的代码
	- Gitlab hook plugin：通过Gitlab发来的hook从而自动创建
	- TAP Plugin：构建完毕后根据TAP(Test Anything Protocol)文件生成图表
	- Publish over FTP
	- Post-Build Script Plugin
	- Email Exension Plugin
	- Cobertura Plugin
	- junit、nosetests

##3.GitLab
- 3.1 安装GitLab
```powershell
# 安装环境依赖
sudo apt-get install curl openssh-server ca-certificates postfix
# 添加GitLab package进仓库
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ce/script.deb.sh | sudo bash
# 下载GitLab，但因为国内网速问题，建议从清华源下载https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/ubuntu/pool/xenial/main/g/gitlab-ce/gitlab-ce_8.10.0-ce.1_amd64.deb
# sudo apt-get install gitlab-ce
sudo dpkg -i gitlab-ce_8.10.0-ce.1_amd64.deb
# 设置并启动GitLab
sudo gitlab-ctl reconfigure

```
## 4.添加节点
方法一：
- Jenkins Master机器通过`ssh-copy-id  IP`的方式，即免密登录Slave节点
- Jenkins ---> Credentials ---> Global credentials ---> Add Credentials：将Jenkins Master的id_rsa添加进去
- slave节点需要安装java环境：`yum -y install java git`
- 所有节点添加时只需通过唯一的**Jenkins Master的id_rsa**作为credentials，就可以添加成功
![Alt text](./1523450246822.png)



**方法X：（淘汰）**
> 总体思路：先将节点的`id_rsa`秘钥添加到jenkins平台的credentials，作为`private key`使用；其次，以用户名+登录密码的方式添加节点，`launch agent`使其online

- 4.1 配置slave节点。在slave节点上执行（此操作为必须，否则报错！！）：
```powershell
yum install -y java  git
ssh-keygen -t rsa
cd ~/.ssh/
cat id_rsa.pub > authorized_keys
chmod 700 authorized_keys
```

- 4.2 添加slave节点的私钥
	- Jenkins ---> Credentials ---> Global credentials ---> Add Credentials
	- Kind中选择`SSH Username with private key`
	- 在Username中输入登录Slave节点的用户名root
	- Private Key中选择Enter directly
	- 输入之前在slave节点创建的id_rsa私钥文件内容

![Alt text](./1510574882589.png)

**注意：**上述操作是添加节点私钥的操作，而下面是通过`用户名/密码`的方式登录节点

- 4.3 系统管理--->管理节点--->新建节点
	- 填写节点名称，勾选Permanent Agent
	- 远程工作目录：/root
	- 用法：至允许运行绑定到这台机器的Job
	- 启动方法：Launch slave agents via SSH
	- Host: 172.16.16.72
	- Credentials: Add credentials ---> kind: Username with password
	- Save

> **注意点如下：**
> 1.直接按如上操作，此时会出现报错`No entry currently exists in the Known Hosts file for this host.`
> 解决方案如下：

```powershell
# 在运行Jenkins的机器上切换jenkins用户
su jenkins
# 如果不知道密码，可以切换到root用户，然后修改jenkins用户密码
su root
passwd jenkins
# 登录jenkins账户后通过ssh登录远程节点
ssh root@172.16.16.72
```
*如上操作的目的在于，在于将远程节点添加进jenkins账户下的known hosts里*

> 2.目标节点在第一次添加过程中，会默认安装jenkins的一些环境，此时出现失败也会导致报错。此时，可以去目标节点手动安装。
> `yum install java`
> 特别注意：如果节点上Java的版本与jenkins平台的Java版本不一致，也会导致节点无法添加，所以需要统一Java版本

## 5.配置邮件

- 步骤：Jenkins页面 ---> 系统管理 ---> 系统设置
- Jenkins Location配置：

![Alt text](./1510552158513.png)

- Extended E-mail Notification配置：
![Alt text](./1510552246956.png)

- 邮件通知配置：
![Alt text](./1510552298983.png)

- 在jenkins job里配置邮箱任务（带附件版）
![Alt text](./1513589945648.png)
- 其中Attachments是相对于workspace的路径，如果workspace的当前路径有所需要的附件，就可以直接添加。
	- 在高级设置里添加failure和success时发送的对象

![Alt text](./1513590099644.png)

## 6. jenkins在运行shell脚本时的报错`sudo: sorry, you must have a tty to run sudo`

需要禁止requiretty
`vim /etc/sudoers`
```
# Defaults    requiretty
root    ALL=(ALL:ALL) ALL
jenkins ALL=(ALL) NOPASSWD: ALL
```

## 7. jenkins使用`Parameterized Trigger plugin`时无法传递变量的bug

> 希望将A项目的变量传递到B项目里继续使用，但发现B项目无法接收

此时需要在B项目里设置一个同名变量（可以为空或设默认值），此时才会将变量继续接收


## 8. jenkins中的环境变量
- jenkins中的环境变量有：
	- 主机中的系统环境变量
	- Master/Slave节点设置的环境变量
	- Job执行时的环境变量（http://ip:port/jenkins/env-vars.html/、参数化构建时的参数也会被设置为环境变量、一些插件提供的环境变量）
- 其中，如果环境变量名称相同，后者会覆盖前者
- 这些环境变量可以在Shell或Batch脚本中被使用，以JOB_NAME环境变量为例：
	- 在Shell中：$JOB_NAME
	- 在Batch中：%JOB_NAME%
	- 在Ant插件中：$JOB_NAME
	- 在Ant的build.xml中：${JOB_NAME}

- 在使用Jenkins的过程中，多次遇到Jenkins job中无法获取Slave上的环境变量的情况
	-	例如：例如，在Jenkins slave上安装了python，但在Jenkins job中使用python命令时，出现如下提示'python'不是内部或外部命令，也不是可运行的程序或批处理文件
	-	而实际上Slave机器的环境变量PATH中已追加了python的环境变量，但是Jenkins job中无法读取到

- 解决办法：
	- 使用绝对路径的命令
	- 在jenkins的job中设置和坦克变量参数
	- 在jenkins的节点配置中设置环境变量
**个人感觉其中最友好的方式是在在Jenkins的节点配置中设置环境变量**

例如：可以设置PATH的值为$PATH，这样PATH就可以读到slave机器上的配置

![Alt text](./1518074084623.png)

## 9. jenkins邮件模板配置

> 在`系统设置`--->`Extended E-mail Notification`--->`Default Content`处填写邮件模板

```vbscript-html
<body>

    <head>
        <STYLE TYPE="text/css">
            BODY {
                background-image: URL(https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1517949726997&di=8454bb73c7f72ccb3329717a1dcb3086&imgtype=0&src=http%3A%2F%2Fpic.58pic.com%2F58pic%2F14%2F90%2F78%2F358PICC58PICHNk_1024.jpg);
                background-position: center;
                background-repeat: no-repeat;
                background-attachment: fixed;
                background-size:100% 100%;
            }
        </STYLE>
    </head>
    <h2 style="color: #5e9ca0; text-align: center;">PS:(本邮件是程序自动发送，请勿回复！)</h2>
    <table class="editorDemoTable" style="height: 273px;" width="430">
        
        <tbody>
            <tr>
                <td nowrap="nowrap" style="border-color: gray;"><span style="color: #000000;">项目名称：</span></td>
                <td nowrap="nowrap" style="border-color: gray;"><span style="color: #000000;">$PROJECT_NAME</span></td>
            </tr>
            <tr>
                <td nowrap="nowrap" style="border-color: gray;"><span style="color: #000000;">构建编号：</span></td>
                <td nowrap="nowrap" style="border-color: gray;"><span style="color: #000000;">$BUILD_NUMBER</span></td>
            </tr>
            <tr>
                <td nowrap="nowrap" style="border-color: gray;"><span style="color: #e83a1e;">构建状态：</span></td>
                <td nowrap="nowrap" style="border-color: gray;"><span style="color: #e83a1e;">$BUILD_STATUS</span></td>
            </tr>
            <tr>
                <td nowrap="nowrap" style="border-color: gray;"><span style="color: #000000;">触发原因：</span></td>
                <td nowrap="nowrap" style="border-color: gray;"><span style="color: #000000;">${CAUSE}</span></td>
            </tr>
            <tr>
                <td nowrap="nowrap" style="border-color: gray;"><span style="color: #000000;">构建日志地址：</span></td>
                <td nowrap="nowrap" style="border-color: gray;"><span style="color: #00ccff;">&nbsp;<a style="color: #00ccff;" title="构建日志" href="${BUILD_URL}console">${BUILD_URL}console</a></span></td>
            </tr>
            <tr>
                <td nowrap="nowrap" style="border-color: gray;"><span style="color: #000000;">构建日志（最后100行）：</span></td>
				<hr size="2" width="100%" align="center" /></hr>
				<td><textarea cols="80" rows="30" readonly="readonly" style="font-family: Courier New">${BUILD_LOG, maxLines=100}</textarea></td>
            </tr>			
   

```

## 10.  Jenkins备份及恢复

> - 自带的backup插件，仅仅是在jenkins节点上进行本地备份，无法实现跨节点备份
> - 通过互信、定时任务，实现跨节点的备份任务
> - 通过安装方法可以获知备份，只需备份2个文件，jenkins平台的**jenkins.war（控制jenkins版本）**文件，和**/root/.jenkins/**目录。

- **备份**

`crontab -e`
```
#!/bin/bash

DATESTR=`date +%Y%m%d-%H%M`
echo ${DATESTR}

base="/data/172.16.24.33_jenkins/jenkins"

# rsync from 172.16.24.33
ssh 172.16.24.33 rm -rf /root/*.tar.gz
ssh 172.16.24.33 tar -zcf jenkins_${DATESTR}.tar.gz /root/.jenkins/*   
rsync -avz root@172.16.24.33:/root/jenkins_${DATESTR}.tar.gz ${base}/

# delete more than 15 days
#for dir in $(find ${base}/ -maxdepth 1 -type d -mtime +15); do
#    rm -rf ${dir}
#done
find ${base}/ -iname "*.tar.gz" -mtime +15 -delete

# make forever backup
forever="/data/172.16.24.33_jenkins/forever"
mkdir -p $forever
if [ ${DATESTR:6:-5} = '28' ]; then
   cp -a ${base}/jenkins_${DATESTR}.tar.gz ${forever}/
   echo "jenkins_${DATESTR}.tar.gz succeed" >> ${forever}/backup.log
else
   echo "jenkins_${DATESTR}.tar.gz failed" >> ${forever}/backup.log
fi

# make system backup
system="/data/172.16.24.33_jenkins/system"
mkdir -p $system
if [ ${DATESTR:6:-5} = '28' ]; then
   ssh 172.16.24.33 mkdir -p /archive
   ssh 172.16.24.33 tar -zcvpf /archive/fullbackup_${DATESTR}.tar.gz  --exclude=/archive --exclude=/mnt --exclude=/proc --exclude=/lost+found --exclude=/dev --exclude=/sys --exclude=/tmp /
   rsync -avz root@172.16.24.33:/archive/fullbackup_${DATESTR}.tar.gz ${system}/
   ssh 172.16.24.33 rm -f /archive/*.tar.gz
   echo "fullbackup_${DATESTR}.tar.gz succeed" >> ${system}/backup.log
else
   echo "fullbackup_${DATESTR}.tar.gz ignored" >> ${system}/backup.log
fi
```

- **双master多slave策略**
1. 将jenkins.war和.jenkins/目录传到备用的master节点
2. 每天用定时任务将.jenkins/目录rsync同步
3. 当主master节点挂掉后，迅速到备份master节点`java -jar jenkins.war`恢复jenkins
4. 备份master节点上的jenkins，拥有本机**id_rsa**的**credentials**，用于每个slave节点添加时使用
5. 注意：此时需要检查各个节点，备份mater节点是否与各个slave节点实现免密


- **恢复**
1. 备份节点配置java环境：`yum install java-1.8.0-openjdk java-1.8.0-openjdk-devel`
2. 启动jenkins：`java -jar jenkins`
3. 检查各个slave节点是否启用

## 11. Jenkins配置Gerrit触发Jobs

> 参考文档：https://github.com/kobeqing/celadon/wiki/Gerrit-Code-Review%E6%90%AD%E5%BB%BA%E9%85%8D%E7%BD%AE

- 安装Gerrit Trigger插件，并配置gerrit信息
![Alt text](./1519286590019.png)

![Alt text](./1519286621399.png)

- 配置Jenkins Job
	- **注意：**username设置的是gerrit平台上实际的jenkins账户
	- 需要将jenkins master的id_rsa.pub放入jenkins账户的SSH Public Keys中
	- SSH Keyfiel Password：无需填写（默认配置有密码）
	- 遇到过的一个BUG有，gerrit trigger插件未升级，及时**Test Configuration**成功仍然无法使用。解决：升级下gerrit trigger插件即可
	- **注意：**当希望给一个项目的多个分支进行Gerrit Trigger时，可以在Gerrit Trigger里的**Dynamic Trigger Configuration**里设置正则，**path：****或者**RegExp：.***
![Alt text](./1523873302074.png)

![Alt text](./1519354608801.png)
**Dynamic Trigger Configuration：**
![Alt text](./1523873799793.png)
**或者**
![Alt text](./1523873815859.png)



- 实现
```
# 下载代码
git clone ssh://sunyan@172.16.24.40:29418/watcher-chinac
# 修改
# 添加
git add .
git commit -m "test"
# 上传代码的2种方式
# 方式1
git push origin HEAD:refs/for/mitaka-chinac
# 此时可能出现没有commit id的报错，通过如下解决。存放一个全局的钩子配置
mkdir -p ~/.git_template/hooks
## 从远程复制钩子到本地
scp -P29418 sunyan@172.16.24.40:hooks/commit-msg  ~/.git_template/hooks
## 添加进git的全局配置
git config --global init.templatedir ~/.git_template

# 方式2
## 添加远程仓库gerrit
git remote add gerrit ssh://sunyan@172.16.24.40:29418/watcher-chinac 
## 推送到远程
git review mitaka-chinac

```

> **方式1和2的区别在于，方式1如果提交的分支和jenkins上配置的分支不一致，则不会触发jenkins上的job。方式2不论分支是否和jenkins一致，都会触发jenkins job。**

## 12 Jenkins任务失败后自动restart jobs (Retry build after failed)

安装插件**Naginator**

## 13 Jenkins无法完整地展示html report问题

> 参考：https://zhuanlan.zhihu.com/p/28080975
> https://kb.froglogic.com/display/KB/Content+Security+Policy+%28CSP%29+for+Web+Report
> - 对于测试报告来说，除了内容的简洁精炼，样式的美观也很重要。常用的做法是，采用HTML格式的文档，并搭配CSS和JS，实现自定义的样式和动画效果（例如展开、折叠等）。
> - 然而，在jenkins中展示html report会出现样式不全的问题

**调整前：**
![Alt text](./1539151981445.png)

**调整后：**
![Alt text](./1539151990412.png)

- 问题分析：出现该现象的原因在于Jenkins中配置的CSP（Content Security Policy）。简单地说，这是Jenkins的一个安全策略，默认会设置为一个非常严格的权限集，以防止Jenkins用户在workspace、/userContent、archived artifacts中受到恶意HTML/JS文件的攻击。默认地，该权限集会设置为：`sandbox; default-src 'none'; img-src 'self'; style-src 'self';`
- 在该配置下，只允许加载：
	- Jenkins服务器上托管的CSS文件
	- Jenkins服务器上托管的图片文件

- **解决方法1**：
	- 安装startup-trigger-plugin和Groovy插件后
	- 新建一个job，该job专门用于Jenkins启动时执行的配置命令；
	- 在Build Triggers模块下，勾选Build when job nodes start；
	- 在Build模块下，Add build step->Execute system Groovy script，在Groovy Script中输入配置命令，System.setProperty("hudson.model.DirectoryBrowserSupport.CSP", "")。

- **解决方法2**：
	- 安装groovy插件
	- 在项目配置页添加构建步骤：`Execute system Groovy script`
	- 输入`System.setProperty("hudson.model.DirectoryBrowserSupport.CSP", "")`

- **解决方法3**（临时）：
	- 系统配置 --> 脚本命令行 --> 输入`System.setProperty("hudson.model.DirectoryBrowserSupport.CSP", "")`
	- 注意：下次重启后失效！

## 14. 以容器方式启动jenkins
> 通过挂载本地目录的方式将数据持久化。

```
docker run -p 8080:8080 -p 50001:50000 -d --env JAVA_OPTS="-Dorg.apache.commons.jelly.tags.fmt.timeZone=Asia/Shanghai" -v /dev/shm/jenkins_home:/var/jenkins_home -v /etc/localtime:/etc/localtime jenkins:2.164.1
```

**2.0版本**

```
# 以root用户执行，将互信的.ssh目录以及jenkins_home映射出来
# --restart=always总是重启，这样就可以实现开机重启docker容器，但因为jenkins服务总是有些莫名的错误，会导致容器不断重启，因此换成如下参数
# --restart=on-failure:10只有在非0状态退出时才从新启动容器，且10s重启一次

docker run -d --restart=on-failure:10 -u root -p 80:8080 -p 50000:50000 --env JAVA_OPTS="-Dorg.apache.commons.jelly.tags.fmt.timeZone=Asia/Shanghai" -v jenkins_data:/var/jenkins_home -v /var/lib/jenkins/.ssh/:/var/jenkins_home/.ssh/ -v /var/lib/jenkins/.ssh/:/var/lib/jenkins/.ssh/ -v /etc/localtime:/etc/localtime --name jenkins jenkins:2.168   
```

