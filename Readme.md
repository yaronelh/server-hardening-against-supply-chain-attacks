# Supply Chain Attacks, Hardening Your Dev Environmen

These days, it's way too easy for a random dependency, a sneaky build script, or a compromised package to wreck your day — or worse, your whole environment.

This checklist walks through a practical hardening approach for a development VM setup focused on supply chain resilience.

---

## Reference Infrastructure

- Windows host  
- Hyper-V virtual machine  
- Ubuntu Server 24.04 LTS guest  
- Development happens inside the Linux guest via SSH  

---

## Why Use a VM Instead of WSL

WSL is convenient but tightly integrated with Windows. A dedicated VM provides stronger isolation and reduces risk if something inside the Linux environment is compromised.

---

## 1. Access and SSH Hardening

- [ ] Disable SSH password authentication (`PasswordAuthentication no`)
- [ ] Disable SSH root login (`PermitRootLogin no`)
- [ ] Enable SSH key authentication (`PubkeyAuthentication yes`)
- [ ] Disable keyboard-interactive auth (`KbdInteractiveAuthentication no`)
- [ ] Disable X11 forwarding (`X11Forwarding no`)
- [ ] Limit SSH users (`AllowUsers admin`)
- [ ] Set `MaxAuthTries 7`
- [ ] Set `LoginGraceTime 30s`
- [ ] Keep `AllowTcpForwarding yes`
- [ ] Set `GatewayPorts no`

---

## 2. Identity, Privilege, and Workspace Separation

- [ ] Keep `admin` as admin-capable account
- [ ] Use `dev` as non-sudo daily account
- [ ] Perform development under `dev`
- [ ] Store projects under `/home/dev/projects`
- [ ] Restrict `.ssh` permissions
- [ ] Restrict `.gnupg` permissions
- [ ] Ensure `/tmp` has sticky bit (1777)
- [ ] Review tool state directories (e.g. `.codex`)
- [ ] Tighten `umask` if needed

---

## 3. Firewall and Network Containment

- [ ] Enable UFW
- [ ] Default deny incoming, allow outgoing
- [ ] Allow SSH on port 22
- [ ] Enable logging
- [ ] Preserve RFC1918 outbound deny rules where applicable
- [ ] Avoid exposing dev ports (3000, 5000, etc.)
- [ ] Prefer SSH port forwarding
- [ ] Bind services to `127.0.0.1`
- [ ] Use NAT networking
- [ ] Avoid Windows portproxy rules unless needed

---

## 4. Platform and Service Footprint Reduction

- [ ] Remove unused tools (e.g. `open-vm-tools`)
- [ ] Remove unnecessary services (e.g. `ModemManager`)
- [ ] Review installed packages periodically
- [ ] Disable unused services

---

## 5. Development Workflow Defaults

- [ ] Use VS Code Remote SSH or similar
- [ ] Use `dev` as default identity
- [ ] Keep app access inside SSH tunnels
- [ ] Standardize localhost binding
- [ ] Document safe access patterns
- [ ] Define rules for opening inbound ports

---

## 6. Supply Chain Tooling and Package Workflow

- [ ] Install `safe-chain`
- [ ] Install `syft`
- [ ] Install `grype`
- [ ] Use safe-chain across admin and dev
- [ ] Wrap pip/npm/pnpm with safe-chain
- [ ] Prefer `pnpm` over `npm`
- [ ] Set `pnpm minimum-release-age`
- [ ] Block exotic subdependencies
- [ ] Define when scans run
- [ ] Validate blocking behavior
- [ ] Prefer pinned dependencies
- [ ] Review Python workflows equally
- [ ] Define safe process for new registries

---

## 7. Environment Strategy and Blast-Radius Reduction

- [ ] Separate stable and experimental environments
- [ ] Separate credentials
- [ ] Define environment roles clearly
- [ ] Keep sensitive data out of experimental envs

---

## 8. Logging, Monitoring, and Recovery Basics

- [ ] Ensure `rsyslog` is running
- [ ] Ensure `journald` available
- [ ] Install and enable Fail2Ban
- [ ] Tune Fail2Ban settings
- [ ] Enable recidive jail
- [ ] Test Fail2Ban regularly
- [ ] Define recovery plan

---

## 9. Kernel and OS-Level Baseline Hardening

- [ ] Set `kernel.yama.ptrace_scope = 1`
- [ ] Set `kernel.kptr_restrict = 1`
- [ ] Set `kernel.dmesg_restrict = 1`
- [ ] Set `fs.protected_hardlinks = 1`
- [ ] Set `fs.protected_symlinks = 1`
- [ ] Review `kernel.unprivileged_userns_clone`
- [ ] Consider hardened `/tmp` mount options

---

## 10. Validation and Housekeeping

- [ ] Confirm only SSH is exposed
- [ ] Verify dev workspace usability
- [ ] Remove Hyper-V checkpoints after stability
- [ ] Apply phased updates
- [ ] Revalidate checklist after changes
- [ ] Review regularly

---

## 11. Suggested Next Actions

- [ ] Document daily workflow
- [ ] Define routine scans (syft/grype)
- [ ] Test safe-chain behavior
- [ ] Review local permissions (umask, etc.)
- [ ] Keep documentation aligned with reality

---

*Source: Converted from uploaded PDF*