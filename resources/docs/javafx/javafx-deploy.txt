## JavaFX程序部署
***

#### 0.准备工作
安装 Inno Setup，为应用程序创建一个单独.exe文件的Windows安装器。生成的.exe执行用户级别的安装（无需管理员权限），也创建一个快捷方式（菜单和桌面）。
* 下载[Inno Setup 5](http://www.jrsoftware.org/isdl.php)以后版本，安装Inno程序到你的计算机上。我们的Ant脚本将使用它自动生成安装器。
* 添加Inno安装路径到Path环境变量中。  
![Path设置](/images/posts/deploy01.png)  
* 安装完成后，需要重启Eclipse。

#### 1.新建JavaFX工程
* AppDeploy工程项目内容如下：  
![AppDeploy](/images/posts/deploy00.png)  

#### 2.编辑build.fxbuild文件
* 编辑如下(1)~(6)部分。  
![编辑build.fxbuild](/images/posts/deploy02.png)  
* 点击(7)部分"仅生成ant build.xml文件"。
	- 出现一个"Not a JDK"对话框，选择"Yes"。
	- 又出现一个"Select a JDK"对话框，选择"Cancel"。

此时，工程项目内容如下：  
![build.xml](/images/posts/deploy03.png)  

#### 3.添加安装程序图标
* 在build目录下创建下面的子目录:
	- build/package/windows (只用于Windows)
* 将相关图标到windows目录中，现在工程项目内容如下：  
![windows](/images/posts/deploy04.png)  
* ==注意==：图标的名称必须精确匹配build.fxbuild中指定的**Application的标题名**。
	- YourAppTitle.ico
	- YourAppTitle-setup-icon.bmp

#### 4.添加资源文件
resources目录不能自动拷贝，必须手动添加它到build目录下:
* 在build目录下创建下面的子目录:
	- build/dist
* 拷贝resources目录（包含应用的图标）到build/dist。
此时，工程项目的内容如下：  
![dist](/images/posts/deploy05.png)  

#### 5.编辑build.xml
e(fx)clipse没有告诉它包含其它资源，例如resources目录和上面添加的安装文件图标，必须手动编辑build.xml文件。
打开build.xml文件，找到路径fxant。添加一行到${basedir}，使得安装器图标可用。
```XML
		<path id="fxant">
			<filelist>
				<file name="${java.home}\..\lib\ant-javafx.jar"/>
				<file name="${java.home}\lib\jfxrt.jar"/>
				<file name="${basedir}"/>
			</filelist>
		</path>
```
找到块fx:resources id="appRes"，文件的更下面位置。为resources添加一行：
```XML
		<fx:resources id="appRes">
			<fx:fileset dir="dist" includes="AppDeploy.jar"/>
			<fx:fileset dir="dist" includes="libs/*"/>
			<fx:fileset dir="dist" includes="resources/**"/>
		</fx:resources>
```
有时候，版本数不能添加到fx:application中，使得安装器总是缺省的版本1.0。为了修复它，手动添加版本号:
```XML
		<fx:application id="fxApplication"
			name="AppDeploy"
			mainClass="ch.makery.address.MainApp"
			version="1.6"
		/>
```
#### 6.运行build.xml
使用ant运行build.xml，右击 build.xml文件| Run As | Ant Build。  
* 命令运行如下：  
![ant](/images/posts/deploy06.png "Ant Build" "width:600px;height:80px")  
构建完成后，选择项目根目录，按F5刷新项目内容，此时工程项目内容如下：  
![ant build](/images/posts/deploy07.png)  
文件夹bundles下的AppDeploy-1.6.exe即为Windows安装文件，此时的安装文件在安装时运行界面如下：  
![app install](/images/posts/deploy08.png)  
可以看出，安装程序不允许用户选择安装目录，程序将默认安装在`C:\Users\**\AppData\Local\application name`目录下。

#### 7.改变安装目录
* 查看刚才Ant Build运行时Eclipse的Console端输出内容：

```Bash
Buildfile: H:\EclipseWorkplace\AppDeploy\build\build.xml

......

使用以下位置的基础 JDK: G:\Java\jdk1.8.0_66\jre
使用以下位置的基础 JDK: G:\Java\jdk1.8.0_66\jre
Using custom package resource [application icon]  (loaded from package/windows/AppDeploy.ico)
Icon File Name: C:\Users\**\AppData\Local\Temp\fxbundler8755168410396838924\windows\AppDeploy.ico
Executable File Name: C:\Users\**\AppData\Local\Temp\fxbundler8755168410396838924\images\win-exe.image\AppDeploy\AppDeploy.exe
  Config files are saved to C:\Users\**\AppData\Local\Temp\fxbundler8755168410396838924\windows. Use them to customize package.
  Using default package resource [InnoSetup 项目文件]  (add package/windows/AppDeploy.iss to the class path to customize)
  Using custom package resource [设置对话框图标]  (loaded from package/windows/AppDeploy-setup-icon.bmp)
Using default package resource [要在填充应用程序映像之后运行的脚本]  (add package/windows/AppDeploy-post-image.wsf to the class path to customize)
Inno Setup 5 Command-Line Compiler
......

Successful compile (66.859 sec). Resulting Setup program filename is:
H:\EclipseWorkplace\AppDeploy\build\deploy\bundles\AppDeploy-1.6.exe
安装程序 (.exe) 已保存到: H:\EclipseWorkplace\AppDeploy\build\deploy\bundles
  配置文件已保存到C:\Users\**\AppData\Local\Temp\fxbundler8755168410396838924\windows。使用这些配置文件可定制程序包。
BUILD SUCCESSFUL
Total time: 1 minute 26 seconds
```
其中，有一句`配置文件已保存到C:\Users\**\AppData\Local\Temp\fxbundler8755168410396838924\windows。使用这些配置文件可定制程序包。`打开`C:\Users\**\AppData\Local\Temp\fxbundler8755168410396838924\windows`文件夹，其内容如下：  

![app install](/images/posts/deploy09.png)  

* 将上述文件夹中的AppDeploy.iss文件拷贝到`package/windows/`目录下。
此时，工程项目的内容如下：  

![app install](/images/posts/deploy10.png)  

* 修改AppDeploy.iss文件内容，以支持用户选择安装目录：
	+ 修改的原内容如下：
```Makefile
DefaultDirName={localappdata}\AppDeploy
DisableStartupPrompt=Yes
DisableDirPage=Yes
DisableProgramGroupPage=Yes
DisableReadyPage=Yes
DisableFinishedPage=Yes
DisableWelcomePage=Yes
```
	+ 修改后内容为：
```Makefile
DefaultDirName={pf}\AppDeploy
DisableStartupPrompt=Yes
DisableDirPage=No
DisableProgramGroupPage=Yes
DisableReadyPage=No
DisableFinishedPage=No
DisableWelcomePage=No
```
	* 上述含义：  
		- DefaultDirName={pf}\AppDeploy:默认安装目录设为C:\Program File\ApplicationName。
		- DisableDirPage=No：允许用户选择安装目录
		- DisableReadyPage=No：显示就绪安装界面
		- DisableFinishedPage=No：显示安装完成界面
		- DisableWelcomePage=No：显示安装程序欢迎界面

* AppDeploy.iss文件完整内容如下：

```Makefile
;This file will be executed next to the application bundle image
;I.e. current directory will contain folder AppDeploy with application files
[Setup]
AppId={{fxApplication}}
AppName=AppDeploy
AppVersion=1.6
AppVerName=AppDeploy 1.6
AppPublisher=makery.ch
AppComments=AppDeploy
AppCopyright=Copyright (C) 2016
;AppPublisherURL=http://java.com/
;AppSupportURL=http://java.com/
;AppUpdatesURL=http://java.com/
DefaultDirName={pf}\AppDeploy
DisableStartupPrompt=Yes
DisableDirPage=No
DisableProgramGroupPage=Yes
DisableReadyPage=No
DisableFinishedPage=No
DisableWelcomePage=No
DefaultGroupName=makery.ch
;Optional License
LicenseFile=
;WinXP or above
MinVersion=0,5.1 
OutputBaseFilename=AppDeploy-1.6
Compression=lzma
SolidCompression=yes
PrivilegesRequired=lowest
SetupIconFile=AppDeploy\AppDeploy.ico
UninstallDisplayIcon={app}\AppDeploy.ico
UninstallDisplayName=AppDeploy
WizardImageStretch=No
WizardSmallImageFile=AppDeploy-setup-icon.bmp   
ArchitecturesInstallIn64BitMode=x64


[Languages]
Name: "english"; MessagesFile: "compiler:Default.isl"

[Files]
Source: "AppDeploy\AppDeploy.exe"; DestDir: "{app}"; Flags: ignoreversion
Source: "AppDeploy\*"; DestDir: "{app}"; Flags: ignoreversion recursesubdirs createallsubdirs

[Icons]
Name: "{group}\AppDeploy"; Filename: "{app}\AppDeploy.exe"; IconFilename: "{app}\AppDeploy.ico"; Check: returnTrue()
Name: "{commondesktop}\AppDeploy"; Filename: "{app}\AppDeploy.exe";  IconFilename: "{app}\AppDeploy.ico"; Check: returnFalse()


[Run]
Filename: "{app}\AppDeploy.exe"; Parameters: "-Xappcds:generatecache"; Check: returnFalse()
Filename: "{app}\AppDeploy.exe"; Description: "{cm:LaunchProgram,AppDeploy}"; Flags: nowait postinstall skipifsilent; Check: returnTrue()
Filename: "{app}\AppDeploy.exe"; Parameters: "-install -svcName ""AppDeploy"" -svcDesc ""AppDeploy"" -mainExe ""AppDeploy.exe""  "; Check: returnFalse()

[UninstallRun]
Filename: "{app}\AppDeploy.exe "; Parameters: "-uninstall -svcName AppDeploy -stopOnUninstall"; Check: returnFalse()

[Code]
function returnTrue(): Boolean;
begin
  Result := True;
end;

function returnFalse(): Boolean;
begin
  Result := False;
end;

function InitializeSetup(): Boolean;
begin
// Possible future improvements:
//   if version less or same => just launch app
//   if upgrade => check if same app is running and wait for it to exit
//   Add pack200/unpack200 support? 
  Result := True;
end;  
```

* 再次运行安装文件，显示如下：  
![app install dir](/images/posts/deploy11.png)  
![app install dir](/images/posts/deploy12.png)  
![app install dir](/images/posts/deploy13.png)  
![app install dir](/images/posts/deploy15.png)  
#### 8.**说明**：
* 存在一个默认设置: UsePreviousAppDir,既安装程序会将默认安装目录设置为上次安装目录，故在进行第二次安装前，需要先卸载之前的安装.  
* 软件安装时如果出现错误(即`重命名package.dll错误`)，请选择**中止**，并注销计算机，然后尝试再次安装。  
	- 可能出现的错误：  
	![app install error](/images/posts/deploy14.png)  
	- 也可以选择忽略，其实安装目录下仅仅缺少了package.dll文件，将[package.dll](/resources/package.dll)文件下载下来，拷贝到软件安装的根目录下即可。
* 所有用JavaFX开发的软件最好都安装在同一目录下，这样可以共用一些文件，如Java运行时等。但弊端是，卸载文件也将共用（仅有一个卸载文件），一旦卸载一个软件，该目录下的所有软件都会被卸载。

#### 参考资料
* [JavaFX 8 教程 - 第七部分：部署](http://code.makery.ch/library/javafx-8-tutorial/zh-cn/part7/)
* [JavaFX Self Installer With Inno Setup 5](http://stackoverflow.com/questions/28370655/javafx-self-installer-with-inno-setup-5-allow-user-to-change-install-directory)






