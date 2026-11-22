*This project has been created as part of the 42 curriculum by wli-ting.*

# Born2beroot

## Description

**Born2beroot** is a 42 system administration project focused on building, hardening, and managing a headless Linux server inside a virtual machine under strict security constraints. The goal is to gain practical experience with virtualization, encrypted disk partitioning, user and privilege management, firewall configuration, mandatory access control, automated system monitoring, and deploying server-side services without a graphical interface.

### Operating System Choice & Main Setup Decisions

I chose **Debian 13 (Trixie)** over Rocky Linux for its stability, lightweight minimal installation footprint, straightforward `apt` package management, and extensive community documentation.

Key configuration choices on this VM (`wli-ting42`):

- **Encrypted LVM Partitioning:** `/boot` resides on an unencrypted primary partition (`sda1`), while the rest of the disk (`sda5`) is encrypted with LUKS (`sda5_crypt`) and divided via LVM into 7 logical volumes: `/`, `swap`, `/home`, `/var`, `/srv`, `/tmp`, and `/var/log`. Isolating `/var/log` and `/tmp` prevents runaway logs or temporary files from filling the root filesystem.
- **User & Privilege Management:** Configured user `wli-ting` in both the `user42` and `sudo` groups, with strict `sudo` rules in `/etc/sudoers.d/` (`passwd_tries=3`, custom wrong-password message, `requiretty`, restricted `secure_path`, and full input/output logging to `/var/log/sudo/`).
- **Password Policy:** Enforced a 30-day maximum age, 2-day minimum age, and 7-day expiration warning in `/etc/login.defs` (and applied to existing accounts via `chage`), paired with `pam_pwquality` requiring at least 10 characters, an uppercase letter, a lowercase letter, a digit, at most 3 consecutive identical characters, at least 7 changed characters (`difok=7`), username rejection, and enforcement for `root`.
- **Network Hardening & MAC:** Enabled AppArmor at boot, configured OpenSSH to listen exclusively on port `4242` with `PermitRootLogin no`, and activated UFW allowing only ports `4242`, `80/tcp`, and `9090/tcp`.
- **Automated Monitoring:** Deployed `/usr/local/bin/monitoring.sh` scheduled via `root`'s crontab (`@reboot` and `*/10 * * * *`) to broadcast live system metrics across all terminals using `wall`.
- **Bonus Services:**
  - **WordPress Stack:** Deployed with `lighttpd`, `php-cgi`, `php-mysql`, and `mariadb` (with `apache2` and `nginx` completely purged).
  - **Cockpit:** Installed as an additional administrative service on port `9090` to provide lightweight, browser-based system inspection, log viewing, and service management without installing a graphical desktop environment.

### Technical Comparisons

#### Debian vs. Rocky Linux
| Criteria | Debian | Rocky Linux |
| :--- | :--- | :--- |
| **Package Manager** | `apt` / `dpkg` (`.deb`) | `dnf` / `rpm` (`.rpm`) |
| **Release Philosophy** | Community-driven, huge repository, prioritizes stability and minimal base installs | Downstream rebuild of RHEL, enterprise-focused with long support lifecycles |
| **Default Security Stack** | AppArmor + UFW (commonly used) | SELinux + `firewalld` |
| **Advantages** | Lightweight, beginner-friendly documentation, broad software availability | Enterprise standard, binary-compatible with RHEL, strict out-of-the-box SELinux policies |
| **Disadvantages** | Older default package versions on stable releases compared to rolling distributions | Steeper learning curve for SELinux and `firewalld` configuration in minimal environments |

#### AppArmor vs. SELinux
| Criteria | AppArmor | SELinux |
| :--- | :--- | :--- |
| **Access Control Model** | Path-based (rules are tied to file paths) | Label-based (rules are tied to security contexts stored on inodes and processes) |
| **Complexity** | Simpler to read, write, and troubleshoot profiles | More complex with a steeper learning curve |
| **Granularity** | Moderate; moving or renaming a file can affect path-based rules if misconfigured | Very high; labels follow files even when moved or renamed |
| **Default Ecosystem** | Debian / Ubuntu | RHEL / Rocky Linux / Fedora |

#### UFW vs. firewalld
| Criteria | UFW (Uncomplicated Firewall) | firewalld |
| :--- | :--- | :--- |
| **Design Approach** | Simple CLI frontend for managing host firewall rules (`iptables`/`nftables`) | Dynamic daemon built around network **zones** (public, home, internal, etc.) |
| **Rule Reloading** | Reloading re-applies the rule set | Supports runtime vs. permanent changes dynamically without dropping active connections |
| **Best Fit** | Single-server setups needing quick, readable port rules | Complex multi-interface or enterprise environments requiring zone-based policies |

