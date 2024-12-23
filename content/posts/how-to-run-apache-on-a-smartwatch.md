---
title: "How to run the Apache webserver on a five year old Android smartwatch"
date: 2019-11-30T11:53:16+01:00
type: posts
tags:
  - apache
  - termux
  - adb
  - android
  - wearables
draft: false
description: The first question you might ask yourself when reading the title of this article is; why would you want to this. My answer to that question is, why wouldn't you?
---

The first question you might ask yourself when reading the title of this article is; why would you want to this. My answer to that question is, why wouldn't you?

I like using technology in ways that makes my live easier. That's why I bought my Motorola Moto 360 in the first place, more than five years ago. It has indeed made my life easier in a few ways, quickly reading a message, discarding when necessary, or pulling out my phone when I need/want to reply. Also controlling my music, checking of a shopping list in the supermarket, even making a quick calculation with the built-in calculator. All these things made me really like my smartwacth. I could live without it more easily than my phone, but still, I wear it almost every day.

Now I have come to a point where I'm looking for a new smartwatch. I get the feeling that my beloved first generation Moto 360 is running on it's last legs. The performance isn't really that great (it hasn't really ever been that great), the battery is deteriorating, I can't even start it up when it has less than about 60% battery, I have to put it on a charger for it to boot up. Also the new features in the newest generations of smartwatches, for example, contact-less payments, built-in GPS, all make me linger for one.

So, this leads me to explaining exactly why I ran a webserver on my Moto 360. As I said in the beginning of this article, how I like using technology in ways that makes my life easier, I also like to explore the limits of what is possible with that technology. Running a webserver on a wearable maybe doesn't really have a direct practical use. But the fact that it is possible, just makes me happy.

Let us now take a look on how this is actually done. To do this you need a few things: an Android smartphone with an Android Wear watched paired with it, and the Wear OS app installed off course. A computer with internet connectivity and the Android SDK Platform-Tools installed. I'm also using an OS with a Linux kernel on that computer. If you are using another operating system, maybe you need to install drivers to connect your phone and being able to use ADB with it, but I have no idea how to do that. Once that's all in place, we can get started.

First of all you need to enable USB debugging on your Android device, you do this be enabling the Developer options, this done by tapping the build number some times in the 'About Phone' section. Then you can go in the developer options and check the checkbox next to 'USB debugging'. You can check if ADB works by plugging you phone in with a USB cable into your computer, and running `adb devices` in your preferred shell. You should see at least one device listed there. On your phone there will probably appear a dialog, asking you to give permission to the computer you are running ADB on. If you trust this computer you can check a checkbox to always trust that computer. It could be that the output of the `adb devices` tells you that it still doesn't have permission for the device. Then a possible solution is running the ADB command with super user privileges, on macOS, or Linux-based OS'es you can do this with the `sudo` utility; `sudo adb devices` for example. Or starting a shell as root, and running all the following ADB commands from there.

You also need to enable ADB debugging on your Android Wear device. This is the same procedure as on your Android phone/tablet.

When that's all done, you can connect to your Android Wear device with the following procedure, this procedure is explained in more detail [here][3]:
- Connect your phone to your computer with a USB cable, and verify ADB is working by running `adb devices`
- Then head over to the Wear companion app, go to **Advanced Settings**
- Enable **Debugging over Bluetooth** in that menu
- Head over to the **Developer options** and enable **ADB debugging** (this settings turns off automatically)
- Now, run these two commands on your computer:
```
adb forward tcp:4444 localabstract:/adb-hub
adb connect 127.0.0.1:4444
```
- That's it, now you should see the following output in the **Advanced Settings** of the Wear OS app:
```
Host: connected
Target: connected
```
And you should also see two devices listed in the output of `adb devices`

Now we are ready to install the [Termux][1] application on our watch. An old Moto 360 doesn't run Wear OS 2.0, so you will have to download the APK manually. Because the Moto 360 uses Android 6.0 as it's base OS, the newest versions of Termux have [given up][2] on Android versions prior to Android 7, so we will need to get the Termux application from [archive.org][4]. Download the file `termux-app-git-debug.apk`, optionally, also download `termux-app-git-debug.apk`, you can fetch some additional device info from Termux with this addon, more information can be found [here][5].

Once you have downloaded the(se) file(s), we can install them using ADB:
```
adb -s 127.0.0.1:4444 install termux-app-git-debug.apk
# optional
adb -s 127.0.0.1:4444 install termux-api-git-debug.apk
```

