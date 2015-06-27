VPS2Arch
========

The fastest way to convert a _VPS_ to [Arch Linux](https://www.archlinux.org/)!

Author
------

[Timothy Redaelli](mailto:tredaelli@archlinux.info)

Description
-----------

This script is used to convert a _VPS_, running another linux distro, to _Arch Linux_.  
It should be **only** used if your _VPS_ provider doesn't provide you an _Arch Linux_ image.

Disclaimer
----------

> I'm not responsible for any damage in your system and/or any violation of the agreement between you and your vps provider.  
> **Use at your own risk!**

How To
------

Download the script on your _VPS_ and execute it with root privileges

**WARNING** The script will **delete** any data in your _VPS_!

	wget http://git.io/vps2arch
	chmod +x vps2arch
	./vps2arch
	
Pro-tip:  if you want to install a 32-bit Arch Linux on your 64-bit _VPS_,
replace the previous line with `linux32 ./vps2arch`.

How does it work?
-----------------

It's Black Magic.
Just kiddin' 😏, the script itself is very simple.

In a nutshell, it will download the _Arch Linux Bootstrap Image_ and (see the [wiki](https://wiki.archlinux.org/index.php/Install_from_existing_Linux#Method_B:_Using_the_Bootstrap_Image_.28recommended.29)),
extract the image to / and configure the _Bootstrap chroot_.

Now, about the **critical** part:

> How can you wipe the system without breaking everything?

It's simple: the script downloads and installs [busybox](http://www.busybox.net/) in the _Bootstrap chroot_.

After that, it will erase all the system directories except from the _Bootstrap chroot_, `/dev`, `/proc`, `/sys` and the like .

Busybox is statically linked, so it can still be used to chroot to the _Bootstrap chroot_ and to install _Arch Linux_.

At this point _Arch Linux_ has been installed, but not configured.
The script will provide a SSH-able system automagically configuring grub, network and restoring the root password from the original system.

Once done doing its job, the script will ask you to manually reboot your _VPS_ and voilà, PROFIT!

Does it really work?
--------------------

Yes, it does!

For the time being, the script has been tested on the following linux distros:

* CentOS 6 (x86 and x86_64)
* CentOS 7 (x86 and x86_64)

on the following _VPS_ providers:

* [CloudAtCost](http://www.cloudatcost.com/)

Theoretically it should also work on **real** computers (running linux), but I think it's not worth it,
because you can install it in the canonical way.

Contributing
------------

If you have any useful modification, please use **Pull requests**.  
If you have successfully used this script on a different _distro_ - _VPS_ combination, please contact me so that I can update the above list.

If you are not a developer, but you still want to contribute, you can donate me an account on your _VPS_ provider and I'll do my best to support it.  
Or you can just donate me some bucks I'll spend to buy a _VPS_ on your provider in order to support it.

Caveats
-------

[OpenVZ](http://openvz.org/), [Virtuozzo](http://www.odin.com/products/virtuozzo/), [Docker](https://www.docker.com/) or any other similar _VPS_ systems are not supported (for the time being).

In other words, it'll only work on **fully virtualized** systems.
