# Rule: Mandatory System Documentation

## Scope & Trigger
This rule applies whenever an agent creates, modifies, installs, or troubleshoots any component, service, configuration, or workaround on this system (`framboise`). This includes, but is not limited to:
- Creating or editing systemd units (system or user: `/etc/systemd/system/`, `~/.config/systemd/user/`)
- Modifying desktop or compositor configs (Hyprland, Caelestia, Quickshell, waybar, foot)
- Adding or modifying user scripts (e.g., in `~/.local/bin/` or custom daemons)
- Installing or configuring system packages, drivers, or kernel modules
- Changing shell configurations (Fish, Bash, environment variables)
- Tuning hardware, bluetooth, audio (PipeWire/WirePlumber), or network settings

---

## Mandatory Actions Required

Whenever an agent makes any of the system changes described above, the agent **MUST**:

### 1. Create or Update a Dedicated Documentation File
Create a new Markdown runbook in `/home/mathieu/docs/<topic-name>.md` (or update an existing one if the change directly pertains to an already documented component).

Every documentation file must strictly follow this structure:
- **Overview**: Concise statement of the problem, target hardware, and affected components.
- **Context & Root Cause**: Technical diagnosis, error logs, timing issues, or protocol details explaining why the change was required.
- **Solution Implemented**: Complete configuration files, scripts, or unit files with exact absolute paths.
- **Verification & Status**: Practical commands to confirm that the service, binding, or fix is active and functioning correctly.
- **Maintenance & Troubleshooting**: Useful commands for inspecting logs (`journalctl`), restarting units, testing status, or manual overrides.

### 2. Update the Index in `AGENTS.md`
Immediately update the **Existing Documentation Index** table in [`/home/mathieu/docs/AGENTS.md`](file:///home/mathieu/docs/AGENTS.md):
- Add the file name and Markdown link.
- Summarize the purpose and scope.
- List key components, services, or config paths touched.

### 3. Syntax & Shell Conventions
- Code snippets must always specify language tags (`lua`, `python`, `ini`, `bash`, `fish`, `text`, etc.).
- Clearly distinguish commands requiring `sudo` from rootless user commands (`systemctl --user`).
- All interactive commands intended for the user to copy/paste must be formatted for **Fish** shell (e.g. `set -gx`, not `export`).

### 4. Update the git
Add files with `git add`, commit them and push to github
