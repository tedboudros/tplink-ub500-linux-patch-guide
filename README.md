# TP-Link UB500 (rtl8761b) Linux <5.16 Kernel Patch Guide


## Step #1. Downloading and patching the kernel
First, find out the kernel version you are currently using.
You can check this by running:
```bash
uname -r
```
If you're using kernel version > 5.16, this patch should already be working out of the box, you do not need to continue with this guide, something else is wrong.

Then download and extract and enter the directory of the according kernel by doing:

> Example kernel version (5.11)

```bash
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.11.tar.xz
tar xpvf linux-5.11.tar.xz
cd linux-5.11/drivers/bluetooth
```

It's now time to edit the `btusb.c` file, which is the file that contains info about the bluetooth dongles' configurations.
```bash
nano btusb.c
```

Add this just before the line that says `/* Silicon Wave based devices */`:


```c
/* Tp-Link UB500 */
{ USB_DEVICE(0x2357, 0x0604), .driver_info = BTUSB_REALTEK },
```

Depending on the kernel version you're using, you also might need to change the following inside the `hci_ldisc.c` file:

```c
static ssize_t hci_uart_tty_read(struct tty_struct *tty, struct file *file,
                 unsigned char __user *buf, size_t nr)
```
to
```c
static ssize_t hci_uart_tty_read(struct tty_struct *tty, struct file *file,
                 unsigned char __user *buf, size_t nr,
                 void **cookie, unsigned long offset)
```

## Step #2. Compiling and updating the kernel

Let's now compile our changes and update the kernel!

```bash
make -C /lib/modules/$(uname -r)/build M=$(pwd) clean
cp /usr/src/linux-headers-$(uname -r)/.config ./
cp /usr/src/linux-headers-$(uname -r)/Module.symvers Module.symvers
make -C /lib/modules/$(uname -r)/build M=$(pwd) modules
sudo cp btusb.ko /lib/modules/$(uname -r)/kernel/drivers/bluetooth
```

**If your system is using secure boot, continue normally to Step #3.
If you don't use secure boot, you can skip the 4th and go to Step #4.**

To check if your system uses secure boot, run:
```bash
sudo mokutil --sb-state
```

## Step #3. Adding a key to the UEFI and signing our kernel module.
Go ahead and make a new `openssl.cnf` file and copy paste the following:
```bash
# This definition stops the following lines choking if HOME isn't
# defined.
HOME                    = .
RANDFILE                = $ENV::HOME/.rnd 
[ req ]
distinguished_name      = req_distinguished_name
x509_extensions         = v3
string_mask             = utf8only
prompt                  = no

[ req_distinguished_name ]
countryName             = GR
stateOrProvinceName     = Attika
localityName            = Athens
0.organizationName      = Example
commonName              = Secure Boot Signing
emailAddress            = example@example.com

[ v3 ]
subjectKeyIdentifier    = hash
authorityKeyIdentifier  = keyid:always,issuer
basicConstraints        = critical,CA:FALSE
extendedKeyUsage        = codeSigning,1.3.6.1.4.1.311.10.3.6,1.3.6.1.4.1.2312.16.1.2
nsComment               = "OpenSSL Generated Certificate"
```
At this point, you should take some time to switch your own details under the `[ req_distinguished_name ]` section.

After that, make the keys:
```bash
openssl req -config ./openssl.cnf \
        -new -x509 -newkey rsa:2048 \
        -nodes -days 36500 -outform DER \
        -keyout "MOK.priv" \
        -out "MOK.der"
```

Install key: (this will ask for a password, make sure you use one that you write down, you will need it later):
```bash
sudo mokutil --import MOK.der
```
Before continuing with this guide, you need to restart your computer.
Just before the boot begins, you should be greeted with a screen that says **Enroll MOK**.

Here you follow the steps and at the end of the prompt, it will ask you to input the password you wrote down before.

-----


To sign the compiled patched file `btusb.ko`, go into the folder we edited and compiled it
`linux-5.11/drivers/bluetooth` and run
```bash
kmodsign sha512 MOK.priv MOK.der btusb.ko
```


## Step #4. Installing the kernel module and the firmware

To install the patched kernel module, do:
```bash
sudo cp btusb.ko /lib/modules/$(uname -r)/kernel/drivers/bluetooth

sudo modprobe -r btusb
sudo modprobe -v btusb
```

After that, the final thing to do would be to install the bluetooth dongle's firmware.

To do that, do:
```bash
sudo mkdir -p /lib/firmware/rtl_bt
sudo curl -s https://raw.githubusercontent.com/Realtek-OpenSource/android_hardware_realtek/rtk1395/bt/rtkbt/Firmware/BT/rtl8761b_fw -o /lib/firmware/rtl_bt/rtl8761b_fw.bin
```

---
## The end...

And that's pretty much it!

Last thing to do is restart your computer one final time, and the bluetooth from your UB500 should be working perfectly fine!

## Huge props to:
https://askubuntu.com/questions/1370663/bluetooth-scan-doesnt-detect-any-device-on-ubuntu-21-10
https://github.com/TheSonicMaster/rtl8761b-fw-installer
https://ubuntu.com/blog/how-to-sign-things-for-secure-boot