#### VirtualBox vs. UTM
| Criteria | VirtualBox | UTM |
| :--- | :--- | :--- |
| **Underlying Tech** | Type-2 hypervisor focused on x86_64 virtualization | Frontend for QEMU and Apple's Virtualization framework |
| **Platform Support** | Cross-platform (Windows, Linux, Intel macOS; limited on Apple Silicon) | Built specifically for macOS (excels on Apple Silicon) and iOS |
| **Use Case** | Standard choice for x86_64 Windows/Linux hosts and VirtualBox `.vdi` workflows | Best choice on Apple Silicon Macs for running ARM VMs or emulating x86_64 |

---

## Instructions

### Prerequisites
- **Oracle VirtualBox** installed on the host machine.
- **Debian 13 (Trixie) netinst ISO** (if reproducing from scratch) or the configured `born2beroot` VM disk (`.vdi`).

### Port Forwarding Configuration (VirtualBox NAT)
To access the VM services from the host machine, configure the following NAT port-forwarding rules in VirtualBox (**Settings → Network → Adapter 1 → Advanced → Port Forwarding**):

| Service | Protocol | Host IP | Host Port | Guest Port |
| :--- | :--- | :--- | :--- | :--- |
| **SSH** | TCP | `127.0.0.1` | `4242` | `4242` |
| **WordPress (Lighttpd)** | TCP | `127.0.0.1` | `1672` | `80` |
| **Cockpit** | TCP | `127.0.0.1` | `9090` | `9090` |

### Running & Accessing the VM
1. Start the `born2beroot` VM in VirtualBox (with **Hard Disk** first in the Boot Order) and enter the LUKS encryption passphrase at boot.
2. **Connect via SSH** from the host terminal:
   ```bash
   ssh wli-ting@127.0.0.1 -p 4242
   ```
   *(Direct SSH login as `root` is disabled by design.)*
3. **Access WordPress** in a host web browser:
   ```text
   http://127.0.0.1:1672
   ```
4. **Access Cockpit** in a host web browser:
   ```text
   https://127.0.0.1:9090
   ```
5. **Clean Shutdown:** Always power off cleanly with `sudo poweroff` rather than saving the machine state.

### Verifying the Disk Signature
With the VM powered off and zero snapshots present, generate the SHA-1 checksum of the `.vdi` file and compare it against `signature.txt`:

- **Windows (PowerShell):**
  ```powershell
  certUtil -hashfile born2beroot.vdi sha1
  ```
- **Linux / macOS:**
  ```bash
  sha1sum born2beroot.vdi
  ```

---

## Resources

### Guides & Community References
- [chlimous — 42 Born2beroot Guide](https://github.com/chlimous/42-born2beroot_guide)
- [Ayouub-aj — Born2beroot](https://github.com/Ayouub-aj/born2beroot)

### Official Documentation
- [Debian Administrator's Handbook & Official Documentation](https://www.debian.org/doc/)
- [Debian Wiki — LVM, AppArmor, and UFW](https://wiki.debian.org/)
- [OpenSSH `sshd_config` Manual](https://man.openbsd.org/sshd_config)
- [PAM `pam_pwquality` Manual](https://manpages.debian.org/trixie/libpam-pwquality/pam_pwquality.8.en.html)
- [Lighttpd Documentation](https://redmine.lighttpd.net/projects/lighttpd/wiki)
- [WordPress Installation Guide](https://developer.wordpress.org/advanced-administration/before-install/howto-install/)
- [Cockpit Project Documentation](https://cockpit-project.org/documentation.html)

### AI Usage Disclosure
An AI assistant (Gemini) was used as a learning and troubleshooting companion during this project for:
- **Debugging & Troubleshooting:** Resolving console keyboard layout issues in VirtualBox, diagnosing a `php` dependency that installed `apache2` over `lighttpd`, fixing SSH login mismatches, and handling a VirtualBox iPXE warm-reboot hang.
- **Compliance Auditing:** Walking through verification commands to audit mandatory and bonus requirements (PAM complexity flags, `chage` expiration settings, `sudo` logging, UFW rules, and active services).
- **Documentation:** Structuring and refining the layout and technical comparison tables of this `README.md`.

All VM installations, configurations, script implementations, and verifications were executed and tested directly inside the virtual machine.