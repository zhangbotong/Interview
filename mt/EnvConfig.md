环境配置

一、办公用品申请

显示器

https://km.sankuai.com/page/886196089

办公耗材

手机大象-工作台-办公用品

二、开发环境安装

homebrew

git

idea

jdk

maven

idea 默认 maven 配置文件：~/.m2/settings.xml

个人账号获取：https://km.sankuai.com/page/505722445

<?xml version="1.0"?>
<settings xmlns="https://maven.apache.org/SETTINGS/1.0.0" xmlns:xsi="https://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="https://maven.apache.org/SETTINGS/1.0.0 https://maven.apache.org/xsd/settings-1.0.0.xsd">

    <!--https://maven.apache.org/ref/3.5.0/maven-settings/settings.html-->

    <servers>
      	<server>
            <id>meituan-nexus-releases</id>
            <username>你的账号</username>
            <password>你的token</password>
        </server>
        <server>
            <id>meituan-nexus-snapshots</id>
            <username>你的账号</username>
            <password>你的token</password>
        </server>
       <server>
            <id>meituan-dianping-server-id</id>
            <username>你的账号</username>
            <password>你的token</password>
        </server>
        <server>
            <configuration>
                <httpsConfiguration>
                    <all>
                        <connectionTimeout>5000</connectionTimeout>
                        <readTimeout>10000</readTimeout>
                    </all>
                </httpsConfiguration>
            </configuration>
        </server>
    </servers>

    <profiles>
        <profile>
            <id>meituan-dianping</id>
            <activation>
                <activeByDefault>true</activeByDefault>
            </activation>
            <repositories>
                <repository>
                    <id>meituan-dianping-releases</id>
                    <name>Repository for releases artifacts</name>
                    <url>https://pixel.sankuai.com/repository/group-releases</url>
                    <snapshots>
                        <enabled>false</enabled>
                    </snapshots>
                    <releases>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                    </releases>
                </repository>
                <repository>
                    <id>meituan-dianping-snapshots</id>
                    <name>Repository for snapshots artifacts</name>
                    <url>https://pixel.sankuai.com/repository/group-snapshots</url>
                    <snapshots>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                    </snapshots>
                    <releases>
                        <enabled>false</enabled>
                    </releases>
                </repository>
            </repositories>

            <pluginRepositories>
                <pluginRepository>
                    <id>meituan-dianping-releases-plugin</id>
                    <name>Repository for plugin releases artifacts</name>
                    <url>https://pixel.sankuai.com/repository/group-releases</url>
                    <snapshots>
                        <enabled>false</enabled>
                    </snapshots>
                    <releases>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                        <checksumPolicy>ignore</checksumPolicy>
                    </releases>
                </pluginRepository>
                <pluginRepository>
                    <id>meituan-dianping-snapshots-plugin</id>
                    <name>Repository for plugin snapshots artifacts</name>
                    <url>https://pixel.sankuai.com/repository/group-snapshots</url>
                    <snapshots>
                        <enabled>true</enabled>
                        <updatePolicy>always</updatePolicy>
                        <checksumPolicy>ignore</checksumPolicy>
                    </snapshots>
                    <releases>
                        <enabled>false</enabled>
                    </releases>
                </pluginRepository>
            </pluginRepositories>
        </profile>
    </profiles>

    <pluginGroups>
        <pluginGroup>org.apache.maven.plugins</pluginGroup>
        <pluginGroup>org.unidal.maven.plugins</pluginGroup>
        <pluginGroup>com.dianping.maven.plugins</pluginGroup>
        <pluginGroup>org.jvnet.hudson.tools</pluginGroup>
    </pluginGroups>

    <activeProfiles>
        <activeProfile>meituan-dianping</activeProfile>
    </activeProfiles>
    <mirrors>
        <mirror>
            <id>meituan-dianping</id>
            <name>meituan-dianping mirror</name>
            <url>https://pixel.sankuai.com/repository/mtdp</url>
            <mirrorOf>*</mirrorOf>
        </mirror>
    </mirrors>
</settings>

海龙setting配置（含offline仓库）

自己原setting配置（不含offline仓库）

/data/webapps/appenv

参考：https://km.sankuai.com/page/1271229597

iterm2, on-my-zsh（可选）

drawio（可选）

三、公司内部工具熟悉

学城--内部文档平台

https://km.sankuai.com/space/driver

code--代码托管平台

https://dev.sankuai.com/code/home?codeArea=mcode

ones--需求研发管理系统（类JIRE）

Avatar

https://avatar.mws.sankuai.com/#/service/detail/info?appkey=com.sankuai.qcs.service.goldeneye&env=prod

RDS--数据库可视化

test权限申请：联系袁瑞东

prod权限：页面申请

https://rds.mws.sankuai.com/dba/sqlEditor

Lion

配置中心

Raptor--监控

https://raptor.mws.sankuai.com/application/rpc?reportType=hour&date=&startDate=20231013160000&endDate=20231013165900&r=23572&ip=All&group=&domain=com.sankuai.qcs.service.goldeneye&type=OctoService

plus

jumpter

四、公司内部中间件

squirrel（封装redis）

clink

五、平台及工具权限添加

联系同事/领导添加各个系统及工具的权限

经营工作台（prod）：https://biz.sankuai.com/noAuth?childRC=configurationCenterList

RDS 数据库权限

jumper申请

六、其他

开发规范

https://km.sankuai.com/page/928322097

开发流程

https://km.sankuai.com/page/928383581

IT办公

https://km.sankuai.com/page/968603055

打卡

早上10点前，总时间大于9h，过闸机自动打卡。

周报

参考文献

https://km.sankuai.com/collabpage/1911268334

https://km.sankuai.com/collabpage/1830634752