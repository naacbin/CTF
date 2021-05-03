## Android build profile
We first need to find the kernel version
```bash
strings lime.dump | grep -i 'Linux version'
Kernel: Linux version 3.18.91+ (android-build@wphr1.hot.corp.google.com) (gcc version 4.9 20140827 (prerelease) (GCC) ) #1 SMP PREEMPT Tue Jan 9 20:30:51 UTC 2018
Linux version 4.4.124+ (forensics@fcsc2021) (gcc version 4.9.x 20150123 (prerelease) (GCC) ) #3 SMP PREEMPT Sun Mar 21 19:15:33 CET 2021
strings lime.dump | grep -i 'Android SDK built'
Dalvik/2.1.0 (Linux; U; Android 8.0.0; Android SDK built for x86_64 Build/OSR1.180418.026)
```

Then I followed this [tutorial](https://gabrio-tognozzi.medium.com/run-android-emulator-with-a-custom-kernel-547287ef708c) to run a custom kernel into Android. Followed by this [one](https://gabrio-tognozzi.medium.com/lime-on-android-avds-for-volatility-analysis-a3d2d89a9dd0) in order to use create the profile.
```bash
git clone https://android.googlesource.com/platform/prebuilts/gcc/linux-x86/x86/x86_64-linux-android-4.9 -b android-security-8.0.0_r54 # branch master doesn't have x86_64-linux-android-4.9-gcc
export PATH=$PATH:~/x86_64-linux-android-4.9/bin
export ARCH=x86_64
export CROSS_COMPILE=x86_64-linux-android-

git clone https://android.googlesource.com/kernel/goldfish/ -b android-goldfish-4.4-dev
cd ~/goldfish
make x86_64_ranchu_defconfig
# Check CONFIG_MODULES=y in .config
make -j16

git clone https://github.com/volatilityfoundation/volatility.git ~/android-volatility
cd ~/android-volatility/tools/linux
```

Edit the Makefile :
```makefile
obj-m += module.o
KDIR ?= ~/goldfish
CCPATH := ~/x86_64-linux-android-4.9/bin
DWARFDUMP := dwarfdump
KVER ?= $(shell uname -r)

-include version.mk

all: dwarf 

dwarf: module.c
	$(MAKE) ARCH=x86_64 CROSS_COMPILE=$(CCPATH)/x86_64-linux-android- -C $(KDIR) CONFIG_DEBUG_INFO=y M=$(PWD) modules
	$(DWARFDUMP) -di module.ko > module.dwarf

clean:
	rm -f module.dwarf
EOF
```

Finally build the profile
```bash
make
zip ~/android-volatility/volatility/plugins/overlays/linux/Golfish-4.4.zip module.dwarf ~/goldfish/System.map
```

### Bonus
If you want to simulate an android device
```bash
# https://stackoverflow.com/questions/60440509/android-command-line-tools-sdkmanager-always-shows-warning-could-not-create-se
mkdir ~/android-sdk && cd ~/android-sdk
mv cmdline-tools tools/ && mkdir cmdline-tools 11111111111111111111&& mv tools/ cmdline-tools
ANDROID_SDK_ROOT=~/android-sdk
export PATH=$PATH:$ANDROID_SDK_ROOT/cmdline-tools/latest/bin:$ANDROID_SDK_ROOT/cmdline-tools/tools/bin:~/android-sdk/platform-tools
adb --version
sdkmanager --update
sdkmanager --list
sdkmanager "platforms;android-26"
sdkmanager "system-images;android-26;google_apis;x86_64"
sdkmanager "build-tools;26.0.0"
avdmanager create avd -n fcsc -k "system-images;android-26;google_apis;x86_64" # Google service permit root
avdmanager list avd
~/android-sdk/emulator/emulator -avd fcsc -kernel ~/goldfish/arch/x86/boot/bzImage -show-kernel -verbose
```

Build your kernel version and bump it to android device
```bash
git clone https://github.com/504ensicsLabs/LiME.git ~/LiME
cd ~/LiME/src
```

Edit the Makefile :
```makefile
obj-m := lime.o
lime-objs := tcp.o disk.o main.o hash.o deflate.o

KVER ?= $(shell uname -r)
KDIR_GOLDFISH ?= ~/goldfish
CCPATH := ~/x86_64-linux-android-4.9/bin
PWD := $(shell pwd)

.PHONY: modules modules_install clean distclean debug

default:
        $(MAKE) ARCH=x86_64 CROSS_COMPILE=$(CCPATH)/x86_64-linux-android- -C $(KDIR_GOLDFISH) EXTRA_CFLAGS=-fno-pic M=$(PWD) modules
        mv lime.ko lime-goldfish.ko

debug:
	KCFLAGS="-DLIME_DEBUG" $(MAKE) CONFIG_DEBUG_SG=y -C $(KDIR) M="$(PWD)" modules
	strip --strip-unneeded lime.ko
	mv lime.ko lime-$(KVER).ko

symbols:
	$(MAKE) -C $(KDIR) M="$(PWD)" modules
	mv lime.ko lime-$(KVER).ko

modules:    main.c disk.c tcp.c hash.c lime.h
	$(MAKE) -C /lib/modules/$(KVER)/build M="$(PWD)" $@
	strip --strip-unneeded lime.ko

modules_install:    modules
	$(MAKE) -C $(KDIR) M="$(PWD)" $@

clean:
	rm -f *.o *.mod.c Module.symvers Module.markers modules.order \.*.o.cmd \.*.ko.cmd \.*.o.d
	rm -rf \.tmp_versions

distclean: mrproper
mrproper:    clean
	rm -f *.ko
```
```bash
make
make clean
```

Push the module and dump the memory
```bash
adb push ~/LiME/src/lime-goldfish.ko /sdcard/lime.ko
adb shell
su
insmod /sdcard/lime.ko "path=/sdcard/lime.dump format=lime"
```

## Ordiphone 0
### Points : *50*
### Description :
> Un nouvel apprenti vient d'effectuer une capture mémoire mais a oublié de noter la date du lancement de celle-ci.

> Pour valider cette première étape, vous devez retrouver la date à laquelle le processus permettant la capture a été lancé. Le flag est au format FCSC{sha256sum(date)}, avec la date au format YYYY-MM-DD HH:MM en UTC.

> lime.dump.7z (180M)

### Solution :
Found time of start in audit log.
```bash
strings lime.dump | grep -i "audit" | grep -i "insmod"
type=1400 audit(1616526815.693:11968): avc: denied { module_load } for pid=4752 comm="insmod" path="/storage/emulated/0/lime.ko" dev="sdcardfs" ino=57349 scontext=u:r:su:s0 tcontext=u:object_r:sdcardfs:s0 tclass=system permissive=1
```
`1616526815` to epoch converter : `2021-03-23 19:13`

Flag : `FCSC{b7dc08558ee16d1acbf54db67263c1d92e9a9d9603e6a1345550c825527adc06}`

---
## Ordiphone 1
### Points : *200*
### Description :
> Le maître d’apprentissage, un peu fou, veut savoir depuis combien de nanosecondes l'ordiphone était allumé lors du lancement de la capture mémoire. Retrouvez cette information, contenue dans la variable real_start_time du processus ayant effectué la capture !

> Le flag est au format FCSC{real_start_time}, où real_start_time est un nombre entier.

> lime.dump.7z (180M)

### Solution :
On this [reddit](https://www.reddit.com/r/kernel/comments/bg6sj5/find_start_time_of_a_process_given_pid/) post we can see that real_start_time is part of a task_struct. 

> As you can see, the real_start_time is a field in the task struct.

By looking on a [MISC article](https://connect.ed-diamond.com/MISC/MISC-076/Volatilisons-Linux-partie-2) about volatility, we learn that commands related to processus volatility use task_struct. The article give an example of the plugins pslist :
```python
init_task_addr = self.addr_space.profile.get_symbol("init_task")
init_task = obj.Object("task_struct", vm = self.addr_space, offset = init_task_addr)
for task in init_task.tasks:
	yield task
```

After looking at the code two ideas came to me. The first one was to write a plugins and the second one was to look at volshell to see if it can interact with task_struct. I found a very nice documentation about [volshell](https://www.tophertimzen.com/resources/cs407/slides/week02_01-volshell.html), the fifth slide explain that we can explore a structure with volshell.
```
volatility --profile=LinuxGolfishx64 -f ./lime.dump linux_volshell
> ps()
insmod           4752   0xffff880011da12c0
> dt("task_struct", 0xffff880011da12c0)
0x8d0 : real_start_time                63951047224
```

Flag : `FCSC{63951047224}`

---
## Ordiphone 2
### Points : *200*
### Description :
> Pour avancer sur cette investigation, vous devez analyser cette capture mémoire et une copie du stockage interne d'un téléphone Android utilisé par un cybercriminel.

> Votre mission est de retrouver les secrets que ce dernier stocke sur son téléphone.

> lime.dump.7z (180M)
>
> sdcard.zip (17MB)

### Solution :
After unzipping the `sdcard.zip` we found a file named secrets that seems to be a LUKS encrypted partition : 
`secrets: LUKS encrypted file, ver 2 [, , sha256] UUID: 26040e95-2800-4129-be0f-4879b4579f22`.

We have a memory dump, so if the partition was open during the dump we could try to extract his AES key with programs like [findaes](https://sourceforge.net/projects/findaes/). We get 6 differents key of 256 bits size. We try to open the luks partition with each one.
```bash
echo "d59cd96303c84041e11f083c5eaefd737eda32362853762ff314380297fdb779" | xxd -r -p > key1
echo "4c23303dc0cf28229979e25b08aab22550f463c94281f3c19ff5914ef54b5ec6" | xxd -r -p > key2
echo "d004723b9319dd084ca8bb8b9619b067b27ca802cc201b1723c357142d003d82" | xxd -r -p > key3
echo "9a6429e9045226def1be293dbe8d385fea08f1636ef0e5a1dc3178e884b2be2f" | xxd -r -p > key4
echo "8f51b80fc7d58913a342ddda460c6f6e951b79bc18524aa8ba101326624fdc3d" | xxd -r -p > key5
echo "fd0090fe95927256ee94bd26adb2a3beaf0f0a7e492a215ef1f35c0fc83cf846" | xxd -r -p > key6
cryptsetup luksOpen secrets secret --master-key-file key3
mount /dev/mapper/secret /mnt/mountpoint 
```
We found that the correct key was the third then we mount the LUKS partition. In this partition we have a `flag.enc` file and the following `script.sh` file :
```bash
aleatoire=$(cat /dev/urandom | head | xxd -p -l 30 | tr -d " ")
echo $aleatoire > /dev/kmsg
aleatoirebis="$aleatoire$(pidof adbd | tr -d ' ')$(pidof vold | tr -d ' ')$(pidof logd | tr -d ' ')"
echo $aleatoirebis | /data/data/com.termux/files/usr/bin/openssl aes-256-cbc -in flag -out flag.enc -pass stdin
/data/data/com.termux/files/usr/bin/shred flag
rm flag#
```

To view the content of /dev/kmsg we use the command `linux_dmesg` and `linux_pslist` to get the pid of the processes.
```bash
volatility --profile=LinuxGolfishx64 -f ./lime.dump linux_dmesg
[63788233965.63] 387e8985bd75be1b922eddaadde934e70465424ab4b0c3da98763c094432

volatility --profile=LinuxGolfishx64 -f ./lime.dump linux_pslist
0xffff8800481192c0 logd                 1529            1               1036            1036   0x00000000482fb000 0
0xffff880047c4a580 vold                 1539            1               0               0      0x00000000483f8000 0
0xffff8800480e0000 adbd                 1581            1               2000            2000   0x0000000048223000 0

volatility --profile=LinuxGolfishx64 -f ./lime.dump linux_dmesg
[63788233965.63] 387e8985bd75be1b922eddaadde934e70465424ab4b0c3da98763c094432


openssl aes-256-cbc -d -in flag.enc -pass pass:"387e8985bd75be1b922eddaadde934e70465424ab4b0c3da98763c094432158115391529" -out flag.png 2>/dev/null
```

Flag: `FCSC{ba5dc3f62c971c212133bb45b76084732c86936b76a026dc89c7b34fd3df29ae}` 