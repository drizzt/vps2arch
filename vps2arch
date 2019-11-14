#!/bin/sh

# Copyright 2015, Timothy Redaelli <tredaelli@archlinux.info>

# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 2 of the License, or
# (at your option) any later version.

# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License at <http://www.gnu.org/licenses/> for
# more details.

set -e

# Gathering informations about actual environment.
if command -v wget >/dev/null 2>&1; then
	_download() { wget -O- "$@" ; }
elif command -v curl >/dev/null 2>&1; then
	_download() { curl -fL "$@" ; }
else
	echo "This script needs curl or wget" >&2
	exit 2
fi

get_worldwide_mirrors() {
	echo "http://mirrors.kernel.org/archlinux"
	_download 'https://www.archlinux.org/mirrorlist/?country=all&protocol=http&protocol=https&ip_version=4' | awk '/^## /{if ($2 == "Worldwide") { flag=1 } else { flag=0 } } /^#Server/ { if (flag) { sub(/\/\$repo.*/, ""); print $3 } }'
}

cpu_type=$(uname -m)

is_openvz() { [ -d /proc/vz -a ! -d /proc/bc ]; }
is_lxc() { grep -aqw container=lxc /proc/1/environ ; }

download() {
	local path="$1" x=
	shift
	for x in $mirrors; do
		_download "$x/$path" && return 0
	done
	return 1
}

download_and_extract_bootstrap() {
	local sha1 filename
	download iso/latest/sha1sums.txt | fgrep "$cpu_type.tar.gz" > "sha1sums.txt"
	read -r sha1 filename < "sha1sums.txt"
	download "iso/latest/$filename" > "$filename"
	sha1sum -c sha1sums.txt || exit 1
	tar -xpzf "$filename"
	rm -f "$filename"
	cp -L /etc/resolv.conf "/root.$cpu_type/etc"

	# Mount options taked from arch-chroot script
	mount -t proc proc -o nosuid,noexec,nodev "/root.$cpu_type/proc"
	mount -t sysfs sys -o nosuid,noexec,nodev,ro "/root.$cpu_type/sys"
	mount -t devtmpfs -o mode=0755,nosuid udev "/root.$cpu_type/dev"
	mkdir -p "/root.$cpu_type/dev/pts" "/root.$cpu_type/dev/shm"
	mount -t devpts -o mode=0620,gid=5,nosuid,noexec devpts "/root.$cpu_type/dev/pts"
	mount -t tmpfs -o mode=1777,nosuid,nodev shm "/root.$cpu_type/dev/shm"
	mount -t tmpfs -o nosuid,nodev,mode=0755 run "/root.$cpu_type/run"
	mount -t tmpfs -o mode=1777,strictatime,nodev,nosuid tmp "/root.$cpu_type/tmp"
	# FIXME support multiple partitions
	mount --bind / "/root.$cpu_type/mnt"
	findmnt /boot >/dev/null && mount --bind /boot "/root.$cpu_type/mnt/boot"
	# Workaround for Debian
	mkdir -p "/root.$cpu_type/run/shm"
	# Workaround for OpenVZ
	rm -f "/root.$cpu_type/etc/mtab"
	cp -L /etc/mtab "/root.$cpu_type/etc/mtab"
}

chroot_exec() {
	chroot "/root.$cpu_type" /bin/bash -c "$*"
}

configure_chroot() {
	local m
	for m in $mirrors; do
		echo 'Server = '"$m"'/$repo/os/$arch'
	done >> "/root.$cpu_type/etc/pacman.d/mirrorlist"
	# Install and initialize haveged if needed
	if ! is_openvz && ! pidof haveged >/dev/null; then
		# Disable signature check, install and launch haveged and re-enable signature checks.
		sed -i.bak "s/^[[:space:]]*SigLevel[[:space:]]*=.*$/SigLevel = Never/" "/root.$cpu_type/etc/pacman.conf"
		chroot_exec 'pacman --noconfirm -Sy haveged && haveged'
		mv "/root.$cpu_type/etc/pacman.conf.bak" "/root.$cpu_type/etc/pacman.conf"
	fi
	chroot_exec 'pacman-key --init && pacman-key --populate archlinux'
	chroot_exec 'pacman --noconfirm -Sy awk'
	# Generate fstab
	chroot_exec 'genfstab /mnt >> /etc/fstab'

	if is_openvz && [ "$kernelver" '<' '2.6.32-042stab111.1' ]; then
		# Use my repository for OpenVZ-patched systemd
		sed -i 's;^#\[testing\]$;[tredaelli-systemd]\nServer = http://pkgbuild.com/~tredaelli/repo/systemd/$arch\n\n&;' "/root.$cpu_type/etc/pacman.conf"
	fi
}

save_root_pass() {
	grep '^root:' /etc/shadow > "/root.$cpu_type/root.passwd"
	chmod 0600 "/root.$cpu_type/root.passwd"
}

