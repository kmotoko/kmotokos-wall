---
# Global vars
layout: posts
title: "Linux kernel hardening: Kernel parameters with sysctl"
date: 2020-01-12 12:00:00 +0300
published: true
tags: ["sysctl", "kernel", "security", "Exec Shield", "YAMA", "BPF-JIT", "ARP", "BOOTP", "hardening", "Linux"]
# Theme specific vars
last_modified_at: 2020-01-12 12:00:00 +0300
thumbnail:
    name: "linux_kernel_hardening_sysctl"
    webp: true
summary: "Linux kernel security can be increased at runtime using sysctl, allowing to mitigate potential vulnerabilities and exploits."
---
In this article, we describe several kernel parameters that have implications in the security of the system and how to harden these parameters. Note that for certain parameters, there are tighter values. However, for the sake of the balance between security and usability, this article does not include the 'strictest' values for those kernel parameters.

## Enable Exec Shield protection
Exec Shield is first introduced by Red Hat. The goal is to protect against stack/buffer/function pointer overflows. In most of the newer Linux distributions, the option to disable Exec Shield through `sysctl` was removed. For example, In Debian 10, if you do `sudo sysctl kernel.exec-shield=1`, you will get `sysctl: cannot stat /proc/sys/kernel/exec-shield: No such file or directory.`. You can check if Exec Shield is enabled in your system via:
```bash
sudo dmesg | grep "Execute Disable"
```
If you see an output similar to:
```
[    0.000000] NX (Execute Disable) protection: active
```
then it is enabled. If it is not active, and you cannot enable it with `sysctl`, it is also possible that the Exec Shield is disabled from the BIOS. Go to your BIOS settings and look for the following keywords: "NX", "XD", "No Execute" or "Execute Disable".
Exec Shield project also brought address space randomization, to enable it:
```bash
sudo sysctl kernel.randomize_va_space=2
```
A value of 2 randomizes heap and the addresses of mmap base, stack and VDSO page. Sources {% cite ExecShield_Wikipedia NXbit_Wikipedia WhatisNXXDfeature_RedHat --file web %}.

## Restrict ptrace call with Yama LSM
Yama Linux Security Module (LSM) brings a function which limits the scope of `ptrace` calls. `ptrace` is used frequently in debugging. From the `ptrace` man page
> The ptrace() system call provides a means by which one process (the "tracer") may observe and control the execution of another process (the "tracee"), and examine and change the tracee's memory and registers.

Thus, it is important to restrict the scope of `ptrace` calls:
```bash
# Restrict ptrace scope
sudo sysctl kernel.yama.ptrace_scope=1
```
A value of 1 means that only the parent process can be debugged.
Sources {% cite Security_ArchWiki ptraceLinuxManPage_dienet --file web %}.

## Restrict core dumps
A core dump is a file which contains valuable information about why a software has crashed/closed unexpectedly. Core dumps can also be generated upon developer's action, e.g. debugging. Although they have many benefits for troubleshooting problems, they might contain sensitive information. Disable processes with elevated privileges from making core dumps:
```bash
sudo sysctl fs.suid_dumpable=0
```
Sources {% cite Coredump_ArchWiki --file web %}.

## Restrict access to kernel pointers
If you want to prevent access to kernel symbol addresses to users without `CAP_SYSLOG`, you can set the parameter via:
```bash
sudo sysctl kernel.kptr_restrict=1
```
If a value of 2 is set, the kernel will hide the symbol addresses regardless of the user privilege. Keep in mind that this might cause problems.
Source {% cite Security_ArchWiki --file web %}.

## Restrict access to kernel logs for unauthorized users
You can prevent access to kernel logs to users without `CAP_SYS_ADMIN` capability via:
```bash
sudo sysctl kernel.dmesg_restrict=1
```
Source {% cite Security_ArchWiki --file web %}.

## Network stack hardening
### Harden BPF Just-in-Time (JIT) compiler
Before the famous Spectre vulnerability, it was a recommended practice to disable BPF Just-in-Time (JIT) compiler. Newer kernels set BPF JIT compiler to always ON, as recommended against Spectre-like vulnerabilities. In addition, Linux kernel has a parameter to harden BPF JIT compiler:
```bash
sudo sysctl net.core.bpf_jit_harden=1
```
which hardens unprivileged code. If you would like to harden all code, use a value of 2.

