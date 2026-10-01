# Voice-Only Omarchy Linux A.I. Workstation — Setup & Use Report

## Executive summary

A 2013 MacBook Air (4 GB RAM) running Omarchy Linux can serve a user with roughly 50% vision loss who cannot use a keyboard or mouse. The design rests on three pillars: an always-listening voice command layer, a large-print high-contrast display, and spoken feedback for anything the user cannot read comfortably.

The administrator does all setup and maintenance remotely over Tailscale. The user never types, clicks, or enters a password during normal use.

The hardest problems are not voice recognition. They are the places where Linux quietly expects a keyboard: the disk-encryption passphrase at boot, the lock screen, and push-to-talk dictation keys. Phase 2 removes each of these before any voice work begins.

## Requirements and constraints

Every requirement below must be met by voice alone, with large-print visuals and spoken confirmation.

| Need | How it is met | Notes |
| --- | --- | --- |
| Open Signal by voice | Command layer phrase "open Signal" | Signal Desktop must be linked to a phone first |
| Talk to an AI assistant | Voice question → AI → spoken answer | Browser-based or a small local script |
| Read, compose, archive email | AI assistant with mail access | Email provider still unknown |
| Write, proofread, file documents | LibreOffice + dictation + AI proofreading | Saving to fixed folders by voice |
| Browse the web | Chromium with large zoom | Allowed through the firewall |
| Secure system | Encryption, firewall, auto-updates, no open ports | See Phase 7 |
| Remote admin by admin | Tailscale + SSH | No port forwarding |

Hardware limits shape every choice:

- **CPU:** dual-core Haswell (4th-gen Intel). Speech models must be small and run locally on the CPU.
- **RAM:** 4 GB, not upgradeable. Keep only one heavy app (browser or LibreOffice) open at a time.
- **Display:** 13.3-inch, 1440×900. Scaling to 1.5–2× leaves little screen space, so one full-screen window at a time is the rule.
- **Wi-Fi:** Broadcom BCM4360 on the proprietary wl driver. Assessed, repaired and redeployed under Omarchy; now working without issue (Phase 1).
- **Microphones:** the built-in mics work, but a USB speakerphone or headset sharply improves recognition.

No keyboard or mouse means: no passwords typed, no hotkeys held, no clicking dialogs. Any step that needs one is either automated, voice-mapped, or handled remotely by Jason.

## Architecture overview

Every input is the user's voice; every output is speech or large print. The always-listening command engine is the hub that launches apps, sends keystrokes, and switches dictation on and off.

```mermaid
flowchart LR
    U["User<br/>speaks commands · hears replies · reads large print"]
    C["Command engine<br/>always listening · wake word 'Computer' · offline Vosk"]
    D["Voxtype dictation<br/>started by command · offline Whisper"]
    A["Apps<br/>Signal · Chromium · LibreOffice Writer · AI assistant + email"]
    P["Piper speech<br/>confirms actions · reads text aloud"]
    L["Large display<br/>scale 1.5–1.75× · high contrast"]
    R["Remote admin over Tailscale (no open ports)<br/>Jason: SSH · screen share · omarchy-update · nightly backups to home lab"]
    U --> C
    U --> D
    C -- "starts / stops" --> D
    C --> A
    D --> A
    A --> P
    A --> L
    P -- "results spoken and shown" --> U
    L --> U
    R -. Tailscale .-> A
    style C stroke-width:3px
```

All speech recognition runs offline on the laptop. Only the AI assistant, email, Signal, and the browser use the internet.

## Phase 1 — Install Omarchy and fix Wi-Fi

Omarchy installs normally on this machine, but the Broadcom Wi-Fi card is not recognised out of the box. Plan for a wired connection or an offline driver bundle before you start.

