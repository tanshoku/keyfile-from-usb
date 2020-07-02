# keyfile-from-usb

#### Mounts an USB drive while in initramfs, searches for a file named "cryptkey" (or anything else you want) and uses that as a key to mount the root partition

 1. generating the keyfile:

  * generate a keyfile: `dd if=/dev/urandom of=cryptkey bs=1024 count=4`;
  
  * add it to luks:  `sudo cryptsetup luksAddKey /dev/sdX cyptkey`;
  
  * now copy it to the usb drive you intend to use as your key (and don't lose it!);

  * it's also reccomended that you keep a backup of yout keyfile somewhere safe

  
 2. you also have to:

  * add an entry to /etc/default/grub, like this one
  `GRUB_CMDLINE_LINUX="quiet cryptopts=source=/dev/sda5,target=sda5_crypt,keyscript=/lib/keyfile-from-usb,key=cryptkey,luks,discard"`

  ```
    explanation of cryptopts=
    source=/dev/sda5                 # $CRYPTTAB_SOURCE; your encrypted root partition, you can also use UUID=...
                                     # this is the default and you probably won't need to change it
    target=sda5_crypt                # $CRYPTTAB_NAME; target expected by the other boot programs, ditto
    keyscript=/lib/keyfile-from-usb  # $CRYPTTAB_OPTIONS; this very script, don't change it unless you know what you're doing
    key=cryptkey                     # $CRYPTTAB_KEY; the name and path relative to / of the keyfile that in your external drive
    luks,discard                     # $CRYPTTAB_OPTIONS; other options such as type and discard for ssd
  
    (please check if these options match with the ones on your /etc/crypttab, otherwise you might end up with a non-bootable sytem!)
  ```

  * change /usr/share/initramfs-tools/scripts/local-top/cryptroot, the line inside the funtion "setup_mapping()" that says
    ```
    # unlock interactively or via keyscript
    run_keyscript "$CRYPTTAB_KEY" "$count" | unlock_mapping
    ```
  * to
    ```
    # unlock interactively or via keyscript
    run_keyscript "$CRYPTTAB_KEY" "$count"
    ```

  * ...because we don't need to pipe the output of this script to "unlock_mapping", since this script does the unlocking by itself, and otherwise we would get
    caught in a loop, where cryptroot fails with "$CRYPTTAB_NAME already exists" (or something like that)

  * add the following modules to your /etc/initramfs-tools/modules:
   `ehci_pci uhci_hcd ehci_hcd usb_storage nls_cp437 vfat fat sd_mod mmc_core scsimod usbcore nls_ascii`
   ...and other modules, as needed;

  * change the `HOOK_SOURCE` variable in `keyfile-from-usb.hook` to wherever the script is

  * copy the hook script to  /usr/share/initramfs-tools/hooks: `sudo cp keyfile-from-usb.hook /usr/share/initramfs-tools/hooks`
 
  * and finally, of course, rebuild your initramfs: `sudo update-initramfs -u`...

  * ...grub menus: `sudo update-grub`

  * and voilà =)


### Notes:
Reasoning: I looked all over and couldn't find a working way of using a keyfile on an usb drive during boot, so I wrote my own script to do that.

This was only tested on Debian Buster, and **I'm not responsible if you make your system unbootable**.

There is an installation script which you can use, but I haven't tested it on other systems.

Have a secondary kernel to boot from in case this fails, and be sure to know how to bring up your system from initramfs/busybox, in case you end up breaking it (which is very likely).

### TODO:
- Add a way to run the installer every time there is an update that changes the initramfs;
- Add support for multiple keyfiles;
- Further automate the setup script to set the hook variables;
- Add a toggle for verbose and quiet operation

### Credits:
This script is mostly based off the scripts and ideas presented in [this page](https://stackoverflow.com/questions/19713918/how-to-load-luks-passphrase-from-usb-falling-back-to-keyboard)
but adapted to work on Debian Buster with cryptsetup 2.0+, and with the advantage that the key can be an actual binary key and not a plaintext passphrase.

-- Elizabeth Bodaneze Aug-2018