I know you are excited now, and eager to start the freshly installed Termux app, but hold on for just one second. The app will try to bootstrap when it's opened for the first time, but it needs internet connectivity for this. You can use the WiFi module in the Moto 360 for this, but to use this, it's best to disable the Bluetooth on your connected phone, otherwise the watch will disable WiFi on itself. But when we disable Bluetooth on the phone, the ADB link is also broken, and we need ADB in the next steps. Here's the solution for this problem, start a TCP/IP ADB server on the watch, this way, we can connect to the ADB server on the watch, over WiFi. The ADB server, listening on TCP/IP can be started this way:
```
adb -s 127.0.0.1 tcpip 5555
```
More details about this, can be found [here][6].

The next step, is installing an openssh server on the watch in Termux, so we can SSH to the Termux environment, and input text more easily. Text can also be inserted via ADB, and we will need to do it this way initially, but that's a pretty tedious process.
To accomplish these steps, you will also need the IP address of your watch (to connect ADB to it), you can find this under **Settings** > **Wi-Fi Settings** > **Advanced** > **IP address**. You can also use the management interface, or nmap, etc.
1. Start the Termux application on your watch, and let it setup, this only happens the first time after installation
2. The first thing you need to do now, is connect to the ADB TCP/IP server on you watch:
```
adb connect <IP address of your watch>:5555
```
3. Now, you can install openssh in Termux, for this, the Termux app needs to be on running in the foreground, and make sure the watch doesn't fall asleep by tapping or swiping on the screen from time to time.
```
adb -s <IP address of your watch>:5555 input text 'pkg install openssh'
adb -s <IP address of your watch>:5555 input keyevent 66
# wait a bit until it asks you to install the necessary packages 
adb -s <IP address of your watch>:5555 input keyevent 66
```
4. Openssh is now installed on the watch, but before you can try to connect to it, you will need to setup a password:
```
adb -s <IP address of your watch>:5555 input text 'passwd'
adb -s <IP address of your watch>:5555 input keyevent 66
adb -s <IP address of your watch>:5555 input text 'termux'
adb -s <IP address of your watch>:5555 input keyevent 66
adb -s <IP address of your watch>:5555 input text 'termux'
adb -s <IP address of your watch>:5555 input keyevent 66
```
5. Now you can start the ssh daemon:
```
adb -s <IP address of your watch>:5555 input text 'sshd'
adb -s <IP address of your watch>:5555 input keyevent 66
```
6. Finally, you can connect to your watch, from a computer with ssh installed in the same network as your watch:
```
ssh -p 8022 <IP address of your watch>
```
When prompted for a password, use `termux`, we set this password in step 4. You may think; don't I need to specifiy a user. The answer is: no. You can use any username you want when SSH'ing to a Termux environment. More information can be found [here][7].

When you are connected to your Termux app over SSH, I recommend creating a termux.properties file in the ~/.termux directory. In this file you can specify which buttons to show on the bottem of your screen. You can configure it to show a button to enter the text `sshd` and then add another button to hit the `enter` key. This way, you can start the ssh daemon, without fiddling around with the ADB text input stuff, this means you also don't need a computer from now on to start the ssh daemon. You can even ssh to your watch when you are on the road, just use the WiFi tethering on your phone, connect your watch to it, start the ssh daemon on your watch, and use an ssh app (you can also use Termux for this) on you phone to ssh to your watch. To create this termux.properties file, execute the following commands on your watch:
```
mkdir .termux
vi .termux/termux.properties
```
Insert the following text in this file:
```
extra-keys = [ \
 ['sshd', 'ENTER'] \
]
```

The final step is the easiest; installing apache. The only thing you need to do is; install the apache package, and start apache:
```
pkg install apache2
apachectl start
```
It's as simple as that, the apache server listens on port 8080, just point any machine on the same network as your watch to your watch's IP address, and you should see `It works!`.

[1]: https://termux.com
[2]: https://wiki.termux.com/wiki/FAQ#Termux_is_no_longer_available_for_Android_5.3F
[3]: https://developer.android.com/training/wearables/apps/debugging#bt-watch
[4]: https://archive.org/details/termux-repositories-legacy
[5]: https://wiki.termux.com/wiki/Termux:API
[6]: https://developer.android.com/studio/command-line/adb#wireless
[7]: https://wiki.termux.com/wiki/Remote_Access#SSH
