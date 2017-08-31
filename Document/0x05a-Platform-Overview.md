# Android Platform Overview

This section introduces Android platform from the architectural point of view. The following four key areas are discussed:

1. Android security architecture
2. Android application structure
3. Inter-process Communication (IPC)
4. Android application publishing

Visit the official [Android developer documentation website](https://developer.android.com/index.html "Android Developer Guide") for more details on the Android platform.

## Android Security Architecture

Android is a Linux-based open-source platform initially developed by Google as a mobile operating system (OS) solution. Today, the platform is a foundation for a wide variety of modern technology such as mobile phones and tablets, wearable tech, and other "smart" devices like TVs. Typical Android builds ship with a range of pre-installed ("stock") apps and support installation of third-party apps through Google Play store and other marketplaces.

The software stack of Android is composed of several different layers. Each layer defines a number of interfaces and offers specific services. 

![Android Software Stack](Images/Chapters/0x05a/android_software_stack.png)

At the lowest level, Android utilizes a variation of Linux Kernel which serves as the foundation for other elements composing the OS. On top the kernel, the Hardware Abstraction Layer (HAL) defines a standard interface for interacting with hardware components built into device. Several HAL implementations are packaged into shared library modules called on by the Android system when required. This is basis for allowing applications to interact with the device’s internal hardware – for example, it grants a stock phone application the ability to use the microphone and speaker of a device.

Android apps are usually written in Java and compiled to Dalvik bytecode, which is somewhat (but not very) different from the traditional Java bytecode. Dalvik bytecode is created by first compiling the Java code to .class files, and then converting the JVM bytecode to Dalvik .dex format using the `dx` tool.

![Java vs Dalvik](Images/Chapters/0x05a/java_vs_dalvik.png)

In current version of Android, this bytecode is executed on the Android runtime (ART). ART is the successor to Android's original runtime, the Dalvik Virtual Machine. The key difference between Dalvik an ART lies in the way the bytecode is executed. 

In Dalvik, bytecode is translated into machine code at execution time, a technique known as *Just In Time* (JIT) compilation. The drawback of JIT compilation is its adverse effect on performance: The compilation step must be performed every time the app is executed. To improve performance, ART introduced *Ahead Of Time* (AOT) compilation. As the name implies, with AOT apps are pre-compiled to machine code once before being executed for the first time. In subsequent runs, the already compiled machine code is executed. This improves performance by a factor of two, while also reducing power consumption.

Android apps do not have direct access to hardware resources, and each app runs in their own sandbox. This allows fine-grained control over resources and apps: for instance, when an app crashes this does impact other apps running on the device. At the same time, the Android runtime controls the maximum amount of system resources allocated to apps, preventing any one app from locking up too many resources.

#### Android Users and Groups

Even though the Android operating system is based on Linux, it does not utilize user accounts in the same way other Unix-like systems do. For instance, it does not have a _/etc/passwd_ file containing the list of users in the system. Instead, Android utilizes the multi-user support of Linux kernel to achieve app sandboxing: With a few exceptions, each app runs as under a separate Linux user, effectively isolating apps from each other and from the rest of the operating system.

The file [system/core/include/private/android_filesystem_config.h](http://androidxref.com/7.1.1_r6/xref/system/core/include/private/android_filesystem_config.h) includes a list of the predefined users and groups used used by system processes. UIDs (userIDs) for other applications are added as they are installed on the system. For more details, check out Bin Chen's [blog post](https://pierrchen.blogspot.mk/2016/09/an-walk-through-of-android-uidgid-based.html "Bin Chen - AProgrammer Blog - Android Security: An Overview Of Application Sandbox") on Android sandboxing.

For example, Android Nougat defines the following system users:

```
    #define AID_ROOT             0  /* traditional unix root user */

    #define AID_SYSTEM        1000  /* system server */
	...
    #define AID_SHELL         2000  /* adb and debug shell user */
	...
    #define AID_APP          10000  /* first app user */
	...
```
## Android Application Structure

### Communication with the Operating System

Android apps interact with system services using the Android Framework, an abstraction layer that offers high-level Java APIs. The majority of these services are invoked via normal Java method calls, and are translated to IPC calls to system services running in the background. Examples of system services include:

    * Connectivity (Wifi, Bluetooth, NFC, ...)
    * Giles
    * Cameras
    * Geolocation (GPS)
    * Microphone

The framework also offers common security functions such as cryptography.

The API specifications change with every new release of Android. Critical bug fixes and security patches are usually propagated to earlier versions as well. The oldest Android version supported at the time of writing this guide, is 4.4 (KitKat), API level 19, while the current version of Android is 7.1 (Nougat), API level 25.

Noteworthy API versions are:

- Android 4.2 Jelly Bean (API 16) in November 2012 (introduction of SELinux);
- Android 4.3 Jelly Bean (API 18) in July 2013 (SELinux becomes enabled by default);
- Android 4.4 KitKat (API 19) in October 2013 (several new APIs and ART is introduced);
- Android 5.0 Lollipop (API 21) in November 2014 (ART by default and many other new features);
- Android 6.0 Marshmallow (API 23) in October 2015 (many new features and improvements, including granting; fine-grained permissions at run time and not all or nothing at installation time);
- Android 7.0 Nougat (API 24-25) in August 2016 (new JIT compiler on ART);
- Android 8.0 O (API 26) beta (major security fixes expected).

#### App Folder Structure

Android apps installed from the Google Play Store or other sources are located at `/data/app/[package-name]`. For example, the Youtube app is found at:

```bash
/data/app/com.google.android.youtube-1/base.apk
```

The Android Package Kit (APK) file is an archive that contains the code and resources needed to run the app. This file is identical to the original, signed app package created by the developer. It is in fact a ZIP archive with the following directory structure:

```
$ unzip base.apk
$ ls -lah
-rw-r--r--   1 sven  staff    11K Dec  5 14:45 AndroidManifest.xml
drwxr-xr-x   5 sven  staff   170B Dec  5 16:18 META-INF
drwxr-xr-x   6 sven  staff   204B Dec  5 16:17 assets
-rw-r--r--   1 sven  staff   3.5M Dec  5 14:41 classes.dex
drwxr-xr-x   3 sven  staff   102B Dec  5 16:18 lib
drwxr-xr-x  27 sven  staff   918B Dec  5 16:17 res
-rw-r--r--   1 sven  staff   241K Dec  5 14:45 resources.arsc
```

- AndroidManifest.xml: Contains the definition of the app’s package name, target and min API version, app configuration, components, user-granted permissions, etc.
- META-INF: contains the app’s metadata.
  - MANIFEST.MF: stores hashes of the app resources.
  - CERT.RSA: the app’s certificate(s).
  - CERT.SF: list of the resources and SHA-1 digest of the corresponding lines in the MANIFEST.MF file.
- assets: directory containing app assets (files used within the Android App like XML, Java Script or pictures) which can be retrieved by the AssetManager.
- classes.dex: classes compiled in the DEX file format understandable by Dalvik virtual machine/Android Runtime. DEX is Java Byte Code for Dalvik Virtual Machine. It is optimized for running on small devices.
- lib: directory containing libraries that are part of the APK, for example the 3rd party libraries that are not part of the Android SDK.
- res: directory containing resources not compiled into resources.arsc.
- resources.arsc: file containing precompiled resources, such as XML files for the layout.

Some resources inside the APK are compressed using non-standard algorithms (e.g. the AndroidManifest.xml). This means that simply unzipping the file won't reveal all information. A better way is to use the tool 'apktool' to unpack and uncompress the files. The following is a list of the files contained in the apk:

```bash
$ apktool d base.apk
I: Using Apktool 2.1.0 on base.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /Users/sven/Library/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
$ cd base
$ ls -alh
total 32
drwxr-xr-x    9 sven  staff   306B Dec  5 16:29 .
drwxr-xr-x    5 sven  staff   170B Dec  5 16:29 ..
-rw-r--r--    1 sven  staff    10K Dec  5 16:29 AndroidManifest.xml
-rw-r--r--    1 sven  staff   401B Dec  5 16:29 apktool.yml
drwxr-xr-x    6 sven  staff   204B Dec  5 16:29 assets
drwxr-xr-x    3 sven  staff   102B Dec  5 16:29 lib
drwxr-xr-x    4 sven  staff   136B Dec  5 16:29 original
drwxr-xr-x  131 sven  staff   4.3K Dec  5 16:29 res
drwxr-xr-x    9 sven  staff   306B Dec  5 16:29 smali
```

- AndroidManifest.xml: file is not compressed anymore and can be opened in a text editor.
- apktool.yml : file contains information about the output of apktool.
- assets: directory containing app assets (files used within the Android App like XML, Java Script or pictures) which can be retrieved by the AssetManager.
- lib: directory containing libraries that are part of the APK, for example the 3rd party libraries that are not part of the Android SDK.
- original: folder contains the MANIFEST.MF file which stores meta data about the contents of the JAR and signature of the APK. The folder is also named as META-INF.
- res: directory containing resources not compiled into resources.arsc.
- smali: directory containing the disassembled Dalvik bytecode in Smali. Smali is a human readable representation of the Dalvik executable. 

Ever app also has a data directory for storing data created during runtime. This directory is found at `/data/data/[package-name]` and has the following structure:

```bash
drwxrwx--x u0_a65   u0_a65            2016-01-06 03:26 cache
drwx------ u0_a65   u0_a65            2016-01-06 03:26 code_cache
drwxrwx--x u0_a65   u0_a65            2016-01-06 03:31 databases
drwxrwx--x u0_a65   u0_a65            2016-01-10 09:44 files
drwxr-xr-x system   system            2016-01-06 03:26 lib
drwxrwx--x u0_a65   u0_a65            2016-01-10 09:44 shared_prefs
```

Apps require no extra permissions to read or write to the returned path, since this path lives in their private storage.

- **cache**: This location is used to cache runtime data, such as WebView caches.
- **code_cache**: The location of the application specific cache directory on the file system designed for storing cached code. On devices running Lollipop or later Android versions, the system will delete any files stored in this location both when your specific application is upgraded, and when the entire platform is upgraded. This location is optimal for storing compiled or optimized code generated by your application at runtime.
- **databases**: This folder stores sqlite database files generated by the app at runtime, e.g. to store user data.
- **files**: This folder is used to store files that are created in the App when using the internal storage.
- **lib**: This folder used to store native libraries written in C/C++. These libraries can have file extension as .so, .dll (x86 support). The folder contains subfolders for the platforms the app has native libraries for:
   * armeabi: Compiled code for all ARM based processors only
   * armeabi-v7a: Compiled code for all ARMv7 and above based processors only
   * arm64-v8a: Compiled code for all ARMv8 arm64 and above based processors only
   * x86: Compiled code for x86 processors only
   * x86_64: Compiled code for x86_64 processors only
   * mips: Compiled code for MIPS processors only
- **shared_prefs**: This folder is used to store the preference file generated by an app at runtime to save current state of the app including data, configuration, session, etc. The file format is XML.


#### Linux UID/GID of Normal Applications

Android leverages Linux user management to isolate apps from each other. This approach is different from how user management is used in traditional Linux environments, where multiple apps are often run by the same user. Android creates a unique user ID (UID) for each Android app, and runs the app as that user in a separate process. Consequently, any one app can only access its own resources. This protection is enforced by the Linux kernel.

Generally apps are assigned UIDs in the range of 10000 (AID_APP) and 99999. Android apps receive a user name based on their UID. For example, the apps with UID 10188 receive the user name u0_a188. If an app requested some permissions and they are granted, the corresponding group ID is added to the process of the app. For example, the user ID of the app below is 10188. It belongs to the group ID 3003 (inet). That is the group related to android.permission.INTERNET permission. The result of the id command is shown below:
```
$ id
uid=10188(u0_a188) gid=10188(u0_a188) groups=10188(u0_a188),3003(inet),9997(everybody),50188(all_a188) context=u:r:untrusted_app:s0:c512,c768
```

The relationship between group IDs and permissions are defined in the file [frameworks/base/data/etc/platform.xml](http://androidxref.com/7.1.1_r6/xref/frameworks/base/data/etc/platform.xml)
```
<permission name="android.permission.INTERNET" >
	<group gid="inet" />
</permission>

<permission name="android.permission.READ_LOGS" >
	<group gid="log" />
</permission>

<permission name="android.permission.WRITE_MEDIA_STORAGE" >
	<group gid="media_rw" />
	<group gid="sdcard_rw" />
</permission>
```

#### The App Sandbox

Apps are executed in the Android Application Sandbox enforcing isolation of the app data and the code execution from other apps on the device. This adds an additional layer of security.

When installing a new app, a new directory named after the app package - `/data/data/[package-name]` - is created. This is directory is used to hold the data of that particular app. Linux directory permissions are set such that the directory can be read and written only by the app's unique UID. 

![Sandbox](Images/Chapters/0x05a/Selection_003.png)

We can confirm this by looking at the file system permissions in the `/data/data` folder. For example, we can see that Google Chrome and Calendar are assigned one directory each and run under different used accounts:

```
drwx------  4 u0_a97              u0_a97              4096 2017-01-18 14:27 com.android.calendar
drwx------  6 u0_a120             u0_a120             4096 2017-01-19 12:54 com.android.chrome
```

Sandboxing can be sidestepped by developer that want their own apps to share a common sandbox. When two apps are signed with the same certificate and explicitly share the same user ID (by including the _sharedUserId_ in their _AndroidManifest.xml_) they can access each other’s data directory. See the following example on how this is achieved in the Nfc app:

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
  package="com.android.nfc"
  android:sharedUserId="android.uid.nfc">
```

##### Zygote

The process called `Zygote` starts up during the [Android initialization process](https://github.com/dogriffiths/HeadFirstAndroid/wiki/How-Android-Apps-are-Built-and-Run "How Android Apps are run"). Zygote is a system service used to launch apps. It opens up a socket in /dev/socket/zygote and listens on it for requests to start new applications. Zygote process is a "base" process that contains all the core libraries that are needed by any app. When Zygote receives a request over its listening socket, it forks a new process which then loads and executes the app-specific code.

##### App Lifecycle

In Android, the lifetime of an app process is controlled by the operating system. A new app process is created when code of the app needs to be run. Android may decide to kill the process when it decides that it is no longer needed, or when it needs to reclaim memory for running other, more important apps. The decision whether a process should be killed is mainly related to the state of the user's interaction with it. In general, there are four states a process can be in:

- A foreground process (e.g running an Activity at the top of the screen or has a running BroadcastReceiver that is currently running (its BroadcastReceiver.onReceive() method is executing).

- A visible process is doing work that the user is currently aware of, so killing it would have a noticeable negative impact on the user experience. 
E.g. is running an Activity that is visible to the user on-screen but not in the foreground. 

- A service process is one holding a Service th`at has been started with the startService() method. Though these processes are not directly visible to the user, they are generally doing things that the user cares about (such as background network data upload or download), so the system will always keep such processes running unless there is not enough memory to retain all foreground and visible processes.

- A cached process is one that is not currently needed, so the system is free to kill it as desired when memory is needed elsewhere. 

Apps must implement event handlers that react to a number of events: for example, the `onCreate` handler is called when the app process is first created. Other callback functions include `onLowMemory`, `onTrimMemory` and `onConfigurationChanged`.

##### Manifest

Every app has a manifest file which embeds content in binary XML format. The name of this file is standardized as AndroidManifest.xml and is the same for every app. It is located in the root tree of the .apk file in which the app is published.

The manifest file describes the app structure as well as its exposed components (activities, services, content providers and intent receivers) and requested permissions. It also contains contain general metadata about the app, like its icon, version number and the theme it uses for the User Experience (UX). It may list other information like the APIs it is compatible with (minimal, targeted and maximal SDK version) and the [kind of storage it can be installed on (external or internal)](https://developer.android.com/guide/topics/data/install-location.html "Define app install location")

Here is an example of a manifest file, including the package name (the convention is to use a url in reverse order, but any string can be used). It also lists the app version, relevant SDKs, required permissions, exposed content providers, used broadcast receivers with intent filters as well as a description of the app and its activities:
```
<manifest
	package="com.owasp.myapplication"
	android:versionCode="0.1" >

	<uses-sdk android:minSdkVersion="12"
		android:targetSdkVersion="22"
		android:maxSdkVersion="25" />

	<uses-permission android:name="android.permission.INTERNET" />

	<provider
		android:name="com.owasp.myapplication.myProvider"
		android:exported="false" />

	<receiver android:name=".myReceiver" >
		<intent-filter>
			<action android:name="com.owasp.myapplication.myaction" />
		</intent-filter>
	</receiver>

	<application
		android:icon="@drawable/ic_launcher"
		android:label="@string/app_name"
		android:theme="@style/Theme.Material.Light" >
		<activity
			android:name="com.owasp.myapplication.MainActivity" >
			<intent-filter>
				<action android:name="android.intent.action.MAIN" />
			</intent-filter>
		</activity>
	</application>
</manifest>
```

The full list of options available in the manifest is listed in the official [Android Manifest file documentation](https://developer.android.com/guide/topics/manifest/manifest-intro.html "Android Developer Guide for Manifest").

#### App Components

Android apps are made of several high-level components. The main components are:

- Activities
- Fragments
- Intents
- Broadcast receivers
- Content providers and services

All these elements are provided by the Android operating system in the form of predefined classes available through APIs.

##### Activities

Activities make up the visible part of any app. One activity exists per screen (e.g. user interface) so an app with three different screens is implementing three different activities. Activities are declared by extending the Activity class. They contain all user interface elements: fragments, views and layouts.

Each activity needs to be declared in the app manifest with the following syntax:

```
<activity android:name="ActivityName">
</activity>
```

Activities not declared in the manifest cannot be displayed, and attempting to launch them would raise an exception.

In the same way as apps do, activities also have their own lifecycle and need to listen to the system changes in order to handle them accordingly. Activities can have the following states: active, paused, stopped and inactive. These states are managed by Android operating system. Accordingly, activities can implement the following event managers:

- onCreate
- onSaveInstanceState
- onStart
- onResume
- onRestoreInstanceState
- onPause
- onStop
- onRestart
- onDestroy

An app may not explicitly implement all event managers in which case default actions are taken. Typically, at least the `onCreate` manager is overridden by the app developers. This is the place where most user interface components are declared and initialized. `onDestroy` may be overridden as well when resources need to be explicitly released (like network connections or connections to databases) or if specific actions need to take place at the end of the app.

##### Fragments

A fragment represents a behavior or a portion of user interface within the Activity. Fragments have been introduced in Android with version Honeycomb 3.0 (API level 11).

Fragments are meant to encapsulate parts of the interface to make re-usability easier and better adapt to different size of screens. Fragments are autonomous entities in a way they embed everything they need to work in themselves (they have their own layout, own buttons etc.). However, they must be integrated in activities to become useful: fragments cannot exist on their own. They have their own life cycle, which is tied to the one of the activity that implements them.

As they have their own life cycle the Fragment class contains event managers, that can be redefined or extended. Such event managers can be onAttach, onCreate, onStart, onDestroy and onDetach. Several others exist; the reader should refer to the [Android Fragment specification](https://developer.android.com/reference/android/app/Fragment.html "Fragment Class") for more details.

Fragments can be implemented easily by extending the Fragment class provided by Android:

```Java
public class myFragment extends Fragment {
	...
}
```

Fragments don't need to be declared in manifest files as they depend on activities.

In order to manage its fragments, an Activity can use a Fragment Manager (FragmentManager class). This class makes it easy to find, add, remove and replace associated fragments.

Fragment Managers can be created simply with the following:

```Java
FragmentManager fm = getFragmentManager();
```

Fragments do not necessarily have a user interface: they can be a convenient and efficient way to manage background operations dealing with the user interface in an app. For instance when a fragment is declared as persistent while its parent activity may be destroyed and created again.

##### Inter-Process Communication

As we already learned, every process on Android has its own sandboxed address space. Inter-process communication (IPC) facilities enable apps to exchange signals and data in a (hopefully) secure way. Instead of relying on the default Linux IPC facilities, IPC on Android is done through Binder, a custom implementation of OpenBinder. Most Android system services, as well as all high-level IPC services, depend on Binder.

The term *Binder* stands for a lot of different things, including:

- Binder Driver - The kernel-level driver
- Binder Protocol - Low-level ioctl-based protocol used to communicate with the binder driver
- IBinder Interface - well-defined behavior Binder objects implement
- Binder object - Generic implementation of the IBinder interface
- Binder service - Implementation of the Binder object. For example, location service, sensor service,...
- Binder client - An object using the binder service

In the Binder framework, a client-server communication model is used. To use IPC functionality, apps call IPC methods in proxy objects. The proxy object transparently *marshalls* the call parameters into a *parcel* and sends a transaction to the Binder server, which is implemented as a character driver (/dev/binder). The server holds a thread pool for handling incoming requests, and is responsible for delivering messages to the destination object. From the view of the client app, all of this looks like a regular method call - all the heavy lifting is done by the binder framework.

![Binder Overview](Images/Chapters/0x05a/binder.jpg)
*Binder Overview. Image source: [Android Binder by Thorsten Schreiber](https://www.nds.rub.de/media/attachments/files/2011/10/main.pdf)*

Services that allow other applications to bind to them are called *bound services*. These services must provide an IBinder interface for use by clients. Developers write interfaces for remote services using the Android Interface Descriptor Language (AIDL).

Servicemanager is a system daemon that manages the registration and lookup of system services. It maintains a list of name/Binder pairs for all registered services. Services are added using the `addService` and retrieved by name using the `getService` static method in `android.os.ServiceManager`:

```
  public static IBinder getService(String name)
```

The list of system services can be queried using the `service list` command.


```
$ adb shell service list
Found 99 services:
0 carrier_config: [com.android.internal.telephony.ICarrierConfigLoader]
1 phone: [com.android.internal.telephony.ITelephony]
2 isms: [com.android.internal.telephony.ISms]
3 iphonesubinfo: [com.android.internal.telephony.IPhoneSubInfo]
```

#### Intents

*Intent messaging* is a framework for asynchronous communication built on top of binder. This framework enables both point-to-point and publish-subscribe messaging. An *Intent* is a messaging object that can be used to request an action from another app component. Although intents facilitate communication between components in several ways, there are three fundamental use cases:

- Starting an activity
    - An Activity represents a single screen in an app. You can start a new instance of an Activity by passing an Intent to startActivity(). The Intent describes the activity to start and carries any necessary data.
- Starting a Service
    - A Service is a component that performs operations in the background without a user interface. With Android 5.0 (API level 21) and later, you can start a service with JobScheduler.
- Delivering a broadcast
    - A broadcast is a message that any app can receive. The system delivers various broadcasts for system events, such as when the system boots up or the device starts charging. You can deliver a broadcast to other apps by passing an Intent to sendBroadcast() or sendOrderedBroadcast().

Intents are messaging components used between apps and components. They can be used by an app to send information to its own components (for instance, start a new activity inside the app) or to other apps, and may be received from other apps or from the operating system. Intents can be used to start activities or services, run an action on a given set of data, or broadcast a message to the whole system. They are a convenient way to decouple components.

There are two types of Intents. Explicit intents specify the component to start by name (the fully-qualified class name). For instance:

```Java
    Intent intent = new Intent(this, myActivity.myClass);
```

Implicit intents are sent to the system with a given action to perform on a given set of data ("http://www.example.com" in our example below). It is up to the system to decide which app or class will perform the corresponding service. For instance:

```Java
    Intent intent = new Intent(Intent.MY_ACTION, Uri.parse("http://www.example.com"));
```

An *intent filter* is an expression in an app's manifest file that specifies the type of intents that the component would like to receive. For instance, by declaring an intent filter for an activity, you make it possible for other apps to directly start your activity with a certain kind of intent. Likewise, if you do not declare any intent filters for an activity, then it can be started only with an explicit intent.

Android uses intents to broadcast messages to apps, like an incoming call or SMS, important information on power supply (low battery for example) or network changes (loss of connection for instance). Extra data may be added to intents (through putExtra / getExtras).

Here is a short list of intents from the operating system. All constants are defined in the Intent class, and the whole list can be found in the official Android documentation:

- ACTION_CAMERA_BUTTON
- ACTION_MEDIA_EJECT
- ACTION_NEW_OUTGOING_CALL
- ACTION_TIMEZONE_CHANGED

In order to improve security and privacy, a Local Broadcast Manager is used to send and receive intents within an app, without having them sent to the rest of the operating system. This is very useful to guarantee sensitive or private data do not leave the app perimeter (geolocation data for instance).

##### Broadcast Receivers

Broadcast Receivers are components that allow to receive notifications sent from other apps and from the system itself. This way, apps can react to events (either internal, from other apps or from the operating system). They are generally used to update a user interface, start services, update content or create user notifications.

Broadcast Receivers need to be declared in the Manifest file of the app. Any Broadcast Receiver must be associated to an intent filter in the manifest to specify which actions it is meant to listen with which kind of data. If they are not declared, the app will not listen to broadcasted messages. However, apps do not need to be started to receive intents: they are automatically started by the system when a relevant intent is raised.

An example of declaring a Broadcast Receiver with an Intent Filter in a manifest is:
```
	<receiver android:name=".myReceiver" >
		<intent-filter>
			<action android:name="com.owasp.myapplication.MY_ACTION" />
		</intent-filter>
	</receiver>
```

When receiving an implicit intent, Android will list all apps that have registered a given action in their filters. If more than one app is matching, then Android will list all those apps and will require the user to make a selection.

An interesting feature concerning Broadcast Receivers is that they be assigned a priority; this way, an intent will be delivered to all receivers authorized to get them according to their priority.

A Local Broadcast Manager can be used to make sure intents are received only from the internal app, and that any intent from any other app will be discarded. This is very useful to improve security.

##### Content Providers

Android is using SQLite to store data permanently: as it is in Linux, data is stored in files. SQLite is an open-source, light and efficient technology for relational data storage that does not require much processing power, making it ideal for use in the mobile world. An entire API is available to the developer with specific classes (Cursor, ContentValues, SQLiteOpenHelper, ContentProvider, ContentResolver, ...).
SQLite is not run in a separate process from a given app, but it is part of it.
By default, a database belonging to a given app is only accessible to this app. However, Content Providers offer a great mechanism to abstract data sources (including databases, but also flat files) for easier use in an app; they also provide a standard and efficient mechanism to share data between apps, including native ones. In order to be accessible to other apps, content providers need to be explicitly declared in the Manifest file of the app that will share it. As long as Content Providers are not declared, they are not exported and can only be called by the app that creates them.

Content Providers are implemented through a URI addressing scheme: they all use the content:// model. Whatever the nature of sources is (SQLite database, flat file, ...), the addressing scheme is always the same, abstracting what sources are and offering a unique scheme to the developer. Content providers offer all regular operations on databases: create, read, update, delete. That means that any app with proper rights in its manifest file can manipulate the data from other apps.

##### Services

Services are components provided by Android operating system (in the form of the Service class) that will perform tasks in the background (data processing, start intents and notifications, ...) without presenting any kind of user interface. Services are meant to run processing on the long term. Their system priorities are lower than the ones active apps have, but are higher than inactive ones. As such, they are less likely to be killed when the system needs resources; they can also be configured to start again automatically when enough resources become available in case they get killed. Activities are executed in the main app thread. They are great candidates to run asynchronous tasks.

##### Permissions

Because any Android app is installed in a sandbox and initially it does not have access to either user information or access to the system components (such as using the camera or the microphone), it provides a system based on permissions where the system has a predefined set of permissions for certain tasks that the app can request.
As an example, if you want your app to use the camera on the phone you have to request the camera permission.
On Android versions before Marshmallow (API 23) all permissions requested by an app were granted at installation time. From Android Marshmallow onwards the user have to approve some permissions during app execution.

###### Protection Levels

Android permissions are ranked in four different categories based on the protection level they offer.
- *Normal*: the lower level of protection. It gives the apps access to isolated application-level feature, with minimal risk to other apps, the user or the system. It is granted during the installation of the App. It is also the default value for the cases when no protection level is specified.
Example: `android.permission.INTERNET`
- *Dangerous*: This permission usually gives the app control over the user data or over the device that impacts the user. This type of permission may not be granted at installation time, leaving it to the user to decide whether the app should have the permission or not.
Example: `android.permission.RECORD_AUDIO`
- *Signature*: This permission is granted only if the requesting app was signed with the same certificate as the app that declared the permission. If the signature matches, the permission is automatically granted.
Example: `android.permission.ACCESS_MOCK_LOCATION`
- *SystemOrSignature*: Permission only granted to the apps embedded in the system image or that were signed using the same certificated as the app that declared the permission.
Example: `android.permission.ACCESS_DOWNLOAD_MANAGER`

###### Requesting Permissions

Apps can request permissions of protection level Normal, Dangerous and Signature by inserting the XML tag `<uses-permission />` to its Android Manifest file.

The example below shows an AndroidManifest.xml sample requesting permission to read SMS messages:
```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.permissions.sample" ...>

    <uses-permission android:name="android.permission.RECEIVE_SMS" />
    <application>...</application>
</manifest>
```
This will enable the app to read SMS messages at install time (before Android Marshmallow - 23) or will enable the app to ask the user to allow the permission at runtime (Android M onwards).

###### Declaring Permissions

Any app is able to expose its features or content to other apps installed on the system. It can expose the information openly or restrict it some apps by declaring a permission.
The example below shows an app declaring a permission of protection level *signature*.
```
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.permissions.sample" ...>

    <permission
    android:name="com.permissions.sample.ACCESS_USER_INFO"
    android:protectionLevel="signature" />
    <application>...</application>
</manifest>
```
Only apps signed with the same developer certificate can use this permission.

###### Enforcing Permissions on Android Components

It is possible to protect Android components using permissions. Activities, Services, Content Providers and Broadcast Receivers – all can use the permission mechanism to protect their interfaces.
*Activities*, *Services* and *Broadcast Receivers* can enforce permissions by entering the attribute *android:permission* inside each tag in AndroidManifest.xml:
```
<receiver
    android:name="com.permissions.sample.AnalyticsReceiver"
    android:enabled="true"
    android:permission="com.permissions.sample.ACCESS_USER_INFO">
    ...
</receiver>
```
*Content Providers* are a little bit different. They allow separate permissions for read, write or access of the Content Provider using a content URI.
- `android:writePermission`, `android:readPermission`: the developer can set separate permissions to read or write.
- `android:permission`: general permission that will control read and write to the Content Provider.
- `android:grantUriPermissions`: true if the Content Provider can be accessed using a content URI, temporarily overcoming the restriction of other permissions and False, if not.

### Signing and Publishing Process

Once an app has been successfully developed, the next step is to publish it to share it with others. However, apps cannot simply be put on a store and shared for several reasons – they need to be signed. This is a convenient way to ensure that apps are genuine and authenticate them to their authors: for instance, an upgrade to an app will only be possible if the update is signed with the same certificate as the original app. Also, this is a way to allow sharing between apps that are signed with the same certificate when signature-based permissions are used.

#### Signing Process

During development, apps are signed with an automatically generated certificate. This certificate is inherently insecure and is used for debugging only. Most stores do not accept this kind of certificate when trying to publish, therefore another certificate, with more secure features, has to be created and used.

When an application is installed onto the Android device, the Package Manager verifies that it has been signed with the certificate included in that APK. If the public key in the certificate matches the key used to sign any other APK on the device, the new APK has the option to share a UID with that APK. This facilitates interaction between multiple applications from the same vendor. Alternatively, it as also possible to specify security permissions the Signature protection level, restricting access to applications signed with the same key.

#### APK Signing Schemes

Android supports two application signing schemes. Starting from Android 7.0, APKs can be verified using the APK Signature Scheme v2 (v2 scheme) or JAR signing (v1 scheme). For backward compatibility, APKs signed with the v2 signature format can be installed on older Android devices, as long as these APKs are also v1-signed. [Older platforms ignore v2 signatures and only verify v1 signatures](https://source.android.com/security/apksigning/ "APK Signing ").

##### JAR Signing (v1 scheme):

In the original version of app signing, the signed APK is actually a standard signed JAR, which must contain exactly the entries listed in `META-INF/MANIFEST.MF`. All entries must be signed using the same certificate. This scheme does not protect some parts of the APK, such as ZIP metadata. The drawback with this scheme is that the APK verifier needs to process untrusted data structures before applying the signature, and discard data not covered by them. Also, the APK verifier must uncompress all compressed files, consuming considerable time and memory.

##### APK Signature Scheme (v2 scheme)

In the APK signature scheme the complete APK is hashed and signed, and an APK Signing Block is created and inserted into the APK. During validation, v2 scheme treats performs signature checking across the entire file. This form of APK verification is faster and offers more comprehensive protection against modification.

<img src="Images/Chapters/0x05a/apk-validation-process.png" width="600px"/>

*[APK signature verification process](https://source.android.com/security/apksigning/v2#verification "APK Signature verification process")*

##### Creating Your Certificate

Android is using the public/private certificates technology to sign Android apps (.apk files): this allows to establish the authenticity of apps and make sure the originator is the owner of the private key. Such certificates can be self-generated and signed. Certificates are bundles that contain different information, the most important from security point of view being keys: a public certificate will contain the public key of the user, and a private certificate will contain the private key of the user. Both public and private certificates are linked together. Certificates are unique and cannot be generated again. This means that, in case one or both certificates are lost, it is not possible to renew them with identical ones, therefore updating an app originally signed with a given certificate will become impossible.

The creator of the app can either reuse an existing private / public key pair that already exists and is stored in an available keystore, or generate a new pair.

Key pairs can be generated by the user with the keytool command (example for a key pair generated for my domain ("Distinguished Name"), using the RSA algorithm with a key length of 2048 bits, for 7300 days = 20 years, and that will be stored in the current directory in the secure file 'myKeyStore.jks'):
```
keytool -genkey -alias myDomain -keyalg RSA -keysize 2048 -validity 7300 -keystore myKeyStore.jks -storepass myStrongPassword
```

Safely storing a secret key and making sure it remains secret during its entire lifecycle is of paramount importance, as any other person who gets access to it would be able to publish updates to your apps with the content that you would not control (therefore being able to create updates to you apps and add insecure features, accessing the content that is shared using signature-based permissions, e.g. only with apps under your control originally). The trust that a user places in an app and its developers is totally based on such certificates, hence its protection and secure management are vital for reputation and customer retention. This is the reason why secret keys must never be shared with other individuals. Keys are stored in a binary file that can be protected using a password: such files are referred to as 'keystores'. Passwords used to protect keystores should be strong and known only by the key creator (-storepass option in the command above, where a strong password will be provided as an argument). For this reason, keys are usually stored on a dedicated build machine with limited access for developers.

Android certificates must have a validity period longer than the one of the associated app (including its updates). For example, Google Play will require that the certificate remains valid until at least Oct 22nd, 2033.

##### Signing an Application

After the developer has generated its own private / public key pair, the signing process can take place. From a high-level point of view, this process is meant to associate the app file (.apk) with the public key of the developer (by encrypting the hash value of the app file with the private key, where only the associated public key can decrypt it to its actual value that anyone can calculate from the .apk file). This guarantees the authenticity of the app (e.g. the fact that the app really comes from the user who claims it) and enforces a mechanism where it will only be possible to upgrade the app with other versions signed with the same private key (e.g. from the same developer).

Many Integrated Development Environments (IDE) integrate the app signing process to make it easier for the user. Be aware that some IDEs store private keys in clear text in configuration files; you should be aware of this and double-check this point in case others are able to access such files, and remove the information if needed.

Apps can be signed from the command line by using the 'apksigner' tool provided in Android SDK (API 24 and higher) or the 'jarsigner' tool from Java JDK in case of earlier Android versions. Details about the whole process can be found in Android official documentation; however, an example is given below to illustrate the point:
```
apksigner sign --out mySignedApp.apk --ks myKeyStore.jks myUnsignedApp.apk
```
In this example, an unsigned app ready for signing ('myUnsignedApp.apk') is going to be signed with a private key from the developer keystore 'myKeyStore.jks' located in the current directory and will become a signed app called 'mySignedApp.apk' ready for release on stores.

###### Zipalign

The `zipalign` tool should always be used to align the APK file before distribution. This tool aligns all uncompressed data within the APK, such as images or raw files, on 4-byte boundaries, that helps to improve memory management during app runtime. If using apksigner, zipalign must be performed before the APK file has been signed.

#### Publishing Process

The Android ecosystem is open, and thus it is possible to distribute apps from anywhere (your own site, any store, etc.). However, Google Play is the most famous, trusted and popular store and is provided by Google itself. Amazon Appstore is the default, trusted store on Kindle devices. If a user wants to install third-party apps from a non-trusted source they must explicitly allow this from the security settings on their device.

Apps can be installed on an Android device from a variety of sources: locally via USB, via Google's official app store (Google Play Store) or from alternative stores.

Whereas other vendors may review and approve apps before they are actually published, Google will simply scan for known malware signatures; this way, a short release time can be expected between the moment when the developer starts the publishing process and the moment when the app is available to users.

Publishing an app is quite straightforward, as the main operation is to make the signed .apk file itself downloadable. On Google Play, it starts with creating an account, and then delivering the app through a dedicated interface. Details are available on Android official documentation at https://developer.android.com/distribute/googleplay/start.html.
