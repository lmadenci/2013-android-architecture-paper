####Written by Gabriel Leake & Lena Madenci
Android Architecture & Boot Process
=============================



##Introduction

Android OS has become the dominant platform in mobile computing, overtaking Apple's iOS.  There is an ever increasing need for functional applications in this system.  The motivation behind this survey was to gain an understanding of how the operating system was structured, how the boot process worked, and how applications were organized.  

##Android Software Stack
The Android software stack is composed of 4 different layers: the Application Layer, Application Framework Layer, Android Runtime Layer, and Libraries. These layers are built on top of a Linux 2.6.4 kernel. 




![alt text](http://www.ecnmag.com/sites/ecnmag.com/files/legacyimages/ECN/Articles/2011/07/ec1109es101_Fig%201%20copy.jpg)

The kernel contains the processes, memory and file system management for its operating system.

*__Library Layer:__* Located on top of the kernel, it provides tools to the upper layers. It is mostly accessed by Application framework services [3].

*__Application Runtime Layer:__* 
This is where the Dalvik Virtual Machine performs. By using services from the kernel, it “provides an environment to host Android applications” [1]. Although Android applications are written in Java, Android uses the Dalvik VM for an application runtime environment because Dalvik code is written to support multiple virtual machines. Dalvik allows for each Android to run its own process and virtual machine. This creates the effect of Sandboxing. The sandboxing allows for managers at the Android Framework Layer to provide security through permissions and intents [12]. Another difference between Dalvik virtual machines and Java virtual machines is that DVMs are register-based, not stack-based. Most virtual machines are stack-based because of its simple implementation, but register-based VMs have around 30% faster performance. Ultimately, there is less memory usage in register-based VMs. This is useful for the Android OS because it is suited for the space-constraint of a mobile device. [12]  

*__Application Framework Layer:__* The application framework layer contains the System Server, which contains managing modules. The Activity Manager Service, or AMS, is a necessary managing module for booting up and launching applications; the AMS communicates with the Linux kernel to determine whether or not forking a new process in necessary. This is done by using the methods startActivity and Process.start() [3]. The AMS is also responsible for saving the state of an activity so that other devices in the operating system can access and restore it [7]. 

*__Application Layer:__* The AMS livesi n the Application layer and manages the core components of Android applications: activities, services, content providers, and broadcast receivers.  


##Booting and Forking a Process

The Zygote process starts at boot time and initializes a Dalvik Virtual Machine. This original Zygote is the process that is the first step in every Android process because each following process is a Zygote fork. The original Zygote listens for spawn requests from the Zygote socket [google code wiki]. Without forking the Zygote process, the sandboxing of DVMs in Android would not be effective; cold starting VMs would take too much time for so many different Vms, which is why the Zygote pre-initializes classes that every app will need. 

An outline of the boot process of Android can be depicted as the following - 


![alt text](http://feb.imghost.us/EOvq.png)


The previous graph, from [2], displays the boot process in Android.

In order to boot up, the Android operating system works from its kernel and up through out the Android Layers. First, when the device is powered on, boot ROM code will begin execution and load the Bootloader onto physical memory (RAM) and then executed [2].  ROM stands for Read Only Memory and basically this memory exists inside the cpu, which makes sense this this is one of the first things the cpu will execute when it powers on. “The boot loader is a special program seperate from the Linux kernel that is used to set up initial memories and load kernel to RAM” [1].  This would be akin to GRUB or uBoot on Linux desktop systems.  

Then, it loads the Linux kernel and starts the init process. Init runs and parses init.rc.  Init.rc loads native services and runs /system/bin/app_process/app_main.cpp.  This is a C++ compiled file that calls AndroidRuntime.start(), this is in frameworks/base/core/jni/AndroidRuntime.cpp.  The start method takes in ZygoteInit and start system-server flag as arguments.   ZygoteInit is in frameworks/base/core/java/com/android/internal/os/ZygoteInit.java .  When ZygoteInit.main() executes it will register the Zygote socket [2].  This will be used by the Zygote to start apps, as the Zygote is constantly listening to the Zygote socket.  The classes listed in frameworks/base/preloaded-classes all get preloaded and startSystemServer() is eventually called.  

From the source code: https://github.com/android/platform_frameworks_base/blob/master/services/java/com/android/server/SystemServer.java 


We can see for ourselves that the SystemServer starts critical processes and services that the Window Manager will need in the run() method: 


![alt text](http://feb.imghost.us/EOyN.png)

These are the services that the SystemServer can instantiate.  Some key ones to note – power manager, activity manager, and window manager.  In addition, if the OS is starting in safe mode, the System Manager will run logic to detect it and then set flags on the Zygote and tell te Activity Manager to enter safe mode:


![alt text](http://feb.imghost.us/EOxt.png)


When the Zygote process starts up, it instantiates a Dalvik virtual machine (in the Android Runtime Layer), loads libraries, and starts listening to the Zygote socket. Once the Zygote process starts, its runs in loop mode continually. We can see this in the ZygoteInit code:


![alt text](http://feb.imghost.us/EOysz.png)
 
The Zygote will run in select loop mode.  A single process will spin and wait for communication to start subsequent applications [4].  We can see that in this while (true) method, the process will continually loop and keep track of how many times it has looped.  The code creates a new file descriptor array and then attempts to read from it.  The server socket contains a file descriptor that is added to the array.  According to [4], new connections are put into array “peers” and the spawn command that gets received over the network is executed by calling the runOnce() method on ZygoteConnection.java.  

In ZygoteInit, further down the while(true) method we have, 


![alt text](http://feb.imghost.us/EP0C.png)

Each peer is a ZygoteConneciton instantiation so the runOnce() method will be called in ZygoteConnection.java. In this method, this line of code will eventually be run which will call into the native fork function and return the pid of the new process:

pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids, parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo, parsedArgs.niceName); 

From listening to the Zygote socket, the Zygote process responds and forks, as shown above. Once the System Server is running, it starts the core platform services that live in the Application Framework Layer, including the Activity Manager Service. Finally, the Activity Service Manager sends messages to the Zygote to start different activities needed to display applications in the Application Layer.

Zygote is the resulting process from init running app_process doing runtime.start and passing in ZygoteInit. Runtime is an instance of AppRuntime which is subclassed from AndroidRuntime. AndroidRuntime.cpp is used for starting the Dalvik runtime environment, which has a start() method to start a virtual machine.

The Dalvik VM uses a register based architecture which means that it uses processor registers, storage that is part of the processor and can be accessed quickly.  These are at the top of the memory hierarchy since they are the fastest way to access data.  Each register holds a single positive integer.  All the Java files that Android uses are converted to .dex format (similar to a jar file, just an archive) using a tool that is part of Dalvik, called dx (5).  The Dalvik VM converts Java bytecode into alternative instructions that the Dalvik VM can understand and use.  The Dalvik VM is optimized to run in a low memory ad resouce constrained environment, which is typical in mobile computing.  How it differs from JVM is that it is not stack based, it uses less space, the interpreter is simplified as it only uses 32-bit.  Dalvik found a way to increase its instruction speed by using a 16 bit instruction set that works on local variables directly, as opposed to executing 8 bit stack instructions.  It was designed to be able to efficiently be instantiated multiple times on a single device. The Dalvik Virtual Machine is what houses application processes that run, each exist inside their own Dalvik VM.  

##Android Application Components

The components of an Android application are described below:

*__Activities:__* An activity is an interface for one single process and is composed of views, which respond to events. An activity is implemented as a class extending from the Activity base class [2]. 
 
*__Services:__* This is a process that runs in the background and does not have a user interface. It continually runs and is considered to be “long-life code” [2].

*__Content Provider:__* A class that shares data with other Android applications. It allows the other applications to store and retrieve the data [2]. The Content Provider is different from the other core components because it communicates with a content Uniform Resource Identifier rather than with intent messages [9].

*__Broadcast Receiver:__* This component acts as a mailbox for intent message broadcasts that are sent from the other components. It is a key component in data sharing within an application or between different applications. 

The above components of an application use intents to interact. An Intent is “a declaration of need” [6].  Intent messages are passed to methods like startActivity, startActivityForResult and startService to start Activities or Services. Broadcast Receivers receive intents when responding system events like incoming phone calls or messages [6]. Intents are checked by an application’s Manifest File. The Manifest File is a mandatory XML file and a configuration field in an application’s root directory [4,10]. It acts as a “contract [that] details on types of components, applications permissions, etc” [8]. It also names the classes that implement the components, capabilities and intents, and determines under what conditions that the application may launch, which process will host it, and if and when it can share data with other applications [8, 10]. These permissions written in the Manifest File are set at install and cannot change until application reinstall. The manifest file, AndroidManifest.xml must be included with each application.  All the components of the application must be declared in the file as well as the user permissions each component or activity requires.  Activity, services, receivers and providers are all declared here.  The manifest file has an intent-filter which is optional to include, but doing so will allow the application to respond to intents from other applications which want to perform an activity.  If the intent the other application is trying to do matches the intent-filter of your activity and the permissions are aligned, then the Application Services Manager will select your app based on the intent->intent-filter match and launch the activity in your app.  

To further elaborate on intents, an intent will bind individual components to one another at runtime and is an object which houses data to be passed between processes. It contains an abstract description of what action is intended to be performed.  The intent object will be passed into Context.startActivity() if an activity should be started.  

##Binders

Linux has multiple inter process communication mechanisms.  As previously mentioned, the Android OS is built on top of Linux and as such has access to using signals, pipes, sockets, semaphores, queuing, and shared memory.  However,  Android has implemented a driver called Binder, which is a reimplementation of OpenBinder, developed by Palm. Inc. [1]. 

The binder implementation file is a C file and is in drivers/misc/binder.c with the include file include/linux/binder.h.  

According to Brian Swetlad, a Senior Software Engineer at Google, the reason the binder was implemented at the kernel level and the reason the existing IPC mechanisms outlined above is because there are certain properties of the binder that are not available using existing IPC.  One of these properties is that the binder avoids copies as it has the kernel copy from the writer into a ring buffer directly inside the reader's address space and will allocate space is necessary.  The second reason is that the binder can manage the lifespan of proxied remote userspace objects, which can be passed or shared among processes [2]. 

In addition to this the Binder framework provides direct and indirect facilities, as shown below [3]:


![alt text](http://feb.imghost.us/EP2p.png)


As the above graph shows – direct facilities include managing, identifying and making calls.  Indirect facilities include using the binder as a token (the binder is unique so it can be used as such) and sending fd of a shared memory.  

From the application perspective, remote object methods can be called as if they were local object methods [3] by using a synchronous system call.  The downside of this is that the calling process will be blocked until a reply is received but the advantage is, “the client has no need to provide a thread method for a asynchronous return message from the client” [3].  The binder facility has the ability to start and stop threads using one and two way messages.  

The system service makes use of the link to death mechanism which is a direct binder facility for notification purposes.  It allows the process to become informed when a specific implementation of a binder interface, linked to a specific process, is terminated.  This is an important feature because the window manager in Android needs to be notified when a window is closed.  Each window has a Binder interface linked to it and a callback for that interface.  The window manager creates a link to death relationship to that callback and thus can be informed when the window closes and react accordingly [3].
  
One of the other interesting things from [3] is that each implementation of the binder is uniquely identifiable.  As such, it can be used as a shared token between processes (so long as the Binder is not published by the service manager), it will only be known between the processes sharing it, so it can be used for security among processes as an access token.


![alt text](http://feb.imghost.us/EP3V.png)

(Image source: http://vineetgupta.com/blog/mobile-platforms-part-1-android) 

The communication model that the Binder framework implements is client-server and is synchronous as stated previously.  A proxy that lives on the client side is used to communicate, this would be the Context shown above. The request that come into a server are worked on as a thread pool exists in the server side [3].  In the example from [3],  “process A is the client and holds the proxy object (Context) which implements the communication with the Binder kernel driver.  Process B is the server process, with multiple Binder threads.  The Binder framework will spawn new threads to handle all incoming requests, until a defined maximum count of threads is reached.  The proxy objects are talking to the Binder driver, that will deliver the message to the destined object.” [3].  

When Android applications want to communicate with one another, to, for example, launch another application, intents are used.  This was described previously but they relate to the binder in that they are sent with Binder IPC.  The intent containing the action of that should be performed, along with data, gets delivered by the intent reference monitor to the Binder that is assigned to the action in the Intent [3]. 

The Java layer that sits on top of the Android middle layer if a framework that provides an API.  The java layer of Android contains the following interfaces and classes which relate to IPC:

-android.app.IActivityManager, 
-android.os.Parcable
-android.os.IBinder
-android.content.ServiceConnection
-android.app.ActivityManager
-android.app.ContextImpl
-android.content.Intent
-android.cotent.ComponentName
-android.os.Parcel
-android.os.Bundle
-Android.os.Binder
-android.os.BinderProxy
-com.android.internal.os.BinderInternal

This layer of the Binder framework wraps the middleware layer so that the Android applications can utilize Binder communication and allows intents to use the Binder framework [3].  

Here is an outline from [3] showing this:


![alt text](http://feb.imghost.us/EP4d.png)


According to [3], the Java API layer “relies on the Binder middleware.  To use the C++ written
middleware from Java, the JNI must be used.  In the source code frameworks/base/core/jni/android_util_Binder.cpp, the mapping between Java and C++ is realized.”  JNI stands for Java Natve Interface and is a programming framework to allow Java cde in a JVM to call and be called by native applications and libraries that are written in other languages [4]





####References (Application Software Stack & Application Components):
1. Abelson, W. F., et al. (2011). "Android in action."
2. Agüero, J., et al. (2009). Does Android Dream with Intelligent Agents? International Symposium  on Distributed Computing and Artificial Intelligence 2008 (DCAI 2008), Springer.
3. Armando, A., et al. (2012). Would you mind forking this process? A denial of service attack on Android (and some countermeasures). Information Security and Privacy Research, Springer: 13-24.
4. Chin, E., et al. (2011). Analyzing inter-application communication in Android. Proceedings of the 9th international conference on Mobile systems, applications, and services, ACM.
5. Enck, W., et al. (2009). "Understanding android security." Security & Privacy, IEEE 7(1): 50-57.
6. Haseman, C. (2008). Android Essentials, apress.
7. Hassan, Z. S. (2008). Ubiquitous computing and android. Digital Information Management, 2008. ICDIM 2008. Third International Conference on, IEEE.
8. Maji, A. K., et al. (2012). An empirical study of the robustness of inter-component communication in Android. Dependable Systems and Networks (DSN), 2012 42nd Annual IEEE/IFIP International Conference on, IEEE.
9. Ongtang, M., et al. (2012). "Semantically rich application‐centric security in Android." Security and Communication Networks 5(6): 658-673.
10. Project, A. O. S. (2013). "The AndroidManifest.xml File." Android Developers. Retrieved July 25, 2013.
11. Shin, W., et al. (2010). A formal model to analyze the permission authorization and enforcement in the Android framework. Social Computing (SocialCom), 2010 IEEE Second International Conference on, IEEE.

####References (Binders):
1. http://0xlab.org/~jserv/android-binder-ipc.pdf
2. https://lkml.org/lkml/2009/6/25/3
3. https://www.nds.rub.de/media/attachments/files/2011/10/main.pdf 
4. http://en.wikipedia.org/wiki/Java_Native_Interface 

####References (Booting):
1. http://www.androidenea.com/2009/06/android-boot-process-from-power-on.html
2. https://sites.google.com/site/tomsgt123/adb-fastboot/understanding-android 
3. http://stackoverflow.com/questions/9153166/understanding-android-zygote-and-dalvikvm 
4. http://elinux.org/Android_Zygote_Startup