backup_old_files() {
	cp -fL /etc/hostname /etc/localtime "/root.$cpu_type/etc/" || true
}

delete_all() {
	# Remove immutable flag from any files / directories
	if command -v chattr >/dev/null 2>&1; then
		find / -type f \( ! -path '/dev/*' -and ! -path '/proc/*' -and ! -path '/sys/*' -and ! -path '/selinux/*' -and ! -path "/root.$cpu_type/*" \) \
			-exec chattr -i {} + 2>/dev/null || true
	fi
	# Delete *all* files from /
	find / \( ! -path '/dev/*' -and ! -path '/proc/*' -and ! -path '/sys/*' -and ! -path '/selinux/*' -and ! -path "/root.$cpu_type/*" \) -delete 2>/dev/null || true
}

install_packages() {
	local packages="base linux lvm2 openssh reflector"
	[ "$bootloader" != "none" ] && packages="$packages $bootloader"
	# XXX Install gptdisk for syslinux. To be removed then FS#45029 will be closed
	[ "$bootloader" = "syslinux" ] && packages="$packages gptfdisk"
	# Black magic!
	"/root.$cpu_type/usr/lib"/ld-*.so --library-path "/root.$cpu_type/usr/lib" \
		"/root.$cpu_type/usr/bin/chroot" "/root.$cpu_type" /usr/bin/pacstrap -M /mnt $packages
	cp -L "/root.$cpu_type/etc/resolv.conf" /etc
}

restore_root_pass() {
	# If the root password is not set, use vps2arch
	if egrep -q '^root:.?:' "/root.$cpu_type/root.passwd"; then
		echo "root:vps2arch" | chpasswd
	else
		sed -i '/^root:/d' /etc/shadow
		cat "/root.$cpu_type/root.passwd" >> /etc/shadow
	fi
}

cleanup() {
	mv "/root.$cpu_type/etc/fstab" "/etc/fstab"
	awk "/\/root.$cpu_type/ {print \$2}" /proc/mounts | sort -r | xargs umount -nl || true
	rm -rf "/root.$cpu_type/"
}

