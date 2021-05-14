---
title: 通过 FileProvider 共享文件
date: 2018-5-26 22:17:00
tags: [Android, FileProvider]
---

最近碰到一个异常，`android.os.FileUriExposedException: file://*** exposed beyond app through Intent.getData()`，了解了一下，原来是 Android 7.0（api level 24）开始，通过 URI 与其他应用共享文件要求 URI 必须是 `content://` 开头的形式。而 [FileProvider][file-provider-doc] 是用来做这件事最简便的方法。

<!-- more -->

# 声明 FileProvider

`FileProvider` 是 `ContentProvider` 的子类，所以需要在 `AndroidManifest.xml` 中进行声明。

```xml
<provider
    android:name="android.support.v4.content.FileProvider"
    android:authorities="com.example.myapp.fileprovider"
    android:grantUriPermissions="true"
    android:exported="false">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/filepaths" />
</provider>
```

将 `android:grantUriPermissions` 设为 `true` 会允许得到访问这个 `FileProvider` 下所有数据的权限。

`meta-data` 这部分将会在后面再细说，其他部分都是常规 `ContentProvider` 的内容。

# 指定可访问的文件

在上面的部分中，`meta-data` 里指定了一个 xml 的资源文件，这个文件便是用来指定可以被访问到的文件的。

在资源文件夹中创建一个 xml 资源文件，根元素如下：

```xml
<paths xmlns:android="http://schemas.android.com/apk/res/android">

</paths>
```

`paths` 内可以添加如下的元素：

* `<files-path name="name" path="path" />`：代表应用内部存储的 `files/` 目录，也就是 `Context.getFilesDir()` 方法所得到的目录。
* `<cache-path name="name" path="path" />`：`Context.getCacheDir()` 方法得到的目录。
* `<external-path name="name" path="path" />`：外部存储目录，与 `Environment.getExternalStorageDirectory()` 方法得到的结果一样。
* `<external-files-path name="name" path="path" />`：对应 `Context.getExternalFilesDir(null)` 方法。
* `<external-cache-path name="name" path="path" />`: 对应 `Context.getExternalCacheDir()` 方法。
* `<external-media-path name="name" path="path" />`: 对应 `Context.getExternalMediaDirs()` 方法。

以上元素都有 `name` 和 `path` 两个属性。

我从 `path` 属性说起。`path` 即是真实的子目录地址段。比如 `<files-path  path="docs/" />` 对应的是 `files/docs/` 目录。那么，如果不想要子目录，而是当前目录呢？用 `.` 啊！

而 `name` 属性则是提供给外部应用的一个虚假的子目录地址段（在 content URI 中使用），主要是为了安全考虑。

# 为 File 生成 Content URI

使用 `FileProvider.getUriForFile()` 方法就可以为在指定目录里的文件生成 content URI。

```java
File imagePath = new File(Context.getFilesDir(), "images");
File newFile = new File(imagePath, "default_image.jpg");
Uri contentUri = FileProvider.getUriForFile(getContext(), "com.mydomain.fileprovider", newFile);
```

上面这段代码，再加上如下的资源文件：

```xml
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <files-path name="my_images" path="images/"/>
</paths>
```

最后得到的 content URI 会是：`content://com.mydomain.fileprovider/my_images/default_image.jpg`。

再回头来看一下。com.mydomain.fileprovider 是 ContentProvider 指定的 authority，同时也是 `getUriForFile()` 方法所需的参数。my_images 是 `files-path` 元素的 `name` 属性，将会映射到真实目录 images/，也就是 `path` 属性。

🙂这里，Android 又有了些小残念，在编写代码的时候无法保证资源文件的字符串和代码中的字符串是一致的。只能等到运行时才能知道。

# 为 URI 获取临时访问权限

通过上面的步骤，我们已得到一个地址正确的 URI，但这个 URI 还不能被其他应用正常使用。有两种方式来给予临时性的读写权限。

第一是使用 `Context.grantUriPermission()` 方法。这个方法需要指定要分享应用的包名。然后传入 `Intent.FLAG_GRANT_READ_URI_PERMISSION` 和 `Intent.FLAG_GRANT_WRITE_URI_PERMISSION` 来指定要授予的权限。给予权限之后，可以使用 `Context.revokeUriPermission()` 来取消；或者等到机器重启之后，这一次的授权也会失效。

这个方法需要事先设置应用的包名，并不是很实用。

第二个方法：可以使用 `Intent.setFlags()` 方法设置上面提到的两个 flag，来授予读写权限。此时，需要将这个 URI 用 `Intent.setData()` 来当成数据传递。由于我们多半是使用 Intent 来传递 URI 的，所以这个方法比较实用。

# 回顾

出于安全考虑，7.0 引入了这个机制。一来可以避免暴露应用内部的文件系统信息，二来不要求文件数据接收方拥有 `READ_EXTERNAL_STORAGE` 权限。出发点是好的，在实现上略有瑕疵，总体感觉还不错，建议使用。

[file-provider-doc]: https://developer.android.com/reference/android/support/v4/content/FileProvider