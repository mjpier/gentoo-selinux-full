# SELinux rules for my custom Gentoo Linux installation (WIP)

Here, SELinux related changes applied to my [custom Gentoo Linux installation](https://github.com/duxsco/gentoo-installation) are documented. This documentation expects for [SELinux already being enabled](https://github.com/duxsco/gentoo-installation#enable-selinux). At this point, the system is in "permissive" mode.

## Enable SELinux

### Kernel command-line parameters

Enable SELinux via kernel command-line parameter in GRUB config:

```bash
rsync -a /etc/default/grub /etc/default/._cfg0000_grub && \
sed -i 's/^GRUB_CMDLINE_LINUX_DEFAULT="\(.*\)"$/GRUB_CMDLINE_LINUX_DEFAULT="\1 lsm=selinux gk.preserverun.disabled=1"/' /etc/default/._cfg0000_grub
```

Append `lsm=selinux gk.preserverun.disabled=1` to the kernel command line parameters in `grub.cfg` and GnuPG sign `grub.conf` in `/efi*`. Alternatively, you can follow the steps in [kernel update](#update-linux-kernel) to rebuild kernel/initramfs and generate new `grub.cfg`. For kernel recreation, I wouldn't delete ccache's cache in order to speed up things.

Reboot the system.

### Relabel

Switch to [mcs policy type](https://wiki.gentoo.org/wiki/SELinux/Policy_store#Switching_active_policy_store) and [relabel the entire system](https://wiki.gentoo.org/wiki/SELinux/Installation#Relabel). But, don't forget to mount `/boot` and all `/efi*` first. Make sure to apply the `setfiles` command on `/boot`, `/var/cache/binpkgs`, `/var/cache/distfiles`, `/var/db/repos/gentoo`, `/var/tmp` and all `/efi*`.

### Users

Add the initial user to the administration SELinux user, and take care of services:
- https://wiki.gentoo.org/wiki/SELinux/Installation#Define_the_administrator_accounts
- https://wiki.gentoo.org/wiki/SELinux/Installation#Supporting_service_administration

In `mcs`, map users to `user_u` by default instead of `unconfined_u`:

```bash
➤ semanage login -l

Login Name           SELinux User         MLS/MCS Range        Service

__default__          unconfined_u         s0-s0                *
➤ semanage login -m -s user_u __default__
➤ semanage login -l

Login Name           SELinux User         MLS/MCS Range        Service

__default__          user_u               s0-s0                *
```

Setup `app-admin/sudo`:

```bash
bash -c 'echo "%wheel ALL=(ALL) TYPE=sysadm_t ROLE=sysadm_r ALL" | EDITOR="tee" visudo -f /etc/sudoers.d/wheel; echo $?'
```

### Logging

Enable logging:

```bash
rc-update add auditd
```

Reboot again.

## OpenRC patch

The OpenRC patch was suggested as a possible solution by [perfinion](https://github.com/perfinion) to fix bootup. Thx for that 🙂 Save the patch:

```bash
➤ tree /etc/portage/patches
/etc/portage/patches
└── sys-apps
    └── openrc
        └── init-early.sh.patch

2 directories, 1 files
```

... and run `emerge -1 sys-apps/openrc`. Such a line should be printed:

```
 * ===========================================================
 * Applying user patches from /etc/portage/patches ...
 * Applying init-early.sh.patch ...                    [ ok ]
 * User patches applied.
 * ===========================================================
```

Reboot the system.

## SSH port label assignment

In the [custom Gentoo Linux installation](https://github.com/duxsco/gentoo-installation), the SSH port has been changed to 50022. This needs to be considered for no SELinux denials to occur:

```bash
➤ semanage port -l | grep -e ssh -e Port
SELinux Port Type              Proto    Port Number
ssh_port_t                     tcp      22
➤ semanage port -a -t ssh_port_t -p tcp 50022
➤ semanage port -l | grep -e ssh -e Port
SELinux Port Type              Proto    Port Number
ssh_port_t                     tcp      50022, 22
```

## GnuPG context assignment

Files in `/boot` and `/efi*` required for booting need to be GnuPG signed (expect for Secure Boot signed EFI binary). For SELinux not to complain, the GnuPG homedir is stored in `/etc/gentoo-installation/gnupg`. In the following, a suitable file context is assigned:

```bash
➤ semanage fcontext -l | grep -e "^/root[[:space:]]" -e "\\\.gnupg"
/home/[^/]+/\.gnupg(/.+)?           all files  user_u:object_r:gpg_secret_t:s0
/home/[^/]+/\.gnupg/S\.gpg-agent.*  socket     user_u:object_r:gpg_agent_tmp_t:s0
/home/[^/]+/\.gnupg/S\.scdaemon     socket     user_u:object_r:gpg_agent_tmp_t:s0
/home/[^/]+/\.gnupg/crls\.d(/.+)?   all files  user_u:object_r:dirmngr_home_t:s0
/home/[^/]+/\.gnupg/log-socket      socket     user_u:object_r:gpg_agent_tmp_t:s0
/home/david/\.gnupg(/.+)?           all files  staff_u:object_r:gpg_secret_t:s0
/home/david/\.gnupg/S\.gpg-agent.*  socket     staff_u:object_r:gpg_agent_tmp_t:s0
/home/david/\.gnupg/S\.scdaemon     socket     staff_u:object_r:gpg_agent_tmp_t:s0
/home/david/\.gnupg/crls\.d(/.+)?   all files  staff_u:object_r:dirmngr_home_t:s0
/home/david/\.gnupg/log-socket      socket     staff_u:object_r:gpg_agent_tmp_t:s0
```

Execute the following in a bash shell:

```bash
semanage fcontext -a -f d -s staff_u -t user_home_dir_t /root
while read -r line; do

    case $(awk '{print $2}' <<<"${line}") in
        regular)
            file_type="f";;
        directory)
            file_type="d";;
        character)
            file_type="c";;
        block)
            file_type="b";;
        socket)
            file_type="s";;
        symbolic)
            file_type="l";;
        named)
            file_type="p";;
        all)
            file_type="a";;
    esac

    selinux_type="$(awk -F':' '{print $(NF-1)}' <<<"${line}")"
    path="$(awk '{print $1}' <<<"${line}")"

    semanage fcontext -a -f "${file_type}" -s staff_u -t "${selinux_type}" "$(sed 's|home/\[\^/\]+/\\\.gnupg|root/\\\.gnupg|' <<<${path})"
    semanage fcontext -a -f "${file_type}" -s staff_u -t "${selinux_type}" "$(sed 's|home/\[\^/\]+/\\\.gnupg|etc/gentoo-installation/gnupg|' <<<${path})"
done < <(semanage fcontext -l | grep "/home/\[\^/\]+/\\\.gnupg")
```

Result:

```bash
➤ semanage fcontext -l | grep -e "^/root[[:space:]]" -e "\\\.gnupg" -e "/etc/gentoo-installation/gnupg"
/etc/gentoo-installation/gnupg(/.+)?           all files  staff_u:object_r:gpg_secret_t:s0
/etc/gentoo-installation/gnupg/S\.gpg-agent.*  socket     staff_u:object_r:gpg_agent_tmp_t:s0
/etc/gentoo-installation/gnupg/S\.scdaemon     socket     staff_u:object_r:gpg_agent_tmp_t:s0
/etc/gentoo-installation/gnupg/crls\.d(/.+)?   all files  staff_u:object_r:dirmngr_home_t:s0
/etc/gentoo-installation/gnupg/log-socket      socket     staff_u:object_r:gpg_agent_tmp_t:s0
/home/[^/]+/\.gnupg(/.+)?                      all files  user_u:object_r:gpg_secret_t:s0
/home/[^/]+/\.gnupg/S\.gpg-agent.*             socket     user_u:object_r:gpg_agent_tmp_t:s0
/home/[^/]+/\.gnupg/S\.scdaemon                socket     user_u:object_r:gpg_agent_tmp_t:s0
/home/[^/]+/\.gnupg/crls\.d(/.+)?              all files  user_u:object_r:dirmngr_home_t:s0
/home/[^/]+/\.gnupg/log-socket                 socket     user_u:object_r:gpg_agent_tmp_t:s0
/home/david/\.gnupg(/.+)?                      all files  staff_u:object_r:gpg_secret_t:s0
/home/david/\.gnupg/S\.gpg-agent.*             socket     staff_u:object_r:gpg_agent_tmp_t:s0
/home/david/\.gnupg/S\.scdaemon                socket     staff_u:object_r:gpg_agent_tmp_t:s0
/home/david/\.gnupg/crls\.d(/.+)?              all files  staff_u:object_r:dirmngr_home_t:s0
/home/david/\.gnupg/log-socket                 socket     staff_u:object_r:gpg_agent_tmp_t:s0
/root                                          directory  staff_u:object_r:user_home_dir_t:s0
/root/\.gnupg(/.+)?                            all files  staff_u:object_r:gpg_secret_t:s0
/root/\.gnupg/S\.gpg-agent.*                   socket     staff_u:object_r:gpg_agent_tmp_t:s0
/root/\.gnupg/S\.scdaemon                      socket     staff_u:object_r:gpg_agent_tmp_t:s0
/root/\.gnupg/crls\.d(/.+)?                    all files  staff_u:object_r:dirmngr_home_t:s0
/root/\.gnupg/log-socket                       socket     staff_u:object_r:gpg_agent_tmp_t:s0
```

Restore:

```bash
restorecon -RFv /root /etc/gentoo-installation/
```

## Non-policy based fixes

Executing `create_policy.sh` resulted in following `ausearch` outputs with non-policy based fixes shows in the following.

### sys-process/cronie

`ausearch` output:

```
----
time->Fri Apr 29 23:25:34 2022
type=PROCTITLE msg=audit(1651267534.933:7): proctitle="/usr/sbin/crond"
type=PATH msg=audit(1651267534.933:7): item=0 name="/var/spool/cron/crontabs/root" inode=641849 dev=00:1c mode=0100600 ouid=0 ogid=460 rdev=00:00 obj=system_u:object_r:unlabeled_t nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1651267534.933:7): cwd="/"
type=SYSCALL msg=audit(1651267534.933:7): arch=c000003e syscall=262 success=yes exit=0 a0=ffffff9c a1=7ffdefcdd160 a2=7ffdefcdd0d0 a3=0 items=1 ppid=1 pid=5437 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="crond" exe="/usr/sbin/crond" subj=system_u:system_r:crond_t key=(null)
type=AVC msg=audit(1651267534.933:7): avc:  denied  { getattr } for  pid=5437 comm="crond" path="/var/spool/cron/crontabs/root" dev="dm-0" ino=641849 scontext=system_u:system_r:crond_t tcontext=system_u:object_r:unlabeled_t tclass=file permissive=1
----
time->Fri Apr 29 23:25:34 2022
type=PROCTITLE msg=audit(1651267534.933:8): proctitle="/usr/sbin/crond"
type=PATH msg=audit(1651267534.933:8): item=0 name="/var/spool/cron/crontabs/root" inode=641849 dev=00:1c mode=0100600 ouid=0 ogid=460 rdev=00:00 obj=system_u:object_r:unlabeled_t nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1651267534.933:8): cwd="/"
type=SYSCALL msg=audit(1651267534.933:8): arch=c000003e syscall=257 success=yes exit=7 a0=ffffff9c a1=7ffdefcdd5d0 a2=800 a3=0 items=1 ppid=1 pid=5437 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="crond" exe="/usr/sbin/crond" subj=system_u:system_r:crond_t key=(null)
type=AVC msg=audit(1651267534.933:8): avc:  denied  { open } for  pid=5437 comm="crond" path="/var/spool/cron/crontabs/root" dev="dm-0" ino=641849 scontext=system_u:system_r:crond_t tcontext=system_u:object_r:unlabeled_t tclass=file permissive=1
type=AVC msg=audit(1651267534.933:8): avc:  denied  { read } for  pid=5437 comm="crond" name="root" dev="dm-0" ino=641849 scontext=system_u:system_r:crond_t tcontext=system_u:object_r:unlabeled_t tclass=file permissive=1
```

Policies:

```bash
➤ sesearch --allow --source crond_t --class file --perm getattr,open,read | grep "_cron_spool_t"
allow crond_t system_cron_spool_t:file { append create getattr ioctl link lock open read rename setattr unlink write }; [ fcron_crond ]:True
allow crond_t system_cron_spool_t:file { getattr ioctl lock open read watch };
allow crond_t user_cron_spool_t:file { append create getattr ioctl link lock open read rename setattr unlink write }; [ fcron_crond ]:True
allow crond_t user_cron_spool_t:file { getattr ioctl lock open read };
```

List file context mapping definitions:

```bash
➤ semanage fcontext -l | grep "/var/spool/cron/crontabs"
/var/spool/cron/crontabs                           directory          system_u:object_r:cron_spool_t
/var/spool/cron/crontabs/.*                        regular file       <<None>>
/var/spool/cron/crontabs/munin                     regular file       system_u:object_r:system_cron_spool_t
```

Modify:

```bash
➤ semanage fcontext -m -f f -s system_u -r object_r -t user_cron_spool_t "/var/spool/cron/crontabs/.*"
```

Restore:

```bash
➤ restorecon -F -v /var/spool/cron/crontabs/*
Relabeled /var/spool/cron/crontabs/root from system_u:object_r:unlabeled_t to system_u:object_r:user_cron_spool_t
```

### net-firewall/nftables

`ausearch` output:

```
----
time->Wed May 11 01:42:40 2022
type=PROCTITLE msg=audit(1652226160.839:7): proctitle=6E6674002D63002D66002F7661722F6C69622F6E667461626C65732F72756C65732D73617665
type=PATH msg=audit(1652226160.839:7): item=0 name="/var/lib/nftables/rules-save" inode=945463 dev=00:1d mode=0100600 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:var_lib_t:s0 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1652226160.839:7): cwd="/"
type=SYSCALL msg=audit(1652226160.839:7): arch=c000003e syscall=257 success=yes exit=4 a0=ffffff9c a1=7ffe15d4cb06 a2=0 a3=0 items=1 ppid=5458 pid=5459 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nft" exe="/sbin/nft" subj=system_u:system_r:iptables_t:s0 key=(null)
type=AVC msg=audit(1652226160.839:7): avc:  denied  { open } for  pid=5459 comm="nft" path="/var/lib/nftables/rules-save" dev="dm-0" ino=945463 scontext=system_u:system_r:iptables_t:s0 tcontext=system_u:object_r:var_lib_t:s0 tclass=file permissive=1
type=AVC msg=audit(1652226160.839:7): avc:  denied  { read } for  pid=5459 comm="nft" name="rules-save" dev="dm-0" ino=945463 scontext=system_u:system_r:iptables_t:s0 tcontext=system_u:object_r:var_lib_t:s0 tclass=file permissive=1
----
time->Wed May 11 01:42:40 2022
type=PROCTITLE msg=audit(1652226160.839:8): proctitle=6E6674002D63002D66002F7661722F6C69622F6E667461626C65732F72756C65732D73617665
type=SYSCALL msg=audit(1652226160.839:8): arch=c000003e syscall=16 success=no exit=-25 a0=4 a1=5401 a2=7ffe15d4c240 a3=20000000 items=0 ppid=5458 pid=5459 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nft" exe="/sbin/nft" subj=system_u:system_r:iptables_t:s0 key=(null)
type=AVC msg=audit(1652226160.839:8): avc:  denied  { ioctl } for  pid=5459 comm="nft" path="/var/lib/nftables/rules-save" dev="dm-0" ino=945463 ioctlcmd=0x5401 scontext=system_u:system_r:iptables_t:s0 tcontext=system_u:object_r:var_lib_t:s0 tclass=file permissive=1
----
time->Wed May 11 01:42:40 2022
type=PROCTITLE msg=audit(1652226160.843:9): proctitle=6E6674002D63002D66002F7661722F6C69622F6E667461626C65732F72756C65732D73617665
type=PATH msg=audit(1652226160.843:9): item=0 name="" inode=945463 dev=00:1d mode=0100600 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:var_lib_t:s0 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1652226160.843:9): cwd="/"
type=SYSCALL msg=audit(1652226160.843:9): arch=c000003e syscall=262 success=yes exit=0 a0=4 a1=7fe2cdf8ef15 a2=7ffe15d35930 a3=1000 items=1 ppid=5458 pid=5459 auid=4294967295 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=(none) ses=4294967295 comm="nft" exe="/sbin/nft" subj=system_u:system_r:iptables_t:s0 key=(null)
type=AVC msg=audit(1652226160.843:9): avc:  denied  { getattr } for  pid=5459 comm="nft" path="/var/lib/nftables/rules-save" dev="dm-0" ino=945463 scontext=system_u:system_r:iptables_t:s0 tcontext=system_u:object_r:var_lib_t:s0 tclass=file permissive=1
```

List file context mapping definitions:

```bash
➤ semanage fcontext -l | grep "/var/lib/.*tables"
/var/lib/ip6?tables(/.*)?                          all files          system_u:object_r:initrc_tmp_t:s0
```

Policies:

```bash
➤ sesearch --allow --source iptables_t --target initrc_tmp_t --class file --perm getattr,ioctl,open,read
allow iptables_t initrc_tmp_t:file { append getattr ioctl lock open read write };
```

Modify:

```bash
semanage fcontext -a -s system_u -r s0 -t initrc_tmp_t "/var/lib/nftables(/.*)?"
```

Restore:

```bash
➤ restorecon -R -F -v /var/lib/nftables
Relabeled /var/lib/nftables from system_u:object_r:var_lib_t:s0 to system_u:object_r:initrc_tmp_t:s0
Relabeled /var/lib/nftables/.keep_net-firewall_nftables-0 from system_u:object_r:var_lib_t:s0 to system_u:object_r:initrc_tmp_t:s0
Relabeled /var/lib/nftables/rules-save from system_u:object_r:var_lib_t:s0 to system_u:object_r:initrc_tmp_t:s0
```

## Creating SELinux policies

I created a script to simplify policy creation for denials printed out by `dmesg` and `ausearch`. Reboot after `semodule -i ...` and create the next SELinux policy. The script creates the `.te` file in the current directory!

```bash
➤ bash ../create_policy.sh
"my_00001_permissive_dmesg-systemd_tmpfiles_t-self.te" has been created!

Please, check the file, create the policy module and install it:
make -f /usr/share/selinux/strict/include/Makefile my_00001_permissive_dmesg-systemd_tmpfiles_t-self.pp
semodule -i my_00001_permissive_dmesg-systemd_tmpfiles_t-self.pp
```

In certain cases, a warning is printed and no `.te` file is created:

```bash
➤ bash ../create_policy.sh
audit2allow printed a warning:


#============= systemd_tmpfiles_t ==============

#!!!! This avc can be allowed using the boolean 'systemd_tmpfiles_manage_all'
allow systemd_tmpfiles_t portage_cache_t:dir { getattr open read relabelfrom relabelto };

Aborting...
```

The policies created with `create_policy.sh` are in the "policy" folder. In addition to building and installing them, enable following booleans:

```bash
setsebool -P allow_mount_anyfile on
setsebool -P systemd_tmpfiles_manage_all on
setsebool -P gpg_read_all_user_content on
setsebool -P gpg_manage_all_user_content on
```