Sources {% cite BerkeleyPacketFilter_Wikipedia Security_ArchWiki bpfIntroducebpf_jit_always_onconfig_LinuxKernelOrganizationGitRepository --file web %}.

### Protect against SYN flood attack
Prevent SYN flood attack, enable SYNcookies. These will kick-in when the `tcp_max_syn_backlog` reached.
```bash
sudo sysctl net.ipv4.tcp_syncookies=1
sudo sysctl net.ipv4.tcp_syn_retries=2
sudo sysctl net.ipv4.tcp_synack_retries=2
sudo sysctl net.ipv4.tcp_max_syn_backlog=4096
```

Sources {% cite Sysctl_ArchWiki LinuxKernelDocumentation-Networking_LinuxKernelOrganization --file web %}.

### Protect against IP spoofing
Enable IP spoofing protection, turn on source route verification.
```bash
sudo sysctl net.ipv4.conf.all.rp_filter=1
sudo sysctl net.ipv4.conf.default.rp_filter=1
```
Sources {% cite Sysctl_ArchWiki LinuxKernelDocumentation-Networking_LinuxKernelOrganization --file web %}.

### Harden Bootstrap Protocol (BOOTP)
Disable BOOTP relay. Bootstrap Protocol (BOOTP) is similar to Dynamic Host Configuration Protocol (DHCP), both are used assign IP addresses to network-attached devices. Note that BOOTP is superseded by DHCP, and only used by legacy devices. To disable packet capturing and forwarding with BOOTP daemon:
```bash
sudo sysctl net.ipv4.conf.all.bootp_relay=0
```
Sources {% cite BootstrapProtocol_Wikipedia Sysctl_ArchWiki LinuxKernelDocumentation-Networking_LinuxKernelOrganization --file web %}.

### Harden Address Resolution Protocol (ARP)
Disable proxy ARP. ARP stands for Address Resolution Protocol. What ARP does is that it maps IP address to a physical network address e.g. media access control (MAC) address. If you do not have a very specific reason, you can safely disable proxy ARP. This means that your computer would not answer ARP queries for an address which is not on the same subnet. You can also improve further by restricting the replies and ignoring the source address:
```bash
# Don't proxy arp for anyone
sudo sysctl net.ipv4.conf.all.proxy_arp=0
# Modes for sending replies in response to received ARP requests that resolve local target IP addresses
# Mode '1': reply only if the target IP address is local address configured on the incoming interface
sudo sysctl net.ipv4.conf.all.arp_ignore=1
# Restriction levels for announcing the local source IP address from IP packets in ARP requests sent on interface
# Mode '2': ignore the source address in the IP packet and try to select local address that we prefer for talks with the target host
sudo sysctl net.ipv4.conf.all.arp_announce=2
```
Sources {% cite AddressResolutionProtocol_Wikipedia Sysctl_ArchWiki LinuxKernelDocumentation-Networking_LinuxKernelOrganization --file web %}.

### Other network hardening
+ Disable packet forwarding. If you are not doing packet forwarding, you can disable it in the kernel via:
```bash
sudo sysctl net.ipv4.ip_forward=0
sudo sysctl net.ipv4.conf.all.forwarding=0
sudo sysctl net.ipv4.conf.default.forwarding=0
sudo sysctl net.ipv6.conf.all.forwarding=0
sudo sysctl net.ipv6.conf.default.forwarding=0
```

+ Disable ICMP redirect acceptance:
```bash
sudo sysctl net.ipv4.conf.all.accept_redirects=0
sudo sysctl net.ipv4.conf.default.accept_redirects=0
sudo sysctl net.ipv4.conf.all.secure_redirects=0
sudo sysctl net.ipv4.conf.default.secure_redirects=0
sudo sysctl net.ipv6.conf.all.accept_redirects=0
sudo sysctl net.ipv6.conf.default.accept_redirects=0
```

