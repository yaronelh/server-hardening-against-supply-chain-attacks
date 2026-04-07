# Supply Chain Attacks, Hardening Your Dev Environmen

These days it's ridiculously easy for a malicious dependency, a compromised package, or a sneaky install script to sneak into your development environment and cause real damage. Supply chain attacks keep getting more common, and as developers we're constantly pulling in code we didn't write ourselves. That's the core problem this setup is meant to tackle: build a development environment that's actually resilient to those risks without making your everyday work painful or slow.

I put this checklist together based on the hardening I did for my own environment. It's ordered from the outside in — starting with how you actually connect to the VM, then moving through accounts, networking, services, daily workflow habits, supply chain protections, and finally ongoing maintenance. The idea is to secure the parts you touch every single day first, before getting into the lower-level stuff.

## Table of Contents

- [Infrastructure](#infrastructure2)
- [Why Use a VM Instead of WSL](#why-use-a-vm-instead-of-wsl)
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
- [Suggested Next Actions](#suggested-next-actions)

## Infrastructure <a name="infrastructure2"></a>

I built this around a Windows host, a proper virtual machine layer, and a Linux guest where all the real development work happens. In simple terms, the setup looks like this:

- Windows host  
- Hyper-V virtual machine  
- Ubuntu Server 24.04 LTS guest  
- Development work done inside the Linux guest over SSH or remote-development tooling  

The whole reason for this structure is to create a cleaner separation between your main workstation and the development environment. If something bad slips in through a dependency, package, extension, or script, it should stay contained inside the Linux guest instead of spreading to your Windows machine.

## Why Use a VM Instead of WSL

WSL 2 does use virtualization, but it's designed for really tight integration between Linux and Windows to make life convenient. You can run Linux tools side-by-side with Windows apps, call back and forth between them, and share files easily. Microsoft even describes it as a lightweight utility virtual machine rather than a fully separate traditional VM.

For a lot of regular development work, that tight integration is a nice feature. But when you're serious about supply chain risks, it's the wrong default tradeoff. A dedicated Hyper-V VM creates a much stronger boundary between the Linux workspace and your Windows host. WSL is intentionally built for easy interoperability, which means if the Linux side gets compromised, there are more practical ways for it to reach Windows files, tools, executables, and other resources.

For the threat model I'm working with here, WSL isn't the right choice for the main development environment. It's not that WSL is broken or useless — it's just optimized for convenience and cross-environment access, not for strong isolation. If containing supply chain compromises, protecting credentials, dealing with malicious build scripts, or limiting damage from hostile dependencies matters to you, then a separate dedicated VM is the safer and more appropriate baseline.

## Access and SSH Hardening

SSH is the main way you get into this VM, and it's also how I handle secure port forwarding to tunnel local web traffic without opening extra network ports. This section comes first because SSH is basically the front door, so hardening it properly gives you the biggest immediate payoff.

Reducing one of the most common internet-facing attack paths by removing password-based SSH logins.\
\- Disable SSH password authentication with `PasswordAuthentication no`

Using a lower-privilege remote access pattern so the root account is not used for direct login.\
\- Disable SSH root login with `PermitRootLogin no`

Replacing password-based remote authentication with SSH keys for stronger access control.\
\- Keep SSH key authentication enabled with `PubkeyAuthentication yes`

Reducing unnecessary authentication paths so there are fewer ways to reach the system remotely.\
\- Disable keyboard-interactive authentication with `KbdInteractiveAuthentication no`

Reducing remote-access features that are not needed for a terminal-based development workflow.\
\- Disable X11 forwarding with `X11Forwarding no`

Reducing exposure by limiting SSH access to the accounts that actually need it.\
\- Limit SSH access with `AllowUsers admin`

Lowering the chance of repeated login guessing without making normal use unnecessarily brittle.\
\- Set `MaxAuthTries 7`

Reducing the amount of time attackers or hung sessions can occupy the login path before authentication completes.\
\- Set `LoginGraceTime 30s`

Supporting secure developer access to local web services without opening extra inbound ports.\
\- Keep `AllowTcpForwarding yes` for development tunnels

Keeping SSH port forwarding limited to the intended client side instead of accidentally sharing forwarded services more widely.\
\- Keep `GatewayPorts no`

Keeping access controls aligned with the real operating model so security policy and daily use do not drift apart.\
\- Review whether `AllowUsers admin` should become `AllowUsers admin dev`

## Identity, Privilege, and Workspace Separation

This section is about least privilege — basically giving each account only the access it actually needs. Day-to-day coding should happen under a regular low-privilege account, while anything that needs admin rights stays in a separate account. That way, if something goes wrong during normal work, the damage stays limited.

Separating administration from routine development so a mistake or compromise in daily work has less reach.\
\- Keep `admin` as the admin-capable account

Reducing the damage a dependency, script, or extension can do by defaulting everyday work to a lower-privilege account.\
\- Keep `dev` as the non-sudo day-to-day account

Turning least privilege into a real protection by using the safer account for actual development work.\
\- Perform routine development under `dev`

Keeping ownership boundaries clear so project files do not inherit unnecessary administrative trust.\
\- Keep project repositories under the development user's workspace, for example `/home/dev/projects`

Protecting remote-access credentials because a stolen private key can bypass many other controls.\
\- Restrict the development user's `.ssh` permissions

Protecting signing material and trust stores because they influence what the system accepts as legitimate.\
\- Restrict the development user's `.gnupg` permissions

Reducing cross-user file abuse in shared temporary space.\
\- Confirm `/tmp` retains the sticky bit, typically mode `1777`

Reducing the chance that automation settings, cached secrets, or local tool state become an easy local target.\
\- Review local automation-tool state directory permissions, for example `.codex`

Making sure newly created files are not more broadly writable than the environment actually requires.\
\- Review whether default `umask` should be tighter than `0002`

## Firewall and Network Containment

This part is about limiting what can reach the VM and what the VM can reach outward. The firewall makes inbound traffic deny-by-default, and using NAT keeps the VM from being too exposed on the network. These controls make it much harder for a compromise to spread.

Creating an independent network boundary so exposed services are not controlled only by application defaults.\
\- Enable UFW

Reducing accidental exposure by treating inbound access as something that must be explicitly allowed.\
\- Keep the UFW default policy at `deny incoming` and `allow outgoing`

Keeping the necessary admin entry point available while still minimizing overall exposure.\
\- Keep SSH explicitly allowed inbound on port `22`

Improving visibility so unexpected traffic patterns can be noticed and investigated.\
\- Keep UFW logging enabled

Making it harder for a compromised tool or dependency to pivot into other internal systems.\
\- Preserve outbound RFC1918 deny rules for `10.0.0.0/8`, `172.16.0.0/12`, and `192.168.0.0/16` if they fit the workflow

Reducing unnecessary network exposure from local application servers that are meant for one developer's use.\
\- Avoid opening common development ports such as `3000`, `5000`, `8000`, and `8080` to the network by default

Using the trusted remote-management channel instead of creating extra paths into the VM.\
\- Prefer SSH local port forwarding for web apps

Keeping development services private by default so test servers do not quietly become network-accessible.\
\- Prefer binding dev services to `127.0.0.1` inside the guest

Limiting how directly the VM can interact with the broader network if something inside it is compromised.\
\- Keep the VM on an internal Hyper-V switch with NAT rather than broad LAN exposure

Preventing the host from silently re-exposing services that the guest itself is trying to keep private.\
\- Keep Windows portproxy rules absent unless intentionally required

## Platform and Service Footprint Reduction

The fewer unnecessary packages and services you have running, the smaller your attack surface. If a piece of software doesn't actually support what the VM is used for, it's just extra maintenance and risk.

Reducing software footprint by removing integration tools that do not match the actual virtualization platform.\
\- Remove `open-vm-tools` from a Hyper-V guest when VMware integration is not needed

Removing background software that serves no real purpose in the intended server role.\
\- Remove `ModemManager` if modem hardware is not part of the VM's role

Reducing long-term attack surface by pruning software that remains only out of habit or neglect.\
\- Periodically review installed packages for platform-mismatched or unused components

Keeping the running system easier to reason about by ensuring each enabled service has a clear purpose.\
\- Check whether any remaining services are enabled without supporting the current use case

## Development Workflow Defaults

Security only sticks if it fits naturally into how you actually work every day. The safe path should feel like the default path, not some annoying extra step you have to remember.

Using remote-development tools that fit the secure access model instead of working around it.\
\- Use VS Code Remote SSH or equivalent SSH-native tooling

Ensuring the safer account is the default in real work, not just in policy.\
\- Use `dev` as the default day-to-day remote development identity

Allowing normal application testing without turning every local dev port into a network-facing service.\
\- Keep application access inside SSH tunnels where possible

Reducing accidental exposure by making private-by-default service binding the normal project behavior.\
\- Standardize localhost binding in project templates and run commands

Helping people choose the safer access pattern consistently instead of inventing one-off exceptions.\
\- Document the approved pattern for viewing local web apps from Windows

Preventing convenience exceptions from quietly becoming permanent new exposure.\
\- Define when opening a non-SSH inbound port is acceptable

## Supply Chain Tooling and Package Workflow

A lot of today's compromises happen right here — through package managers, dependencies, and install scripts. This section adds some practical guardrails around the commands that bring in external code.

Adding guardrails around the commands most likely to pull untrusted code into the environment.\
\- Install `safe-chain`

Improving visibility into what is actually installed so suspicious or vulnerable components are easier to spot.\
\- Install `syft`

Catching known-risk components before they blend into normal development work unnoticed.\
\- Install `grype`

Avoiding gaps where protections exist in one shell but not in the account that actually performs the risky action.\
\- Make `safe-chain` available in both `admin` and `dev` contexts

Placing controls at the point where untrusted dependencies are most often introduced.\
\- Wrap `pip3`, `npm`, and `pnpm` through `safe-chain`

Reducing dependency-management risk by preferring tooling with stricter and more reviewable behavior.\
\- Prefer `pnpm` over `npm` for JavaScript work when the project supports it

Creating a buffer against sudden malicious or hijacked package releases by avoiding immediate adoption.\
\- Keep `pnpm` `minimum-release-age=10080`

Limiting dependency resolution paths that are harder to audit and easier to abuse.\
\- Keep `pnpm` `block-exotic-subdeps=true`

Making security tooling useful in practice by deciding exactly when it should be part of normal work.\
\- Document exactly when `syft` and `grype` should run

Increasing consistency so checks happen at predictable moments instead of only when someone remembers.\
\- Define whether scans should happen before install, after install, before commit, or before deployment

Building confidence that protections really work under normal developer behavior, not just in theory.\
\- Validate blocking behavior for wrapped package managers once all intended package managers are present

Making dependency changes easier to review and less likely to shift silently over time.\
\- Prefer pinned dependency versions where practical

Avoiding a false sense of coverage by hardening all major language ecosystems used on the VM, not just one.\
\- Review Python package workflow with the same rigor as JavaScript workflow

Reducing the chance that urgent convenience decisions become the weakest point in the supply chain.\
\- Decide on a safe process for introducing new package registries or third-party install scripts

## Environment Strategy and Blast-Radius Reduction

When something does get through, you want to limit how much damage it can do. Keeping daily work separate from riskier experiments helps contain the fallout.

Containing the fallout of risky testing by not giving every experiment access to the same trusted environment.\
\- Keep separate stable and experimental development environments

Limiting how far a compromise can spread by keeping trust and credentials separated between environments.\
\- Keep credentials separated between those environments

Turning environment separation into a usable practice instead of an abstract idea.\
\- Define what kinds of work belong in the stable VM versus the experimental VM

Reducing exposure of valuable information by keeping high-trust data out of higher-risk workspaces.\
\- Decide what data or secrets should never enter the experimental environment

## Logging, Monitoring, and Recovery Basics

You need some basic logging and monitoring so you can actually see what's happening and recover if things go wrong, without making the whole setup too complicated to maintain.

Keeping enough operational history to understand what happened when something goes wrong.\
\- Keep `rsyslog` present and running

Improving resilience in troubleshooting by not depending on a single logging path.\
\- Keep systemd journal available

Reducing repetitive hostile traffic without requiring constant manual intervention.\
\- Keep `Fail2Ban` installed and enabled

Tuning automated defenses so they are strong enough to matter but realistic enough for everyday use.\
\- Tune `Fail2Ban` to `bantime = 1h`, `findtime = 10m`, `maxretry = 7`, `backend = systemd`, and `banaction = nftables`

Improving response to repeated abuse by treating persistent offenders more seriously than casual noise.\
\- Enable both `sshd` and `recidive` jails, with `recidive maxretry = 3`, `recidive bantime = 1w`, and `recidive findtime = 1d`

Avoiding silent defensive failure by checking that the protection still works after changes and updates.\
\- Periodically test `fail2ban-client status` and config validation

Reducing operational risk by deciding in advance how to recover from mistakes without undoing the whole hardening model.\
\- Define a simple recovery plan for lockouts or bad hardening changes

## Kernel and OS-Level Baseline Hardening

These are some lower-level kernel and OS tweaks that make certain kinds of local abuse or post-compromise poking around harder, without usually breaking your normal tools.

Reducing what untrusted local code can observe about other running processes.\
\- Keep `kernel.yama.ptrace_scope = 1`

Limiting low-level system information that can help an attacker understand or target the kernel more effectively.\
\- Keep `kernel.kptr_restrict = 1`

Reducing exposure of sensitive system details that are useful for debugging but also useful for attackers.\
\- Keep `kernel.dmesg_restrict = 1`

Making certain filesystem abuse techniques harder to use in multi-user or semi-trusted environments.\
\- Keep `fs.protected_hardlinks = 1`

Reducing a class of file-redirection tricks that can be used to target higher-trust processes.\
\- Keep `fs.protected_symlinks = 1`

Balancing tighter isolation against developer-tool compatibility before changing a setting that can break workflows.\
\- Review `kernel.unprivileged_userns_clone` carefully before changing it

Looking for extra containment in temporary storage without adopting settings that create constant friction.\
\- Review whether hardened mount options for `/tmp` and `/var/tmp` are practical

## Validation and Housekeeping

Hardening isn't a "set it and forget it" thing — you have to verify it actually works and keep it from drifting as your tools and workflow evolve.

Verifying that the real network-facing posture matches the intended design, not just the configuration on paper.\
\- Confirm that only SSH is publicly exposed

Preserving usability so the hardened workflow remains the one people actually keep using.\
\- Verify that the development workspace is functioning in practice

Reducing clutter and overhead after the recovery window closes and the change is considered stable.\
\- Merge or delete the Hyper-V checkpoint after the stability window

Maintaining the security baseline over time instead of freezing it at the moment of first hardening.\
\- Apply deferred phased package upgrades when they become available

Keeping documentation aligned with reality as the toolchain and workflow evolve.\
\- Revalidate this checklist after major tooling changes

Preventing gradual drift by revisiting the hardening model on a recurring basis.\
\- Review the checklist on a recurring schedule

## Suggested Next Actions

These are the most useful follow-up steps once the basics are done. They help turn the checklist into something you actually use day-to-day without it becoming dead weight.

Making the intended secure workflow easy to follow and easy to teach.\
\- Write down the official day-to-day workflow for `dev`, SSH tunneling, and localhost-bound services

Increasing the odds that security tooling helps in real life by making it small enough to run regularly.\
\- Define a lightweight routine for running `syft` and `grype` on active projects

Confirming that controls behave the way the team expects before relying on them.\
\- Test `safe-chain` behavior intentionally with representative package operations

Improving the local baseline by treating several small but meaningful controls as one deliberate review.\
\- Review `umask`, `.codex` permissions, and user namespace settings together as one local-hardening pass

Keeping strategy and execution aligned so the written guidance stays connected to the real hardening work.\
\- Keep refining the article outline separately from this checklist
