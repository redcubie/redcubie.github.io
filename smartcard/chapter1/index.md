# The saga of the virtual smart card
© redcubie, 2023

## Preface
The following was in large part written late at night, so there may be some errors. I may refactor it at a later date.

## Introduction
The goal of this project was to learn about smart cards and how to use them for authentication (for example through kerberos).
The first step in smart card infrastructure is planning and buying the hardware. As finding the correct cards may be difficult and shipping may take a long time, especially with Chinese vendors (cards may also be mislabeled), I opted to try it out using a virtual smart card instead.

Virtual smart cards are practically free and can be easily reconfigured or replicated, but for real applications, virtual smart cards are insecure, as the secrets are stored on your hard drive (or in some cases, only your computer's working memory), the solution used in this project is also insecure in that the driver exposes any smart card readers on your system to the network without any protection.

## The story
After choosing to go for virtual smart cards, I found instructions for installing a virtual smart card driver. \[[link](https://github.com/OpenSC/OpenSC/wiki/Smart-Card-Simulation)\] The driver is called `vpcd` and supports all 3 major platforms. This project was conducted on a Linux platform (Ubuntu 22.04 LTS) and thus I will focus on that platform, but the information should be applicable on others as well with minor changes.
The `vpcd` driver is installed as a plugin to `pcscd` (the PC/SC daemon; PC/SC - Personal Computer/Smart Card). [see appendix 1 for installation guide]
The instructions I followed showed building and installing the driver from source, but I later learned it might also available as a package, so if you want to try it out, that may be an option for you, but compiling the driver went smoothly, so it should not be an issue.

The next piece of the puzzle is called jCardSim. It's a smart card simulator written in java which can run JavaCard applets and supports a few different virtual smart card reader backends. (Windows users may need to use a different backend)
JavaCard is, as the name suggests, a standard for smart card applications written in Java. The pros and cons of JavaCard are out of scope for this text.
Most, if not all, of the issues faced were related to jCardSim.

Though jCardSim can run JavaCard applets, it doesn't come with any and the user must provide their own. In the instructions I followed, a few different applets were shown. \[[instructions](https://github.com/OpenSC/OpenSC/wiki/Smart-Card-Simulation)\] I went for the first in the list, IsoApplet which, as the name implies, is an applet that conforms to some ISO smart card specifications. This also allows it to work with the most tools.

The guide pointed me to use a fork of jCardSim that's an older version guaranteed to work. \[[arekinath/jcardsim](https://github.com/arekinath/jcardsim)\] That fork did not compile, as `tools.jar` (a file removed from OpenJDK near Java 9; I have Java 18 installed, more about that later) was missing. In the pull requests of that repository \[[hko-s/jcardsim](https://github.com/hko-s/jcardsim)\] was another fork that fixed that error by upgrading the version of one of the libraries used. With these fixes, the compile succeeded.

Next came installing the IsoApplet. This was also documented in the base instructions I followed. The process was mostly straightforward. I `git clone`-d the repository \[[arekinath/jcardsim](https://github.com/arekinath/jcardsim)\], but as the next step was from the windows part of the instructions (they reused the applet install sections), the paths needed converting. (In short: Windows uses backwards slashes while Linux uses forward slashes) As only relative paths were used, this was pretty simple. After changing the slashes in the path, it still didn't find the files. Turns out the author changes their domain name which meant the paths would also change. After changing that as well, the class files were compiled. There was a deprecation warning, but it could be ignored.

Now I had to create a configuration file for jCardSim which was pretty easy, just containing a few rows for IsoApplet and a few for the connection parameters to the virtual smart card driver.
Now it's finally time to run the simulator. [see command in appendix 1 step 10] The simulator ran fine and the card responded to commands.
The instructions included a command that sends some magic to the card to initialize the applet, without which other programs won't be able to talk to the card. After that, I could initialize the card with `pkcs15-init`. [see appendix 2]

But it turned out that this version of the simulator didn't support saving the card's state, so one would have to re-initialize the card before each use. That is very impractical, so I searched on. Turned out that the main repository of jCardSim already got this functionality added in a pull request at some point. I decided to download that pull request as a patch and apply it to my downloaded fork that compiles. This didn't really work, as the codebase had gone through significant changes.

So now I decided to just use the main repository of jCardSim.

Trying to compile this version of the simulator, I was met with the same error as the first one, so I decided to just download the fix commit from the working version and apply it. This worked and now the official version also compiled.

Now I had to figure out how the persistence worked. By reading the code, I figure out I had to define a variable called `persistentSimulatorRuntime.dir` in the configuration file to tell it where to store its data. After adding that, nothing changed.
After further investigation by reading the tests around the persistent runtime, I figured out that it must be passed to the initializer when initializing the simulator runtime. This is done in the virtual card reader interface class that is called when running the simulator. In this case, it's `com.licel.jcardsim.remote.VSmartCard`. The addition to load the persistent runtime was rather simple.
Next came the weird errors when running it with existing persistence data (i.e. the second time or later). After much head scratching and trying to debug it in Visual Studio Code, I figured out that the errors don't happen when debugging and realized that VSCode used Java 11 while manually running it used Java 18. Manually pointing to Java 11 when running fixed this issue.

Now that persistence worked, I tried to actually start using the virtual card. I initialized the applet and the PKCS#15 store, then I tried to generate a key. But, key generation failed with a generic error from the card. (During looking for the source of this error, I also found an open pull request in the repository for IsoApplet that aimed for better compatibility with the persistence runtime of jCardSim so I also download and applied that as a patch.)

After some debugging, I figured out that the card was trying to respond to one of the requests, but setting the length of the output was failing. Funnily enough, the newer version silently discarded any exceptions during runtime, but the old one just printed the trace out which helped a lot in finding the issue. It turned out that if the request was not an extended size, that the simulator limited the response to 258 bytes (this is probably something in a standard somewhere), but the card was trying to send 270 bytes. The limit in extended mode was the 16-bit *INTMAX* bytes. Replacing the conditional statement with a hard coded limit to the bigger of the two fixed this issue and it now worked fully.

In addition to the aforementioned issues, there was also a continuous issue with connecting the simulator to the virtual smart card reader. The simulator would usually give a connection refused error when trying to start it, until a client application tried to use the card, thereafter it would be able to connect. But after some period of inactivity, the simulator would start spewing out a lot of socket I/O errors. This made using a non-persistent simulator much more impractical and made debugging and use annoying. (When running the client applications while the simulator wasn't connected to the virtual reader, you get a card not present error. Then you'd have to start the simulator and rerun the client application.) It turned out that the service file for `pcscd` started it with the `--auto-exit` flag, which perfectly explained the symptoms I saw. To temporarily disable this, you can just run the daemon yourself without this flag. For a more permanent solution, you may remove the flag from the service file.

## Summary
No TLDR exists at this time.


## Appendix 1: setup guide

Any files used in this guide will be distributed with this guide in case the originals disappear from the internet.

### Step 0 - install dependencies
pcsc, openjdk, ant, git

### Step 1 - install vpcd
There are two ways to install `vpcd` (the virtual smart card reader driver) on Linux:

#### Option 1 - install using package manager
On Ubuntu there is a package named `virtualsmartcard-vpcd`, you can install it by running 

```sh
sudo apt install virtualsmartcard-vpcd
```

If you use another Linux distribution, there may be an equivalent package in your package manager's repositories. If you cannot find it, continue with option 2.

#### Option 2 - install from source
Get the source from GitHub, change into the driver's directory, then configure, build and install the driver:

```sh
git clone https://github.com/frankmorgner/vsmartcard.git
cd vsmartcard/virtualsmartcard
autoreconf -vis && ./configure 
make
sudo make install
```


### Step 2 - restart pcscd
Make sure to restart the PC/SC daemon (`pcscd`) service after installing the driver. (varies by Linux distribution)

```sh
sudo systemctl restart pcscd.service
```

If your distribution's service file for this service sets it to automatically exit after some idle period, you should disable that feature by removing `--auto-exit` from its commandline.

For Ubuntu, you can check by running `systemctl status pcscd.service` and checking the `Process` line.

You have two options for disabling this feature.

#### Option 1 - temporary
Stop the service by running

```sh
sudo systemctl stop pcscd.service
```
And run the service

```sh
sudo pcscd --foreground
```

#### Option 2 - permanent
Find the service file and remove the `--auto-exit` flag from its commandline.

This is left as an exercise for the reader.

### Step 3 - download JavaCard development kit
Get the code from GitHub

```sh
git clone https://github.com/martinpaljak/oracle_javacard_sdks.git
```
Store the location in an environment variable (persists for this terminal session)

```sh
export JC_CLASSIC_HOME=$PWD/oracle_javacard_sdks/jc305u3_kit
```

### Step 4 - download jCardSim
Get the source and change into it

```sh
git clone https://github.com/licel/jcardsim
cd jcardsim
```

### Step 5 - apply patches to jCardSim
Required at the time of writing, but the issues may get fixed by the maintainers of this repository in the future.

#### Fix for build error regarding tools.jar
This file was removed in Java 9, so building with anything newer will fail.

This patch was found from [a pull request](https://github.com/arekinath/jcardsim/pull/1)

Download it by running 

```sh
wget https://github.com/arekinath/jcardsim/pull/1/commits/1364e2f0ee1fd80972e67f29d08a4fa4fb8c8e17.diff -O jcardsim_toolsjar.patch
```

Apply it

```sh
git apply jcardsim_toolsjar.patch
```

#### Fix for APDU max length issues
This patch was written by me and will be distributed with this document.

```sh
git apply jcardsim_apdulen.patch
```

#### Patch to enable persistence
This patch was written by me and will be distributed with this document.

```sh
git apply jcardsim_persistence.patch
```

#### Print an trace on errors (optional)
This patch was written by me and will be distributed with this document.

```sh
git apply jcardsim_printerrors.patch
```


### Step 6 - build jCardSim
Override Java version (won't build if your Java is too new)

```sh
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```


Initialize the build system

```sh
mvn initialize
```
Now build the simulator

```sh
mvn clean install
```

### Step 6 - download IsoApplet
Download and change into the directory

```sh
git clone https://github.com/philipWendland/IsoApplet
cd IsoApplet
```

### Step 7 - patch IsoApplet

#### Better support for simulation
This patch originates from [someone else's pull request](https://github.com/philipWendland/IsoApplet/pull/31)


Download it by running 

```sh
wget https://github.com/philipWendland/IsoApplet/pull/31.patch -O isoapplet_sim.patch
```

Apply it

```sh
git apply isoapplet_sim.patch
```

#### Change JavaCard SDK version
This patch was written by me and will be distributed with this document.

```sh
git apply isoapplet_sdkver.patch
```

### Step 8 - build IsoApplet
Build and force use of Java 11, because 18 is not supported

```sh
JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64 ant
```

### Step 9 - copy jCardSim config file
Copy the file `jcardsim_isoapplet.cfg` to the jCardSim directory.

### Step 10 - run the simulator
Force use of Java 11, as the default on my system, Java 18, causes fatal errors

```sh
/usr/lib/jvm/java-11-openjdk-amd64/bin/java -classpath 'target/jcardsim-3.0.5-SNAPSHOT.jar:IsoApplet/src' com.licel.jcardsim.remote.VSmartCard jcardsim_isoapplet.cfg
```


## Appendix 2 - working with the card

### Dependecies

pkcs15-tools, opensc-tools, openssl, libengine-openssl-pkcs11

### Step 0 - initialize IsoApplet
```sh
opensc-tool --card-driver default --send-apdu 80b800001a0cf276a288bcfba69d34f310010cf276a288bcfba69d34f3100100
```

### Step 1 - initialize PKCS#15 structure on card
You will be asked to create a PIN and PUK. This will create PIN with ID 0xff.

Some applications may require the card to have a serial number. You can specify one by adding `--serial` followed by a space and 32 hexadecimal characters (the serial number is 16 bytes long) to the command. You may also include a label for the card by adding `--label` followed by a space and, in quotation marks, the label text.

```sh
pkcs15-init --create-pkcs15
```

For the following sections, you may read the manual entry for `pkcs15-init` to see more options.

### Step 2 - create PIN
You may create additional PINs for use with different keys.


### Step 3 - generate keys
You may choose another key algorithm (see supported algorithms and info [here](https://github.com/philipWendland/IsoApplet/wiki/Initialization#chose-a-key-type-and-length)). The `--auth-id` parameter chooses which PIN this key will be protected with and you may choose either the main user PIN (ID ff) or one you created yourself. You may choose a label for this key. You can also set the ID for this key, this example uses ID 1, but if you want more keys, you will have to give them different IDs.

```sh
pkcs15-init --generate-key "rsa/2048" --auth-id "FF" --label "myKey" --id "1"
```

### Step 4 - create certificate
Creating a certificate for a smart card involves generating a certificate signing request (CSR) using the smart card, getting it signed by a Certificate Authority (CA) and uploading it to the card.

#### Generating a CSR
Run this command (replace 01 with the ID of your key). You will be asked to provide some additional information to add to your certificate, follow the prompts. You may add a password for securing the request for transit to the CA.

```sh
openssl req -engine pkcs11 -new -key id_01 -keyform engine -out cert.csr
```
Now you can provide the generated file to the CA of your choice, which may or may not be you. 

#### Getting it signed
This is left as an exercise to the reader. If you want a graphical interface, you can use XCA, otherwise you can use the `openssl` command line interface.

### Uploading the certificate
After obtaining your signed certificate, you can upload it to the card by running

```sh
pkcs15-init --store-certificate John_Doe.crt
```

### Step 5 - verify info
The following command will list info about your card. You can now verify that your PINs, keys and certificates are properly stored.

```sh
pkcs15-tool --list-data-objects
```

### Next steps
You now have a working smart card with a keypair and a corresponding certificate.

You can use it to test out other applications that can make use of a smart card.

## Historical info
Copies of the related repositories and patches are included alongside this document for historical value.
### Commit hashes at the time of writing

|Repository|Commit hash|
|-|-|
|https://github.com/martinpaljak/oracle_javacard_sdks|e305a1a0b9bf6b9a8b0c91a9aad8d73537e7ff1b|
|https://github.com/licel/jcardsim|b54b18bade77eaaf93a57f86c764bb17c5854fef|
|https://github.com/philipWendland/IsoApplet|b22f0eea290630acc3b01644bafe053be7d1c029|

### File downloads
- [oracle_javacard_sdks.zip](attachments/oracle_javacard_sdks.zip)
- [jcardsim.zip](attachments/jcardsim.zip)
    - [jcardsim_apdulen.patch](attachments/jcardsim_apdulen.patch)
    - [jcardsim_persistence.patch](attachments/jcardsim_persistence.patch)
    - [jcardsim_printerrors.patch](attachments/jcardsim_printerrors.patch)
    - [jcardsim_toolsjar.patch](attachments/jcardsim_toolsjar.patch)
- [IsoApplet.zip](attachments/IsoApplet.zip)
    - [isoapplet_sdkver.patch](attachments/isoapplet_sdkver.patch)
    - [isoapplet_sim.patch](attachments/isoapplet_sim.patch)
- [jcardsim_isoapplet.cfg](attachments/jcardsim_isoapplet.cfg)