configure_bootloader() {
	local root_dev=$(findmnt -no SOURCE /) root_devs= tmp= needs_lvm2=0
	case $root_dev in
	/dev/mapper/*) needs_lvm2=1 ;;
	esac

	if [ $needs_lvm2 -eq 1 ]; then
		# Some distro doesn't use lvmetad by default
		sed -i.bak 's/use_lvmetad = 1/use_lvmetad = 0/g' /etc/lvm/lvm.conf
	fi

	if [ "$bootloader" = "grub" ]; then
		# If you are still using eth* as interface name, disable "strange" ifnames
		grep -q '^[[:space:]]*eth' /proc/net/dev && \
			sed -i.bak 's/GRUB_CMDLINE_LINUX_DEFAULT="/&net.ifnames=0 /' /etc/default/grub

		# Disable "graphic" terminal output
		sed -i.bak 's/^#GRUB_TERMINAL_OUTPUT=console/GRUB_TERMINAL_OUTPUT=console/' /etc/default/grub

		if [ $needs_lvm2 -eq 1 ]; then
			local vg
			vg=$(lvs --noheadings $root_dev | awk '{print $2}')
			root_dev=$(pvs --noheadings | awk -v vg="$vg" '($2 == vg) { print $1 }')
		fi
		for root_dev in $root_dev; do
			tmp=$(lsblk -npsro TYPE,NAME "$root_dev" | awk '($1 == "disk") { print $2}')
			case " $root_devs " in
			*" $tmp "*) 	;;
			*)		root_devs="${root_devs:+$root_devs }$tmp"	;;
			esac
		done
		for root_dev in $root_devs; do
			grub-install --target=i386-pc --recheck --force "$root_dev"
		done
		grub-mkconfig > /boot/grub/grub.cfg
	elif [ "$bootloader" = "syslinux" ]; then
		# If you are still using eth* as interface name, disable "strange" ifnames
		grep -q '^[[:space:]]*eth' /proc/net/dev && tmp="net.ifnames=0"
		syslinux-install_update -ami
		sed -i "s;\(^[[:space:]]*APPEND.*\)root=[^[:space:]]*;\1root=$root_dev${tmp:+ $tmp};" /boot/syslinux/syslinux.cfg
	fi

	if [ $needs_lvm2 -eq 1 ]; then
		mv /etc/lvm/lvm.conf.bak /etc/lvm/lvm.conf
		sed -i '/HOOKS/s/block/& lvm2/' /etc/mkinitcpio.conf
		mkinitcpio -p linux
	fi
}

configure_network() {
	local gateway dev ip

	read -r dev gateway <<-EOF
		$(awk '$2 == "00000000" { ip = strtonum(sprintf("0x%s", $3));
			printf ("%s\t%d.%d.%d.%d", $1,
			rshift(and(ip,0x000000ff),00), rshift(and(ip,0x0000ff00),08),
			rshift(and(ip,0x00ff0000),16), rshift(and(ip,0xff000000),24)) ; exit }' < /proc/net/route)
	EOF

	set -- $(ip addr show dev "$dev" | awk '($1 == "inet") { print $2 }')
	ip=$@

	# FIXME Not supported for "P2P" interfaces, such as venet, yet
	if [ "$network" = "systemd-networkd" ]; then
		cat > /etc/systemd/network/default.network <<-EOF
			[Match]
			Name=$dev

			[Network]
			Gateway=$gateway
		EOF
		for ip in $ip; do
			echo "Address=$ip"
		done >> /etc/systemd/network/default.network
		systemctl enable systemd-networkd
	elif [ "$network" = "netctl" ]; then
		cat > /etc/netctl/default <<-EOF
			Interface=$dev
			Connection=ethernet
			IP=static
			Address=($ip)
		EOF
		if [ "$gateway" = "0.0.0.0" ]; then
			echo 'Routes=(0.0.0.0/0)'
		else
			echo "Gateway=$gateway"
		fi >> /etc/netctl/default
		netctl enable default
	fi

	systemctl enable sshd
}

finalize() {
	# OpenVZ hacks
	if is_openvz; then
		local kernelver
		read -r _ _ kernelver _ < /proc/version
		if [ "$kernelver" '>' '3.10' ]; then
			# Virtuozzo 7 works with systemd, but it needs /etc/resolvconf/resolv.conf.d directory
			mkdir -p /etc/resolvconf/resolv.conf.d
		elif [ "$kernelver" '<' '2.6.32-042stab111.1' ]; then
			# Use my repository for OpenVZ-patched systemd
			sed -i 's;^#\[testing\]$;[tredaelli-systemd]\nServer = http://pkgbuild.com/~tredaelli/repo/systemd/$arch\n\n&;' /etc/pacman.conf
		fi
	fi

	# Enable SSH login for user root (#3)
	sed -i '/^#PermitRootLogin\s/s/.*/&\nPermitRootLogin yes/' /etc/ssh/sshd_config
	
	# Run reflector to get updated mirrors
	
	cat <<-EOF
	
		Reflector: Rating the 35 most recently synced HTTPS servers, sorting them by download speed
		
	EOF
	
	reflector -l 35 -p https --sort rate --save /etc/pacman.d/mirrorlist

	cat <<-EOF
		Hi,
		your VM has successfully been reimaged with Arch Linux.

		This script configured $bootloader as bootloader and $network for networking.

		When you are finished with your post-installation, you'll need to reboot the VM the rough way:
		# sync ; reboot -f

		Then you'll be able to connect to your VM using SSH and to login using your old root password (or "vps2arch" if you didn't have a root password).
	EOF
}

bootloader=grub
network=systemd-networkd
mirrors=

while getopts ":b:m:n:h" opt; do
	case $opt in
	b)
		if ! [ "$OPTARG" = "grub" -o "$OPTARG" = "syslinux" -o "$OPTARG" = "none" ]; then
			echo "Invalid bootloader specified" >&2
			exit 1
		fi
		bootloader="$OPTARG"
		;;
	m)
		mirrors="${mirrors:+$mirrors }$OPTARG"
		;;
	n)
		if ! [ "$OPTARG" = "systemd-networkd" -o "$OPTARG" = "netctl" -o "$OPTARG" = "none" ]; then
			echo "Invalid networking configuration system specified" >&2
			exit 1
		fi
		network="$OPTARG"
		;;
	h)
		cat <<-EOF
			usage: ${0##*/} [options]

			  Options:
			    -b (grub|syslinux)           Use the specified bootloader. When this option is omitted, it defaults to grub.
			    -n (systemd-networkd|netctl) Use the specified networking configuration system. When this option is omitted, it defaults to systemd-networkd.
			    -m mirror                    Use the provided mirror (you can specify this option more than once).

			    -h                           Print this help message

			    Warning:
			      On OpenVZ containers the bootloader will be not installed and the networking configuration system will be enforced to netctl.
		EOF
		exit 0
		;;
	:)
		printf "%s: option requires an argument -- '%s'\n" "${0##*/}" "$OPTARG" >&2
		exit 1
		;;
	?)
		printf "%s: invalid option -- '%s'\n" "${0##*/}" "$OPTARG" >&2
		exit 1
		;;
	esac
done
shift $((OPTIND - 1))

[ -z "$mirrors" ] && mirrors=$(get_worldwide_mirrors)


if is_openvz; then
	bootloader=none
	network=netctl
elif is_lxc; then
	bootloader=none
fi

cd /
download_and_extract_bootstrap
configure_chroot
save_root_pass
backup_old_files
delete_all
install_packages
restore_root_pass
cleanup
configure_bootloader
configure_network
finalize
