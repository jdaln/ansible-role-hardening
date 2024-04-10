# Hardening - the Ansible role

An [Ansible](https://www.ansible.com/) role to make a AlmaLinux, Debian, or
Ubuntu server a bit more secure.
[systemd edition](https://freedesktop.org/wiki/Software/systemd/).

Requires Ansible >= 2.15.

[AlmaLinux 8](https://wiki.almalinux.org/release-notes/#almalinux-8),
[AlmaLinux 9](https://wiki.almalinux.org/release-notes/#almalinux-9),
[Debian 11](https://www.debian.org/releases/bullseye/),
[Debian 12](https://www.debian.org/releases/bookworm/),
[Ubuntu 20.04](https://releases.ubuntu.com/focal/) and
[Ubuntu 22.04](https://releases.ubuntu.com/jammy/) are supported.

There are also hardened Ubuntu AWS Amazon Machine Images and Azure virtual
machine images available in the [konstruktoid/hardened-images](https://github.com/konstruktoid/hardened-images)
repository, built using [Packer](https://www.packer.io/) and this role.

> **Note**
>
> Do not use this role without first testing in a non-operational environment.

> **Note**
>
> There is a [SLSA](https://slsa.dev/) artifact present under the
> [slsa action workflow](https://github.com/konstruktoid/ansible-role-hardening/actions/workflows/slsa.yml)
> for verification.

## Dependencies

None.

## Examples

### Requirements

```yaml
---
roles:
  - name: konstruktoid.hardening
    version: v2.0.4
    src: https://github.com/konstruktoid/ansible-role-hardening.git
    scm: git
```

### Playbook

```yaml
---
- name: Import and use the hardening role
  hosts: localhost
  any_errors_fatal: true
  tasks:
    - name: Import the hardening role
      ansible.builtin.import_role:
        name: konstruktoid.hardening
      vars:
        sshd_admin_net:
          - 10.0.2.0/24
          - 192.168.0.0/24
          - 192.168.1.0/24
        suid_sgid_permissions: false
```

### Local playbook using git checkout

```yaml
---
- name: Checkout and configure konstrukoid.hardening
  hosts: localhost
  any_errors_fatal: true
  tasks:
    - name: Clone hardening repository
      become: true
      tags:
        - always
      block:
        - name: Install git
          ansible.builtin.package:
            name: git
            state: present

        - name: Checkout konstruktoid.hardening
          become: true
          ansible.builtin.git:
            repo: https://github.com/konstruktoid/ansible-role-hardening
            dest: /etc/ansible/roles/konstruktoid.hardening
            version: v2.0.4

        - name: Remove git
          ansible.builtin.package:
            name: git
            state: absent

    - name: Include the hardening role
      ansible.builtin.include_role:
        name: konstruktoid.hardening
      vars:
        sshd_allow_groups:
          - ubuntu
          - vagrant
        sshd_login_grace_time: 60
        sshd_max_auth_tries: 10
        sshd_use_dns: false
        sshd_update_moduli: true
```

## Note regarding UFW firewall rules

Instead of resetting `ufw` every run and by doing so causing network traffic
disruption, the role deletes every `ufw` rule without
`comment: ansible managed` task parameter and value.

The role also sets default deny policies, which means that firewall rules
needs to be created for any additional ports except those specified in
the `sshd_ports` and `ufw_outgoing_traffic` variables.

## Task Execution and Structure

See [STRUCTURE.md](STRUCTURE.md) for tree of the role structure.

## Role testing

See [TESTING.md](TESTING.md).

## Role Variables with defaults

### ./defaults/main/aide.yml

```yaml
manage_aide: true

aide_checksums: sha512
```

If `manage_aide: true`, then [AIDE](https://aide.github.io/) will be installed
and configured.

`aide_checksums` modifies the AIDE `Checksums` variable. Note that the
`Checksums` variable might not be present depending on distribution.

[aide.conf(5)](https://linux.die.net/man/5/aide.conf)

### ./defaults/main/auditd.yml

```yaml
manage_auditd: true

auditd_apply_audit_rules: true
auditd_action_mail_acct: root
auditd_admin_space_left_action: suspend
auditd_disk_error_action: suspend
auditd_disk_full_action: suspend
auditd_enable_flag: 2
auditd_flush: incremental_async
auditd_max_log_file: 20
auditd_max_log_file_action: rotate
auditd_mode: 1
auditd_num_logs: 5
auditd_space_left: 75
auditd_space_left_action: email
grub_audit_backlog_cmdline: audit_backlog_limit=8192
grub_audit_cmdline: audit=1
```

If `manage_auditd: true`, then the [Linux Audit System](https://github.com/linux-audit/audit-userspace)
will configured and enabled at boot using GRUB.

When `auditd_apply_audit_rules: 'yes'`, the role applies the auditd rules
from the included template file.

`auditd_action_mail_acct` should be a valid email address or alias.

`auditd_admin_space_left_action` defines what action to take when the system has
detected that it is low on disk space. `suspend` will cause the audit daemon to
stop writing records to the disk.

`auditd_enable_flag` sets the enabled flag. If `0` is passed, temporarily disable auditing.
`1` will enable auditing and `2` will lock the audit configuration.

`auditd_flush: sync` tells the audit daemon to keep both the data and meta-data
fully sync'd with every write to disk.

`auditd_max_log_file_action` sets what action to take when the system has
detected that the max file size limit has been reached. E.g. the `rotate` option
will cause the audit daemon to rotate the logs. The `keep_logs` option is
similar to `rotate` except it does not use the `num_logs` setting. This prevents
audit logs from being overwritten.

`auditd_space_left_action` tells the system what action to take when the system
has detected that it is low on disk space. `email` means that it will send a
warning to the email account specified in `action_mail_acct` as well as
sending the message to syslog.

`auditd_mode` sets `auditd` failure mode, 0=silent 1=printk 2=panic.

[auditd.conf(5)](https://man7.org/linux/man-pages/man5/auditd.conf.5.html)

### ./defaults/main/compilers.yml

```yaml
manage_compilers: true

compilers:
  - as
  - cargo
  - cc
  - cc-[0-9]*
  - clang-[0-9]*
  - gcc
  - gcc-[0-9]*
  - go
  - make
  - rustc
```

If `manage_compilers: true`, then the listed compilers will be restricted
to the root user.

### ./defaults/main/crypto_policies.yml

```yaml
set_crypto_policy: true
crypto_policy: DEFAULT:NO-SHA1
```

Set and use [cryptographic policies](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/security_hardening/using-the-system-wide-cryptographic-policies_security-hardening)
if `/etc/crypto-policies/config` exists and `set_crypto_policy: true`.

### ./defaults/main/disablewireless.yml

```yaml
disable_wireless: false
```

If `true`, turn off all wireless interfaces.

### ./defaults/main/dns.yml

```yaml
manage_resolved: true

dns:
  - 1.1.1.2
  - 9.9.9.9
fallback_dns:
  - 1.0.0.2
  - 149.112.112.112
dnssec: allow-downgrade
dns_over_tls: opportunistic
```

If `manage_resolved: true`, configure [systemd-resolved](https://www.freedesktop.org/software/systemd/man/latest/resolved.conf.html).

IPv4 and IPv6 addresses to use as system and fallback DNS servers.
If `dnssec` is set to "allow-downgrade" DNSSEC validation is attempted, but if
the server does not support DNSSEC properly, DNSSEC mode is automatically
disabled.

If `dns_over_tls` is true, all connections to the server will be encrypted if
the DNS server supports DNS-over-TLS and has a valid certificate.

### ./defaults/main/ipv6.yml

```yaml
disable_ipv6: false
sysctl_net_ipv6_conf_accept_ra_rtr_pref: 0

ipv6_disable_sysctl_settings:
  net.ipv6.conf.all.disable_ipv6: 1
  net.ipv6.conf.default.disable_ipv6: 1

ipv6_sysctl_settings:
  net.ipv6.conf.all.accept_ra: 0
  net.ipv6.conf.all.accept_redirects: 0
  net.ipv6.conf.all.accept_source_route: 0
  net.ipv6.conf.all.forwarding: 0
  net.ipv6.conf.all.use_tempaddr: 2
  net.ipv6.conf.default.accept_ra: 0
  net.ipv6.conf.default.accept_ra_defrtr: 0
  net.ipv6.conf.default.accept_ra_pinfo: 0
  net.ipv6.conf.default.accept_ra_rtr_pref: 0
  net.ipv6.conf.default.accept_redirects: 0
  net.ipv6.conf.default.accept_source_route: 0
  net.ipv6.conf.default.autoconf: 0
  net.ipv6.conf.default.dad_transmits: 0
  net.ipv6.conf.default.max_addresses: 1
  net.ipv6.conf.default.router_solicitations: 0
  net.ipv6.conf.default.use_tempaddr: 2
```

If `disable_ipv6: true`, IPv6 will be disabled and related sysctl settings
configured.

`ipv6_sysctl_settings` is the `sysctl` configuration used if the host is using
IPv6.

[sysctl.conf](https://linux.die.net/man/5/sysctl.conf)

### ./defaults/main/journal.yml

```yaml
rsyslog_filecreatemode: "0640"

journald_compress: true
journald_forwardtosyslog: false
journald_storage: persistent
```

`rsyslog_filecreatemode` sets the creation mode with which rsyslogd creates
new files, see
[rsconf1_filecreatemode](https://www.rsyslog.com/doc/configuration/action/rsconf1_filecreatemode.html).

`journald_compress` can take a boolean value. If enabled (the default), data
objects are compressed before they are written to the file system.

`journald_storage` controls where to store journal data. One of `volatile`,
`persistent`, `auto` and `none`.

`journald_forwardtosyslog` control whether log messages received by the journal
daemon shall be forwarded to a traditional syslog daemon.

See [journald.conf](https://www.freedesktop.org/software/systemd/man/latest/journald.conf.html)
for more information.

### ./defaults/main/limits.yml

```yaml
limit_nofile_hard: 1024
limit_nofile_soft: 512
limit_nproc_hard: 1024
limit_nproc_soft: 512
```

Set maximum number of processes and open files.

### ./defaults/main/misc.yml

```yaml
reboot_ubuntu: false
redhat_signing_keys:
  - 567E347AD0044ADE55BA8A5F199E2F91FD431D51
  - 47DB287789B21722B6D95DDE5326810137017186
epel7_signing_keys:
  - 91E97D7C4A5E96F17F3E888F6A2FAEA2352C64E5
epel8_signing_keys:
  - 94E279EB8D8F25B21810ADF121EA45AB2F86D6A1
epel9_signing_keys:
  - FF8AD1344597106ECE813B918A3872BF3228467C
```

If `reboot_ubuntu: true` an Ubuntu node will be rebooted if required.

`redhat_signing_keys` are the [RedHat Product Signing Keys](https://access.redhat.com/security/team/key/).

The `epel7_signing_keys`, `epel8_signing_keys` and `epel9_signing_keys` are
release specific [Fedora EPEL signing keys](https://getfedora.org/security/).

### ./defaults/main/module_blocklists.yml

```yaml
fs_modules_blocklist:
  - cramfs
  - freevxfs
  - hfs
  - hfsplus
  - jffs2
  - squashfs
  - udf

misc_modules_blocklist:
  - bluetooth
  - bnep
  - btusb
  - can
  - cpia2
  - firewire-core
  - floppy
  - ksmbd
  - n_hdlc
  - net-pf-31
  - pcspkr
  - soundcore
  - thunderbolt
  - usb-midi
  - usb-storage
  - uvcvideo
  - v4l2_common

net_modules_blocklist:
  - atm
  - dccp
  - sctp
  - rds
  - tipc
```

Kernel modules to be [blacklisted](https://wiki.debian.org/KernelModuleBlacklisting)
and disabled using a fake install.

> **Note**
>
> Disabling the `usb-storage` module will disable all USB
> storage devices. If such devices are needed [USBGuard](#defaultsmainusbguardyml),
> or a similar tool, should be configured accordingly.

### ./defaults/main/mount.yml

```yaml
hide_pid: 2
process_group: root
```

`hide_pid` sets `/proc/<pid>/` access mode.

The `process_group` setting configures the group authorized to learn processes
information otherwise prohibited by `hidepid=`.

[/proc mount options](https://www.kernel.org/doc/html/latest/filesystems/proc.html#mount-options)

### ./defaults/main/ntp.yml

```yaml
manage_timesyncd: true

fallback_ntp:
  - ntp.netnod.se
  - ntp.ubuntu.com
ntp:
  - 2.pool.ntp.org
  - time.nist.gov
```

If `manage_timesyncd: true`, then configure systemd
[timesyncd](https://manpages.ubuntu.com/manpages/jammy/man8/systemd-timesyncd.service.8.html),
otherwise installing a NTP client is recommended.

### ./defaults/main/packagemgmt.yml

```yaml
apt_hardening_options:
  - Acquire::AllowDowngradeToInsecureRepositories "false";
  - Acquire::AllowInsecureRepositories "false";
  - Acquire::http::AllowRedirect "false";
  - APT::Get::AllowUnauthenticated "false";
  - APT::Get::AutomaticRemove "true";
  - APT::Install-Recommends "false";
  - APT::Install-Suggests "false";
  - APT::Periodic::AutocleanInterval "7";
  - APT::Sandbox::Seccomp "1";
  - Unattended-Upgrade::Remove-Unused-Dependencies "true";
  - Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
```

Configure the [APT suite of tools](https://manpages.debian.org/bookworm/apt/apt.conf.5.en.html).

### ./defaults/main/packages.yml

```yaml
automatic_updates: true
system_upgrade: true

packages_blocklist:
  - apport*
  - autofs
  - avahi*
  - avahi-*
  - beep
  - git
  - pastebinit
  - popularity-contest
  - prelink
  - rpcbind
  - rsh*
  - rsync
  - talk*
  - telnet*
  - tftp*
  - tuned
  - whoopsie
  - xinetd
  - yp-tools
  - ypbind

packages_debian:
  - acct
  - apparmor-profiles
  - apparmor-utils
  - apt-show-versions
  - audispd-plugins
  - auditd
  - cracklib-runtime
  - debsums
  - gnupg2
  - haveged
  - libpam-apparmor
  - libpam-cap
  - libpam-modules
  - libpam-pwquality
  - libpam-tmpdir
  - lsb-release
  - needrestart
  - openssh-server
  - postfix
  - rsyslog
  - sysstat
  - systemd-journal-remote
  - tcpd
  - vlock
  - wamerican

packages_redhat:
  - audispd-plugins
  - audit
  - cracklib
  - gnupg2
  - haveged
  - libpwquality
  - openssh-server
  - needrestart
  - postfix
  - psacct
  - python3-dnf-plugin-post-transaction-actions
  - rsyslog
  - rsyslog-gnutls
  - systemd-journal-remote
  - vlock
  - words

packages_ubuntu:
  - fwupd
  - secureboot-db
  - snapd
```

`automatic_updates: true` will install and configure
[dnf-automatic](https://dnf.readthedocs.io/en/latest/automatic.html)
or [unattended-upgrades](https://wiki.debian.org/UnattendedUpgrades),
depending on the distribution.

`system_upgrade: true` will run `apt upgrade` or
`dnf update` if required.

Packages to be installed depending of distribution
and packages to be removed (`packages_blocklist`).

### ./defaults/main/password.yml

```yaml
manage_faillock: true

faillock:
  admin_group: []
  audit: true
  deny: 5
  dir: /var/run/faillock
  even_deny_root: true
  fail_interval: 900
  local_users_only: true
  no_log_info: false
  nodelay: true
  root_unlock_time: 600
  silent: false
  unlock_time: 600

login_defs:
  login_retries: 5
  login_timeout: 60
  pass_max_days: 60
  pass_min_days: 1
  pass_warn_age: 7

password_remember: 5

pwquality:
  dcredit: -1
  dictcheck: 1
  dictpath: ""
  difok: 8
  enforce_for_root: true
  enforcing: 1
  gecoscheck: 1
  lcredit: -1
  local_users_only: true
  maxclassrepeat: 4
  maxrepeat: 3
  minclass: 4
  minlen: 15
  ocredit: -1
  retry: 3
  ucredit: -1
  usercheck: 1
  usersubstr: 3
```

`manage_faillock: true` will enable the faillock library.

`password_remember` set the size of the password history that the user will not be able to reuse.

Configure the [pam_faillock](https://manpages.ubuntu.com/manpages/lunar/en/man5/faillock.conf.5.html) library.

Configure the [login.defs](https://manpages.ubuntu.com/manpages/lunar/en/man5/login.defs.5.html) configuration.

Configure the [libpwquality](https://manpages.ubuntu.com/manpages/noble/man5/pwquality.conf.5.html) library.

### ./defaults/main/rkhunter.yml

```yaml
manage_rkhunter: true

rkhunter_allow_ssh_prot_v1: false
rkhunter_allow_ssh_root_user: false
rkhunter_mirrors_mode: "0"
rkhunter_update_mirrors: true
rkhunter_web_cmd: curl -fsSL
```

If `manage_rkhunter: true`, then configure [rkhunter](https://rkhunter.sourceforge.net/).

### ./defaults/main/sshd.yml

```yaml
manage_ssh: true

sshd_accept_env: LANG LC_*
sshd_admin_net:
  - 192.168.0.0/24
  - 192.168.1.0/24
sshd_allow_agent_forwarding: false
sshd_allow_groups:
  - sudo
sshd_allow_tcp_forwarding: false
sshd_allow_users:
  - "{{ ansible_user | default(lookup('ansible.builtin.env', 'USER')) }}"
sshd_authentication_methods: any
sshd_authorized_principals_file: /etc/ssh/auth_principals/%u
sshd_banner: /etc/issue.net
sshd_ca_signature_algorithms:
  - ecdsa-sha2-nistp256
  - ecdsa-sha2-nistp384
  - ecdsa-sha2-nistp521
  - ssh-ed25519
  - rsa-sha2-256
  - rsa-sha2-512
  - ssh-rsa
sshd_kbd_interactive_authentication: false
sshd_ciphers:
  - chacha20-poly1305@openssh.com
  - aes256-gcm@openssh.com
  - aes256-ctr
sshd_client_alive_count_max: 1
sshd_client_alive_interval: 200
sshd_compression: false
sshd_config_d_force_clear: false
sshd_config_force_replace: false
sshd_debian_banner: false
sshd_deny_groups: []
sshd_deny_users: []
sshd_gssapi_authentication: false
sshd_host_key_algorithms:
  - ssh-ed25519-cert-v01@openssh.com
  - ssh-rsa-cert-v01@openssh.com
  - ssh-ed25519
  - ssh-rsa
  - ecdsa-sha2-nistp521-cert-v01@openssh.com
  - ecdsa-sha2-nistp384-cert-v01@openssh.com
  - ecdsa-sha2-nistp256-cert-v01@openssh.com
  - ecdsa-sha2-nistp521
  - ecdsa-sha2-nistp384
  - ecdsa-sha2-nistp256
sshd_host_keys_files: []
sshd_host_keys_group: root
sshd_host_keys_mode: "0600"
sshd_host_keys_owner: root
sshd_hostbased_authentication: false
sshd_ignore_rhosts: true
sshd_ignore_user_known_hosts: true
sshd_kerberos_authentication: false
sshd_kex_algorithms:
  - curve25519-sha256@libssh.org
  - ecdh-sha2-nistp521
  - ecdh-sha2-nistp384
  - ecdh-sha2-nistp256
  - diffie-hellman-group-exchange-sha256
sshd_listen:
  - "0.0.0.0"
sshd_log_level: VERBOSE
sshd_login_grace_time: 20
sshd_macs:
  - hmac-sha2-512-etm@openssh.com
  - hmac-sha2-256-etm@openssh.com
  - hmac-sha2-512
  - hmac-sha2-256
sshd_match_addresses: {}
sshd_match_groups: {}
sshd_match_local_ports: {}
sshd_match_users: {}
sshd_max_auth_tries: 3
sshd_max_sessions: 3
sshd_max_startups: 10:30:60
sshd_password_authentication: false
sshd_permit_empty_passwords: false
sshd_permit_root_login: false
sshd_permit_tunnel: false
sshd_permit_user_environment: false
sshd_ports:
  - 22
sshd_print_last_log: true
sshd_print_motd: false
sshd_print_pam_motd: false
sshd_rekey_limit: 512M 1h
sshd_required_ecdsa_size: 521
sshd_required_rsa_size: 4096
sshd_sftp_enabled: true
sshd_sftp_only_chroot: true
sshd_sftp_only_chroot_dir: "%h"
sshd_sftp_only_group: ""
sshd_sftp_subsystem: internal-sftp -f LOCAL6 -l INFO
sshd_strict_modes: true
sshd_syslog_facility: AUTH
sshd_tcp_keep_alive: false
sshd_trusted_user_ca_keys_base64: ""
sshd_trusted_user_ca_keys_file: /etc/ssh/trusted-user-ca-keys.pem
sshd_update_moduli: false
sshd_use_dns: false
sshd_use_pam: true
sshd_use_privilege_separation: sandbox
sshd_x11_forwarding: false
```

If `manage_ssh: true` the SSH client and the SSH daemon will be configured.

> **Note**
>
> `CASignatureAlgorithms`, `Ciphers`, `HostKeyAlgorithms`, `KexAlgorithms` and `MACs`
> will be configured as defined by cryptographic policies if
> `/etc/crypto-policies/config` exists and `set_crypto_policy: true`.

For a explanation of the options not described below, please read
[https://man.openbsd.org/sshd_config](https://man.openbsd.org/sshd_config).

Only the network(s) defined in `sshd_admin_net` are allowed to connect to
`sshd_ports`.
Note that additional rules need to be set up in order to allow access to
additional services.

OpenSSH login is allowed only for users whose primary group or supplementary
group list matches one of the patterns in `sshd_allow_groups`. OpenSSH login
is also allowed for users in `sshd_allow_users`. To do the opposite and deny
access, use the `sshd_deny_groups` and `sshd_deny_users` parameters, which in
turn have priority over the previous parameters.

`sshd_allow_agent_forwarding` specifies whether ssh-agent(1) forwarding is
permitted.

`sshd_allow_tcp_forwarding` specifies whether TCP forwarding is permitted.
The available options are `true` or `all` to allow TCP forwarding,
`false` to prevent all TCP forwarding, `local` to allow local (from the
perspective of ssh(1)) forwarding only or `remote` to allow remote forwarding
only.

`sshd_authentication_methods` specifies the authentication methods that must
be successfully completed in order to grant access to a user.

`sshd_log_level` gives the verbosity level that is used when logging messages.

`sshd_max_auth_tries` and `sshd_max_sessions` specifies the maximum number of
SSH authentication attempt s permitted per connection and the maximum number of
open shell, login or subsystem (e.g. `sftp`) sessions permitted per network
connection.

`sshd_password_authentication` specifies whether password authentication is
allowed.

`sshd_ports` specifies the port(s) number that sshd(8) listens on.

`sshd_required_rsa_size`, `RequiredRSASize`, will only be set if SSH version
is higher than 9.1.

`sshd_config_d_force_clear` force clear directory `/etc/ssh/sshd_config.d`.
Default: `false`.

`sshd_config_force_replace` force replace configuration file
`/etc/ssh/sshd_config`. Default: `false`.

> **Note**
>
> By default, the role checks whether the directory `/etc/ssh/sshd_config.d`
> exists and whether it is linked via the `Include` parameter in the
> `/etc/ssh/sshd_config` file, if so, an additional configuration file is
> created in `/ etc/ssh/sshd_config.d`, if not, the `/etc/ssh/sshd_config`
> file is overwritten.

> **Warning**
>
> If any `sshd_match_(users|groups|addresses|local_ports)` or
> `sshd_sftp_only_group` parameters is set, the value `true` will be implicit.

`sshd_host_keys_files` host keys for sshd. If empty `['/etc/ssh/ssh_host_rsa_key',
'/etc/ssh/ssh_host_ec dsa_key', '/etc/ssh/ssh_host_ed25519_key']` will be used,
as far as supported by the installed sshd version.

`sshd_host_keys_owner` set owner of host keys for sshd.

`sshd_host_keys_group` set group of host keys for sshd.

`sshd_host_keys_mode` set permission of host keys for sshd.

`sshd_match_users` add a conditional block for users. If all of the criteria
on the Match line are satisfied, the rules/parameters defined on the following
lines override those set in the global section of th e config file, until either
another Match line or the end of the file.

Expected configuration structure:

```yaml
sshd_match_users:
  - user: <username>
    rules:
      - <parameter sshd> <value>
      - <parameter sshd> <value>
```

Example, allow `ubuntu` user access through password authentication and allow
`ansible` user access without a banner:

```yaml
sshd_match_users:
  - user: ubuntu
    rules:
      - AllowUsers ubuntu
      - AuthenticationMethods password
      - PasswordAuthentication yes
  - user: ansible
    rules:
      - AllowUsers ansible
      - Banner none
```

`sshd_match_groups` add a conditional block for groups. More details and
examples in the parameter description `sshd_match_users`.

Expected configuration structure:
```yaml
sshd_match_groups:
  - group: <groupname>
    rules:
      - <parameter sshd> <value>
      - <parameter sshd> <value>
```

`sshd_match_addresses` add a conditional block for adddresses. More details and
examples in the parameter description `sshd_match_users`.

Expected configuration structure:

```yaml
sshd_match_addresses:
  - address: <ip>
    rules:
      - <parameter sshd> <value>
      - <parameter sshd> <value>
```
`sshd_match_local_ports` add a conditional block for ports. More details and
examples in the parameter description `sshd_match_users`.

Expected configuration structure:

```yaml
sshd_match_ports:
  - port: <port>
    rules:
      - <parameter sshd> <value>
      - <parameter sshd> <value>
```

`sshd_print_pam_motd` specifies whether printing of the MOTD via `pam`.

`sshd_sftp_enabled` specifies whether enabled sftp configuration.

`sshd_sftp_subsystem` sets the subsystem for file transfer daemon.

`sshd_sftp_only_group` specifies the name of the group that will have access
restricted to the sftp service only.

`sshd_sftp_only_chroot` specifies group access will be via chroot isolation.

`sshd_sftp_only_chroot_dir` specifies the chroot directory. Accepts the tokens
`%%` (a literal `%`), `%h` (home directory of the user), and `%u` (username).

`sshd_syslog_facility` sets the facility code that is used when logging messages
from sshd.

`sshd_update_moduli`, if set to `true`, will download a updated
[moduli file](https://man7.org/linux/man-pages/man5/moduli.5.html)
from the [konstruktoid/ssh-moduli](https://github.com/konstruktoid/ssh-moduli)
repository.

### ./defaults/main/suid_sgid_blocklist.yml

```yaml
suid_sgid_permissions: true
suid_sgid_blocklist:
  - 7z
  - aa-exec
  - ab
  - agetty
  - alpine
  - ansible-playbook
  - ansible-test
  - aoss
  - apt
  - apt-get
  [...]
```

If `suid_sgid_permissions: true` loop through `suid_sgid_blocklist` and remove
any SUID/SGID permissions.

A complete file list is available in
[defaults/main/suid_sgid_blocklist.yml](defaults/main/suid_sgid_blocklist.yml)
and is based on the work by [@GTFOBins](https://github.com/GTFOBins).

### ./defaults/main/sysctl.yml

```yaml
manage_sysctl: true
sysctl_conf_dir: "{{ '/usr/lib/sysctl.d' if usr_lib_sysctl_d_dir else '/etc/sysctl.d' }}"

sysctl_dev_tty_ldisc_autoload: 0

ipv4_sysctl_settings:
  net.ipv4.conf.all.accept_redirects: 0
  net.ipv4.conf.all.accept_source_route: 0
  net.ipv4.conf.all.log_martians: 1
  net.ipv4.conf.all.rp_filter: 1
  net.ipv4.conf.all.secure_redirects: 0
  net.ipv4.conf.all.send_redirects: 0
  net.ipv4.conf.all.shared_media: 0
  net.ipv4.conf.default.accept_redirects: 0
  net.ipv4.conf.default.accept_source_route: 0
  net.ipv4.conf.default.log_martians: 1
  net.ipv4.conf.default.rp_filter: 1
  net.ipv4.conf.default.secure_redirects: 0
  net.ipv4.conf.default.send_redirects: 0
  net.ipv4.conf.default.shared_media: 0
  net.ipv4.icmp_echo_ignore_broadcasts: 1
  net.ipv4.icmp_ignore_bogus_error_responses: 1
  net.ipv4.ip_forward: 0
  net.ipv4.tcp_challenge_ack_limit: 2147483647
  net.ipv4.tcp_invalid_ratelimit: 500
  net.ipv4.tcp_max_syn_backlog: 20480
  net.ipv4.tcp_rfc1337: 1
  net.ipv4.tcp_syn_retries: 5
  net.ipv4.tcp_synack_retries: 2
  net.ipv4.tcp_syncookies: 1

generic_sysctl_settings:
  fs.protected_fifos: 2
  fs.protected_hardlinks: 1
  fs.protected_symlinks: 1
  fs.suid_dumpable: 0
  kernel.core_uses_pid: 1
  kernel.dmesg_restrict: 1
  kernel.kptr_restrict: 2
  kernel.panic: 60
  kernel.panic_on_oops: 60
  kernel.perf_event_paranoid: 3
  kernel.randomize_va_space: 2
  kernel.sysrq: 0
  kernel.unprivileged_bpf_disabled: 1
  kernel.yama.ptrace_scope: 2
  net.core.bpf_jit_harden: 2

conntrack_sysctl_settings:
  net.netfilter.nf_conntrack_max: 2000000
  net.netfilter.nf_conntrack_tcp_loose: 0
```

If `manage_sysctl: true`, then update the `sysctl` configuration.

`sysctl_conf_dir` is where the `sysctl` configuration will be placed.

See [sysctl.conf](https://linux.die.net/man/5/sysctl.conf) and
the [kernel documentation](https://www.kernel.org/doc/Documentation/sysctl/).

### ./defaults/main/templates.yml

```yaml
adduser_conf_template: etc/adduser.conf.j2
common_account_template: etc/pam.d/common-account.j2
common_auth_template: etc/pam.d/common-auth.j2
common_password_template: etc/pam.d/common-password.j2
coredump_conf_template: etc/systemd/coredump.conf.j2
faillock_conf_template: etc/security/faillock.conf.j2
hardening_rules_template: etc/audit/rules.d/hardening.rules.j2
hosts_allow_template: etc/hosts.allow.j2
hosts_deny_template: etc/hosts.deny.j2
initpath_sh_template: etc/profile.d/initpath.sh.j2
issue_template: etc/issue.j2
journald_conf_template: etc/systemd/journald.conf.j2
limits_conf_template: etc/security/limits.conf.j2
login_defs_template: etc/login.defs.j2
login_template: etc/pam.d/login.j2
logind_conf_template: etc/systemd/logind.conf.j2
logrotate_conf_template: etc/logrotate.conf.j2
motd_template: etc/motd.j2
pwquality_conf_template: etc/security/pwquality.conf.j2
resolved_conf_template: etc/systemd/resolved.conf.j2
rkhunter_template: etc/default/rkhunter.j2
ssh_config_template: etc/ssh/ssh_config.j2
sshd_config_template: etc/ssh/sshd_config.j2
sysctl_ipv6_config_template: etc/sysctl/sysctl.ipv6.conf.j2
sysctl_main_config_template: etc/sysctl/sysctl.main.conf.j2
system_conf_template: etc/systemd/system.conf.j2
timesyncd_conf_template: etc/systemd/timesyncd.conf.j2
tmp_mount_template: etc/systemd/tmp.mount.j2
user_conf_template: etc/systemd/user.conf.j2
useradd_template: etc/default/useradd.j2
```

Paths in order to support overriding the default [role templates](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html).

### ./defaults/main/ufw.yml

```yaml
manage_ufw: true

ufw_outgoing_traffic:
  - 22
  - 53
  - 80
  - 123
  - 443
  - 853
```

`manage_ufw: true` installs and configures `ufw` with related rules.
Set it to `false` in order to install and configure a firewall manually.

`ufw_outgoing_traffic` opens the specific `ufw` ports, allowing outgoing
traffic.

### ./defaults/main/umask.yml

```yaml
session_timeout: 900
umask_value: "077"
```

`session_timeout` sets, in seconds, the
[TMOUT](https://www.gnu.org/software/bash/manual/bash.html#index-TMOUT)
environment variable.

`umask_value` sets the default
[umask value](https://manpages.ubuntu.com/manpages/jammy/man2/umask.2.html).

### ./defaults/main/usbguard.yml

```yaml
manage_usbguard: true

usbguard_configuration_file: /etc/usbguard/usbguard-daemon.conf
usbguard_rulefile: /etc/usbguard/rules.conf

usbguard_auditbackend: LinuxAudit
usbguard_auditfilepath: /var/log/usbguard/usbguard-audit.log
usbguard_authorizeddefault: none
usbguard_devicemanagerbackend: uevent
usbguard_deviceruleswithport: false
usbguard_hidepii: false
usbguard_implicitpolicytarget: block
usbguard_inserteddevicepolicy: apply-policy
usbguard_ipcaccesscontrolfiles: /etc/usbguard/IPCAccessControl.d/
usbguard_ipcallowedgroups:
  - plugdev
  - root
  - wheel
usbguard_ipcallowedusers:
  - root
usbguard_presentcontrollerpolicy: keep
usbguard_presentdevicepolicy: apply-policy
usbguard_restorecontrollerdevicestate: false
```

`manage_usbguard: true` installs and configures
[USBGuard](https://usbguard.github.io/).

A policy will be generated if any rules can be listed and a policy doesn't
yet exist.

See the [configuration documentation](https://usbguard.github.io/documentation/configuration.html)
regarding the available options.

### ./defaults/main/users.yml

```yaml
delete_users:
  - games
  - gnats
  - irc
  - list
  - news
  - sync
  - uucp
```

Users to be removed.

## Recommended Reading

[Comparing the DISA STIG and CIS Benchmark values](https://github.com/konstruktoid/publications/blob/master/ubuntu_comparing_guides_benchmarks.md)

[Center for Internet Security Linux Benchmarks](https://www.cisecurity.org/cis-benchmarks/)

[Common Configuration Enumeration](https://nvd.nist.gov/cce/index.cfm)

[DISA Security Technical Implementation Guides](https://public.cyber.mil/stigs/downloads/?_dl_facet_stigs=operating-systems%2Cunix-linux)

[SCAP Security Guides](https://complianceascode.github.io/content-pages/guides/index.html)

[Security focused systemd configuration](https://github.com/konstruktoid/hardening/blob/master/systemd.adoc)

## Contributing

Do you want to contribute? Great! Contributions are always welcome,
no matter how large or small. If you found something odd, feel free to submit a
issue, improve the code by creating a pull request, or by
[sponsoring this project](https://github.com/sponsors/konstruktoid).

## License

Apache License Version 2.0

## Author Information

[https://github.com/konstruktoid](https://github.com/konstruktoid "github.com/konstruktoid")
