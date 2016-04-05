---
title: DroidPlugin插件化应用分析
date: 2016-04-25 14:08
tags: 
  - 插件化 
categories: Android
---
[TOC]
## <font color=#C4573C size=5 face="黑体">简介</font>
DroidPlugin 是360手机助手在Android系统上实现的一种新的插件机制:它可以在无需安装、修改的情况下运行APK文件,此机制对改进大型APP的架构，实现多团队协作开发具有一定的好处
详情请查看DroidPlugin的[github地址](https://github.com/Qihoo360/DroidPlugin)
## <font color=#C4573C size=5 face="黑体">背景</font>
> * 将项目中某个相对独立的功能模块分解出来
> * 例如：语音搜索功能模块独立出来，这样减少了项目中依赖包的数量，减少了项目中某些包的层层依赖关系，将语音搜索模块独立成一个单独的apk
> * 可以独立开发语音搜索模块
> * 假设主项目为Host，依赖library为DroidPlugin，独立出来的语音模块生成为voice.apk

## <font color=#C4573C size=5 face="黑体">应用</font>
 这里将独立出来的语音搜索模块打包成单独的apk，放在项目的assets目录下

项目中的设置步骤：
 > * 导入DroidPlugin相关的library
> * 在自己的Application当中添加如下代码
```java
 @Override
public void onCreate() {
    super.onCreate();
    //这里必须在super.onCreate方法之后，顺序不能变
    PluginHelper.getInstance().applicationOnCreate(getBaseContext());
}

@Override
protected void attachBaseContext(Context base) {
    PluginHelper.getInstance().applicationAttachBaseContext(base);
    super.attachBaseContext(base);
}
```
> * 配置主应用AndroidManifest.xml中相关的组件和权限
这里需要注意DroidPlugin library中需要的权限以及组件要在主项目Host中声明
还有就是Host中的权限只能比voice的多，不能少，否则会有问题
> * 读取assets里面的voice.apk并复制到sd上或者其他的可读取存储位置，然后再使用DroidPlugin进行安装，这里将需要用到的一些方法封装在PluginUtils工具类中。
> * 当voice.apk通过插件式的形式安装完成以后，我们就可以通过如下方式启动voice.apk
```java
PluginUtils.startActivity(SearchResultActivity.this,"com.storm.smart.voice");
```
> * 在Application或者MainActivty中加载voice.apk
```java
	/**
     * 通过DroidPlugin插件安装voice.apk
     * 从sd卡根目录读取
     */
    private void droidPluginInstall() {
        BfExecutor.getInstance().execute(new Runnable() {
                @Override
                public void run() {
                    if (PluginUtils.copyApkFromAssets(getApplicationContext(), "voice.apk",
                            Environment.getExternalStorageDirectory().getAbsolutePath() + "/voice.apk")) {
                        PluginUtils.installApk(getApplicationContext(),
                                Environment.getExternalStorageDirectory().getAbsolutePath() + "/voice.apk",
                                "com.storm.smart.voice");
                    }
                }
            });
    }
    
	/**
	 * 通过DroidPlugin插件安装voice.apk
	 * 从data下读取，需要权限（有可能手机没有sd卡）
	 */
	private void droidPluginInstall() {
		BfExecutor.getInstance().execute(new Runnable() {
			@Override
			public void run() {
				String path = "/data/data/" + getApplicationContext().getPackageName() + "/files" + "/voice.apk";
				try {
					String command = "chmod " + "666" + " " + path;
					Runtime runtime = Runtime.getRuntime();
					runtime.exec(command);
				} catch (IOException e) {
					e.printStackTrace();
				}
				if (PluginUtils.copyApkFromAssets(getApplicationContext(), "voice.apk", path)) {
					PluginUtils.installApk(getApplicationContext(), path, "com.storm.smart.voice");
				}
			}
		});
	}
```
> * PluginUtils全部代码如下：
```java
public class PluginUtils {

    public static void startActivity(Activity activity, String packageName){
        PackageManager pm = activity.getPackageManager();
        Intent intent = pm.getLaunchIntentForPackage(packageName);
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        activity.startActivity(intent);
    }
    
    /**
     * 删除apk
     * @param activity
     * @param packageName
     */
    public static  void doUninstall( Activity activity, String packageName) {
        if (!PluginManager.getInstance().isConnected()) {
            Toast.makeText(activity, "服务未连接", Toast.LENGTH_SHORT).show();
        } else {
            try {
                PluginManager.getInstance().deletePackage(packageName, 0);
                Toast.makeText(activity, "删除完成", Toast.LENGTH_SHORT).show();
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
    }
    
    /**
     * 应该在线程中安装，此方法仅供测试
     * @param activity
     * @param apkPath
     * @param packageName
     */
    public static boolean  installApk(Context context, String apkPath, String packageName) {
        if (!PluginManager.getInstance().isConnected()) {
            //installTips(context,"插件服务正在初始化，请稍后再试。。。");
            return false;
        }
        try {
            if (PluginManager.getInstance().getPackageInfo(packageName, 0) != null) {
                //installTips(context,"已经安装了，不能再安装");
            } else {
               int re = PluginManager.getInstance().installPackage(apkPath, 0);
               if(re == PluginManager.INSTALL_FAILED_NO_REQUESTEDPERMISSION){
                   //安装失败，文件请求的权限太多
                   //installTips(context,"安装失败，文件请求的权限太多");
               }else{
                   //安装完成
                   //installTips(context,"安装完成");
                   return true;
               }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }    
    
    /**安装提示
     * @param context
     * @param tips
     */
    public static void installTips(Context context,String tips){
    	 Looper.prepare();//1、初始化Looper 
         Toast.makeText(context, tips, Toast.LENGTH_SHORT).show();
         Looper.loop();//4、启动消息循环
    }
    
    /** 将assets文件下的apk读取到读取到指定路径
     * @param context
     * @param fileName
     * @param path
     * @return
     */
    public static boolean copyApkFromAssets(Context context, String fileName, String path) {  
        boolean copyIsFinish = false;  
        try {  
            InputStream is = context.getAssets().open(fileName);  
            File file = new File(path);  
            file.createNewFile();  
            FileOutputStream fos = new FileOutputStream(file);  
            byte[] temp = new byte[1024];  
            int i = 0;  
            while ((i = is.read(temp)) > 0) {  
                fos.write(temp, 0, i);  
            }  
            fos.close();  
            is.close();  
            copyIsFinish = true;  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
        return copyIsFinish;  
    }  
    
}

```
## <font color=#C4573C size=5 face="黑体">总结</font>
最终我们会发现在/data/data/【host packageName】下有一个Plugin文件夹，文件夹中有个【voice packageName】文件夹，里面有voice的相关数据
这样voice.apk就以插件的方式嵌入到我们的主应用中了。





