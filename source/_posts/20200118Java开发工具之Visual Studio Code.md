---
title: Java开发工具之Visual Studio Code
tags:
  - Java
  - 环境变量
categories: 技术
abbrlink: 21078
date: 2020-01-18 15:00:00
---



## Java开发工具之Visual Studio Code

> 子曰：工欲善其事，必先利其器。

> 先聊点使用这个开发Java的背景故事,从事Java开发的程序就不得不提 伟大的开发利器IDEA了,其一就是:但是最近idea民间激活崩盘的厉害,一不小心就不能使用了,本人也是,三天两头去找激活码激活.再就是:本人开发的工作环境条件比较苛刻,不能使用任何收费软件(原因....).于是我去试用了一两周的eclipse软件开发.非常极其的痛苦,非常慢的打开速度,加写代码时候不定时的卡顿,弱鸡的代码的提示,还有莫名其妙的红色的X一直在你面前闪!码代码的体验太过糟糕了.于是在新年前最后一个周末,我尝试找新的编码工具.


### 1.Visual Studio Code

Microsoft在2015年4月30日Build 开发者大会上正式宣布了 Visual Studio Code 项目：一个运行于 Mac OS X、Windows和 Linux 之上的，针对于编写现代 Web 和云应用的跨平台源代码编辑器。

[下载](https://code.visualstudio.com/) 安装 一步步 傻瓜式安装就行了

> 作为Java程序员使用微软的东西.)耻辱啊!! 嗯 真香!~


### 2.打开Visual Studio Code下载对应的插件

![](https://gitee.com/yttrium2016/img/raw/master/20200118171224769.PNG)

以下是我自己根据需求下载的插接

 - Java Extension Pack ***Java开发环境***
 - Spring Boot Extension Pack ***Spring开发提示***
 - Maven for Java  ***Java包管理工具***
 - Git Project Manager ***Git管理工具***
 - FreeMarker ***FreeMarker代码高亮***
 - Lombok Annotations Support for VS Code ***Lombok 可以少写好多Java的get set 方法***
 - Tomcat for Java ***Tomcat工具(还是要自己下载TOMCAT)***
 - XML ***XML文件支持***
 - Checkstyle for Java ***Java语法高亮 检测***
 - HTML Snippets ***html代码提示***
 - Eclipse Keymap ***eclipse快捷键 可以选择idea快捷键***
 - Debugger for Java ***Java调试用的***

### 3.打开Visual Studio Code对应的设置

	{
	    "editor.suggestSelection": "first",
	    "vsintellicode.modify.editor.suggestSelection": "automaticallyOverrodeDefaultValue",
	    "maven.executable.path": "C:\\Development\\apache-maven-3.6.3\\bin\\mvn",
	    "java.configuration.maven.userSettings": "C:\\Development\\apache-maven-3.6.3\\conf\\settings.xml",
	    "search.exclude": {
	        "**/node_modules": true,
	        "**/bower_components": true,
	        "**/target": true,
	        "**/logs": true
	    },
	    "java.jdt.ls.vmargs": "-noverify -Xmx1G -XX:+UseG1GC -XX:+UseStringDeduplication -javaagent:\"C:\\Users\\Y-PC\\.vscode\\extensions\\gabrielbb.vscode-lombok-0.9.9/server/lombok.jar\"",
	    "emmet.triggerExpansionOnTab": true,
	    "files.associations": {
	        "*.js": "html",
	        "*.vue": "html",
	        "*.ftl": "html"
	    }
	}


### 4.Java类快速建立的配置

	设置 -> 用户代码块 -> Java

	{
		"Java Class": {
			"prefix": "java",
			"body": [
				"package ${1:包名};",
				"",
				"/**",
				"* ${2:备注}",
				"* Created By 杨振宇",
				"* Date: $CURRENT_YEAR/$CURRENT_MONTH/$CURRENT_DATE",
				"* Time: $CURRENT_HOUR:$CURRENT_MINUTE:$CURRENT_SECOND",
				"*/",
				"public class ${3:ClassName} {",
				"",
				"}"
			],
			"description": "Java Class"
		}
	}

	使用 -> java文件建立后 输入java -> Tab 就会出现对应的代码块了


> 目前就这点配置 以后待补充