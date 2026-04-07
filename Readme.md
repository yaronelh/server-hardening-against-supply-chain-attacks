# Supply Chain Attacks, Hardening Your Dev Environmen

These days it's ridiculously easy for a malicious dependency, a compromised package, or a sneaky install script to sneak into your development environment and cause real damage. Supply chain attacks keep getting more common, and as developers we're constantly pulling in code we didn't write ourselves. That's the core problem this setup is meant to tackle: build a development environment that's actually resilient to those risks without making your everyday work painful or slow.

I put this checklist together based on the hardening I did for my own environment. I encourage you to copy this article to your favorite AI in your development environment and see where you stand with your exposure to security risks, specifically supply chain attacks.

## Table of Contents

- [Infrastructure](#infrastructure)
- [Drop WSL and move to a full VM environment](#drop-wsl-and-move-to-a-full-vm-environment)
- [Access and SSH Hardening](#access-and-ssh-hardening)
- [Identity, Privilege, and Workspace Separation](#identity-privilege-and-workspace-separation)
- [Firewall and Network Containment](#firewall-and-network-containment)
- [Platform and Service Footprint Reduction](#platform-and-service-footprint-reduction)
- [Development Workflow Defaults](#development-workflow-defaults)
- [Supply Chain Tooling and Package Workflow](#supply-chain-tooling-and-package-workflow)
- [Environment Strategy and Blast-Radius Reduction](#environment-strategy-and-blast-radius-reduction)
- [Logging, Monitoring, and Recovery Basics](#logging-monitoring-and-recovery-basics)
- [Kernel and OS-Level Baseline Hardening](#kernel-and-os-level-baseline-hardening)
- [Validation and Housekeeping](#validation-and-housekeeping)

## Infrastructure

I built this around a Windows host, a proper virtual machine layer, and a Linux guest where all the real development work happens. In simple terms, the setup looks like this:

- Windows 11 pro
- Hyper-V virtual machine  
- Ubuntu Server 24.04 LTS guest  
- Development work done inside the Linux guest over SSH or remote-development tooling  

The whole reason for this structure is to create a cleaner separation between your main workstation and the development environment. If something bad slips in through a dependency, package, extension, or script, it should stay contained inside the Linux guest instead of spreading to your Windows machine.

## Drop WSL and move to a full VM environment

WSL 2 does use virtualization, but it's designed for really tight integration between Linux and Windows to make life convenient. You can run Linux tools side-by-side with Windows apps, call back and forth between them, and share files easily. Microsoft even describes it as a lightweight utility virtual machine rather than a fully separate traditional VM.

For a lot of regular development work, that tight integration is a nice feature. But when you're serious about supply chain risks, it's the wrong approach. The isolation of WSL is loose (intentionally as it's built for comfortable development), with it you can easily find yourself compromising your own Windows environment.  A dedicated Hyper-V VM creates a much stronger boundary between the Linux workspace and your Windows host. WSL is intentionally built for easy interoperability, which means if the Linux side gets compromised, there are more practical ways for it to reach Windows files, tools, executables, and other resources.

if your environment has been compromised by a supply chain attack, or other forms of attacks. The fact that it can reach outside of the WSL into Windows becomes a liability, not justifying the risk for the comfort. WSL isn't the right choice for the main development environment. It's not that WSL is broken or useless it's just optimized for convenience and cross-environment access, not for strong isolation. If containing supply chain compromises, protecting credentials, dealing with malicious build scripts, or limiting damage from hostile dependencies matters to you, then a separate dedicated VM the proper way to go.

Let's go over the complete process explaining the steps of each goal and how we're going to harden our Linux environment.

## Access and SSH Hardening

SSH is the main way you get into this VM, and it's also how I handle secure port forwarding to tunnel local web traffic without opening extra network ports. This section comes first because SSH is basically the front door, so hardening it properly gives you the biggest immediate payoff.

**Goal:** Reducing one of the most common internet-facing attack paths by removing password-based SSH logins.\
**How:** Disable SSH password authentication with `PasswordAuthentication no`

**Goal:** Using a lower-privilege remote access pattern so the root account is not used for direct login.\
**How:** Disable SSH root login with `PermitRootLogin no`

**Goal:** Replacing password-based remote authentication with SSH keys for stronger access control.\
**How:** Keep SSH key authentication enabled with `PubkeyAuthentication yes`

**Goal:** Reducing unnecessary authentication paths so there are fewer ways to reach the system remotely.\
**How:** Disable keyboard-interactive authentication with `KbdInteractiveAuthentication no`

**Goal:** Reducing remote-access features that are not needed for a terminal-based development workflow.\
**How:** Disable X11 forwarding with `X11Forwarding no`

**Goal:** Reducing exposure by limiting SSH access to the accounts that actually need it.\
**How:** Limit SSH access with `AllowUsers admin`

**Goal:** Lowering the chance of repeated login guessing without making normal use unnecessarily brittle.\
**How:** Set `MaxAuthTries 7`

**Goal:** Reducing the amount of time attackers or hung sessions can occupy the login path before authentication completes.\
**How:** Set `LoginGraceTime 30s`

**Goal:** Supporting secure developer access to local web services without opening extra inbound ports.\
**How:** Keep `AllowTcpForwarding yes` for development tunnels

**Goal:** Keeping SSH port forwarding limited to the intended client side instead of accidentally sharing forwarded services more widely.\
**How:** Keep `GatewayPorts no`

**Goal:** Keeping access controls aligned with the real operating model so security policy and daily use do not drift apart.\
**How:** Review whether `AllowUsers admin` should become `AllowUsers admin dev`

## Identity, Privilege, and Workspace Separation

This section is about least privilege basically giving each account only the access it actually needs. Day-to-day coding should happen under a regular low-privilege account, while anything that needs admin rights stays in a separate account. That way, if something goes wrong during normal work, the damage stays limited.

**Goal:** Separating administration from routine development so a mistake or compromise in daily work has less reach.\
**How:** Keep `admin` as the admin-capable account

**Goal:** Reducing the damage a dependency, script, or extension can do by defaulting everyday work to a lower-privilege account.\
**How:** Keep `dev` as the non-sudo day-to-day account

**Goal:** Turning least privilege into a real protection by using the safer account for actual development work.\
**How:** Perform routine development under `dev`

**Goal:** Keeping ownership boundaries clear so project files do not inherit unnecessary administrative trust.\
**How:** Keep project repositories under the development user's workspace, for example `/home/dev/projects`

**Goal:** Protecting remote-access credentials because a stolen private key can bypass many other controls.\
**How:** Restrict the development user's `.ssh` permissions

**Goal:** Protecting signing material and trust stores because they influence what the system accepts as legitimate.\
**How:** Restrict the development user's `.gnupg` permissions

**Goal:** Reducing cross-user file abuse in shared temporary space.\
**How:** Confirm `/tmp` retains the sticky bit, typically mode `1777`

**Goal:** Reducing the chance that automation settings, cached secrets, or local tool state become an easy local target.\
**How:** Review local automation-tool state directory permissions, for example `.codex`

**Goal:** Making sure newly created files are not more broadly writable than the environment actually requires.\
**How:** Review whether default `umask` should be tighter than `0002`

## Firewall and Network Containment

This part is about limiting what can reach the VM and what the VM can reach outward. The firewall makes inbound traffic deny-by-default, and using NAT keeps the VM from being too exposed on the network. These controls make it much harder for a compromise to spread.

**Goal:** Creating an independent network boundary so exposed services are not controlled only by application defaults.\
**How:** Enable UFW

**Goal:** Reducing accidental exposure by treating inbound access as something that must be explicitly allowed.\
**How:** Keep the UFW default policy at `deny incoming` and `allow outgoing`

**Goal:** Keeping the necessary admin entry point available while still minimizing overall exposure.\
**How:** Keep SSH explicitly allowed inbound on port `22`

**Goal:** Improving visibility so unexpected traffic patterns can be noticed and investigated.\
**How:** Keep UFW logging enabled

**Goal:** Making it harder for a compromised tool or dependency to pivot into other internal systems.\
**How:** Preserve outbound RFC1918 deny rules for `10.0.0.0/8`, `172.16.0.0/12`, and `192.168.0.0/16` if they fit the workflow

**Goal:** Reducing unnecessary network exposure from local application servers that are meant for one developer's use.\
**How:** Avoid opening common development ports such as `3000`, `5000`, `8000`, and `8080` to the network by default

**Goal:** Using the trusted remote-management channel instead of creating extra paths into the VM.\
**How:** Prefer SSH local port forwarding for web apps

**Goal:** Keeping development services private by default so test servers do not quietly become network-accessible.\
**How:** Prefer binding dev services to `127.0.0.1` inside the guest

**Goal:** Limiting how directly the VM can interact with the broader network if something inside it is compromised.\
**How:** Keep the VM on an internal Hyper-V switch with NAT rather than broad LAN exposure

**Goal:** Preventing the host from silently re-exposing services that the guest itself is trying to keep private.\
**How:** Keep Windows portproxy rules absent unless intentionally required

## Platform and Service Footprint Reduction

The fewer unnecessary packages and services you have running, the smaller your attack surface. If a piece of software doesn't actually support what the VM is used for, it's just extra maintenance and risk.

**Goal:** Reducing software footprint by removing integration tools that do not match the actual virtualization platform.\
**How:** Remove `open-vm-tools` from a Hyper-V guest when VMware integration is not needed

**Goal:** Removing background software that serves no real purpose in the intended server role.\
**How:** Remove `ModemManager` if modem hardware is not part of the VM's role

**Goal:** Reducing long-term attack surface by pruning software that remains only out of habit or neglect.\
**How:** Periodically review installed packages for platform-mismatched or unused components

**Goal:** Keeping the running system easier to reason about by ensuring each enabled service has a clear purpose.\
**How:** Check whether any remaining services are enabled without supporting the current use case

## Development Workflow Defaults

Security only sticks if it fits naturally into how you actually work every day. The safe path should feel like the default path, not some annoying extra step you have to remember.

**Goal:** Using remote-development tools that fit the secure access model instead of working around it.\
**How:** Use VS Code Remote SSH or equivalent SSH-native tooling

**Goal:** Ensuring the safer account is the default in real work, not just in policy.\
**How:** Use `dev` as the default day-to-day remote development identity

**Goal:** Allowing normal application testing without turning every local dev port into a network-facing service.\
**How:** Keep application access inside SSH tunnels where possible

**Goal:** Reducing accidental exposure by making private-by-default service binding the normal project behavior.\
**How:** Standardize localhost binding in project templates and run commands

**Goal:** Helping people choose the safer access pattern consistently instead of inventing one-off exceptions.\
**How:** Document the approved pattern for viewing local web apps from Windows

**Goal:** Preventing convenience exceptions from quietly becoming permanent new exposure.\
**How:** Define when opening a non-SSH inbound port is acceptable

## Supply Chain Tooling and Package Workflow

A lot of today's compromises happen right here through package managers, dependencies, and install scripts. This section adds some practical guardrails around the commands that bring in external code.

**Goal:** Adding guardrails around the commands most likely to pull untrusted code into the environment.\
**How:** Install `safe-chain`

**Goal:** Improving visibility into what is actually installed so suspicious or vulnerable components are easier to spot.\
**How:** Install `syft`

**Goal:** Catching known-risk components before they blend into normal development work unnoticed.\
**How:** Install `grype`

**Goal:** Avoiding gaps where protections exist in one shell but not in the account that actually performs the risky action.\
**How:** Make `safe-chain` available in both `admin` and `dev` contexts

**Goal:** Placing controls at the point where untrusted dependencies are most often introduced.\
**How:** Wrap `pip3`, `npm`, and `pnpm` through `safe-chain`

**Goal:** Reducing dependency-management risk by preferring tooling with stricter and more reviewable behavior.\
**How:** Prefer `pnpm` over `npm` for JavaScript work when the project supports it

**Goal:** Creating a buffer against sudden malicious or hijacked package releases by avoiding immediate adoption.\
**How:** Keep `pnpm` `minimum-release-age=10080`

**Goal:** Limiting dependency resolution paths that are harder to audit and easier to abuse.\
**How:** Keep `pnpm` `block-exotic-subdeps=true`

**Goal:** Making security tooling useful in practice by deciding exactly when it should be part of normal work.\
**How:** Document exactly when `syft` and `grype` should run

**Goal:** Increasing consistency so checks happen at predictable moments instead of only when someone remembers.\
**How:** Define whether scans should happen before install, after install, before commit, or before deployment

**Goal:** Building confidence that protections really work under normal developer behavior, not just in theory.\
**How:** Validate blocking behavior for wrapped package managers once all intended package managers are present

**Goal:** Making dependency changes easier to review and less likely to shift silently over time.\
**How:** Prefer pinned dependency versions where practical

**Goal:** Avoiding a false sense of coverage by hardening all major language ecosystems used on the VM, not just one.\
**How:** Review Python package workflow with the same rigor as JavaScript workflow

**Goal:** Reducing the chance that urgent convenience decisions become the weakest point in the supply chain.\
**How:** Decide on a safe process for introducing new package registries or third-party install scripts

## Environment Strategy and Blast-Radius Reduction

When something does get through, you want to limit how much damage it can do. Keeping daily work separate from riskier experiments helps contain the fallout.

**Goal:** Containing the fallout of risky testing by not giving every experiment access to the same trusted environment.\
**How:** Keep separate stable and experimental development environments

**Goal:** Limiting how far a compromise can spread by keeping trust and credentials separated between environments.\
**How:** Keep credentials separated between those environments

**Goal:** Turning environment separation into a usable practice instead of an abstract idea.\
**How:** Define what kinds of work belong in the stable VM versus the experimental VM

**Goal:** Reducing exposure of valuable information by keeping high-trust data out of higher-risk workspaces.\
**How:** Decide what data or secrets should never enter the experimental environment

## Logging, Monitoring, and Recovery Basics

You need some basic logging and monitoring so you can actually see what's happening and recover if things go wrong, without making the whole setup too complicated to maintain.

**Goal:** Keeping enough operational history to understand what happened when something goes wrong.\
**How:** Keep `rsyslog` present and running

**Goal:** Improving resilience in troubleshooting by not depending on a single logging path.\
**How:** Keep systemd journal available

**Goal:** Reducing repetitive hostile traffic without requiring constant manual intervention.\
**How:** Keep `Fail2Ban` installed and enabled

**Goal:** Tuning automated defenses so they are strong enough to matter but realistic enough for everyday use.\
**How:** Tune `Fail2Ban` to `bantime = 1h`, `findtime = 10m`, `maxretry = 7`, `backend = systemd`, and `banaction = nftables`

**Goal:** Improving response to repeated abuse by treating persistent offenders more seriously than casual noise.\
**How:** Enable both `sshd` and `recidive` jails, with `recidive maxretry = 3`, `recidive bantime = 1w`, and `recidive findtime = 1d`

**Goal:** Avoiding silent defensive failure by checking that the protection still works after changes and updates.\
**How:** Periodically test `fail2ban-client status` and config validation

**Goal:** Reducing operational risk by deciding in advance how to recover from mistakes without undoing the whole hardening model.\
**How:** Define a simple recovery plan for lockouts or bad hardening changes

## Kernel and OS-Level Baseline Hardening

These are some lower-level kernel and OS tweaks that make certain kinds of local abuse or post-compromise poking around harder, without usually breaking your normal tools.

**Goal:** Reducing what untrusted local code can observe about other running processes.\
**How:** Keep `kernel.yama.ptrace_scope = 1`

**Goal:** Limiting low-level system information that can help an attacker understand or target the kernel more effectively.\
**How:** Keep `kernel.kptr_restrict = 1`

**Goal:** Reducing exposure of sensitive system details that are useful for debugging but also useful for attackers.\
**How:** Keep `kernel.dmesg_restrict = 1`

**Goal:** Making certain filesystem abuse techniques harder to use in multi-user or semi-trusted environments.\
**How:** Keep `fs.protected_hardlinks = 1`

**Goal:** Reducing a class of file-redirection tricks that can be used to target higher-trust processes.\
**How:** Keep `fs.protected_symlinks = 1`

**Goal:** Balancing tighter isolation against developer-tool compatibility before changing a setting that can break workflows.\
**How:** Review `kernel.unprivileged_userns_clone` carefully before changing it

**Goal:** Looking for extra containment in temporary storage without adopting settings that create constant friction.\
**How:** Review whether hardened mount options for `/tmp` and `/var/tmp` are practical

## Validation and Housekeeping

Hardening isn't a "set it and forget it" thing you have to verify it actually works and keep it from drifting as your tools and workflow evolve.

**Goal:** Verifying that the real network-facing posture matches the intended design, not just the configuration on paper.\
**How:** Confirm that only SSH is publicly exposed

**Goal:** Preserving usability so the hardened workflow remains the one people actually keep using.\
**How:** Verify that the development workspace is functioning in practice

**Goal:** Reducing clutter and overhead after the recovery window closes and the change is considered stable.\
**How:** Merge or delete the Hyper-V checkpoint after the stability window

**Goal:** Maintaining the security baseline over time instead of freezing it at the moment of first hardening.\
**How:** Apply deferred phased package upgrades when they become available

**Goal:** Keeping documentation aligned with reality as the toolchain and workflow evolve.\
**How:** Revalidate this checklist after major tooling changes

**Goal:** Preventing gradual drift by revisiting the hardening model on a recurring basis.\
**How:** Review the checklist on a recurring schedule

We're reached the end, by now you should have a pretty decently secured Linux development environment wrapped in a VM when you're developing from Windows. I try to cover as many points as possible, and there is even more that can be done (Network adapter switches, Host key creation/setting, Hyper-V Settings etc). This will bring you to a good, hardened system that you can work from. 

Copy and paste this article into your favorite AI to evaluate the security of your systems and where gaps still exist 🙂.
