---
layout: post
title:  "如何在 Android Studio 下与 Eclipse 协同开发"
date:   2015-05-01 00:01:00
tags:
    - 技巧
    - Android
---
最近开始的一个合作项目的同事用的都是 eclipse，然而自从 Android Studio 发布的那天起就不断关注着并且在去年成功的转移到了 Android Studio 下，在被 Android Studio 惯坏了的情况下再回到丑陋效率低下的 eclispe 自然是相当不情愿的，遂研究下了如何保持工作目录不变在 AS 环境下开发。

具体操作如下从 svn checkout 项目之后，不要选择从 eclipse adt 导入，这样会自动生成 gradle 并将工作目录重新安排与 AS 项目一致。那么导入的项目依然与 eclipse 下保持一致后，在项目中新建 build.gradle 文件（可以从其他项目中拷贝过来），文件内容大概是这样：

	// Top-level build file where you can add configuration options common to all sub-projects/modules.

	buildscript {
	    repositories {
	        jcenter()
	    }
	    dependencies {
	        classpath 'com.android.tools.build:gradle:1.0.1'

	        // NOTE: Do not place your application dependencies here; they belong
	        // in the individual module build.gradle files
	    }
	}

	allprojects {
	    repositories {
	        jcenter()
	    }
	}


	apply plugin: 'com.android.application'

	android {
	    compileSdkVersion 19
	    buildToolsVersion "21.1.1"

	    defaultConfig {
	        applicationId ""
	        minSdkVersion 14
	        targetSdkVersion 19
	        versionCode 1
	        versionName "1.0"
	    }
	    buildTypes {
	        release {
	            minifyEnabled false
	            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
	        }
	    }
	}

	dependencies {
	    compile fileTree(dir: 'libs', include: ['*.jar'])
	}

中间去掉一部分信息，基本就是这样的一个结构了，我们来看 Android 官方关于 Gradle 的一些[说明][gradle]，其中 3.2.1 节 `Configuring the Structure` 已经告诉我们怎么做了。也就是在 android scope 中添加如下的 code，代表一个目录的映射。
	
    sourceSets {
        main {
            manifest.srcFile 'AndroidManifest.xml'
            java.srcDirs = ['src']
            resources.srcDirs = ['src']
            aidl.srcDirs = ['src']
            renderscript.srcDirs = ['src']
            res.srcDirs = ['res']
            assets.srcDirs = ['assets']
        }

        androidTest.setRoot('tests')
    }

这些工作完成之后其实也就 OK 了，点击下工具栏中的 Sync Project with Gradle Files，很快你的项目已经被 gradle 构建完毕，就可以愉快的在 AS 下与 eclipse 的小伙伴协作了，有木有很开心。

当然最后你还需要做的是 svn ignore 的配置，把所有于项目无关的统统 ignore，避免对同事照成影响。 over～

[gradle]:http://tools.android.com/tech-docs/new-build-system/user-guide#TOC-Configuring-the-Structure