+ Disable ICMP redirect sending:
```bash
sudo sysctl net.ipv4.conf.all.send_redirects=0
sudo sysctl net.ipv4.conf.default.send_redirects=0
```

+ Disable source routing:
```bash
sudo sysctl net.ipv4.conf.all.accept_source_route=0
sudo sysctl net.ipv4.conf.default.accept_source_route=0
sudo sysctl net.ipv6.conf.all.accept_source_route=0
sudo sysctl net.ipv6.conf.default.accept_source_route=0
```

+ Disable the logging of bogus responses from certain routers. From the kernel docs:
> Some routers violate RFC1122 by sending bogus responses to broadcast frames.  Such violations are normally logged via a kernel warning. If this is set to TRUE, the kernel will not give such warnings, which will avoid log file clutter.

  To ignore such bogus responses:
```bash
sudo sysctl net.ipv4.icmp_ignore_bogus_error_responses=1
```
  In addition, you can ignore ICMP ECHO and TIMESTAMP requests from broadcasts via:
```bash
sudo sysctl net.ipv4.icmp_echo_ignore_broadcasts=1
```

+ Protect against time-wait assassination hazards in TCP:
```bash
sudo sysctl net.ipv4.tcp_rfc1337=1
```

Sources {% cite Sysctl_ArchWiki LinuxKernelDocumentation-Networking_LinuxKernelOrganization --file web %}.

### Apply the settings to new connections
Lastly, ensure that the subsequent connections use the new configurations:
```bash
# Comes last
sudo sysctl net.ipv4.route.flush=1
sudo sysctl net.ipv6.route.flush=1
```

## Make the sysctl configuration permanent
The commands applied by `sudo sysctl <parameter>=<value>` are only preserved for the current session. To make them persist across the reboots, you need to put parameter:value pairs (without `sudo sysctl`) into a config file, e.g. `/etc/sysctl.d/99-sysctl.conf` {% cite Sysctl_ArchWiki --file web %}. Therefore, create it and fill with the desired values. Below is the convenience snippet which includes all the parameters shown in the previous parts, ready-to-be used in a configuration file:

```toml
###
### SYSTEM SECURITY ###
###

# Enable address Space Randomization
kernel.randomize_va_space = 2

# Restrict core dumps
fs.suid_dumpable = 0

# Hide kernel pointers
kernel.kptr_restrict = 1

# Restrict access to kernel logs
kernel.dmesg_restrict = 1

# Restrict ptrace scope
kernel.yama.ptrace_scope = 1

###
### Deprecated/Not-in-use keys for security
###

# The contents of /proc/<pid>/maps and smaps files are only visible to
# readers that are allowed to ptrace() the process
# kernel.maps_protect = 1

# Enable ExecShield
# kernel.exec-shield = 1

###
### NETWORK SECURITY ###
###

# Harden BPF JIT compiler
net.core.bpf_jit_harden = 1

# Prevent SYN attack, enable SYNcookies (they will kick-in when the max_syn_backlog reached)
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_max_syn_backlog = 4096

# Disable packet forwarding
net.ipv4.ip_forward = 0
net.ipv4.conf.all.forwarding = 0
net.ipv4.conf.default.forwarding = 0
net.ipv6.conf.all.forwarding = 0
net.ipv6.conf.default.forwarding = 0

# Enable IP spoofing protection
# Turn on source route verification
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Disable Redirect Acceptance
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Disable Redirect Sending
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Disable IP source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

# Don't relay bootp
net.ipv4.conf.all.bootp_relay = 0

# Disable proxy ARP
net.ipv4.conf.all.proxy_arp = 0
net.ipv4.conf.all.arp_ignore = 1
net.ipv4.conf.all.arp_announce = 2

# Mitigate time-wait assassination hazards in TCP
net.ipv4.tcp_rfc1337 = 1

# Enable bad error message Protection
net.ipv4.icmp_ignore_bogus_error_responses = 1

# Enable ignoring broadcasts request
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Ensure that subsequent connections use the new values
# PUT TO THE END
net.ipv4.route.flush = 1
net.ipv6.route.flush = 1
```
After the configuration file is set, apply the settings via: `sudo sysctl --system`

## References
{% bibliography --file web --cited_in_order %}
