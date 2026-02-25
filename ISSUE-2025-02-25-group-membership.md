# Issue: Group Membership Not Applied After usermod

**Date:** 2025-02-25
**Status:** Open
**Priority:** Medium

## Problem

After running `sudo usermod -aG incus-admin $USER`, the group membership is not immediately effective:

```bash
# Step 1: Add user to group
sudo usermod -aG incus-admin $USER

# Step 2: Try to use incus immediately
incus admin init --minimal
# Error: Failed to connect to local daemon... permission denied
```
This happens on every new bootstrap attempt unless the user has fully logged out/in since the last `usermod`.

## Real-World Experience

**Important:** In practice, `logout/login` was NOT sufficient. The group membership only became effective after a **full system reboot**.

This may be due to:
- Systemd user sessions caching group info
- Display manager (GDM/SDDM) not fully restarting on logout
- Some background processes holding old group state

**Recommendation for documentation:** Tell users to reboot after `usermod` for first-time setup.
This happens on every new bootstrap attempt unless the user has fully logged out/in since the last `usermod`.

## Root Cause

Linux only applies group membership changes to:
- **New** login sessions
- **New** shells started after the change

Group changes do NOT affect:
- Current shell session
- Already running tmux sessions
- Any running processes

## Current Workaround

The bootstrap currently uses `newgrp incus-admin` as a workaround:

```bash
sudo usermod -aG incus-admin $USER
newgrp incus-admin  # Temporary fix - only affects current shell
incus admin init --minimal  # Works now
```

However, `newgrp` only works within that specific shell invocation and doesn't persist.

## Potential Solutions

### Option A: Document in SKILL.md (Minimal)
Add a note explaining that:
- First-time bootstrap requires logout/login or `newgrp`
- After that, group membership persists across sessions

### Option B: Detect and Warn in Skill
Add a check at bootstrap start:

```bash
# Check if user is in incus-admin group
if ! groups | grep -q incus-admin; then
  echo "ERROR: User not in incus-admin group."
  echo "Run: sudo usermod -aG incus-admin \$USER"
  echo "Then: logout and login again, OR run: newgrp incus-admin"
  exit 1
fi
```

### Option C: Auto-apply via newgrp (Current Behavior)
Keep the current workaround but document it better.

### Option D: Use sg Command
Alternative to newgrp that might work better in scripts:

```bash
sg incus-admin -c "incus admin init --minimal"
```

## Recommendation

Option A + B: Add detection check and clear documentation.

The group membership issue only affects **first-time bootstrap**. Once the user has logged out/in once, the problem never recurs. The skill should:
1. Check group membership before bootstrap
2. Provide clear instructions if missing
3. Document this as a one-time setup requirement

## Files to Update

- [ ] `SKILL.md` — Add prerequisite check and note
- [ ] `SKILL.ru.md` — Same in Russian

---

*Created during bootstrap testing session*
