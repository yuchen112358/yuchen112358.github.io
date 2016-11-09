错误信息如下：
```Bash
javac] Compiling 22 source files to C:\Users\Hassan\Desktop\Programming\workspace\Raktar_vevo 2.7\build\classes
[javac] warning: [options] bootstrap class path not set in conjunction with -source 1.7
[javac] Note: C:\Users\Hassan\Desktop\Programming\workspace\Raktar_vevo 2.7\build\src\application\SajátKészlet.java uses unchecked or unsafe operations.
[javac] Note: Recompile with -Xlint:unchecked for details.
[javac] 1 warning
 [copy] Copying 12 files to C:\Users\Hassan\Desktop\Programming\workspace\Raktar_vevo 2.7\build\classesinit

    -fx-tasks:
  [taskdef] Could not load definitions from resource com/sun/javafx/tools/ant/antlib.xml. It could not be found.
do-deploy:
     [copy] Copying 20 files to C:\Users\Hassan\Desktop\Programming\workspace\Raktar_vevo 2.7\dist\libs

BUILD FAILED
C:\Users\Hassan\Desktop\Programming\workspace\Raktar_vevo 2.7\build.xml:217: Problem: failed to create task or type javafx:com.sun.javafx.tools.ant:resources
Cause: The name is undefined.
Action: Check the spelling.
Action: Check that any custom tasks/types have been declared.
Action: Check that any <presetdef>/<macrodef> declarations have taken place.
No types or tasks have been defined in this namespace yet


Total time: 22 seconds 
```
**解决方法如下：**  
此时我们需要进行如下设置：
```
Prefereces->Java->Installed JREs, and check it as "separate jre" in External Tools Configuration->JRE in case of Eclipse
```
* 设置1：Window->Preferences->Java->Installed JREs  

![设置1](/images/posts/jfx-jre01.png)  

* 设置2：Run > External Tools > External Tool Configuration  

![设置2-1](/images/posts/jfx-jre00.png)  

![设置2-2](/images/posts/jfx-jre02.png)  

#### 参考资料
* [JAVAFx Build Failed](http://stackoverflow.com/questions/24840414/javafx-build-failed)