1. Flash the current Omarchy ISO to USB and boot the MacBook holding the Option key.
2. Connect a USB Ethernet adapter, or USB-tether a phone. Omarchy treats a tethered phone as a wired link ([omarchy-broadcom-wifi](https://github.com/Kick-buttowzkii/omarchy-broadcom-wifi)).
3. Complete the install. Use a strong disk-encryption passphrase; Phase 2 handles unlocking it without a keyboard.
4. Install the driver against Omarchy's own kernel: `sudo pacman -Syu broadcom-wl-dkms linux-omarchy-headers`. Omarchy runs `linux-omarchy`, so prebuilt Arch modules will not load ([source](https://github.com/Kick-buttowzkii/omarchy-broadcom-wifi)).
5. Unload the conflicting open drivers and load `wl`: `sudo modprobe -r b43 bcma brcmsmac ssb && sudo modprobe wl` ([Matt Ferrante](https://heyferrante.com/installing-omarchy-on-macbook-air-2012)).
6. Blacklist `b43`, `bcma`, `brcmsmac` and `ssb` in `/etc/modprobe.d/` so the fix survives reboots, then connect to Wi-Fi.

**No Ethernet at all?** Download `broadcom-wl-dkms`, `dkms`, and the matching `linux-omarchy-headers` on another machine, copy them to a USB stick, and install with `pacman -U` ([source](https://github.com/Kick-buttowzkii/omarchy-broadcom-wifi)).

**Update discipline:** always update with `omarchy-update`, never a bare `pacman -Syu`. A kernel update without matching headers silently breaks Wi-Fi, and with it, Jason's remote access. Keep the USB driver bundle in a drawer next to the laptop as a rescue kit.

**Status:** the built-in Wi-Fi has since been assessed, repaired and redeployed under Omarchy, and now works without issue. The driver remains the proprietary `wl` module built by DKMS, so the update discipline above still applies, and its residual security risk is handled by the compensating controls in Patching and vulnerability management.

## Phase 2 — Remove keyboard dependencies

Five places in a normal Omarchy session expect typing or a held key. Each needs a deliberate fix before the machine goes to the user.

| Keyboard dependency | Fix | Trade-off |
| --- | --- | --- |
| Disk-encryption passphrase at boot | See options below | Security vs. independence |
| Login after boot | Omarchy logs straight into Hyprland after the disk unlocks; confirm autologin is active | None, since the disk is encrypted |
| Lock screen after idle | Remove the lock action from `~/.config/hypr/hypridle.conf`; keep only screen dimming, or nothing | An unattended laptop stays open |
| Keyring prompts (Signal, browser) | Set the login keyring's password to blank so it never asks | Secrets rely on disk encryption only |
| Password prompts for admin tasks | Never trigger them locally; Jason does all `sudo` work over SSH | User cannot install software (intended) |

**Boot unlock options** (the 2013 Air has no TPM chip, so automatic TPM unlocking is not possible):

1. **Rare reboots, helper unlocks (recommended).** Leave the machine on and plugged in. Disable suspend on AC power. After the occasional reboot, a trusted person types the passphrase once.
2. **USB key file.** Add a LUKS key slot backed by a file on a small USB stick left in the port. Fully hands-free, but anyone who takes the laptop also takes the key.
3. **Remote unlock over SSH in early boot.** Possible with an initramfs SSH server, but it needs a wired USB Ethernet adapter and is fragile on this hardware. Only worth it if reboots are frequent.

**Power settings:** never sleep on AC power, keep the lid open, and turn the display off (not lock) after a long idle. Any voice command wakes it back up.

## Phase 3 — Low-vision display configuration

With about half of normal vision, the user benefits most from enlargement and strong contrast, backed by speech. A full screen reader is a fallback, not the primary tool (see Phase 5).

Tune these with the user sitting in front of the screen, because the right values depend on their specific condition.

| Setting | Starting value | Where |
| --- | --- | --- |
| Display scale | 1.5, then test 1.75 | Hyprland monitor setting for `eDP-1` |
| Theme | A high-contrast Omarchy theme; test dark vs. light | Omarchy menu → Style → Theme |
| Terminal font | 18–20 pt | Terminal config (Jason only) |
| Top bar | Larger font, fewer modules (clock, battery, Wi-Fi, mic status) | Bar config |
| Notifications | Large font, 10-second timeout, also spoken aloud | Notification daemon config |
| Chromium | Default page zoom 175%, minimum font size raised | Chromium settings → Appearance |
| LibreOffice | Zoom 175%, large icons, high-contrast application colours | Tools → Options → View / Application Colours |
| Signal Desktop | Zoom level raised to maximum comfortable | Signal settings → Appearance |
| Screen magnifier | Hyprland cursor zoom, toggled by voice ("zoom in", "zoom out") | `hyprctl keyword cursor:zoom_factor 2` |

Window rules matter as much as size. Force every app to open full-screen, one at a time, so nothing ever opens off to the side or behind another window.

Config syntax differs between Omarchy versions; recent releases moved Hyprland to a Lua-based config. Check the version on the machine before editing.

## Phase 4 — Voice control

Voice input is split into two engines: a small always-listening **command** engine and a larger on-demand **dictation** engine. One engine doing both would either mishear commands or be too slow for this CPU.

### Dictation: Voxtype (built into Omarchy)

Omarchy 3.3 and later ship optional local dictation powered by Voxtype, running fully offline ([Linuxiac](https://linuxiac.com/arch-based-omarchy-3-3-brings-ai-dictation-hibernation/)). Install it from the Omarchy menu: Install → AI → Dictation.

- Out of the box it is push-to-talk on F9 ([omarchy-speech-orb](https://github.com/nodrej/omarchy-speech-orb)). The user cannot press F9, so the command engine starts and stops it instead: `voxtype record start` / `voxtype record stop`.
- Use the smallest English model that gives acceptable accuracy. The Haswell GPU is not a useful accelerator, so expect a one- to three-second delay per sentence.
- Add the user's contacts, street names, and other proper nouns to `~/.config/voxtype/vocabulary.txt` so they are spelled correctly ([omarchy-dictation](https://github.com/krosdai/omarchy-dictation)).

### Commands: an always-listening phrase engine

Commands need an offline recogniser that listens continuously for a fixed list of phrases and runs a shell command for each. [Numen](https://git.sr.ht/~geb/numen) fits this machine well: it is built for Wayland, uses small offline Vosk models, and maps phrases to actions in plain-text files. Confirm it still builds on current Arch before committing to it.

- Start every command with a wake word such as "Computer" to stop TV or conversation from triggering actions.
- Run it as a `systemd --user` service so it starts at login and restarts if it crashes.
- Each phrase runs a command: launch an app, send a key combination, start dictation, or speak a status.

### Microphone

Buy a USB speakerphone with built-in noise suppression and place it 30–60 cm from the user. Recognition quality depends more on the microphone than on the model. Set it as the default PipeWire input and lock its gain so nothing changes it.

### If voice alone proves too unreliable

A single large USB foot switch or big-button switch, mapped to "start/stop listening", is a common accessibility fallback. It is neither a keyboard nor a mouse, but confirm the user can and wants to use one.

## Phase 5 — Speech output and feedback

The machine must confirm every action out loud, because the user may not see a small change on screen. Speech output has three jobs: confirm, read, and describe.

- **Voice:** install Piper, an offline neural text-to-speech engine that runs well on older CPUs. Wrap it in a one-line `say` script that every other component calls. Fall back to `espeak-ng` if Piper is too slow.
- **Confirm:** play a short distinct sound when a command is recognised, when dictation starts, when it stops, and on any error. Then speak the result: "Signal is open", "Email archived".
- **Read:** a "read this" command copies the current selection or document text and speaks it. A "stop reading" command kills playback immediately.
- **Describe:** a "what's on my screen" command takes a screenshot (`grim`) and asks the AI assistant to describe it aloud. This is the most useful single tool for partial vision.
- **Notifications:** have the notification daemon pipe each new notification to `say`, so new Signal messages are announced by sender name.

**Screen reader (Orca):** install it and route it through Piper, but treat it as a secondary tool. Orca reads well inside Chromium and LibreOffice, but its navigation is keyboard-driven and it has limited awareness of the Hyprland desktop itself. Map only "screen reader on" and "screen reader off" to voice.

## Phase 6 — Applications

Each app gets a small, fixed set of voice commands built from its keyboard shortcuts. Fewer, reliable commands beat a large vocabulary the user cannot remember.

### Signal

Install Signal Desktop and link it to the user's phone with a QR scan during setup. Signal Desktop cannot run without a primary phone account.

- "Open Signal" → launches or focuses it full-screen.
- "Message *name*" → opens the conversation search, types the name, selects the first match, then starts dictation.
- "Send it" → presses Enter. "Cancel message" → clears the box.
- "Read messages" → speaks the last few messages in the open chat.
- "Answer call" / "Hang up" → Signal's call shortcuts. Verify the current shortcuts in Signal's keyboard-shortcut list.

### AI assistant

Run a small voice loop script for everyday questions: "Computer, ask…" records the question, Voxtype transcribes it, an AI API answers, and Piper speaks the reply. This feels like a conversation and needs no screen at all.

For longer work, keep a browser AI chat pinned as an app window. Dictate into it and use "read this" for answers. Store the API key in a file in the user's home directory with `600` permissions, and set a monthly spending cap with the provider.

### Email (provider still unknown)

The AI assistant handles mail through tool calls: list new mail, read a message, draft a reply, archive. How it connects depends on the provider (IMAP with an app password, or the provider's API).

- "Check my email" → speaks sender and subject for each new message.
- "Read the one from *name*" → speaks the body, summarised if long.
- "Reply saying…" → AI drafts, then reads the draft back aloud.
- **Nothing sends without an explicit "send it" after the read-back.** This one rule prevents most AI mistakes from leaving the house.
- "Archive that" → moves it out of the inbox. Never permanently delete by voice.

### LibreOffice Writer

- "New letter" / "New note" → opens a template at 175% zoom with large fonts.
- Dictation fills the document; "new paragraph", "undo that", "delete last sentence" handle edits.
- "Proofread this" → sends the text to the AI, which speaks a short list of fixes and offers "apply fixes".
- "Save as *name*" → saves into a fixed folder (`Documents/Letters`, `Documents/Notes`, `Documents/Forms`) so filing never needs a file dialog.
- "Open my last letter" → reopens the newest file in that folder.

### Chromium

- "Open *site name*" → a short list of named favourites (news, weather, bank, pharmacy).
- "Scroll down / up", "go back", "zoom in / out", "read this page" (reader mode, then spoken).
- Close the browser before opening LibreOffice. With 4 GB of RAM, running both alongside the speech engines will cause slowdowns.

## Phase 7 — Security and remote administration

The laptop exposes nothing to the internet. Jason reaches it only through Tailscale, and the user's account has no admin rights. This phase is the baseline; the security posture sections that follow give the detailed controls, playbooks and verification checks.

### Accounts

- **User account:** runs the desktop session and autologin. Remove it from the `wheel` group so it cannot `sudo`.
- **Admin account (Jason):** key-based SSH only, member of `wheel`, never used for the desktop session.

### Tailscale

1. Install `tailscale`, enable `tailscaled`, and run `tailscale up --ssh`.
2. In the Tailscale admin console, tag the device and write an ACL so only Jason's devices can reach it.
3. Disable key expiry for this device. Otherwise it drops off the tailnet every few months and nobody on-site can re-authenticate it.
4. For screen sharing, run `wayvnc` bound to the Tailscale IP only. Jason sees exactly what the user sees when troubleshooting by phone.

### Firewall

Deny all incoming traffic except on the `tailscale0` interface; allow all outgoing so browsing, Signal, and email work. Confirm `ufw` is enabled and check `ufw status verbose` after every major Omarchy update.

### Updates and recovery

- Jason runs `omarchy-update` over SSH every week or two. Do not enable unattended updates, because of the Wi-Fi driver risk in Phase 1.
- After a kernel update, run `dkms status` and confirm the `wl` module built **before** rebooting. A failed build plus a reboot means no Wi-Fi and no remote access.
- Omarchy's filesystem snapshots allow rolling back a bad update from the boot menu, but that needs someone on-site.

### Backups

Back up `~/Documents` and the Signal Desktop data folder nightly over Tailscale to the home lab's storage, using `restic` or `rsync` from a `systemd` timer. Test a restore once before handover.

### Online safety

- Install uBlock Origin in Chromium and save the few website logins the user needs in the browser's password manager, so they never type a password.
- Instruct the AI assistant to warn aloud about links or requests for money, codes, or passwords in email and Signal.

## Security posture — threat model

This machine carries more risk than a normal laptop because accessibility forces four deliberate weakenings: no lock screen, autologin, a blank keyring, and an AI agent that reads untrusted mail. Every control below exists to compensate for one of those.

**Assets:** the user's email, Signal history, documents, saved website logins, the AI API key, and the Tailscale foothold into Jason's home lab.

| Threat | Likelihood | Impact | Primary controls |
| --- | --- | --- | --- |
| Phishing or scam via email/Signal, acted on by voice | High | High | AI warnings, read-back with recipient, no voice payments, DNS filtering |
| Prompt injection: email or web text steers the AI agent | High | High | Tool allowlist, no free-text to shell, human confirmation on every send |
| Audio injection: TV, video, or visitor speaks a command | Medium | Medium | Wake word, confirmation on high-impact actions, pause listening during media |
| Laptop stolen while powered on (unlocked session) | Medium | High | Remote LUKS wipe, Tailscale removal, credential rotation runbook |
| Laptop stolen powered off | Medium | Low | LUKS2 full-disk encryption, EFI firmware password |
| Vulnerable proprietary Wi-Fi driver exploited over the air | Low | High | Documented risk acceptance plus compensating controls (Patching section) |
| Malicious or compromised package (AUR, plugins, install scripts) | Low | High | Official repos first, review PKGBUILDs, no Omarchy plugins on this machine |
| Tailscale or email account takeover | Low | High | MFA, Tailnet Lock, ACLs that stop the laptop reaching the home lab |
| Laptop used as a pivot into the home lab | Low | High | One-way ACLs, append-only backup target |
| Ransomware or wiper | Low | Medium | Non-admin user, append-only off-device backups |

The highest risks are social: scams and manipulated AI actions. Technical hardening matters, but the AI and voice guardrails carry the most weight.

## Physical, firmware and boot security

With no lock screen, a running laptop is an open session. Physical controls protect the machine when it is off; remote controls protect it when it is on.

1. **Update firmware before wiping macOS.** Apple no longer ships firmware updates for this model, so install the last macOS release and security update it supports first. That leaves the EFI at its final patched version.
2. **Set an EFI firmware password.** Boot into macOS Recovery (Command-Option-R for Internet Recovery works even after Linux is installed) and use the Firmware Password Utility. This blocks booting from USB or recovery without the password. Record it in Jason's password manager; a lost firmware password needs Apple service.
3. **LUKS2 with Argon2id.** Confirm with `cryptsetup luksDump`. Use a passphrase of 5+ random words. Back up the LUKS header to Jason's encrypted storage only if you accept that anyone holding it plus the passphrase can decrypt.
4. **Avoid the USB key-file unlock** unless reboots become frequent. It turns full-disk encryption into a lock with the key taped to it.
5. **Thunderbolt DMA protection.** Add `intel_iommu=on` to the kernel command line in the Limine config and confirm DMAR entries in `dmesg`. This limits direct-memory-access attacks through the Thunderbolt port on a running machine.
6. **Disable unused hardware.** Turn Bluetooth off and mask `bluetooth.service`. If the camera is never used, blacklist the `uvcvideo` module or cover the lens.
7. **Remote wipe capability.** If the laptop is stolen while powered on and online, Jason runs `cryptsetup luksErase` on the LUKS partition over SSH, then powers it off. All key slots are destroyed and the data becomes unrecoverable.

**Compensating controls for the missing lock screen:** keep the laptop in a private room, set screen-off (not suspend) after 10 minutes so content is not on display, and treat the blank keyring as acceptable only because the disk is encrypted at rest.

## Accounts, authentication and remote access

The user account must be unable to change the system, and Jason's access must require more than one factor.

| Control | Setting |
| --- | --- |
| User account | Not in `wheel`; no `sudo`; owns only its home directory |
| Admin account | Separate account for Jason; in `wheel`; `sudo` requires its password; no desktop login |
| Root | Locked: `passwd -l root` |
| SSH | Ed25519 keys only; `PasswordAuthentication no`; `PermitRootLogin no`; `AllowUsers` set to the admin account; listen on the Tailscale IP only |
| Tailscale SSH (if used instead) | ACL action `check`, so sessions require recent re-authentication by Jason's identity provider |
| Tailscale account | MFA on the identity provider; Tailnet Lock enabled so a stolen account cannot add rogue nodes |
| Key expiry | Disabled for this node only (needed for unattended operation); Tailnet Lock compensates |
| Failed logins | `pam_faillock` enabled for the admin account |
| sudo logging | `Defaults logfile=/var/log/sudo.log` and all `sudo` use reviewed weekly |
| Screen sharing | `wayvnc` not running by default; started over SSH when needed, bound to the Tailscale IP, with TLS and a password |
| Email and AI provider accounts | MFA on the primary account; the laptop holds only an app password or scoped API key, never the main password |

The user never learns or types any of these credentials. Jason holds them all in his own password manager.

## Network segmentation

Traffic rules are one-way: Jason can reach the laptop, but the laptop can reach nothing on the tailnet except the backup target. A compromised laptop then cannot pivot into the home lab.

### Tailscale policy (example)

```json
{
  "tagOwners": {
    "tag:a11y-laptop": ["jason@example.com"],
    "tag:backup":      ["jason@example.com"]
  },
  "acls": [
    {"action": "accept", "src": ["jason@example.com"], "dst": ["tag:a11y-laptop:22,5900"]},
    {"action": "accept", "src": ["tag:a11y-laptop"],   "dst": ["tag:backup:8000"]}
  ],
  "ssh": [
    {"action": "check", "src": ["jason@example.com"], "dst": ["tag:a11y-laptop"], "users": ["admin"]}
  ]
}
```

Replace the identity, port 8000 (a restic REST server), and account name with real values. Test with `tailscale ping` and a refused connection from the laptop to any other node.

### Host firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0 to any port 22 proto tcp
sudo ufw allow in on tailscale0 to any port 5900 proto tcp
sudo ufw logging low
sudo ufw enable
```

### DNS and name resolution

- Use a filtering resolver such as Quad9 with DNS-over-TLS in `/etc/systemd/resolved.conf` (`DNS=9.9.9.9#dns.quad9.net`, `DNSOverTLS=yes`). This blocks many known phishing and malware domains before the user reaches them.
- Set `LLMNR=no` and `MulticastDNS=no` in the same file.
- If Tailscale would override DNS, either set the same resolver as the tailnet's global nameserver or run `tailscale set --accept-dns=false`.

### Services and Wi-Fi

- Disable anything listening that is not needed: check with `ss -tulpn`. Typical candidates are `avahi-daemon` and a network-exposed `cups`; keep CUPS on localhost only if the user prints.
- If the home router supports it, put the laptop on a separate SSID or VLAN from smart-home devices, using WPA2 or WPA3.

## Patching and vulnerability management

The built-in Wi-Fi is now working and stays in service. Its remaining risk is a known flaw in the driver code rather than a functional fault, so it is managed by verification, a written risk acceptance and compensating controls.

### Built-in Wi-Fi: status and residual risk

The proprietary `wl` driver has two published heap buffer overflows, CVE-2019-9501 and CVE-2019-9502, triggered by malformed Wi-Fi frames ([CERT/CC VU#166939](https://kb.cert.org/vuls/id/166939/)). In the worst case a nearby unauthenticated attacker could run code on the machine, though denial of service is the more typical outcome ([Debian tracker](https://security-tracker.debian.org/tracker/CVE-2019-9502)). The packaged Linux driver is still version 6.30.223.271, so treat it as unmaintained.

**Current status:** assessed, repaired and redeployed under Omarchy; functioning without issue. A working connection confirms the driver loads and associates correctly. It does not by itself change the code, so the CVEs above still apply unless the running driver has changed.

**Verify which driver is running** and record the result:

```bash
lspci -k | grep -A3 -i network   # "Kernel driver in use"
modinfo -F version wl            # wl version, if wl is in use
dkms status                      # wl built for the running kernel
```

If the output shows `wl` 6.30.223.271, keep the card and apply the controls below. If it shows a different, maintained driver, record that and close this finding.

**Compensating controls while `wl` is in use:**

1. **Write a risk acceptance.** One paragraph: driver, version, CVEs, controls below, review date. Re-review every 6 months.
2. **Join one network only.** Keep a single saved profile for the home network with WPA2-AES or WPA3 and a long passphrase. Delete every other saved network and disable auto-connect to open networks. This keeps the card off untrusted access points.
3. **Keep the laptop at home.** The attack needs radio proximity. A stationary machine on one trusted network has far less exposure than a travelling one.
4. **Turn off Wake-on-WLAN** and anything else the card does beyond normal association.
5. **Watch for driver faults.** Alert on `wl` errors, kernel oopses or repeated disconnects in the forwarded logs. These are the visible symptoms of the denial-of-service form of the bug.
6. **Keep the fallback ready.** A USB adapter with a mainline driver stays on the shelf as a drop-in replacement if a new `wl` flaw is published or the driver stops building.

### Routine patching

| Task | Cadence | Command / method |
| --- | --- | --- |
| System update | Weekly | `omarchy-update` over SSH |
| Known-CVE check on installed packages | Weekly, before updating | `arch-audit` (Arch security tracker) |
| CPU microcode | With system updates | Confirm `intel-ucode` is installed and loaded by the bootloader |
| CPU mitigations | After kernel updates | `grep . /sys/devices/system/cpu/vulnerabilities/*` shows mitigations active |
| Post-update service check | After every update | Voice services, firewall, Tailscale and Wi-Fi all running |
| Reboot | Only when Jason or a helper can unlock the disk | Planned, never automatic |

### Supply chain

- Prefer official Arch and Omarchy repositories. For any AUR package (the command engine may be one), read the PKGBUILD before building and pin the version.
- Do not install Omarchy plugins or community scripts on this machine. Its threat model is different from a developer laptop.
- Keep all custom scripts and configs in a private Git repo that Jason controls, and deploy from there, so any local tampering shows as a diff.

## AI assistant and voice-channel security

The AI agent reads untrusted content (email, web pages, documents) and can act. Any of that content can contain instructions aimed at the AI, so the agent's power must be limited by code, not by its prompt.

### AI agent guardrails

| Control | Implementation |
| --- | --- |
| Tool allowlist | The agent gets only: list mail, read mail, draft reply, archive, read a document, proofread. No forward, delete, rule creation, contact editing, file deletion, or shell access |
| Human confirmation | Sending is a separate step outside the AI: the script speaks the recipient address, subject and body, then waits for "send it" |
| Recipient restrictions | Replies only to the original sender. A new or changed recipient triggers a spoken warning and a second confirmation |
| No data exfiltration paths | The agent cannot fetch arbitrary URLs or attach files from outside `~/Documents` |
| Untrusted-content framing | The system prompt marks email and web text as data, and tells the agent to report embedded instructions aloud instead of following them |
| Scam detection | The agent flags requests for money, gift cards, codes, passwords, urgency, or "don't tell anyone", and suggests calling a trusted person |
| No financial actions | No banking, payments or purchases through the agent or by voice. These stay with a trusted person |
| Least-privilege credentials | Email via an app password or OAuth scope limited to mail; AI API key in its own project with a hard monthly spend limit |
| Audit trail | Every tool call (time, tool, target, confirmation) is logged to a file Jason reviews weekly |

### Voice channel guardrails

- **Fixed phrases only.** The command engine maps exact phrases to fixed scripts. Dictated text is passed as quoted data, never evaluated or spliced into a shell command.
- **Tiered confirmation.** Read-only actions (time, read email) run at once. Actions that send, call, archive or change anything require a spoken confirmation.
- **Audio injection.** A TV, video or visitor can say the wake word. Pause the command engine while media plays in the browser, and keep the confirmation step for every outbound action.
- **No admin by voice.** No voice phrase can install software, change network or firewall settings, or disable security services.
- **Microphone privacy.** Recognition stays on the device and no audio is stored. Show a mic indicator in the top bar and provide "Computer, stop listening" to mute fully.

## Application hardening

Each app is locked down with settings the user cannot change, because the user cannot judge a security prompt they cannot read comfortably.

### Chromium (managed policy)

Save as `/etc/chromium/policies/managed/a11y-hardening.json`, owned by root. Confirm at `chrome://policy`.

```json
{
  "ExtensionInstallBlocklist": ["*"],
  "ExtensionInstallForcelist": ["<content-blocker-extension-id>"],
  "SafeBrowsingProtectionLevel": 2,
  "DownloadRestrictions": 1,
  "HttpsOnlyMode": "force_enabled",
  "DefaultNotificationsSetting": 2,
  "DefaultPopupsSetting": 2,
  "DefaultGeolocationSetting": 2,
  "AutofillCreditCardEnabled": false,
  "PasswordManagerEnabled": true,
  "PasswordLeakDetectionEnabled": true,
  "BrowserSignin": 0,
  "DeveloperToolsAvailability": 2
}
```

The blocklist stops the user being talked into installing a malicious extension; only the content blocker Jason allows is force-installed. Website notifications are blocked because fake "virus alert" notifications are a common scam. Check that Safe Browsing is actually active on this Chromium build at `chrome://safe-browsing`.

### Signal

- On the user's phone: set a Signal PIN and turn on Registration Lock, so the number cannot be re-registered by someone else.
- Review linked devices monthly; only the phone and this laptop should appear.
- Turn on default disappearing messages if the user is comfortable, to limit what a stolen laptop exposes.

### Email

- MFA on the account; the laptop uses an app password or scoped OAuth token only.
- Turn on the provider's strongest spam and phishing filtering.
- Add Jason as recovery contact where the provider supports it, so a locked or hijacked account can be recovered.

### LibreOffice

- Set macro security to Very High (Tools → Options → Security → Macro Security) and lock the setting in the system-wide configuration so it cannot be lowered.
- Attachments from email open read-only, and only after the AI assistant has read out the sender.

## Logging, monitoring and backup integrity

Jason cannot watch the screen, so the machine must record what happened and ship it off the device where a compromise cannot erase it.

### Logging

- Make the journal persistent: `Storage=persistent` in `/etc/systemd/journald.conf`.
- Forward logs over Tailscale to a collector in the home lab (rsyslog or a Wazuh agent). Add that one port to the laptop's outbound ACL.
- Add `auditd` watches on security-relevant files, for example:

```bash
-w /etc/sudoers -p wa -k priv
-w /etc/sudoers.d/ -p wa -k priv
-w /etc/ssh/sshd_config -p wa -k ssh
-w /etc/ufw/ -p wa -k firewall
-w /etc/chromium/policies/ -p wa -k browser-policy
-w /home/user/.config/systemd/user/ -p wa -k persistence
-w /home/user/.config/autostart/ -p wa -k persistence
```

(Replace `user` with the real account name.) The last two catch the most common persistence locations for anything run as the user.

### Weekly review (about 15 minutes, over SSH)

- SSH and `sudo` logins: `journalctl -u sshd --since -7d` and the sudo log
- Audit hits: `ausearch -k <key> --start week-ago`
- AI tool-call log: any send to a new recipient, any refused action
- `arch-audit` output and pending updates
- Tailscale admin console: device list unchanged, no new keys

### Backup integrity

- Run a restic REST server on the home lab with `--append-only`. The laptop can add snapshots but cannot delete or overwrite them, so ransomware on the laptop cannot destroy the backups.
- Run pruning and `restic check` from the server side only, monthly.
- Restore a sample file every quarter, and keep a second copy off-site for disaster recovery.

## Incident response

Write these playbooks down before handover, and give the household one instruction: if anything seems wrong, say "Computer, call for help" or phone Jason.

### Laptop lost or stolen

1. Check `tailscale status`. If the laptop is online, SSH in, run `cryptsetup luksErase` on the encrypted partition, then `systemctl poweroff`.
2. Remove the node from the tailnet and revoke its keys in the Tailscale admin console.
3. Revoke the email app password and the AI API key.
4. On the phone, unlink the laptop from Signal (Settings → Linked devices).
5. Change every website password saved in Chromium.
6. Report the theft if needed, and log the time and actions taken.

### Suspected compromise (odd behaviour, unknown sends, unexpected logins)

1. Isolate: tighten the ACL so only Jason's SSH reaches the laptop and the laptop reaches nothing, including the backup server.
2. Preserve evidence: copy the journal, audit log, AI tool-call log and shell histories to the home lab before changing anything.
3. Rotate credentials: email, AI key, saved web logins, and Signal (re-link after rebuild).
4. Rebuild rather than clean: reinstall Omarchy, redeploy configs from the Git repo, restore documents from a snapshot taken before the suspected date.
5. Review the email account's sent items, forwarding rules and recovery settings for attacker changes.

### Scam or manipulation (user acted on a fraudulent message)

1. Contact the bank or card issuer straight away if money or codes were shared.
2. Change any password that was disclosed, and check email forwarding rules.
3. Add the sender's domain or number to block lists, and update the AI's scam warning examples.

## Further hardening (second assessment)

A second review found eight gaps the first baseline did not close. Most come from one fact: the laptop sits in a shared home with an open, unlocked session.

| Priority | Gap | Remediation |
| --- | --- | --- |
| High | Anyone can plug a keystroke-injection device or USB stick into the open session | USBGuard: allow only the devices attached at setup, block everything else; Jason approves new devices over SSH |
| High | The internal keyboard and trackpad work in the unlocked session | Optionally disable them inside Hyprland; Jason re-enables over SSH. Disk unlock at boot is unaffected |
| High | The AI agent runs with all of the user's rights | Run it as a separate service account inside a hardened systemd unit (below) |
| High | "What's on my screen" sends screenshots to a cloud AI, which may capture email or banking pages | Speak a notice before sending; refuse when a banking or password page is focused; save nothing locally; check the provider's data-retention settings |
| Medium | `/boot` is unencrypted and this Mac has no Secure Boot, so a tampered kernel or initramfs could capture the disk passphrase | Firmware password (already set); weekly SHA-256 check of the boot partition against a stored record |
| Medium | Default kernel settings expose more than this machine needs | Hardening sysctls (below) |
| Medium | Chromium can request the microphone and camera | Add `AudioCaptureAllowed: false` and `VideoCaptureAllowed: false` to the Chromium policy; Signal Desktop is unaffected |
| Low | Secrets can leak through crash dumps and clipboard history | Set `Storage=none` in `/etc/systemd/coredump.conf`; clear the clipboard after "read this"; no persistent clipboard history |

### USBGuard

```bash
sudo pacman -S usbguard
# attach every device the user needs first (speakerphone, foot switch)
sudo sh -c 'usbguard generate-policy > /etc/usbguard/rules.conf'
sudo systemctl enable --now usbguard
usbguard list-devices            # later: usbguard allow-device -p <id>
```

The internal keyboard and trackpad on this MacBook sit on the USB bus, so confirm they appear as allowed before enabling, or helpers lose them.

### Hardened service for the AI agent

Run the agent as its own system account, with access to `~/Documents` and its own credential file only:

```ini
[Service]
User=a11y-agent
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/home/user/Documents
PrivateTmp=yes
PrivateDevices=yes
CapabilityBoundingSet=
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
SystemCallFilter=@system-service
LockPersonality=yes
MemoryDenyWriteExecute=yes
```

Check the result with `systemd-analyze security <unit>`. If the agent is a Node or Python runtime that needs JIT memory, remove `MemoryDenyWriteExecute` and note it.

### Kernel hardening sysctls

Save as `/etc/sysctl.d/90-a11y-hardening.conf` and apply with `sysctl --system`:

```ini
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.unprivileged_bpf_disabled = 1
net.core.bpf_jit_harden = 2
kernel.yama.ptrace_scope = 2
kernel.kexec_load_disabled = 1
fs.suid_dumpable = 0
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv4.conf.all.rp_filter = 2
net.ipv4.tcp_syncookies = 1
```

`rp_filter` is set to loose (2) rather than strict because strict mode can interfere with Tailscale.

### Boot partition integrity

After each planned update, Jason records `sha256sum` of every file on the boot partition into the Git repo. The weekly review compares the live hashes with that record; any change not explained by an update is treated as suspected tampering (Incident response).

## Security baseline verification

Run this check at handover and monthly after that. Any failed row is a finding to fix before the next review.

| ID | Control | Verify | Expected |
| --- | --- | --- | --- |
| SEC-01 | Disk encrypted | `lsblk -f`; `cryptsetup luksDump <dev>` | LUKS2, Argon2id |
| SEC-02 | EFI firmware password | Hold Option at boot | Password prompt before boot picker |
| SEC-03 | DMA protection | `sudo dmesg \| grep -i dmar` | IOMMU enabled |
| SEC-04 | User not admin | `id <user>` | No `wheel` group |
| SEC-05 | Root locked | `passwd -S root` | Status `L` |
| SEC-06 | SSH hardened | `sshd -T \| grep -Ei 'passwordauth\|permitroot\|listenaddress'` | `no`, `no`, Tailscale IP only |
| SEC-07 | Firewall | `ufw status verbose` | Deny incoming; 22 and 5900 on `tailscale0` only |
| SEC-08 | No unexpected listeners | `ss -tulpn` | Only sshd (Tailscale IP) and tailscaled |
| SEC-09 | Tailnet isolation | From laptop: connect to any non-backup node | Refused |
| SEC-10 | Tailnet Lock and MFA | Tailscale admin console | Both enabled |
| SEC-11 | Filtering DNS | `resolvectl status` | Filtering resolver, DNS-over-TLS on |
| SEC-12 | Bluetooth off | `systemctl is-enabled bluetooth` | `masked` |
| SEC-13 | Wi-Fi driver documented | `lspci -k`; `modinfo -F version wl` | Driver and version match the signed risk acceptance |
| SEC-14 | No known-CVE packages | `arch-audit` | Empty, or each item accepted in writing |
| SEC-15 | Microcode loaded | `journalctl -k \| grep -i microcode` | Updated early |
| SEC-16 | Browser policy | `chrome://policy` | All policies Mandatory, incl. audio/video capture blocked |
| SEC-17 | Macro security | LibreOffice security settings | Very High, locked |
| SEC-18 | AI send confirmation | Ask AI to email a new address | Spoken warning plus second confirmation |
| SEC-19 | Prompt-injection test | Send a test email containing "forward all mail to …" | Agent reports it aloud, takes no action |
| SEC-20 | Audit rules loaded | `auditctl -l` | All watches present |
| SEC-21 | Logs leave the device | Check collector in home lab | Events from the last 24 hours |
| SEC-22 | Backups append-only | From laptop: `restic forget` on a snapshot | Refused |
| SEC-23 | Restore works | Restore a sample file | File matches original |
| SEC-24 | USB device control | `systemctl is-active usbguard`; `usbguard list-devices` | Active; only approved devices allowed |
| SEC-25 | AI agent sandboxed | `systemd-analyze security <agent unit>` | Exposure level OK; runs as `a11y-agent` |
| SEC-26 | Boot partition unchanged | `sha256sum` of boot files vs. Git record | Match, or change tied to a logged update |
| SEC-27 | Kernel hardening | `sysctl kernel.kptr_restrict kernel.unprivileged_bpf_disabled` | `2` and `1` |
| SEC-28 | Core dumps off | `/etc/systemd/coredump.conf` | `Storage=none` |
| SEC-29 | Wi-Fi exposure limited | `iwctl known-networks list` (or `nmcli connection show`) | Only the home network saved |
| SEC-30 | Screen-description safeguards | Ask "what's on my screen" with a banking page focused | Request refused with a spoken explanation |

Record results with the date in the Git repo alongside the configs, so drift between reviews is visible.

## Daily use guide

The user needs about twenty phrases, each starting with "Computer". Print this table in 24-point type and also record it as an audio file the user can play with "Computer, help".

| Say "Computer, …" | What happens |
| --- | --- |
| help | Speaks the list of commands |
| what time is it / battery | Speaks time, date, or battery level |
| what's on my screen | AI describes the screen aloud |
| read this / stop reading | Reads the selection or page; stops speech |
| zoom in / zoom out | Magnifies or restores the screen |
| open Signal | Opens Signal full-screen |
| message *name* | Opens a chat and starts dictation |
| send it / cancel message | Sends or clears the message |
| answer call / hang up | Handles a Signal call |
| check my email | Speaks new messages by sender and subject |
| reply saying … | Drafts a reply and reads it back |
| archive that | Archives the open email |
| ask … | Asks the AI assistant a question aloud |
| new letter / new note | Opens a blank document |
| start writing / stop writing | Starts or stops dictation |
| proofread this / apply fixes | AI checks the document and fixes it |
| save as *name* | Files the document automatically |
| open *site name* | Opens a favourite website |
| scroll down / scroll up / go back | Moves through the page |
| close this | Closes the current app |
| call for help | Sends Jason a Signal alert with a screenshot |

**Typical session:** "Computer, check my email" → "Computer, read the one from Sarah" → "Computer, reply saying I'll be there at three" → hear the draft → "Computer, send it".

Add commands only when the user asks for them. Each new phrase is one more thing to remember and one more chance of a false trigger.

## Testing and acceptance checklist

The machine is ready for handover when every item passes with the keyboard and trackpad physically covered.

- [ ] Cold boot reaches a working desktop with no typing (or with the chosen unlock method)
- [ ] Wi-Fi reconnects after a reboot and after the router restarts
- [ ] No lock screen, keyring prompt, or password dialog appears after 2 hours idle
- [ ] Every phrase in the daily use guide works 9 times out of 10, spoken at normal volume from the user's chair
- [ ] TV or conversation in the room for 30 minutes triggers no commands
- [ ] Dictating a 3-sentence Signal message needs no more than one correction
- [ ] An email reply is drafted, read back, and only sent after "send it"
- [ ] A letter is dictated, proofread, and saved into the right folder by voice alone
- [ ] The user can read the bar, notifications, and documents comfortably at the chosen scale
- [ ] Browser and LibreOffice each run for 30 minutes alongside the speech engines without heavy slowdown
- [ ] Jason can SSH in and view the screen over Tailscale from outside the home network
- [ ] A port scan from outside the tailnet shows no open ports
- [ ] A test file restores from backup
- [ ] After `omarchy-update` and reboot, Wi-Fi and all voice services come back on their own

## Risks, limitations and open items

The biggest risk is a kernel update that breaks Wi-Fi, because it also cuts off Jason's only way in.

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Scam or phishing acted on by voice | Money or account loss | AI scam warnings, no financial actions by voice, filtering DNS |
| Prompt injection steers the AI agent | Mail sent or data leaked | Tool allowlist, confirmation outside the AI, recipient checks |
| Kernel update leaves the `wl` module unbuilt | No Wi-Fi, no remote access | Update via `omarchy-update`; `dkms status` before every reboot; USB rescue kit on-site |
| Built-in Wi-Fi driver flaw exploited (CVE-2019-9501/9502) | Remote code execution or crash | Risk acceptance; one trusted network; laptop stays home; fault alerting; USB adapter as fallback |
| Laptop stolen while running | Open session exposed | Remote `luksErase`, credential rotation playbook |
| Boot partition tampered with (evil maid) | Disk passphrase captured | EFI firmware password, `/boot` hash check, USBGuard |
| Reboot needs the disk passphrase | User stuck at boot screen | Rare reboots plus helper, or USB key file (Phase 2) |
| Misheard or injected voice command | Wrong action | Wake word, tiered confirmation, no admin by voice, pause during media |
| Laptop used to pivot into home lab | Wider compromise | One-way Tailscale ACLs, append-only backups |
| 4 GB RAM runs out | Slowdowns, crashes | One heavy app at a time; smallest speech models |
| Omarchy changes its configs | Custom settings overwritten | Keep all custom configs in a Git repo and re-check after updates |
| Battery ageing on a 2013 laptop | Sudden shutdowns | Keep it on AC power; replace the battery if needed |

**Open items**

- [ ] Identify the user's email provider and choose IMAP or API access
- [ ] Choose the boot-unlock method with the user and household
- [ ] Confirm the user has a phone to link Signal Desktop to
- [ ] Test dark vs. light theme and the display scale with the user present
- [ ] Confirm Numen (or an alternative command engine) builds on the current Omarchy release
- [ ] Decide whether a foot switch is an acceptable fallback
- [ ] Verify the running Wi-Fi driver and sign the risk acceptance for the built-in card
- [ ] Update macOS firmware and set the EFI firmware password before wiping
- [ ] Enable MFA and Tailnet Lock on the Tailscale account
- [ ] Stand up the append-only restic server and log collector in the home lab
- [ ] Pass every row of the security baseline verification table

**Sources** (accessed October 2026)

- [omarchy-broadcom-wifi — offline Broadcom driver on Omarchy](https://github.com/Kick-buttowzkii/omarchy-broadcom-wifi)
- [Installing Omarchy on a MacBook Air 2012 — Matt Ferrante](https://heyferrante.com/installing-omarchy-on-macbook-air-2012)
- [Omarchy 3.3 adds Voxtype dictation — Linuxiac](https://linuxiac.com/arch-based-omarchy-3-3-brings-ai-dictation-hibernation/)
- [omarchy-speech-orb — Voxtype on Omarchy](https://github.com/nodrej/omarchy-speech-orb)
- [omarchy-dictation — Voxtype vocabulary file](https://github.com/krosdai/omarchy-dictation)
- [CERT/CC VU#166939 — Broadcom Wi-Fi driver vulnerabilities](https://kb.cert.org/vuls/id/166939/)
- [Debian security tracker — CVE-2019-9502](https://security-tracker.debian.org/tracker/CVE-2019-9502)
