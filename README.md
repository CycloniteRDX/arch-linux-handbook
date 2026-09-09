# Arch Linux Handbook

A topic-oriented reference for understanding, operating, and repairing the
Arch Linux systems documented in the companion runbook and post-install
repositories.

## Project status

Stable first reference edition with active extensions. The core subsystem,
operation, diagnosis, and recovery guides are published and reviewed, and the
daily-command reference records the first hardware-validated workstation
baseline. Further articles are added by topic rather than by repeating the
installation sequence. The modular-desktop guide now also records the
hardware-validated chapter 23 refinement of Kitty, Mako, Fuzzel, swaylock, and
the swayidle system-resume path; the accepted static swaybg policy; and the
hardware-validated chapter 24 tuigreet presentation. The shell and editor
foundation now also records chapter 25's validated Bash, Nano, Micro, and Vim
packages, while the workstation guide distinguishes BlueZ service state from
the selected adapter-off-at-boot policy. The boot guide now records chapter
26's hardware-validated RogueOS Plymouth presentation, its reuse of packaged
Arch assets, and the current systemd TPM2-token discovery behavior observed in
the textual fallback. The final desktop review now also records the completed
GTK/Qt consistency pass, Papirus-backed Mako icon lookup, on-demand Bluetooth
policy, and successful global lifecycle validation on 2026-09-09.
The concise clean-install reconstruction belongs to post-install chapter 28.
The handbook remains the explanatory and diagnostic layer and does not
duplicate that command sequence.

## Purpose

The handbook answers questions that would interrupt a concise installation or
post-install procedure:

- What does this command change?
- Why was this design selected?
- Which alternatives exist, and what are their trade-offs?
- How can the result be inspected or diagnosed?
- How can a mistake be recovered safely?

It is not a second installation runbook. Articles should stand on their own
and may be read in any order.

For routine workstation use, keep the
[daily and occasional command cheatsheet](guides/maintenance-and-recovery/27-daily-and-occasional-command-cheatsheet.md)
close at hand.

## Content policy

- Prefer ArchWiki and upstream documentation as technical sources.
- Separate facts from project-specific recommendations.
- State assumptions and risks explicitly.
- Explain commands before presenting destructive examples.
- Keep ordinary examples short.
- Make diagnostic scripts optional and narrowly scoped.
- Never require a large script merely to validate a routine manual step.
- Include recovery instructions when a guide discusses a risky change.
- Do not store credentials, private keys, Secure Boot private material, or
  machine-specific secrets.

## Planned guide families

See the [handbook index](guides/README.md) for the working taxonomy.

## Related repositories

- [Arch Linux Runbook](https://github.com/CycloniteRDX/arch-linux-runbook)
  provides the concise ISO-to-TTY installation path.
- [Arch Linux Post-install](https://github.com/CycloniteRDX/arch-linux-post-install)
  provides the ordered workstation build.
- [Niri Dotfiles](https://github.com/CycloniteRDX/niri-dotfiles) contains the
  user configuration produced by that build.
