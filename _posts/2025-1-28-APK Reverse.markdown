---
title: "Android APK Analysis: A Beginner's Guide"
layout: post
date: 2025-01-28
description: "Learn how to analyze Android APK files, explore their structure, and understand essential components like Smali, AndroidManifest.xml, and more."
categories: [Android, Reverse Engineering]
tags: [APK, Smali, Reverse Engineering, Android]
---

## APK File **Structure**

APK files are the package file format used by the Android operating system for distributing and installing mobile applications. They are essentially ZIP files containing all the resources needed for the application to run.

To explore the contents of an APK file, simply rename the file extension from `.apk` to `.zip`, then unzip it. Upon unzipping, you'll see the following structure:

```bash
AndroidManifest.xml
META-INF/
assets/
resources.arsc
lib/
classes.dex
res/
```



> All of these files are in binary format and not human-readable, but we can convert them to readable formats using specific tools.
{:.prompt-tip}

### Key Files

- **`AndroidManifest.xml`**: The manifest file in binary XML format.
- **`classes.dex`**: The application code compiled in DEX format.
- **`resources.arsc`**: Precompiled application resources in binary XML.
- **`res/`**: Contains resources not compiled into `resources.arsc`.
- **`assets/`**: Optional folder containing application assets retrievable by `AssetManager`.
- **`lib/`**: Optional folder for compiled native code libraries.
- **`META-INF/`**: Contains metadata, including the `MANIFEST.MF` file and APK signatures.


## Smali

Smali is the assembly language for the Dalvik Virtual Machine (DVM) used in Android. It provides a human-readable version of DEX files, which can be partially reversed to reconstruct the original source code.

