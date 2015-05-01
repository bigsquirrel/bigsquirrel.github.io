---
layout: post
title:  "使用 gradle 脚本解决 utf-8 bom 的问题"
date:   2015-05-01 10:00:00
tags:
    - bug
    - Android
    - gradle
---
碰到了这么一个编译错误

	Error:(1, 1) error: illegal character: \65279

Google 之后得知是作死的 Windows 下 UTF-8 编码 with BOM or NOBOM 的问题，而 javac 并不认这个东西，那么解决方案也是很理所当然的，将编码方式换成 NOBOM 就 OK 了，其中 Notepad++ 等一系列编辑器就支持这个功能，但后来发现其实可以写一个 gradle 脚本来帮我们处理这件事。
	
	task convertSource << {
	    // convert sources files in source set to normalized text format
	    android.sourceSets.main.java.srcDirs.each { File fileDir ->
	        // read first "raw" line via BufferedReader
	        FileTree tree = fileTree(dir:fileDir)
	        tree.each { File file ->
	            def r = new BufferedReader(new FileReader(file))
	            String s = r.readLine()
	            r.close()
	            // get entire file normalized
	            String text = file.text
	            // get first "normalized" line
	            String normalizedLine = new StringReader(text).readLine()
	            if (s != normalizedLine) {
	                println "rename: $file"
	                File target = new File(file.getParentFile(), file.getName() + '.bak')
	                if (!target.exists() && file.renameTo(target))
	                    file.setText(text)
	                else
	                    println "failed to rename or target already exists"
	            }
	        }
	    }
	} // end task

这段脚本代码源是从 [stackoverflow][gradle_code] 中经过一些修改，原来的代码 copy 过来会报错：
	
	* What went wrong:
	Execution failed for task ':convertSource'.
	> Could not find property 'main' on SourceSet container.

从意思上看就是 scope 不对，访问不到 sourceSets 的 main，附上我的 sourceSets 配置：

	android {
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
	}

从经验上猜测是作用域的问题，那么我想当然的把那段 task 代移到 android scope 中，但依然不行。其实 gradle 也是一门语言，要精通还是挺困难的，于是边看语法边摸索，这时候 `println` 真是最好的 debug 工具，在出问题的脚本代码前面加上 android 域后，能够正常打印出路径了，但却提示是一个 directory，继续翻遍历一个目录的语法，于是加上了上面那段 tree 的代码，成功的打印出了目录下的每个文件也就是说问题解决了，同时 stackoverflow 上的这段脚本也成功的将编码问题解决，这段脚本其实也很简单，我不确定这是不是调用的 Java 代码，但看起来简直就是。

总算可以在 Android Studio 下码了，也不枉我折腾一个晚上加早晨。

ps: 修改的代码看起来其实很奇怪，因为
	
	println android.sourceSets.main.java.srcDirs

得到的是一个路径没错，但是被 `[]` 包裹，而 each 应该是可以遍历目录的，不太清楚为什么不可以。这里

	android.sourceSets.main.java.srcDirs.each

的作用是去掉 `[]`，为什么我这样做？用字符串操作不就可以达到目的吗？理论上肯定是有这样的方法，但在我 println 后得到了正确的路径实现了目的，我也不想再去找什么其他的方法替代了。

[gradle_code]:http://stackoverflow.com/questions/17084727/disable-encoding-checking-in-java-gradle-project
