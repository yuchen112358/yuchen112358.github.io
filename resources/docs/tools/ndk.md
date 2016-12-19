1\. 切换至"Project"项目视图，将app/src目录下的androidTest和test文件夹删除；

![](/images/posts/ndk00.png)  
2\. 将build.gradle（Module:app）中的depencies中的相关test选项注释掉；
```Java
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    compile 'com.android.support:appcompat-v7:24.0.0'
    compile 'com.android.support.constraint:constraint-layout:1.0.0-alpha1'
//    testCompile 'junit:junit:4.12'
//    androidTestCompile 'com.android.support.test.espresso:espresso-core:2.2.2'
//    androidTestCompile 'com.android.support.test:runner:0.5'
//    androidTestCompile 'com.android.support:support-annotations:24.0.0'
}
```
##### 注意以上两步骤也可不做！！！
3\. 新建一个java类EnUtils, 增加两个native方法。如图:

![](/images/posts/ndk01.png)  

4\. 点击Build->Rebuild Project。之后，在build/intermediates/classes/debug目录下就会生成 EnUtils.class文件。,结果如下显示：

![](/images/posts/ndk02.png)  

5\. 进入到debug目录下(`cd app/build/intermediates/classes/debug/`)，执行javah命令,即`javah -jni io.github.wzzju.jnistudy.EnUtils`，就会在app/build/intermediates/classes/debug/目录下生成一个io_github_wzzju_jnistudy_EnUtils.h头文件。

![](/images/posts/ndk04.png)  

6\.在src/main目录下，创建jni目录，把之前生成的头文件复制到jni目录下。

![](/images/posts/ndk05.png)  

7\. 在jni目录下，创建c源文件，文件名可随意。函数实现，如图:

![](/images/posts/ndk06.png)  
这一步之后就可以Rebuild Project了，但会出错，依据提示修改gradle即可。

8\.配置gradle
1. 在gradle.properties中增加一行android.useDeprecatedNdk=true
2. 在app的build.gradle的defaultConfig中增加
```
ndk {
            moduleName "EnUtilsName" //so名字
            abiFilters "armeabi", "armeabi-v7a", "x86", "x86_64"
       }
```

并注释掉`testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"`此句,也可以不注释掉。

![](/images/posts/ndk07.png)  
![](/images/posts/ndk08.png)  

9\. 再次Rebuild Project，就生成了so文件,如图:

![](/images/posts/ndk09.png)  

10\. 如何使用调用so库文件?
即，在EnUtils类中添加加载so库的代码，名称要和build.gradle中定义的ndk moduleName一致。
```Java
package io.github.wzzju.jnistudy;

/**
 * Created by yuchen on 16-7-2.
 */

public class EnUtils {

    static {
        System.loadLibrary("EnUtilsLib");
    }

    public static native String getVer();

    public static native int max(int a, int b);

}
```
Java中调用，如下图：

![](/images/posts/ndk10.png)  

![](/images/posts/ndk11.png)  

10\. 运行，lldb始终配置不对，点击Run时会跳出如下对话框：

![](/images/posts/ndk12.png)  

点击"Continue Anyaway"。

###### 下面将要介绍，编译生成的so文件如何单独使用。
1. 删除jni目录，以及目录下的.h .c文件
2. build.gradle中ndk配置也删除掉。
3. 在src/main下创建jniLibs目录，再将ndk/debug/lib下的4个so目录复制到jniLibs目录下。
4. 再重新Clean Project, Rebuild Project即可。

#### 参考资料：

* [Android Studio NDK JNI开发入门记录](http://m.blog.csdn.net/article/details?id=51221823)