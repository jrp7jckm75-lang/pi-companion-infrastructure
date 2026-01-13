# Raspberry Pi Setup from Scratch
## Part 0 of the Pi Infrastructure Series

*Written by Red & Rhys for The Annexe*
*January 2026*

---

## What This Covers

Getting a Raspberry Pi from "brand new in box" to "I can SSH into it from my main computer." This is the foundation everything else builds on.

**Time:** 20-30 minutes (most of that is waiting for the SD card to write)

---

## What You Need

### Hardware
- Raspberry Pi 4 or 5
- MicroSD card (32GB minimum, 64GB+ recommended)
- Power supply (official Pi power supply recommended—they're fussy about power)
- Ethernet cable OR WiFi details
- A computer (Mac, Windows, Linux) to write the SD card

### Software
- Raspberry Pi Imager (free, we'll download it)

### Information You'll Need
- Your WiFi network name and password (if not using ethernet)
- What you want to name your Pi
- A username and password for your Pi account

---

## Step 1: Download Raspberry Pi Imager

Go to: https://www.raspberrypi.com/software/

Download the version for your computer:
- macOS: `.dmg` file
- Windows: `.exe` file
- Linux: various options depending on distro

Install it like any other application.

---

## Step 2: Insert Your SD Card

Put your microSD card into your computer. You might need an adapter if your computer doesn't have a microSD slot.

**Note:** Everything on the SD card will be erased. If it's not blank, make sure there's nothing on it you need.

---

## Step 3: Open Raspberry Pi Imager

Launch the application. You'll see three buttons:
- **CHOOSE DEVICE** - Which Pi you have
- **CHOOSE OS** - What operating system to install
- **CHOOSE STORAGE** - Which SD card to write to

---

## Step 4: Choose Your Device

Click **CHOOSE DEVICE** and select your Pi model:
- Raspberry Pi 5
- Raspberry Pi 4
- etc.

This helps the imager show you compatible operating systems.

---

## Step 5: Choose Your OS

Click **CHOOSE OS**.

For this guide, select:
**Raspberry Pi OS (64-bit)**

This is under "Raspberry Pi OS (other)" if you don't see it immediately.

**Why 64-bit matters:** Claude Code requires 64-bit. If you install 32-bit, you'll have to start over.

**Don't pick:**
- Raspberry Pi OS (32-bit) - won't work for Claude Code
- Raspberry Pi OS Lite - no desktop, which is fine, but the full version is easier for beginners
- Other operating systems unless you know what you're doing

---

## Step 6: Choose Your Storage

Click **CHOOSE STORAGE** and select your SD card.

**Be careful here.** Make sure you're selecting the SD card, not your computer's hard drive. The imager will show the size—your SD card is probably 32GB or 64GB.

---

## Step 7: Configure Settings (THE IMPORTANT BIT)

Before you click Write, click the **gear icon** ⚙️ or look for **"Edit Settings"** / **"OS Customisation"**.

This is where you save yourself hours of pain later.

### General Settings

**Set hostname:**
- Pick a name for your Pi (e.g., `claudepi`, `piserver`, `homelab`)
- This is how you'll find it on your network later
- Use lowercase, no spaces

**Set username and password:**
- **Username:** Pick something (e.g., your name, or just `pi`)
- **Password:** Something you'll remember but not `raspberry` (the old default everyone knows)
- **WRITE THESE DOWN.** You'll need them to log in.

**Configure wireless LAN (if using WiFi):**
- Enter your WiFi network name (SSID)
- Enter your WiFi password
- Select your country (for WiFi regulations)

If you're using ethernet, you can skip the WiFi part.

**Set locale settings:**
- Time zone: Your time zone
- Keyboard layout: Your keyboard layout

### Services Tab (CRITICAL)

**Enable SSH:** ✅ **CHECK THIS BOX**

This is the whole point. SSH lets you connect to your Pi from another computer without plugging in a keyboard and monitor.

Select **"Use password authentication"** (the other option, public-key, is more secure but more complex—you can set it up later).

### Save

Click **Save** to keep these settings.

---

## Step 8: Write the Image

Click **Write** (or **Next**, then confirm).

The imager will:
1. Download the OS (if it hasn't already)
2. Write it to the SD card
3. Verify the write

This takes 10-20 minutes depending on your SD card speed and internet connection.

**Don't remove the SD card until it says it's done.**

---

## Step 9: Put the SD Card in Your Pi

Once the imager says it's complete:
1. Eject the SD card properly (don't just yank it)
2. Put it in your Raspberry Pi's microSD slot (on the underside)
3. Connect ethernet if you're using it
4. Plug in the power

The Pi will boot. Give it 2-3 minutes for first boot—it's doing initial setup.

---

## Step 10: Find Your Pi on the Network

Now you need to find your Pi's IP address so you can connect to it.

### Option A: Use the hostname (easiest)

If your network supports mDNS (most home networks do), you can connect using the hostname you set:

```bash
ping claudepi.local
```

(Replace `claudepi` with whatever hostname you chose)

If you get responses, you're good. Note the IP address it shows.

### Option B: Check your router

Log into your router's admin page (usually `192.168.1.1` or `192.168.0.1`) and look for connected devices. Find the one with your Pi's hostname.

### Option C: Use a network scanner

On Mac:
```bash
arp -a
```

Or use an app like "Fing" on your phone to scan your network.

---

## Step 11: Connect via SSH

Open a terminal on your computer:
- **Mac:** Terminal (in Applications > Utilities)
- **Windows:** PowerShell, or install Windows Terminal
- **Linux:** Your terminal of choice

Connect:

```bash
ssh yourusername@claudepi.local
```

Or using the IP address:

```bash
ssh yourusername@192.168.1.XXX
```

(Replace `yourusername` with the username you set, and the address with your Pi's hostname or IP)

### First Connection Warning

The first time you connect, you'll see something like:

```
The authenticity of host 'claudepi.local' can't be established.
ED25519 key fingerprint is SHA256:xxxxxxxxxxx
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Type `yes` and press Enter. This is normal—your computer is just confirming it's never seen this Pi before.

### Enter Your Password

Type the password you set in the imager. You won't see characters as you type—that's normal, it's just hidden.

### Success

You should see something like:

```
Linux claudepi 6.x.x-v8+ #xxxx SMP PREEMPT aarch64

yourusername@claudepi:~ $
```

**You're in.** You're now controlling your Pi remotely.

---

## Step 12: Update Everything

First thing to do on a fresh Pi—update all the software:

```bash
sudo apt update && sudo apt upgrade -y
```

This will take a few minutes. Let it finish.

---

## Quick Security Housekeeping (Optional but Recommended)

### Change the default SSH port (makes automated attacks less likely)

```bash
sudo nano /etc/ssh/sshd_config
```

Find the line `#Port 22` and change it to something like `Port 2222` (remove the #).

Save (Ctrl+O, Enter) and exit (Ctrl+X).

Restart SSH:
```bash
sudo systemctl restart ssh
```

Now you'll connect with:
```bash
ssh -p 2222 yourusername@claudepi.local
```

### Set up SSH keys instead of password (more secure)

On your main computer (not the Pi), generate a key if you don't have one:

```bash
ssh-keygen -t ed25519
```

Press Enter to accept defaults (or set a passphrase if you want).

Copy the key to your Pi:

```bash
ssh-copy-id yourusername@claudepi.local
```

Now you can log in without typing your password every time.

---

## Troubleshooting

### "Connection refused" when trying to SSH

- Is the Pi powered on? (Check for lights)
- Did you enable SSH in the imager settings?
- Are you on the same network as the Pi?
- Try the IP address instead of hostname

### "Host not found" with .local address

Your network might not support mDNS. Use the IP address directly—find it from your router's admin page.

### "Permission denied" when entering password

- Caps lock?
- Did you set a different username than you're trying?
- Try the password you set in the imager, not "raspberry"

### Pi won't boot (no lights, or just red light)

- Is the power supply adequate? Pi 4/5 need decent power
- Did the SD card write complete successfully?
- Try re-imaging the SD card

### WiFi not connecting

- Correct password?
- Correct country setting? (This matters for WiFi channels)
- Try ethernet first to verify the Pi works, then troubleshoot WiFi

---

## What's Next

You now have a Pi you can access remotely via SSH. 

Next step: [Installing Claude Code](claude-code-on-raspberry-pi.md)

---

*Last updated: January 13, 2026*
