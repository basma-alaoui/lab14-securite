LAB 14 : Bypass Root Detection sur Android : Techniques Dynamiques avec Frida, Objection et Hooks Natif

Dynamic Techniques: Frida, Objection, Medusa
Course: Mobile Application Security

             1. Legal Disclaimer

Use these techniques only on devices and applications that you own or for which
you have explicit written permission (audit, research, training). Rooting or
patching a device may void the warranty, erase data, degrade certain services
(e.g., Widevine), and increase security risks. The author is not responsible
for any misuse.

             2. Environment Setup

Install the required tools on your PC:

  pip install frida-tools objection

Ensure ADB (Android Debug Bridge) is installed and accessible.

Check device connection:

  adb devices

You should see your device or emulator listed as "device".

Verify Frida installation:

  frida --version
  objection version

              3. Start frida-server on the Android Device

Frida requires a server running on the Android device.

3.1 Identify the device architecture

  adb shell getprop ro.product.cpu.abi

Common values: arm64-v8a, armeabi-v7a, x86, x86_64.

3.2 Download the matching frida-server

From Frida releases (https://github.com/frida/frida/releases), download the
binary that matches:
- your Frida version (e.g., 17.9.5)
- the device architecture (e.g., android-x86_64 for x86_64 emulators)

Important: Do not use the Windows .exe version on Android.

3.3 Push, set permissions, and run

  adb push frida-server /data/local/tmp/
  adb shell chmod 755 /data/local/tmp/frida-server
  adb shell /data/local/tmp/frida-server &

3.4 Verify frida-server is working

  frida-ps -U

If the command lists running processes (e.g., Calendar, Settings), the setup
is successful.

               4. Bypass Root Detection with Frida (Custom Script)

Many applications detect root by executing shell commands (e.g., "which su",
"ls /system/bin/su") using Runtime.exec. Frida can hook this method to block
suspicious commands.

Example Frida script (save as bypass_root.js):

  Java.perform(function() {
      var Runtime = Java.use("java.lang.Runtime");

      Runtime.exec.overload('[Ljava.lang.String;').implementation = function(args) {
          var cmd = args ? Array.from(args).join(' ') : '';
          if (cmd.includes("su") || cmd.includes("busybox") || cmd.includes("magisk")) {
              console.log("[+] Blocked Runtime.exec: " + cmd);
              return this.exec(['echo', 'blocked']);
          }
          return this.exec(args);
      };
  });

Run the script:

  frida -U -l bypass_root.js com.pwnsec.firestorm

Replace com.pwnsec.firestorm with your target package name.

                       5. Simpler Alternative with Objection

Objection is a runtime exploration tool that wraps Frida and provides ready-
to-use commands.

                       5.1 Attach to the application

  objection -n com.pwnsec.firestorm start

Once in the interactive shell, disable root detection:

  android root disable

Objection automatically hooks common root detection methods (file checks,
properties, Runtime.exec). You can also run "android sslpinning disable" if
needed.

                      5.2 List available commands inside Objection

  help
  env

                     6. Medusa – Modular Automation Framework

Medusa automates the injection of Frida scripts and offers pre-built modules
for root bypass, SSL unpinning, etc.

                     6.1 Installation

  git clone https://github.com/Ch0pin/medusa.git
  cd medusa

                     6.2 Help and options

  python medusa.py --help

                    6.3 Run Medusa on the target application

  python medusa.py -p com.pwnsec.firestorm

After entering the interactive Medusa console, type:

  run module root_bypass

Other useful commands: help, dump, modules list.


              7. When to Prefer Magisk (System-level Hiding)

Dynamic methods (Frida / Objection / Medusa) are temporary – they stop after
the application restarts or the device reboots. Magisk with MagiskHide (or
Zygisk) provides persistent root hiding at the system level.

- Quick analysis, one-time bypass → Frida / Objection
- Application checks root very early at startup → Magisk (or try --spawn with Frida)
- Need long-term, reboot-proof hiding → Magisk
- Avoid modifying system partitions → Frida / Objection

                8. Troubleshooting

Problem: frida-ps -U shows no processes
Solution: frida-server not running or wrong architecture. Verify with
        "adb shell ps | grep frida". Kill and restart with correct binary.

Problem: objection -n <package> fails with "unable to find process"
Solution: Wrong package name or app not running. Use "objection -l" to list
        installed packages. Start the app manually.

Problem: Application crashes after Frida injection
Solution: Version mismatch between Frida client and server. Upgrade both to
        same version. Try --spawn mode.

Problem: adb devices does not list any device
Solution: USB debugging disabled or driver issue. Enable Developer Options →
        USB debugging. Reconnect cable.

Problem: Root detection still active after "android root disable"
Solution: Application uses native (C/C++) checks or custom detection. Write a
        custom Frida script to hook native functions using Interceptor.attach.

                         9. Quick Checklist

- ADB detects the device (adb devices)
- Correct frida-server binary (architecture matching) is pushed and running
- frida-ps -U lists processes
- Objection can attach to the target application
- android root disable command executes without errors
- The application no longer displays root warnings or closes abnormally

                        
Always use these skills responsibly and legally.
