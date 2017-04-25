# Gradle打造高效开发环境
本文试图解决开发过程中常见的问题

这些问题我们每个人、每一天都会遇到，非常普遍，以至于我们都觉得那不是问题

### 当前开发过程中常见的问题
#### 启动工程

eclipse或idea打开工程 >> 注释maven排除配置文件的代码 >> 配置tomcat >> 点击debug按钮 >> 工程启动

* 这个步骤有多少人一次成功？（尤其是qianbao工程）
* 有多少次启动失败是因为缺少配置文件？
* 为什么昨天启动还好好的，今天就启动不了了？
* 如果要启动多个工程势必要打开多个IDE，你的电脑不卡吗？
* 换个电脑或换个人是不是要重新来一遍？
* maven排除配置文件是为了方便测试和运维人员，为什么开发环境会受影响？

#### 接口联调

每次接口联调都是噩梦，qianboa666与qbao混乱，登录基本每次都有问题，153服务器上的配置文件被改来改去，每次出问题都要从头排查，服务器hosts，nginx，tomcat，cas.properties、domain.properties，ip白名单，……

现在的服务器都有部署脚本,每次部署前要先修改它的svn路径再执行，可算作某种程度上的自动化，虽然把svn路径作为参数很容易，再加上配置文件替换、清空日志、重启tomcat、集群部署等功能也不是很难，但还远远不够

主要缺点是它没有被版本控制系统管理起来，在多人使用时依然会造成混乱；其次，配置文件替换的代码与工程代码不同步，难于维护

## Gradle
Gradle使用groovy语言定制的DSL，groovy采用java语法，ruby思想，基于jvm，灵活实用，简单易学
![groovy](/home/ma/图片/groovy.png)

Gradle借鉴了ant和maven，兼具ant灵活的task和maven强大的依赖管理，上可以完成复杂的全套环境（dev/test/pre/prd）发布工作，下可以替代maven管理本地工程，DevOps利器，用简单的命令完成复杂的重复性工作
![Why Gradle](/home/ma/图片/why-gradle.png)

### 启动本地工程
	svn co http://192.168.7.237/ngsvnroot/message/trunk message
	cd message
	gradle api:debug

### 自动部署
gradle api:deploy
![gradle deploy 1](/home/ma/图片/gradle-deploy1.png)
![gradle deploy 2](/home/ma/图片/gradle-deploy2.png)

### 部署到指定环境
gradle api:deploy -Pserver=116

### 部署指定分支
gradle api:deploy -Pserver=116 -PsvnUrl=http://192.168.7.237/ngsvnroot/message/branches/message-20150624-test

### 你需要做什么？
下载gradle，解压，生成ssh key，done！


### 替代maven
	project(':app'){
		apply plugin: 'war'
		apply plugin: 'org.akhikhl.gretty'

		gretty {
			httpPort = ports[project.name]
			debugPort = httpPort + 1
			jvmArgs = ['-Xmx1024M', '-XX:PermSize=128M', '-XX:MaxPermSize=256M']
			servletContainer = 'jetty7'
		}

		sourceSets {
			main { resources.srcDirs = ['src/main/resources', 'src/main/native2ascii'] }
		}

	    war {
	        processResources{
	            if(!hasCommand('build')){ return }
	            exclude '**/redis.properties'
	        }
	    }

		dependencies {
			compile(
				project(":common"),
				"org.springframework.security:spring-security-config:${securityVersion}",
				fileTree(dir: 'lib' , include: '*.jar' )
			)
		}
	}
### 超越maven，一切皆是代码
	task compileServer(dependsOn: getSvnUrl) << {
		description '更新代码，编译，打包'
		ssh.run {
			session(remoteServer) {
				println ">>>>>>> svn代码准备完完毕，下面开始编译"
				def mavenCompile = [
						"cd $branchDir",
						"export JAVA_HOME=$javaPath",
						"$gradleCommand ${project.name}:build -x test"
				]
				if(!project.hasProperty('nocompile')){
					execute mavenCompile.join(' && ')
				}
				exists = execute "[ -f $warPath ] && echo yes || echo no"
				if(exists == 'no'){
					throw new RuntimeException("不存在$warPath 编译失败")
				}
			} // end session
		}
	}

	task deployCode(dependsOn: compileServer) << {
		description '修改配置文件'
		ssh.run {
			session(remoteServer) {
				println ">>>>>>> 准备${projectName}/${project.name}工程发布目录"
				execute "[ -d $deployRootDir ] || mkdir -p $deployRootDir"
				execute "rm -rf $deployRootDir/*"
				println ">>>>>>> 解压 $warPath 到发布目录 $deployRootDir"
				execute "unzip -oq $warPath -d $deployRootDir"
				println ">>>>>>> 准备配置文件，域名替换"
				execute "cp $branchDir/common/src/main/resources/*.properties $deployRootConfDir"
				replaceConfFile.each { confFile ->
					replaceUrl.each{ key,value ->
						execute "sed -i 's@$key@'$value'@g' $deployRootConfDir/$confFile"
					}
				}
			} // end session
		}
	}

### 兼容当前的测试 & 运维
只需替换`mvn clean install -Dmaven.skip.test=true`为`gradle build -x test`即可，war包在${project}/build/libs下面

### 目前已经添加gradle支持的工程
* qianbao 
* goods 
* user 
* enterprise 
* order
* message
* search （删除了maven的pom.xml文件，完全使用gradle）
* cas
* leipai

### 关于开发效率
在项目很紧张的情况下还要抽时间学习、维护gradle，岂不是又增加了工作量？

提高效率的关键在于降低反馈成本，如果部署联调的成本太高，开发人员就会推迟部署联调，直到在本机完成所有的功能，在这期间依赖他的人就只能等，无形之中降低了开发效率

gradle可以保证部署的正确性，如果接口调不通，不用怀疑环境，一定是代码的问题，单就这一点就可以节省开发人员大把的时间

目前的gradle部署脚本还不支持集群、软件自动安装等功能，虽无法媲美Capistrano，但已经足够成熟

## One more thing——复杂环境部署
上述gradle脚本已经解决了代码的配置文件管理问题，但多数线上应用都是以集群的方式运行，仅user app系统就有13台tomcat服务器，更不用说qianbao，leipai等系统了，这么多机器仅ip地址的维护就是个不小的问题

解决这个问题常用的办法是逻辑分组，以下面的yaml文件为例

	---
	cluster_name: user

	hosts:
		user1:
		  module: app
		  ip: 192.168.7.1
		  compile: true
		  domain: qianbao666
		  port: 8080
		  tomcat_path: /usr/local/tomcat_app

		user2:
		  module: app
		  ip: 192.168.7.2
		  domain: qianbao666
		  port: 8080
		  tomcat_path: /usr/local/tomcat_app
	
		user3:
		  module: api
		  ip: 192.168.7.3
		  compile: true
		  domain: qbao
		  port: 7070
		  tomcat_path: /usr/local/tomcat_api

这是一份服务器清单，通过这份清单可以生成nginx配置文件，如果要在集群中新加一台机器，直接加个服务配置节点，重新部署，部署过程中生成新的nginx配置文件，并完成nginx重启，tomcat启动，日志归档，可用性检查等常规工作
