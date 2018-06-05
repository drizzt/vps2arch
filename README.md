VPS2Arch
========

The fastest way to convert a _VPS_ to [Arch Linux](https://www.archlinux.org/)!

Author
------

[Timothy Redaelli](mailto:timothy@fsfe.org)

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

	wget http://tinyurl.com/vps2arch
	chmod +x vps2arch
	./vps2arch

Some _Debian_ _VPS_ images don't have the _CA certificates_ installed, so `wget` will give you the following errors:

	ERROR: The certificate of `gitlab.com' is not trusted.
	ERROR: The certificate of `gitlab.com' hasn't got a known issuer.

To fix them just execute `apt-get install ca-certificates` and then re-execute the previous commands.

How does it work?
-----------------

It's Black Magic.
Just kiddin' 😏, the script itself is very simple.

In a nutshell, it will download the _Arch Linux Bootstrap Image_ and (see the [wiki](https://wiki.archlinux.org/index.php/Install_from_existing_Linux#Method_B:_Using_the_Bootstrap_Image_.28recommended.29)),
extract the image to / and configure the _Bootstrap chroot_.

Now, about the **critical** part:

> How can you wipe the system without breaking everything?

It's simple: using `ld.so` from the _Bootstrap chroot_ to launch the `chroot` tool.

Since it will erase all the system directories except from the _Bootstrap chroot_, `/dev`, `/proc`, `/sys` and the like,
the only way to launch a command inside the _Bootstrap chroot_ is to using ld.so from the _Bootstrap chroot_ itself.

At this point _Arch Linux_ has been installed, but not configured.
The script will provide a SSH-able system automagically configuring grub (or syslinux), network and restoring the root password from the original system (or by using `vps2arch` as password if no root password was set).

Once done doing its job, the script will ask you to manually reboot your _VPS_ and voilà, PROFIT!

Does it really work?
--------------------

Yes, it does!

On the [Tested VPS Providers](https://gitlab.com/drizzt/vps2arch/wikis/Tested-VPS-Providers) wiki page you can find a list of **Tested VPS Providers**.

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

_IPv6_ currently is not supported. If you need to use it, please configure it manually.