![](https://1.bp.blogspot.com/-JhditcLrkBc/XbhSEl1lEEI/AAAAAAAAAN4/8JAua7PwxUg_unZ3rN8C3QfXUgjJ83miACLcBGAsYHQ/s1600/Screenshot_2.png)

### Baksmaling

Baksmaling is the process of converting the DEX file to Smali


## How to get APK files

There are several ways to obtain APK files:
1. Download from the Google Play Store, then pull apk from Android device.
2. Download from third-party app stores. such as (Apk-Mirror, Apkpure)

> Be cautious when downloading APKs from third-party sources, as they may contain malware or be tampered with.
> In fact this is why I suggest using the Google Play Store.
{:.prompt-warning}

### How to pull apk using adb  ?


I wrote a bash script to pull the apk from the device/emulator using adb.
you can find the script [Here](https://gist.github.com/pwnxpl0it/62ed6fa634db5a4c3f058a21853bf390)

Just choose package name and it will store the apk into `apks/PKGNAME.apk`

<script src="https://gist.github.com/pwnxpl0it/62ed6fa634db5a4c3f058a21853bf390.js"></script>

I recommend using this method for security reasons, as it's safer than downloading from third-party sources.

## Decompiling an APK File

To analyze an APK, you need to decompile it into a readable format. Here are some tools to get started:

### Tools


1. **`apktool`** (For decoding resources and rebuilding APKs)
2. **`jadx`** (For decompiling Java bytecode to readable source code)
3. **`Smali`/`Baksmali`** (For working with DEX bytecode)

#### APKtool

```bash
# Decompile an APK using apktool
apktool d app.apk
```

Apktool will decode the APK file into a folder with the same name as the source APK file, containing the decompiled resources and manifest file, also the smali code.


> This is very useful for analyzing the resources and structure of the application, especially when you don't have the source code, such as during "black-box" security assessments.
{:.prompt-tip}


This tool extracts Smali code only. To retrieve the original source code, we need additional tools.


#### Using `jadx`

`jadx` is a command-line tool for decompiling Android applications and exploring their source code.



```
jadx -d [path-output-folder] [path-apk-or-dex-file]

```

#### Using `jadx-gui`

`jadx-gui` is a graphical tool for decompiling APK files and exploring their structure.

**Steps:**

1. Open `jadx-gui`.
2. Load the APK file.
3. Browse the decompiled Java classes and XML resources.

![](https://user-images.githubusercontent.com/118523/142730720-839f017e-38db-423e-b53f-39f5f0a0316f.png)

#### Dex2Jar

Another method is to use [Dex2Jar](https://github.com/pxb1988/dex2jar)

To Convert an APK to a JAR file, then you can view it in a Java decompiler.

```
d2j-dex2jar.sh /path/application.apk
```

Once you have the JAR file, simply open it with JD-GUI and you’ll see its Java code.

## AndroidManifest.xml

AndroidManifest.xml is the configuration file for the application, it contains all the information about the application like:

| Tag              | Description                                                                                                                                                                          |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Manifest tag     | contains android installation mode, package name, build versions                                                                                                                     |
| Permissions      | custom permission and protection level                                                                                                                                               |
| uses-permissions | requests a permission that must be granted in order for it to operate, full list of permission api can refer here.                                                                   |
| uses-feature     | Declares a single hardware or software feature that is used by the application.                                                                                                      |
| Application      | The declaration of the application. Will contains all the activity                                                                                                                   |
| Activity         | Declares an activity that implements part of the application visual user interface.                                                                                                  |
| intent-filter    | Specifies the types of intents that an activity, service, or broadcast receiver can respond to.                                                                                      |
| service          | Declare a service as one of the application components.                                                                                                                              |
| receiver         | Broadcast receivers enable applications to receive intents that are broadcast by the system or by other applications, even when other components of the application are not running. |
| provider         | Declares a content provider component. A content provider is a subclass of ContentProvider that supplies structured access to data managed by the application.                       |


## META-INF

Contains certificates needed for every application to check if it has been tampered with or not (Integrity check)

### Certificate Signing Process

![](https://connectivity-staging.s3.us-east-2.amazonaws.com/boundary/2020/11/800px-Digital_Signature_diagram_acdx_ccimages.png)

When an APK is signed:
1. The APK file's hash is generated.
2. Public and private key pairs are created.
3. The private key encrypts the hash.
4. The public key decrypts the hash and compares it to a newly generated hash to verify integrity.

If the hashes don't match, Android will refuse to install the APK. However, custom certificates can bypass this.

## The `lib` folder

This folder contains shared object (`.so`) libraries compiled from native code (C/C++). These libraries are architecture-specific (e.g., `arm`, `x86`, `x86_64`) and are equivalent to Windows DLL files. They can be reverse-engineered, though the process is more complex.


## Components of Android Applications

In this section, I will provide a brief overview of each component without going into too much detail.

- Activity
- Broadcast Receiver
- Deep Link
- Content Provider
- WebView
- Services

### Intents

An intent is a message passed between application components, often used to request actions from other components. Exported intents can be accessed by other applications.

> The `intent-filter` defines the actions the activity will respond to.
{:.prompt-tip}

### Activity

An activity represents a UI screen. The first activity launched is called the `Main Activity`, and it is defined in the `AndroidManifest.xml` file as follows:

```xml
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

### Broadcast Receiver

A Broadcast Receiver is a component that listens for system-wide broadcast announcements.

It can be used to listen for system events like:
- `Battery Low`
- `SMS Received`
- `Phone Booted`

It can also be used to send broadcasts to other components of the application.

Example of a Broadcast Receiver in the `AndroidManifest.xml` file:

```xml
<receiver android:name=".MyBroadcastReceiver">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```

In the above example, the `MyBroadcastReceiver` will listen for the `BOOT_COMPLETED` event and then execute some action.

### Deep Link

Deep Linking allows external sources to open specific pages within the app directly. It is useful for opening the application from a web link or another application.

Deep links are defined in the `AndroidManifest.xml` file:

```xml
<activity android:name=".DeepLinkActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="http" />
        <data android:host="example.com" />
        <data android:pathPrefix="/gizli" />
    </intent-filter>
</activity>
```

In this example, the `DeepLinkActivity` will be opened when the user clicks a link that starts with `http://example.com/gizli`.

Example: `whatsapp://send?text=Hello%20World`

This is how the Deep Link would be defined in the `AndroidManifest.xml` file:

```xml
<activity android:name=".DeepLinkActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data android:scheme="whatsapp" />
        <data android:host="send" />
    </intent-filter>
</activity>
```

### Content Provider

Content Providers are used to share data between applications. They allow applications to store and retrieve data from a central repository.

Example of defining a Content Provider:

```xml
<provider
    android:name=".MyContentProvider"
    android:authorities="com.example.mycontentprovider"
    android:exported="true" />
</provider>
```

To access the Content Provider from another application:

```java
Cursor cursor = getContentResolver().query(
    Uri.parse("content://com.example.mycontentprovider/data"),
    null, null, null, null);
```

### WebView

A WebView is used to display web content within an application. It is defined in the layout file and configured in Java/Kotlin code.

Example:

```java
WebView webView = findViewById(R.id.webview);
webView.loadUrl("https://example.com");
```

### Services

Services are used to perform long-running operations in the background without user interaction. They are useful for:

- Playing music in the background
- Downloading files
- Running network operations

Example of defining a Service in `AndroidManifest.xml`:

```xml
<service android:name=".MyService" />
```

Example of a basic Service implementation:

```java
public class MyService extends Service {
    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        // Perform background task
        return START_STICKY;
    }
}
```

### Summary Table

| Component          | Purpose                                             | Example Use Case                        |
|-------------------|-------------------------------------------------|--------------------------------|
| Activity          | Represents a screen UI                         | Login screen, Home screen     |
| Broadcast Receiver | Listens for system-wide events                  | Detecting SMS received        |
| Deep Link        | Opens app pages via external links               | Opening an app from a website |
| Content Provider | Shares data between applications                 | Sharing contacts or files     |
| WebView         | Displays web content within an app               | Embedding a webpage          |
| Services        | Runs background tasks without UI interaction     | Playing music, fetching data  |

