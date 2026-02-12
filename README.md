# Steam Deck: Gamescope Deep Dive
[![Last Updated](https://img.shields.io/badge/last%20updated-February%202025-blue)](https://github.com/yourusername/gamescope-deep-dive)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

## Introduction

Comprehensive look into user-facing side of Gamescope with commands and examples. This is being primarily used-tested on Steam Deck on Wayland display‑server protocol. If you are using an X11 display‑server protocol, some of these commands might not function.

All examples and commands are produced with help of Claude, please note that I have not gone through and tested all of the commands listed here.

This is first and foremost used as a learning document. You are welcome to correct any errors found within it. I will try to keep this updated with any new info.
 **Gamescope Version:** 3.16.14.5

<a id="top"></a>
## Table of Contents

### TL;DR Section
- [Quick Start / TL;DR](#quick-start--tldr)

### Display Stack & Architecture
- [First: The Display Stack](#first-the-display-stack)
  - [Layer 1: You Click "Play" in Steam](#layer-1-you-click-play-in-steam)
  - [Layer 2: Gamescope Initializes (Nested Mode)](#layer-2-gamescope-initializes-nested-mode)
  - [Layer 3: Your Wrapper Script Executes](#layer-3-your-wrapper-script-executes)
  - [Layer 4: Proton Initializes](#layer-4-proton-initializes)
  - [Layer 5: Vulkan Layer Stack Loads](#layer-5-vulkan-layer-stack-loads)
  - [Layer 6: Gamescope Composites the Frame](#layer-6-gamescope-composites-the-frame)
  - [Layer 7: Desktop Compositor Displays](#layer-7-desktop-compositor-displays)
  - [What Changes in Gaming Mode (Embedded Mode)](#what-changes-in-gaming-mode-embedded-mode)
  - [Essential Read 1](#essential-read-1)

### DRM and Hardware Control
- [How DRM Actually Works](#how-drm-actually-works)
  - [Why You Can't Send DRM Output from TTY1 to Display on TTY2](#why-you-cant-send-drm-output-from-tty1-to-display-on-tty2)
  - [What "Switching TTYs" Actually Does](#what-switching-ttys-actually-does)

### Gamescope Headless Mode
- [What is Gamescope Headless Mode?](#what-is-gamescope-headless-mode)
  - [Why It Won't Work for TTY1 → TTY2](#why-it-wont-work-for-tty1--tty2)
  - [Problem 1: Headless = No Display Output](#problem-1-headless--no-display-output)
  - [Problem 2: TTYs Don't Share Framebuffers](#problem-2-ttys-dont-share-framebuffers)
  - [Problem 3: DRM/KMS is Exclusive](#problem-3-drmkms-is-exclusive)

### Gamescope Capabilities: CAP_SYS_NICE
 - [What is CAP_SYS_NICE?](#what-is-cap_sys_nice)
   - [Step 1: Check If Gamescope Has CAP_SYS_NICE](#step-1-check-if-gamescope-has-cap_sys_nice-capability)
   - [Step 2: Check Runtime Messages](#step-2-check-gamescopes-runtime-messages)
   - [Step 3: Check Process Priority](#step-3-check-gamescopes-actual-process-priority-while-running)
   - [Step 4: Check Thread Priorities (Advanced)](#step-4-check-thread-priorities-advanced)
   - [Step 5: Check Systemd Service](#step-5-check-systemd-service-configuration-gaming-mode)
   - [When CAP_SYS_NICE Matters](#when-cap_sys_nice-matters-most)
   - [How to Grant CAP_SYS_NICE](#how-to-grant-cap_sys_nice-properly)
- [Important Caveats](#️-important-caveats)
- [Quick Diagnostic Script](#quick-diagnostic-script)
- [Understanding Thread Breakdown](#understanding-thread-breakdown)

### Execution Flow
- [Command Breakdown: Execution Flow](#command-breakdown-execution-flow)
  - [Part 1: Environment Variables (Before gamescope)](#part-1-environment-variables-before-gamescope)
  - [Part 2: Gamescope and Its Arguments](#part-2-gamescope-and-its-arguments)
  - [Part 3: The Separator `--`](#part-3-the-separator---)
  - [Part 4: env Command](#part-4-env-command)
  - [Part 5: Your Wrapper Script](#part-5-your-wrapper-script)
  - [Part 6: %command% Expansion](#part-6-command-expansion)
- [Complete Execution Flow with Locations](#complete-execution-flow-with-locations)
  - [Step 1: Steam Client (TTY2, in KDE Plasma)](#step-1-steam-client-tty2-in-kde-plasma)
  - [Step 2: Gamescope (TTY2, nested in Plasma's Wayland)](#step-2-gamescope-tty2-nested-in-plasmas-wayland)
  - [Step 3: env (Inside gamescope's Xwayland, TTY2)](#step-3-env-inside-gamescopes-xwayland-tty2)
  - [Step 4: Your Wrapper (Inside gamescope's Xwayland, TTY2)](#step-4-your-wrapper-inside-gamescopes-xwayland-tty2)
  - [Step 5: Proton (Inside gamescope's Xwayland, TTY2)](#step-5-proton-inside-gamescopes-xwayland-tty2)
  - [Step 6: Your Game (Inside gamescope's Xwayland, TTY2)](#step-6-your-game-inside-gamescopes-xwayland-tty2)
- [Physical Location: Where Each Process Actually Runs](#physical-location-where-each-process-actually-runs)
- [Environment Variable Inheritance Table](#environment-variable-inheritance-table)

### Environment Variables
- [Do Exported Variables Get Passed to Games? Do They Persist?](#do-exported-variables-get-passed-to-games-do-they-persist)
  - [Do Environment Variables Pass to Child Processes?](#do-environment-variables-pass-to-child-processes)
  - [You Can Verify This](#you-can-verify-this)
  - [How Steam Launch Options Interact](#how-steam-launch-options-interact)
- [Do Exported Variables Persist?](#do-exported-variables-persist)
  - [Scenario A: Export in Terminal/Script (Temporary)](#scenario-a-export-in-terminalscript-temporary)
  - [Scenario B: Export in Shell RC File (Persistent)](#scenario-b-export-in-shell-rc-file-persistent)
  - [Scenario C: Export in Login Script (Persistent, All Sessions)](#scenario-c-export-in-login-script-persistent-all-sessions)
  - [Scenario D: Systemd Service Environment (Permanent)](#scenario-d-systemd-service-environment-permanent)
- [Q1: Do the "steamscope launcher variables" only affect Steam?](#q1-do-the-steamscope-launcher-variables-only-affect-steam)
  - [Variables That Only Affect Steam Client](#variables-that-only-affect-steam-client)
  - [Variables That Affect Gamescope/Games](#variables-that-affect-gamescopegames)
  - [How to Actually Enable FSR/NIS Upscaling](#how-to-actually-enable-fsrnis-upscaling)
  - [Simplified Script Example](#simplified-script-example)
  - [Verification](#verification)

### TTY Deep Dive
- [What is a TTY? Historical Context → Modern Implementation](#historical-context--modern-implementation)
- [Types of TTYs in Modern Linux](#types-of-ttys-in-modern-linux)
  - [1. Virtual Console TTYs (/dev/tty1 through /dev/tty63)](#1-virtual-console-ttys-devtty1-through-devtty63)
  - [2. Pseudo-TTYs (PTY) (/dev/pts/0, /dev/pts/1, etc.)](#2-pseudo-ttys-pty-devpts0-devpts1-etc)
  - [3. Special TTY Devices](#3-special-tty-devices)
- [Critical Question: What Do TTYs Inherit and Share?](#critical-question-what-do-ttys-inherit-and-share)
  - [✅ SHARED Across ALL TTYs (System-Wide)](#-shared-across-all-ttys-system-wide)
  - [❌ NOT SHARED (TTY-Specific)](#-not-shared-tty-specific)
- [Practical Example: What Happens When You Switch from TTY2 → TTY3](#practical-example-what-happens-when-you-switch-from-tty2--tty3)

### Using TTY to utilize Direct Mode
  - [Running Gamescope on TTY2 (Current Setup)](#running-gamescope-on-tty2-current-setup)
  - [Running Gamescope on TTY3 (Your Proposed Solution)](#running-gamescope-on-tty3-your-proposed-solution)
- [Smart Two-TTY Approach](#smart-two-tty-approach)
  - [TTY1 Script (Starts Gamescope)](#tty1-script-starts-gamescope)
  - [TTY2 (Opens Steam and Connects to TTY1's Gamescope)](#tty2-opens-steam-and-connects-to-tty1s-gamescope)
  - [What's Happening Here](#whats-happening-here)

### Network Transparency & Backend Modes
- [Key Insight: X11/Xwayland is Network-Transparent](#key-insight-x11xwayland-is-network-transparent)
  - [Why This Does NOT Work with Pure Wayland](#why-this-does-not-work-with-pure-wayland)
- [Gamescope Backend Modes](#gamescope-backend-modes)
  - [`--backend wayland` or `--backend sdl` (Nested Mode)](#--backend-wayland-or---backend-sdl-nested-mode)
  - [`--backend drm` or `-e` (Embedded Mode)](#--backend-drm-or--e-embedded-mode)
  - [Complete Comparison](#complete-comparison)
  - [Auto-Detection Logic](#auto-detection-logic)

### Setup & Configuration
- [Setting Up Auto-Login on TTY2](#setting-up-auto-login-on-tty2)
- [Known Issues & Limitations](#known-issues--limitations)
- [Potential Issues & Solutions](#potential-issues--solutions)
  - [Issue 1: "DISPLAY=:0 connection refused"](#issue-1-display0-connection-refused)
  - [Issue 2: No input on Steam](#issue-2-no-input-on-steam)
  - [Issue 3: Audio might not work](#issue-3-audio-might-not-work)
  - [Your Error Message Explained](#your-error-message-explained)

### Emergency Recovery: Stuck TTY or Gamescope
  - [If Gamescope is Frozen/Stuck](#if-gamescope-is-frozenstuck)
  - [If You're Completely Stuck on a TTY](#if-youre-completely-stuck-on-a-tty)
  - [Preventing TTY Lock-ups](#preventing-tty-lock-ups)
  - [If Gamescope Won't Start](#if-gamescope-wont-start)

### VSync & Performance
  - [Gamescope + VSync Clarification](#gamescope--vsync-clarification)
  - [`-f` / `--fullscreen`](#-f----fullscreen)
  - [`--immediate-flips`](#--immediate-flips)

### FSR/NIS Upscaling
  - [How FSR/NIS Works in Gamescope](#how-fsrnis-works-in-gamescope)
  - [Upscaling Filters](#upscaling-filters)
  - [Common Upscaling Ratios](#common-upscaling-ratios)

### Frame Generation (lsfg-vk)
  - [Essential Read 2](#essential-read-2)
  - [Decky plugin or Manual install](#decky-plugin-or-manual-install)

### Testing & Debugging
  - [Logging to the console](#logging-to-the-console)
  - [Mangohud and gamescope](#MANGOHUD)
  - [gamescopectl (must have an active gamescope session to utilize)](#gamescopectl-must-have-an-active-gamescope-session-to-utilize)
  - [Debug Tips](#debug-tips)
- [How MESA_VK_WSI_PRESENT_MODE Works with Gamescope](#how-mesa_vk_wsi_present_mode-works-with-gamescope)
  - [Test Configurations](#test-configurations)
  - [Test 1: Immediate + Immediate (Lowest Latency)](#test-1-immediate--immediate-lowest-latency)
  - [Test 2: Mailbox + Immediate (Balanced)](#test-2-mailbox--immediate-balanced)
  - [Test 3: Relaxed Inside Gamescope (Smooth Priority)](#test-3-relaxed-inside-gamescope-smooth-priority)
  - [Current Call](#current-call)
  - [Environment Variables and Flags](#environment-variables-and-flags)
### Github Gamescope Discussions
  - [Wayland backend: add host-to-client clipboard sync](#wayland-backend-add-host-to-client-clipboard-sync)
  - [Issues with the WSI layer, hdr, and steam overlay / input failing to hook](#issues-with-the-wsi-layer-hdr-and-steam-overlay--input-failing-to-hook)
  - [Stuttery input, but perfect frametiming](#stuttery-input-but-perfect-frametiming)
  - [gamescope having CAP_SYS_NICE break the steam overlay in nested mode](#gamescope-having-cap_sys_nice-break-the-steam-overlay-in-nested-mode)
  - [Huge lag spikes while moving my cursor in Path of Exile 2](#huge-lag-spikes-while-moving-my-cursor-in-path-of-exile-2)
### Glossary
  - [Core Terms](#Core-Terms)
  - [Gamescope Terms](#Gamescope-Terms)
  - [Performance Terms](#Performance-Terms)
### Resources
- [Resources](#resources-1)

---

## Quick Start / TL;DR

**If you just want to:**

### Get lowest latency gaming (Competitive)
```bash
# Switch to empty TTY (Ctrl+Alt+F3), then:
gamescope -e -W 1920 -H 1080 --immediate-flips -- steam -gamepadui
```
[Full explanation - Complete Execution Flow with Locations](#complete-execution-flow-with-locations)

### Use frame generation (lsfg-vk)
```bash
# In Steam launch options:
ENABLE_GAMESCOPE_WSI=1 gamescope -W 1920 -H 1080 -r 120 -- /home/deck/lsfg-vk %command%
```
[Full explanation: Execution Flow](#frame-generation-lsfg-vk-1)

### Debug why gamescope isn't working
1. [Check your backend mode](#gamescope-backend-modes)
2. [Verify environment variables](#verification)
3. [Check CAP_SYS_NICE](#gamescope-capabilities-cap_sys_nice)

**Common issues:**
- Black screen? → [Backend Modes](#gamescope-backend-modes)
- No audio? → [Issue 3: Audio](#issue-3-audio-might-not-work)
- Gamescope stuck? → [Use gamescopectl shutdown](#gamescopectl-must-have-an-active-gamescope-session-to-utilize)

---  
[← Back to top](#top)

#### First: The Display Stack

### The Complete Display Stack: Steam Deck Desktop Mode

Here's what happens when you click "Play"

#### Layer 1: You Click "Play" in Steam

```
Steam Client (Desktop Mode)
  ↓
Reads launch options
  ↓
Executes: gamescope -W 1920 -H 1080 -f --backend wayland --mangoapp -- env /home/deck/lsfg-vk %command%
```

#### Layer 2: Gamescope Initializes (Nested Mode)

Gamescope starts in nested mode because you're in Desktop Mode:

```
KDE Plasma Wayland Compositor (Your Desktop)
  ↓
Creates a Wayland window for gamescope
  ↓
Gamescope runs as a "microcompositor" INSIDE this window
  ↓
Gamescope creates its own internal Wayland server: "gamescope-0"
  ↓
Gamescope spawns Xwayland server (for X11 app compatibility): ":0" or ":1"
```

#### Layer 3: Your Wrapper Script Executes

```
env /home/deck/lsfg-vk %command%
  ↓
Runs your wrapper script which exports:
  - DISABLE_VK_LAYER_VALVE_steam_fossilize_1=1
  - DISABLE_VK_LAYER_VALVE_steam_overlay_1=1
  - LSFGVK_PROFILE=test
  ↓
Then executes whatever %command% expands to (Proton + game exe)
```

#### Layer 4: Proton Initializes

```
Proton (Steam's Wine-based compatibility layer)
  ↓
Sets up Wine prefix
  ↓
Loads DXVK (DirectX → Vulkan translation) or VKD3D (DX12 → Vulkan)
  ↓
Initializes Vulkan with environment variables from your wrapper
```

#### Layer 5: Vulkan Layer Stack Loads

This is where things get interesting! The Vulkan loader builds a chain of layers:

```
Application (MOTORSLICE.exe)
  ↓
DXVK/VKD3D (DirectX → Vulkan translation)
  ↓
VK_LAYER_LSFGVK_frame_generation (lsfg-vk layer - YOUR FRAME GENERATION)
  ↓
VK_LAYER_FROG_gamescope_wsi_x86_64 (Gamescope's WSI layer)
  ↓
Mesa Vulkan Driver (RADV for AMD)
  ↓
Linux Kernel DRM/KMS
  ↓
AMD GPU Hardware
```

**What each layer does:**

**lsfg-vk layer:**
- Intercepts vkQueuePresentKHR calls
- Reads frames from the game
- Uses Lossless.dll algorithm to generate interpolated frames
- Presents multiple frames for each game frame (based on your multiplier)

**Gamescope WSI layer:**
- Intercepts swapchain creation
- Redirects presentation to gamescope's internal compositor instead of directly to your desktop
- Enables gamescope to use async Vulkan compute for compositing, so you see frames quickly even if the GPU is busy with the next game frame

**Mesa RADV driver:**
- Translates Vulkan commands to AMD GPU instructions
- Handles MESA_VK_WSI_PRESENT_MODE (immediate/mailbox/fifo)

#### Layer 6: Gamescope Composites the Frame

```
Game renders frame → DXVK/VKD3D → lsfg-vk generates 3x frames → Gamescope WSI layer
  ↓
Gamescope's internal compositor receives frames
  ↓
Gamescope composites (if needed):
  - Upscales (from 1920x1080 game to 1920x1080 output in your case)
  - Overlays MangoHud (via --mangoapp)
  - Applies FSR/NIS if specified
  ↓
Gamescope presents to its Wayland buffer
```

**With `--backend wayland`:**
- Gamescope uses Wayland protocols to present to KDE Plasma
- Creates a proper Wayland swapchain

**With `--backend sdl`:**
- Gamescope uses SDL2 library for presentation
- May bypass some Wayland overhead

#### Layer 7: Desktop Compositor Displays

```
Gamescope's Wayland surface
  ↓
KDE Plasma Wayland Compositor (KWin)
  ↓
Composites gamescope window with desktop elements (taskbar, etc.)
  ↓
Presents to DRM/KMS
  ↓
Linux Kernel sends buffer to display controller
  ↓
Steam Deck LCD Panel (800p @ 60Hz)
  ↓
Your eyeballs!
```

### What Changes in Gaming Mode (Embedded Mode)

In Gaming Mode, the stack is MUCH simpler:

```
Application
  ↓
DXVK/VKD3D
  ↓
lsfg-vk layer
  ↓
Mesa RADV
  ↓
Gamescope (runs in EMBEDDED MODE - no nested window)
  ↓
DRM/KMS DIRECTLY (no desktop compositor!)
  ↓
LCD Panel
```

**Why embedded mode is faster:**
- Gamescope can use DRM/KMS to directly flip game frames to the screen, removing compositor overhead
- No desktop compositor (KDE Plasma) in between
- Nested sessions incur performance overhead from running a compositor within a compositor
- `--immediate-flips` actually WORKS in embedded mode

---

## Essential Read 1:

Before you move on I recommend you read Gamescope Compatibility section from the maker of Lossless Scaling for linux\
https://github.com/PancakeTAS/lsfg-vk/wiki/Gamescope-Compatibility/760be621450d283c0ac73b6e65e9bc0d9fb6a14e#understanding-how-gamescope-works

---
[← Back to top](#top)
## How DRM Actually Works

The Direct Rendering Manager resides in kernel space and controls the physical display hardware directly. Here's what actually happens:

```
DRM Master Process (gamescope on TTY1)
  ↓
Takes EXCLUSIVE control of /dev/dri/card0
  ↓
Sends frames directly to GPU → Display Pipeline → Your LCD Panel
```

**Critical point:** DRM doesn't render "to a TTY" - it renders to the physical display hardware. TTYs are just sessions that can request control of that hardware.

### Why You Can't Send DRM Output from TTY1 to Display on TTY2

When gamescope runs in DRM mode (embedded mode with `-e` or `--backend drm`):

1. It becomes the DRM master - exclusive control of the GPU
2. It controls the physical display - not a virtual framebuffer
3. Only ONE process can be DRM master at a time

```
TTY1: gamescope -e (DRM master)
  ↓
Controls /dev/dri/card0 EXCLUSIVELY
  ↓
GPU output goes to PHYSICAL DISPLAY
  ↓
The display shows whatever TTY1's DRM master is rendering

TTY2: KDE Plasma
  ↓
CANNOT be DRM master (TTY1 already has it)
  ↓
CANNOT see the display (TTY1 controls it)
  ↓
Is effectively "blind" while TTY1 is active
```

**Only ONE TTY can control the display at a time!**

### What "Switching TTYs" Actually Does

When you press `Ctrl+Alt+F2`:

1. Kernel switches active VT (virtual terminal)
2. TTY2 becomes active and takes control of:
   - Input devices (keyboard/mouse)
   - DRM master (if it has a compositor)
   - Physical display output
3. TTY1 becomes inactive and:
   - Loses input
   - Loses DRM master
   - Its processes keep running but can't display anything

**The display hardware can only show ONE TTY at a time!**

---
[← Back to top](#top)
## What is Gamescope Headless Mode?

Headless mode allows gamescope to work without a display, useful for remote gaming scenarios where you stream the output to another device.

**What it actually does:**

```
Game → Gamescope (--backend headless) → NO DISPLAY OUTPUT
                                        ↓
                                   Encoders only (for streaming)
```

**Headless mode:**
- ✅ Renders frames in GPU memory
- ✅ Runs Vulkan/compositing as normal
- ✅ Can encode frames for streaming (Moonlight, Steam Link, Sunshine)
- ❌ Does NOT output to any TTY/monitor
- ❌ Does NOT create a framebuffer you can see

```bash
# On a headless server (no monitor connected)
gamescope --backend headless -e -W 1920 -H 1080 -- steam -gamepadui

# From another device, stream via Steam Link or Moonlight
# You see the game on your phone/tablet, server has no display
```

### Why It Won't Work for TTY1 → TTY2

Your idea seems to be:
1. Run gamescope headless on TTY1
2. Somehow "pipe" or "display" the output on TTY2

**This fundamentally doesn't work because:**

#### Problem 1: Headless = No Display Output

```
TTY1: gamescope --backend headless -- game
      ↓
      Renders to GPU memory
      ↓
      NO OUTPUT TO ANY TTY (not even TTY1!)
```

Headless mode **intentionally doesn't draw to the screen**. It only:
- Keeps frames in GPU buffers
- Optionally encodes them for streaming
- Does NOT use DRM/KMS to display on a monitor

#### Problem 2: TTYs Don't Share Framebuffers

Each TTY has its own exclusive framebuffer:

```
TTY1: /dev/fb0 (controlled by TTY1's session)
TTY2: /dev/fb0 (controlled by TTY2's session when active)
```

When you switch TTYs, the kernel switches which TTY owns the display. There's no mechanism to "share" or "pipe" framebuffer output between TTYs.

#### Problem 3: DRM/KMS is Exclusive

Gamescope embedded mode uses DRM/KMS, which requires exclusive access to the display connector. Only one application can control DRM at a time:

- If gamescope on TTY1 grabs DRM → TTY2 can't display anything
- If KDE on TTY2 grabs DRM → TTY1's gamescope can't use embedded mode

**Note:** Headless mode could be experimental and buggy. Not recommended for local gaming.

---
[← Back to top](#top)
## Gamescope Capabilities: CAP_SYS_NICE

### What is CAP_SYS_NICE?

CAP_SYS_NICE is a Linux capability that allows a process to set higher scheduling priorities for itself. For gamescope, this means it can run critical threads with realtime FIFO scheduling, which significantly reduces latency and improves frame pacing.

**Quick links:**
- [How to check if you have it](#step-1-check-if-gamescope-has-cap_sys_nice-capability)
- [How to enable it](#how-to-grant-cap_sys_nice-properly)
- [When you need it](#when-cap_sys_nice-matters-most)

---

### Step 1: Check If Gamescope Has CAP_SYS_NICE Capability

Run this command:
```bash
getcap $(which gamescope)
```

**Expected outputs:**

**If CAP_SYS_NICE is set:**
```
/usr/bin/gamescope cap_sys_nice=eip
```

**If NOT set (most common in Desktop Mode):**
```
(no output or "= cap_sys_nice+eip")
```

**Next steps:**
- If set: [Verify it's working](#step-3-check-gamescopes-actual-process-priority-while-running)
- If not set: [Learn when you need it](#when-cap_sys_nice-matters-most) or [How to enable it](#how-to-grant-cap_sys_nice-properly)

---

### Step 2: Check Gamescope's Runtime Messages

When you launch gamescope, watch the console output:
```bash
gamescope -W 1920 -H 1080 -f --backend wayland -- vkcube
```

**If CAP_SYS_NICE is missing, you'll see:**
```
No CAP_SYS_NICE, falling back to regular-priority compute and threads.
Performance will be affected.
```

**If CAP_SYS_NICE is present:**
```
(no warning about CAP_SYS_NICE)
```

---

### Step 3: Check Gamescope's Actual Process Priority While Running

While gamescope is running, open another terminal and run:
```bash
ps -eo pid,ni,comm | grep gamescope
```

**Output explanation:**
```
  PID  NI COMMAND
 12345   0 gamescope         ← Nice value 0 (normal priority, CAP_SYS_NICE not working)
 12345 -10 gamescope         ← Nice value -10 (elevated priority, CAP_SYS_NICE working!)
```

- `NI = 0`: Normal priority (CAP_SYS_NICE not working or not set)
- `NI = -10` or lower: Elevated priority (CAP_SYS_NICE working)

**Want more detail?** See [Thread Priorities (Advanced)](#step-4-check-thread-priorities-advanced)

---

### Step 4: Check Thread Priorities (Advanced)

For a more detailed view of gamescope's thread scheduling:
```bash
ps -eLo pid,tid,class,ni,comm | grep gamescope
```

**Expected output with CAP_SYS_NICE working:**
```
  PID   TID CLS  NI COMMAND
12345 12345  TS -10 gamescope        ← Main thread with nice -10
12345 12346  FF   0 gamescope:cs0    ← Realtime FIFO thread (compute)
12345 12347  FF   0 gamescope:cs1    ← Realtime FIFO thread
```

**Without CAP_SYS_NICE:**
```
  PID   TID CLS  NI COMMAND
12345 12345  TS   0 gamescope        ← Normal priority
12345 12346  TS   0 gamescope:cs0    ← No realtime priority
```

**Legend:**
- `CLS = TS`: Normal time-sharing scheduler
- `CLS = FF`: FIFO realtime scheduler (best for gamescope)
- `NI`: Nice value (lower = higher priority)

**Learn more:** [Understanding Thread Breakdown](#understanding-thread-breakdown)

---

### Step 5: Check Systemd Service Configuration (Gaming Mode)
```bash
systemctl cat gamescope-session.service
```

You'll likely see something like:
```ini
[Service]
AmbientCapabilities=CAP_SYS_NICE CAP_SYS_RESOURCE
CapabilityBoundingSet=CAP_SYS_NICE CAP_SYS_RESOURCE
```

This shows that in Gaming Mode, CAP_SYS_NICE is enabled by default through systemd.

---

### When CAP_SYS_NICE Matters Most

**You NEED it for:**
- High framerate targets (120+ FPS)
- VRR/Adaptive sync scenarios
- CPU-bound games
- Frame generation (like lsfg-vk!) - you want consistent scheduling

**It matters less for:**
- 30-60 FPS targets
- GPU-bound games
- Single-threaded games

**Ready to enable it?** [Jump to How to Grant CAP_SYS_NICE](#how-to-grant-cap_sys_nice-properly)

---

### How to Grant CAP_SYS_NICE Properly

If you want to enable it:
```bash
sudo setcap 'CAP_SYS_NICE=+eip' $(which gamescope)
```

Verify it worked:
```bash
getcap $(which gamescope)
```

Should show: `/usr/bin/gamescope cap_sys_nice=eip`

**⚠️ IMPORTANT:** Read the [caveats below](#️-important-caveats) before enabling!

**Verify it's working:** [Check Process Priority](#step-3-check-gamescopes-actual-process-priority-while-running)

---

### ⚠️ Important Caveats

Using `setcap` causes some Vulkan environment variables to be ignored, such as `MESA_VK_DEVICE_SELECT`. Also, it can stop the Steam In-Game Overlay from working.

**Workaround for overlay:**

You can restore Steam functionality by bypassing `LD_PRELOAD` on gamescope and passing it to the command:
```bash
gamescope -- env LD_PRELOAD="$LD_PRELOAD" %command%
```

**Note:** Your current wrapper already has `LD_PRELOAD=""` which clears it, so you might lose Steam overlay if you enable CAP_SYS_NICE.

**Related:** See [Verification section](#verification) for checking environment variables

---

### Quick Diagnostic Script

Save this as `~/check-gamescope-priority.sh`:
```bash
#!/bin/bash
echo "=== Gamescope CAP_SYS_NICE Check ==="
echo ""
echo "1. Checking binary capabilities:"
getcap $(which gamescope)
echo ""
echo "2. Gamescope running processes:"
ps -eo pid,ni,comm | grep gamescope
echo ""
echo "3. Gamescope thread scheduling:"
ps -eLo pid,tid,class,ni,comm | grep gamescope | head -5
echo ""
echo "=== Interpretation ==="
echo "NI = 0: Normal priority (no CAP_SYS_NICE)"
echo "NI < 0: Elevated priority (CAP_SYS_NICE working)"
echo "CLS = TS: Normal scheduler"
echo "CLS = FF: Realtime FIFO scheduler (best)"
```

Make it executable:
```bash
chmod +x ~/check-gamescope-priority.sh
```

Run it:
```bash
~/check-gamescope-priority.sh
```

**What the output means:** [Understanding Thread Breakdown](#understanding-thread-breakdown)

---

### Understanding Thread Breakdown

Example output with CAP_SYS_NICE enabled:
```
  PID   TID CLS  NI COMMAND
45061 45061  TS -20 gamescope-wl     ← Main Wayland compositor thread (MAX PRIORITY)
45061 45062  TS   0 gamescope-eis    ← EIS (input emulation) - normal priority
45061 45063  TS   0 gamescope_img    ← Image processing - normal priority
45061 45064  TS -20 gamescope        ← Core gamescope thread (MAX PRIORITY)
45061 45086  TS -20 gamescope-pw     ← PipeWire audio thread (MAX PRIORITY)
45061 45087  TS -20 gamescope-xwm    ← X Window Manager thread (MAX PRIORITY)
45061 45089  TS -20 gamescope-wait   ← Wait/sync thread (MAX PRIORITY)
```

**Key threads that benefit from high priority:**
- `gamescope-wl`: Main compositor thread handling frame composition
- `gamescope`: Core rendering and frame delivery
- `gamescope-pw`: Audio synchronization (critical for AV sync)
- `gamescope-xwm`: Window management (input handling)
- `gamescope-wait`: Frame timing and vsync coordination

**Threads that don't need high priority:**
- `gamescope-eis`: Input emulation (best effort is fine)
- `gamescope_img`: Image processing (can be delayed)

**Back to top:** [CAP_SYS_NICE Table of Contents](#table-of-contents---cap_sys_nice-section)

---
[← Back to top](#top)
#### Next: Execution Flow

### Command Breakdown: Execution Flow

Your command:

```bash
PROTON_LOG=1 LD_PRELOAD="" SteamDeck=0 gamescope -W 1920 -H 1080 -f --backend wayland --mangoapp -- env /home/deck/lsfg-vk %command%
```

#### Part 1: Environment Variables (Before gamescope)

```bash
PROTON_LOG=1 LD_PRELOAD="" SteamDeck=0
```

**Where these are set:** In the parent process (Steam client) before launching gamescope

**What happens:**
1. Steam reads your launch options
2. Steam's shell sets these environment variables
3. Steam then executes the gamescope command
4. Gamescope inherits these variables

**Scope:** These variables are available to:
- ✅ Gamescope process
- ✅ Everything gamescope launches (inherited down the chain)

#### Part 2: Gamescope and Its Arguments

```bash
gamescope -W 1920 -H 1080 -f --backend wayland --mangoapp
```

**Where this runs:** Steam client executes gamescope as a subprocess

**What happens:**
1. Steam spawns gamescope with these arguments
2. Gamescope parses the arguments:
   - `-W 1920 -H 1080` → Internal resolution
   - `-f` → Fullscreen mode
   - `--backend wayland` → Use Wayland nested mode
   - `--mangoapp` → Load MangoHud overlay

**Initiated by:** Steam client (the parent process)

#### Part 3: The Separator `--`

```bash
--
```

**What this means:** "End of gamescope arguments, everything after this is THE COMMAND TO RUN INSIDE GAMESCOPE"

This is a standard Unix convention for separating program arguments from the command it should execute.

**Example breakdown:**

```bash
gamescope [gamescope args] -- [command to run inside gamescope]
```

- Everything before `--` → Gamescope configuration
- Everything after `--` → What gamescope should launch

#### Part 4: env Command

```bash
env /home/deck/lsfg-vk %command%
```

**Where this runs:** Inside gamescope's Xwayland session

**What happens:**
1. Gamescope finishes initializing
2. Gamescope creates its internal Xwayland server (e.g., `DISPLAY=:1`)
3. Gamescope executes whatever comes after `--`, which is: `env /home/deck/lsfg-vk %command%`

**What `env` does:**
- `env` is a command that runs another command with a modified environment
- When used with no arguments like this, it just executes the command that follows
- It's basically a no-op here (could be removed)

**Initiated by:** Gamescope (after it's set up its environment)

**Process hierarchy at this point:**

```
Steam (TTY2, your desktop session)
  └─ gamescope (TTY2, nested in Plasma)
      └─ Xwayland server (DISPLAY=:1, created by gamescope)
          └─ env (running inside gamescope's X server)
              └─ /home/deck/lsfg-vk (your wrapper script)
                  └─ %command% (expanded by Steam to: proton run game.exe)
```

**Key Insight: The env Command Is Redundant**

In your command:

```bash
gamescope ... -- env /home/deck/lsfg-vk %command%
```

The `env` command here is doing nothing useful. You could simplify to:

```bash
gamescope ... -- /home/deck/lsfg-vk %command%
```

**Why `env` might have been included:**
- Old habit from when people used `env` to set variables: `env VAR=value command`
- Copy-pasted from example that used it differently
- Harmless but unnecessary in your case

#### Part 5: Your Wrapper Script

```bash
/home/deck/lsfg-vk
```

**Where this runs:** Inside gamescope's Xwayland session

**What it does:**

```bash
#!/bin/bash
export DISABLE_VK_LAYER_VALVE_steam_fossilize_1=1
export DISABLE_VK_LAYER_VALVE_steam_overlay_1=1
export LSFGVK_PROFILE=test
exec "$@"  # Executes whatever arguments were passed (the %command% part)
```

**Initiated by:** The env command (which was initiated by gamescope)

**Environment at this point:**
- ✅ Has variables from Steam (PROTON_LOG, SteamDeck, etc.)
- ✅ Has variables from gamescope (DISPLAY, WAYLAND_DISPLAY if nested)
- ✅ Adds new variables (DISABLE_VK_LAYER_*, LSFGVK_PROFILE)

#### Part 6: %command% Expansion

```bash
%command%
```

**What this is:** A Steam placeholder that gets replaced by Steam

**Where expansion happens:** In the Steam client, BEFORE launching anything

**What it expands to:** The actual game launch command, something like:

```bash
/home/deck/.local/share/Steam/compatibilitytools.d/proton-ge/proton run /path/to/game.exe
```

**Example:**

```bash
# What you write in Steam:
env /home/deck/lsfg-vk %command%

# What Steam actually executes:
env /home/deck/lsfg-vk /home/deck/.local/share/Steam/compatibilitytools.d/proton-ge/proton run /home/deck/.local/share/Steam/steamapps/common/MOTORSLICE/motorslice.exe
```

**Initiated by:** Your wrapper script's `exec "$@"` runs this expanded command

---
[← Back to top](#top)
### Complete Execution Flow with Locations

Let me trace the full path of execution:

#### Step 1: Steam Client (TTY2, in KDE Plasma)

```
Steam reads launch options
  ↓
Sets environment: PROTON_LOG=1 LD_PRELOAD="" SteamDeck=0
  ↓
Expands %command% to actual game path
  ↓
Executes: gamescope -W 1920 -H 1080 -f --backend wayland --mangoapp -- env /home/deck/lsfg-vk [full game command]
```

#### Step 2: Gamescope (TTY2, nested in Plasma's Wayland)

```
Gamescope receives command and arguments
  ↓
Parses its own arguments (-W, -H, -f, --backend, --mangoapp)
  ↓
Initializes Wayland backend (connects to Plasma's wayland-0)
  ↓
Creates nested Xwayland server (DISPLAY=:1 or :2)
  ↓
Launches command after '--': env /home/deck/lsfg-vk [full game command]
```

#### Step 3: env (Inside gamescope's Xwayland, TTY2)

```
env command runs
  ↓
No environment modifications (env used without args)
  ↓
Executes: /home/deck/lsfg-vk [full game command]
```

#### Step 4: Your Wrapper (Inside gamescope's Xwayland, TTY2)

```
/home/deck/lsfg-vk script runs
  ↓
Exports additional environment variables:
  - DISABLE_VK_LAYER_VALVE_steam_fossilize_1=1
  - DISABLE_VK_LAYER_VALVE_steam_overlay_1=1
  - LSFGVK_PROFILE=test
  ↓
exec "$@" replaces itself with: proton run motorslice.exe
```

#### Step 5: Proton (Inside gamescope's Xwayland, TTY2)

```
Proton initializes
  ↓
Sets up Wine prefix
  ↓
Loads DXVK/VKD3D
  ↓
Loads Vulkan layers (including lsfg-vk because LSFGVK_PROFILE=test)
  ↓
Executes: motorslice.exe
```

#### Step 6: Your Game (Inside gamescope's Xwayland, TTY2)

```
MOTORSLICE.exe runs
  ↓
Renders frames via DXVK → Vulkan
  ↓
lsfg-vk intercepts and generates frames
  ↓
Presents to gamescope's Xwayland
  ↓
Gamescope composites and presents to Plasma
  ↓
Plasma displays on your monitor
```

---

### Physical Location: Where Each Process Actually Runs

All of these processes are running **on TTY2** (your KDE Plasma session):

```
TTY2 (KDE Plasma Desktop Session)
├─ Steam client
│   └─ gamescope (nested window in Plasma)
│       └─ Xwayland server (created by gamescope)
│           └─ env
│               └─ lsfg-vk wrapper
│                   └─ proton
│                       └─ MOTORSLICE.exe
```

They're all on the same TTY! Gamescope is running in nested mode as a window inside your Plasma desktop.

### Environment Variable Inheritance Table

Here's what each process sees:

| Variable | Set By | Steam | Gamescope | Wrapper | Proton | Game |
|----------|--------|-------|-----------|---------|--------|------|
| PROTON_LOG=1 | Launch options | ✅ | ✅ | ✅ | ✅ | ✅ |
| LD_PRELOAD="" | Launch options | ✅ | ✅ | ✅ | ✅ | ✅ |
| SteamDeck=0 | Launch options | ✅ | ✅ | ✅ | ✅ | ✅ |
| DISPLAY=:1 | Gamescope | ❌ | ✅ | ✅ | ✅ | ✅ |
| ENABLE_GAMESCOPE_WSI=1 | Gamescope | ❌ | ✅ | ✅ | ✅ | ✅ |
| DISABLE_VK_LAYER_* | Wrapper | ❌ | ❌ | ✅ | ✅ | ✅ |
| LSFGVK_PROFILE=test | Wrapper | ❌ | ❌ | ✅ | ✅ | ✅ |

---
[← Back to top](#top)
#### Next: Learn About Variables

### Do Exported Variables Get Passed to Games? Do They Persist?

Great question! Let me break this down:

#### Do Environment Variables Pass to Child Processes?

**Yes!** Environment variables are inherited by child processes.

When you do this:

```bash
export LSFGVK_PROFILE=test
steam -gamepadui
```

**The inheritance chain:**

```
bash (has LSFGVK_PROFILE=test)
  └─ steam (inherits LSFGVK_PROFILE=test)
      └─ game launcher (inherits LSFGVK_PROFILE=test)
          └─ proton (inherits LSFGVK_PROFILE=test)
              └─ game.exe (inherits LSFGVK_PROFILE=test!)
```

**Every child process inherits the parent's environment!**

#### You Can Verify This

While a game is running, check its environment:

```bash
# Find the game process
ps aux | grep -i motorslice

# Check its environment (replace PID with actual PID)
cat /proc/<PID>/environ | tr '\0' '\n' | grep LSFGVK
```

You'll see `LSFGVK_PROFILE=test` in the game's environment!

#### How Steam Launch Options Interact

When you set launch options in Steam:

```bash
/home/deck/lsfg-vk %command%
```

What Steam does:

```bash
# Steam internally runs:
/home/deck/lsfg-vk /path/to/proton run game.exe

# Your wrapper script exports more variables:
export LSFGVK_PROFILE=test  # (already inherited from parent, or set here)
exec "$@"  # Runs: /path/to/proton run game.exe

# The game inherits EVERYTHING:
# - Variables from your login shell
# - Variables exported in the TTY session
# - Variables exported in wrapper scripts
```

**Bottom line:** Variables "stack" and get inherited all the way down!

### Do Exported Variables Persist?

This depends on WHERE you export them:

#### Scenario A: Export in Terminal/Script (Temporary)

```bash
# In a terminal or script
export LSFGVK_PROFILE=test
steam
```

**Persistence:** Only for that shell session and its children

- ✅ Steam (child process) gets the variable
- ✅ Games launched from Steam get the variable
- ❌ If you close the terminal and open a new one, variable is GONE
- ❌ Other terminals don't have the variable

#### Scenario B: Export in Shell RC File (Persistent)

Add to `~/.bashrc`:

```bash
export LSFGVK_PROFILE=test
```

**Persistence:** Every new bash shell gets it

- ✅ Every terminal you open has the variable
- ✅ Logins on any TTY get the variable
- ✅ Survives reboots (read from `~/.bashrc` each time)

#### Scenario C: Export in Login Script (Persistent, All Sessions)

Add to `~/.bash_profile` or `~/.profile`:

```bash
export LSFGVK_PROFILE=test
```

**Persistence:** Every login session gets it

- ✅ TTY logins get the variable
- ✅ SSH sessions get the variable
- ✅ Desktop Mode login gets it (if using bash)

#### Scenario D: Systemd Service Environment (Permanent)

For gamescope-session or custom systemd services:

```ini
[Service]
Environment="LSFGVK_PROFILE=test"
```

**Persistence:** Only for that specific service

- ✅ Always set when service starts
- ✅ Survives reboots
- ❌ Only applies to that service's processes

---

### Q1: Do the "steamscope launcher variables" only affect Steam?

**Yes, mostly!** Let me break down each variable:

#### Variables That Only Affect Steam Client

These variables tell the Steam client to enable certain UI features and functionality:

**STEAM_GAMESCOPE_FANCY_SCALING_SUPPORT=1**
- What it does: Enables FSR/NIS upscaling controls in Steam's UI
- Effect: Shows the "Performance" menu in Gaming Mode with FSR slider
- Does it affect games? NO - it just shows/hides the UI controls
- Do you need it? Only if you want Steam's UI to show upscaling options

**STEAM_GAMESCOPE_COLOR_MANAGED=1**
- What it does: Enables HDR/color management controls in Steam's UI
- Effect: Shows color profile options in Steam settings
- Does it affect games? NO - just UI controls
- Do you need it? Only if using HDR or color profiles

**STEAM_MULTIPLE_XWAYLANDS=1**
- What it does: Tells Steam to support per-game Xwayland isolation
- Effect: Each game gets its own Xwayland server (better stability)
- Does it affect games? YES - improves game isolation, prevents crashes
- Do you need it? YES - this is actually useful for stability

**STEAM_GAMESCOPE_VRR_SUPPORTED=1** (commented out in your script)
- What it does: Shows VRR (Variable Refresh Rate) toggle in Steam UI
- Do you need it? Only if you have a VRR/FreeSync/G-Sync monitor

#### Variables That Affect Gamescope/Games

**ENABLE_GAMESCOPE_WSI=1**
- What it does: Enables gamescope's Wayland/Vulkan WSI layer
- Effect: Allows gamescope to intercept Vulkan swapchains
- Does it affect games? YES - critical for gamescope to work
- Do you need it? YES - you already discovered this!

#### How to Actually Enable FSR/NIS Upscaling

FSR/NIS is controlled by gamescope command-line flags, not environment variables:

```bash
# Enable FSR upscaling
gamescope -W 1920 -H 1080 -w 1280 -h 800 -F fsr -- game

# Enable NIS upscaling
gamescope -W 1920 -H 1080 -w 1280 -h 800 -F nis -- game

# No upscaling (linear)
gamescope -W 1920 -H 1080 -w 1280 -h 800 -F linear -- game
```

The `-F` (filtering) flag controls the upscaling algorithm. This works whether or not `STEAM_GAMESCOPE_FANCY_SCALING_SUPPORT` is set!

**Your use case (TTY with manual gamescope):**
- You're launching gamescope manually with your own flags
- You're not using Steam's UI controls
- You DON'T need `STEAM_GAMESCOPE_FANCY_SCALING_SUPPORT=1`

#### Simplified Script Example

```bash
#!/bin/bash

# ===== CRITICAL VARIABLES (Actually needed) =====

# Disable conflicting Steam Vulkan layers
export DISABLE_VK_LAYER_VALVE_steam_fossilize_1=1
export DISABLE_VK_LAYER_VALVE_steam_overlay_1=1

# Enable Gamescope WSI (REQUIRED for gamescope to hook Vulkan)
export ENABLE_GAMESCOPE_WSI=1

# Enable per-game Xwayland isolation (RECOMMENDED for stability)
export STEAM_MULTIPLE_XWAYLANDS=1

# lsfg-vk profile
export LSFGVK_PROFILE=test

# ===== OPTIONAL VARIABLES (Only if using Steam UI features) =====

# Uncomment ONLY if you want Steam's FSR UI controls:
# export STEAM_GAMESCOPE_FANCY_SCALING_SUPPORT=1

# Uncomment ONLY if you want Steam's HDR UI controls:
# export STEAM_GAMESCOPE_COLOR_MANAGED=1

# ===== LAUNCH GAMESCOPE =====

gamescope -e \
  -W 1280 -H 800 \
  -r 60 \
  --immediate-flips \
  --adaptive-sync \
  --mangoapp \
  -- steam -gamepadui
```

#### Verification

While `gamescope + vkcube` is running, check what environment it actually sees:

```bash
cat /proc/$(pgrep gamescope)/environ | tr '\0' '\n' | grep -E 'LSFGVK|MESA|ENABLE'
```

You should see your variables like:

```
LSFGVK_PROFILE=test
DISABLE_VK_LAYER_VALVE_steam_fossilize_1=1
MESA_VK_WSI_PRESENT_MODE=immediate
```

If you DON'T see them, it means CAP_SYS_NICE is causing environment variable sanitization.

---
[← Back to top](#top)
## What is a TTY?

### Historical Context → Modern Implementation

**Evolution:**
- **1900s:** Electro-mechanical teletypewriters for stock tickers
- **1960s-70s:** Mainframe terminals - physical keyboards/screens connected via serial cables
- **1970s:** Unix introduced the `/dev/tty` abstraction to represent these devices
- **Present:** Virtual TTYs - software emulation of physical terminals

### Types of TTYs in Modern Linux

#### 1. Virtual Console TTYs (/dev/tty1 through /dev/tty63)

These are virtual text consoles accessible from the keyboard, emulated in hardware using the screen and keyboard connected to your computer.

**On Steam Deck / Modern Linux:**
- `/dev/tty1` - Usually graphical login (GDM/SDDM)
- `/dev/tty2` - KDE Plasma Wayland session (your Desktop Mode)
- `/dev/tty3-tty6` - Text-only login prompts
- `/dev/tty7+` - Additional sessions if needed

**How to access:**
- `Ctrl+Alt+F1` → TTY1 (login screen)
- `Ctrl+Alt+F2` → TTY2 (your graphical desktop)
- `Ctrl+Alt+F3` → TTY3 (text console)
- etc.

#### 2. Pseudo-TTYs (PTY) (/dev/pts/0, /dev/pts/1, etc.)

A PTY enables bi-directional communication similar to pipes, but provides a terminal interface to any process that requires it.

**Used for:**
- Terminal emulator windows (Konsole, GNOME Terminal)
- SSH sessions
- tmux / screen virtual terminals
- Any software-based terminal

When you open Konsole in Desktop Mode and type `tty`, you'll see something like `/dev/pts/0`.

#### 3. Special TTY Devices

- `/dev/tty` - Always refers to YOUR current TTY (whatever you're on)
- `/dev/tty0` - Currently active virtual console
- `/dev/console` - System console (kernel messages, boot logs)
- `/dev/ttyS0`, `/dev/ttyUSB0` - Serial/USB TTYs (physical ports)

### Critical Question: What Do TTYs Inherit and Share?

This is the KEY question for your gamescope use case! Here's the breakdown:

#### ✅ SHARED Across ALL TTYs (System-Wide)

These things are global and available to all TTYs:

**1. Kernel and System Services**
- Network stack - WiFi, Ethernet, routing tables
- systemd services - NetworkManager, PulseAudio/PipeWire, bluetooth, etc.
- Hardware drivers - GPU, CPU, storage
- Mounted filesystems - /home, /, all your files
- Running daemons - SSH server, database servers, web servers

**Example:** If you connect to WiFi on TTY2 (Desktop Mode), TTY3 will also have network access. NetworkManager is a system service, not per-TTY.

**2. User Processes (Within Same User)**
- Processes started with `&` (background jobs) continue running
- Systemd user services (`systemctl --user`)
- Session-wide services (PipeWire audio, if configured)

**3. /tmp and Shared Memory**
- `/tmp` is the same across all TTYs
- `/dev/shm` (shared memory) is global
- Unix sockets can be accessed from any TTY

#### ❌ NOT SHARED (TTY-Specific)

These things are isolated per TTY session:

**1. Login Session**

Each TTY gets its own login session managed by systemd-logind.

**2. Environment Variables**

Each TTY session has its own environment:

```bash
# TTY2 (Desktop Mode with KDE)
$ echo $XDG_SESSION_TYPE
wayland
$ echo $DISPLAY
:0

# TTY3 (Text console)
$ echo $XDG_SESSION_TYPE
tty
$ echo $DISPLAY
(empty - no X11/Wayland)
```

**This means:**
- `LSFGVK_PROFILE=test` set on TTY2 **does NOT** appear on TTY3
- `PATH`, `HOME`, `USER` are usually the same (from `/etc/profile`)
- Custom exports in `~/.bashrc` only apply to that shell

**3. Graphics/Display Server**

**CRITICAL FOR YOUR USE CASE:**

Each TTY can run its own **independent** graphics session:

**TTY2 (KDE Plasma Desktop Mode):**
```
Wayland Compositor (KWin) running
Xwayland server for X11 apps
Display: :0 or wayland-0
All GPU resources allocated to this session
```

**TTY3 (Text-only):**
```
NO graphical server
Framebuffer console only (text mode)
Direct DRM/KMS access available
```

**TTY4 (If you run gamescope there):**
```
Gamescope running in embedded mode
Direct DRM/KMS access
Independent from TTY2's compositor
```

**Key insight:** The main difference between the Linux console and graphical terminal emulators is the shells in the Linux console are attached directly to TTY devices, whereas shells in a graphical terminal emulator are attached to pseudo-TTYs.

**4. Audio Session (Depends on Configuration)**

Modern Linux uses **PipeWire** or **PulseAudio**:
- **System-wide mode:** Audio server runs globally, all TTYs can access it
- **Per-session mode (default on Steam Deck):** Each login session gets its own audio

On Steam Deck, systemd spawns getty services on demand as you switch to VTs, and each gets its own PipeWire session.

**Test this:** Play music on TTY2, switch to TTY3 - you might lose audio depending on PipeWire config.

**5. Input Devices**

The keyboard and mouse are **grabbed** by whichever TTY has focus:
- When on TTY2 (KDE), your mouse/keyboard control KDE
- Switch to TTY3, KDE **loses** input - TTY3 gets exclusive access
- Gamepads connected via USB **stay** connected system-wide, but events go to active TTY

---

### Practical Example: What Happens When You Switch from TTY2 → TTY3

#### You're on TTY2 (KDE Plasma Desktop Mode):

```
TTY2 Active:
├─ KDE Plasma Wayland Compositor (running)
├─ Steam Client (running)
├─ Discord (running)
├─ Firefox (running)
└─ Input: Keyboard/Mouse → KDE

System Services (always running):
├─ NetworkManager (WiFi connected)
├─ PipeWire (audio server)
├─ systemd-logind (session manager)
└─ Steam downloads (background daemon)
```

#### You Press Ctrl+Alt+F3 → Switch to TTY3:

```
TTY3 Now Active:
├─ Text login prompt OR shell (if already logged in)
├─ Framebuffer console (80x25 text mode or higher with KMS)
└─ Input: Keyboard only (no mouse in text mode)

TTY2 Still Running But Inactive:
├─ KDE Plasma (frozen, no redraws)
├─ Steam Client (running but can't display)
├─ Discord (running, still receives messages)
├─ Firefox (running, tabs still load)
└─ Input: NONE (switched to TTY3)

System Services (still running):
├─ NetworkManager ✅ (WiFi still connected!)
├─ PipeWire ✅ (might mute TTY2's audio, depends on config)
├─ systemd-logind ✅ (managing both sessions)
└─ Steam downloads ✅ (continue in background)
```

**Key insight:** TTY2 applications **keep running** but **can't display anything** because the GPU's output is now showing TTY3.

---
[← Back to top](#top)
## What This Means for Your Gamescope Setup

### Running Gamescope on TTY2 (Current Setup):

```
TTY2:
├─ KDE Plasma Wayland (base compositor)
│   └─ Gamescope (nested window)
│       └─ Steam + Game
└─ Networking: ✅ Shared
└─ GPU: ✅ Shared (but KDE owns the display)
```

**Display stack:**
```
Game → Gamescope → KDE Plasma → DRM/KMS → LCD
```

**Latency:** ~20-40ms extra from KDE compositor

### Running Gamescope on TTY3 (Your Proposed Solution):

```
TTY3:
└─ Gamescope (embedded mode, direct DRM)
    └─ Steam + Game
└─ Networking: ✅ STILL AVAILABLE!
└─ GPU: ✅ Exclusive access (no compositor overhead)
```

**Display stack:**
```
Game → Gamescope → DRM/KMS → LCD
```

**Latency:** Minimal, direct to hardware

---

## Smart Two-TTY Approach

### TTY1 Script (Starts Gamescope):

```bash
#!/usr/bin/bash
export XDG_SESSION_TYPE=x11

tmpdir="$(mktemp -p "$XDG_RUNTIME_DIR" -d -t gamescope.XXXXXXX)"
socket="${tmpdir:+$tmpdir/startup.socket}"

/usr/bin/gamescope -W 2560 -H 1440 --default-touch-mode 4 --hide-cursor-delay 3000 --fade-out-duration 200 -R $socket
```

### TTY2 (Opens Steam and Connects to TTY1's Gamescope):

```bash
DISPLAY=:0 steam -gamepadui -steamos3 -steampal -steamdeck
```

### What's Happening Here:

This is **NOT headless mode** - it's actually:

1. **TTY1:** Runs gamescope in **embedded DRM mode** (takes over the display)
2. **TTY2:** Runs Steam which **connects to gamescope's X server** via `DISPLAY=:0`
3. **Result:** Gamescope displays on screen (via TTY1's DRM), Steam UI runs from TTY2

**The magic:** `DISPLAY=:0` tells Steam on TTY2 to connect to the X server that gamescope created on TTY1!

**This is brilliant!** You get:
- ✅ Gamescope's low-latency DRM output (on TTY1)
- ✅ Steam running from TTY2 (easier to debug/restart)
- ✅ The display shows gamescope's output (from TTY1)

---
[← Back to top](#top)
## Key Insight: X11/Xwayland is Network-Transparent

X11 (and Xwayland) uses a **client-server architecture**:

```
X Server (gamescope on TTY1, :0)
    ↑
    | Unix socket: /tmp/.X11-unix/X0
    |
X Clients (Steam on TTY2, connects via DISPLAY=:0)
```

The X Window System is an architecture that was designed at its core to run over a network. X11's client-server model uses network-transparent protocols:

```
X Server (gamescope on TTY1)
    ↕ Unix socket or TCP
X Clients (Steam on TTY2)
```

The client and server communicate via:
- `/tmp/.X11-unix/X0` (Unix domain socket) - accessible from any TTY
- Or TCP (`localhost:6000`) - also accessible anywhere

**This is why `DISPLAY=:0 steam` on TTY2 can connect to gamescope's Xwayland server on TTY1!**

### Why This Does NOT Work with Pure Wayland

Wayland does not offer network transparency by itself. Wayland's design is fundamentally different:

```
Wayland Compositor (runs on TTY1)
    ↕ Direct shared memory buffers (wl_buffer)
Wayland Clients (must run in same session)
```

**Key differences:**

1. **Wayland uses shared memory buffers, not network protocols**
   - The internal type of wl_buffer object is implementation dependent, but the content data must be shareable between the client and the compositor
   - Shared memory (`/dev/shm`) is per-session, not global

2. **No socket-based communication like X11**
   - Wayland socket: `$XDG_RUNTIME_DIR/wayland-0` (e.g., `/run/user/1000/wayland-0`)
   - This socket is session-specific and typically only accessible to processes in that session

3. **Security by design prevents cross-session access**
   - Wayland, by design, addresses security flaws by sandboxing application access to both graphical output and input, ensuring that no process outside the compositor can view or control other applications unless explicitly permitted

---

## Gamescope Backend Modes

### `--backend wayland` or `--backend sdl` (Nested Mode)

**Requires:** Existing compositor (KDE Plasma, GNOME, etc.)

**What it does:**
```
Gamescope connects to existing Wayland/X11 compositor
    ↓
Creates a window inside that compositor
    ↓
Becomes a "nested compositor" (compositor within compositor)
```

**Example:** Your TTY1 test - Plasma is the host compositor, gamescope is a client

### `--backend drm` or `-e` (Embedded Mode)

**Requires:** Direct access to GPU via DRM/KMS (no existing compositor)

**What it does:**
```
Gamescope talks DIRECTLY to GPU via /dev/dri/card0
    ↓
Takes exclusive control of display
    ↓
No other compositor needed
```

### Complete Comparison

| TTY | Graphical Session | Command | Backend | Result |
|-----|-------------------|---------|---------|--------|
| TTY1 | ✅ KDE Plasma | `gamescope --backend wayland vkcube` | Wayland (nested) | ✅ Works - creates window in Plasma |
| TTY1 | ✅ KDE Plasma | `gamescope -e vkcube` | DRM (embedded) | ❌ Fails - Plasma already owns DRM |
| TTY2 | ❌ None | `gamescope --backend wayland vkcube` | Wayland (nested) | ❌ Fails - no compositor to connect to |
| TTY2 | ❌ None | `gamescope -e vkcube` | DRM (embedded) | ✅ Works - direct GPU access |

### Auto-Detection Logic

When you run gamescope without specifying `--backend`, it:

1. Checks for `$WAYLAND_DISPLAY` → If set, use `--backend wayland`
2. Checks for `$DISPLAY` → If set, use `--backend x11`
3. Otherwise → Use `--backend drm` (embedded mode)

**On TTY1 (Plasma):**
```bash
$ echo $WAYLAND_DISPLAY
wayland-0
# gamescope auto-uses --backend wayland (nested)
```

**On TTY2 (text console):**
```bash
$ echo $WAYLAND_DISPLAY
(empty)
$ echo $DISPLAY
(empty)
# gamescope auto-uses --backend drm (embedded)
```

---
[← Back to top](#top)
## Setting Up Auto-Login on TTY2

Set up autologin so TTY2 automatically runs gamescope when you switch to it:

```bash
sudo systemctl edit getty@tty2
```

Add these lines:

```ini
[Service]
ExecStart=
ExecStart=-/sbin/agetty -o '-p -f -- \\u' --noclear --autologin deck %I $TERM
```

Save and exit. Then create `~/.bash_profile` or edit existing one:

```bash
# Auto-start gamescope on TTY2
if [[ "$(tty)" == "/dev/tty2" ]]; then
    exec ~/.local/bin/steamscope
fi
```

---

## Known Issues & Limitations

Issues with TTY gamescope mode include: game overlay does not work, switching to your DE while a game is running can cause gamescope to crash, giving gamescope CAP_SYS_NICE causes some environment variables to be ignored, and simultaneous HDR and VRR can cause stuttering.

**Workarounds:**
- Don't switch TTYs while a game is loading
- Exit games cleanly before switching back to desktop
- Use `gamescopectl shutdown` to close gamescope cleanly from desktop

---

## Potential Issues & Solutions

### Issue 1: "DISPLAY=:0 connection refused"

Gamescope might create `:1` instead of `:0`

### Issue 2: No input on Steam

You need to be on TTY1 for keyboard/mouse input to work:

```bash
Ctrl+Alt+F1
```

### Issue 3: Audio might not work

PipeWire sessions are per-TTY. Export the correct PipeWire socket:

```bash
# On TTY2 before launching Steam
export PIPEWIRE_RUNTIME_DIR=/run/user/1000
DISPLAY=:0 steam -gamepadui
```

### Your Error Message Explained

```
Gamescope fails to create Wayland objects
```

**This error means:**
1. Gamescope tried to use `--backend wayland` (nested mode)
2. It looked for a Wayland compositor to connect to
3. It tried to create Wayland client objects (windows, surfaces)
4. No Wayland socket exists on TTY2
5. Object creation fails → error

**Solution:** Use `-e` or `--backend drm` on TTY2!

---
[← Back to top](#top)
### Gamescope + VSync Clarification

Gamescope does its own compositing and presentation, so in-game VSync settings are often ignored. Gamescope controls the final presentation to your display.

**Key points:**
- Fullscreen windows in Wayland/X11 typically use compositor redirection
- The desktop compositor (KDE Plasma) enforces its own presentation timing
- This adds a VSync layer that prevents immediate tearing flips

### `-f` / `--fullscreen`

**Makes gamescope's window fullscreen on your desktop**

- In nested mode (Desktop Mode), this tells your desktop compositor (KDE Plasma/Wayland) to make the gamescope window take over the full screen
- When using `-f`, tearing is disabled and VSync is enforced
- Think of it as "windowed fullscreen" mode

**WITH `-f`:**
```
Gamescope requests fullscreen → KDE Plasma makes the window cover entire screen → VSync enforced → --immediate-flips ignored
```

The behavior of `-f` is to use the entire display as the output resolution regardless of `-W` and `-H`, which means your specified resolution becomes suggestions rather than hard limits.

**WITHOUT `-f`:**
```
Gamescope creates a regular window → KDE Plasma treats it like any window → You can resize, minimize, etc.
```

### `--immediate-flips`

- Enables immediate flips in embedded mode, which may result in tearing
- Only works in embedded/DRM mode, NOT nested mode
- Bypasses VSync to show frames immediately as they're ready
- Does NOT work with `-f` because `-f` enforces VSync

**The two flags are mutually exclusive in nested mode** - using them together results in tearing being disabled

---
[← Back to top](#top)
## Emergency Recovery: Stuck TTY or Gamescope

### If Gamescope is Frozen/Stuck

#### Method 1: gamescopectl (Preferred)

From another TTY or SSH:
```bash
# Switch to TTY with working terminal (Ctrl+Alt+F3)
gamescopectl shutdown
```

[More gamescopectl commands](#gamescopectl-must-have-an-active-gamescope-session-to-utilize)

#### Method 2: Kill Process
```bash
# Find gamescope PID
ps aux | grep gamescope

# Kill it (replace PID)
kill -9 [PID]

# Or kill all gamescope instances
pkill -9 gamescope
```

#### Method 3: Force TTY Switch

Sometimes switching TTYs multiple times helps:
```bash
Ctrl+Alt+F1  # Try desktop
Ctrl+Alt+F2  # Try another TTY
Ctrl+Alt+F7  # Traditional graphical login
```

---

### If You're Completely Stuck on a TTY

#### Recovery Steps:

**1. Try to get to a working TTY**
- Press `Ctrl+Alt+F2` (usually Desktop Mode on Steam Deck)
- Or `Ctrl+Alt+F1` (login screen)

**2. If display is frozen but you can type:**
```bash
# Blind typing - you won't see output but it might work:
# Type carefully:
sudo systemctl restart gdm
# Or on Steam Deck:
sudo systemctl restart sddm
```

**3. SSH from another device** (if enabled):
```bash
# From another computer on same network:
ssh deck@steamdeck.local

# Then kill gamescope:
pkill -9 gamescope

# Restart display manager:
sudo systemctl restart sddm
```

**4. Hard reset** (last resort):
- Hold power button for 10+ seconds
- Steam Deck will force shutdown
- Your data is safe, but unsaved game progress will be lost

---

### Preventing TTY Lock-ups

**1. Test in nested mode first:**
```bash
# Safe - runs in window, easy to close
gamescope --backend wayland -- vkcube
```

**2. Set up SSH access** (highly recommended):
```bash
# Enable SSH
sudo systemctl enable sshd
sudo systemctl start sshd

# Set password if not already set
passwd
```

**3. Learn the TTY switching shortcuts:**
- `Ctrl+Alt+F1` through `F7` - Switch between TTYs
- `Ctrl+Alt+Backspace` - Kill X server (some distros)
- `Alt+SysRq+K` - Kill all on current console (if enabled)

**4. Keep one TTY with a working shell:**
- Always have TTY3 logged in with a shell
- From there you can kill stuck processes on other TTYs

---

### If Gamescope Won't Start

**Check these in order:**

1. **Backend mode issue?**
```bash
   # Are you on Desktop Mode (TTY with Plasma)?
   # Use nested mode:
   gamescope --backend wayland -- game
   
   # Are you on empty TTY?
   # Use embedded mode:
   gamescope -e -- game
```
   [Full backend explanation](#gamescope-backend-modes)

2. **DRM already in use?**
```bash
   # Check what's using DRM:
   lsof /dev/dri/card0
   
   # Kill conflicting process or switch to nested mode
```

3. **Missing dependencies?**
```bash
   # Check gamescope can run:
   gamescope --help
   
   # Check for errors:
   gamescope -e -- vkcube
```

4. **Permissions issue?**
```bash
   # Check you're in video group:
   groups
   
   # Add yourself if missing:
   sudo usermod -a -G video $USER
   # Log out and back in
```

---
[← Back to top](#top)
## FSR/NIS Upscaling

### How FSR/NIS Works in Gamescope

Gamescope can upscale games from a lower internal resolution to your display resolution, improving performance while maintaining visual quality.

**Basic syntax:**
```bash
gamescope -w [internal width] -h [internal height] -W [output width] -H [output height] -F [filter] -- game
```

### Upscaling Filters

#### FSR (FidelityFX Super Resolution)
```bash
gamescope -w 1280 -h 800 -W 1920 -H 1080 -F fsr -- game
```
- Best quality/performance balance
- AMD's open-source upscaling
- Recommended for most use cases

#### NIS (NVIDIA Image Scaling)
```bash
gamescope -w 1280 -h 800 -W 1920 -H 1080 -F nis -- game
```
- NVIDIA's upscaling algorithm
- Works on any GPU
- Sharper than FSR, may show more artifacts

#### Linear (No Upscaling)
```bash
gamescope -w 1280 -h 800 -W 1920 -H 1080 -F linear -- game
```
- Simple bilinear scaling
- Fastest, but blurrier
- Use when you don't want upscaling algorithms

### Common Upscaling Ratios

| Internal Res | Output Res | Scaling Factor | Use Case |
|--------------|------------|----------------|----------|
| 1280x720 | 1920x1080 | 1.5x | Balanced performance |
| 960x540 | 1920x1080 | 2.0x | Maximum performance |
| 1280x800 | 2560x1600 | 2.0x | Steam Deck OLED |
| 800x600 | 1600x1200 | 2.0x | Retro games |

### Steam UI Controls

To enable FSR controls in Steam's Gaming Mode UI:
```bash
export STEAM_GAMESCOPE_FANCY_SCALING_SUPPORT=1
```

> **Note**: This only shows/hides the UI controls. The actual upscaling is controlled by gamescope flags.

[Related: Environment Variables](#variables-that-only-affect-steam-client)

---
[← Back to top](#top)
### Frame Generation (lsfg-vk)
#### Essential Read 2
- Please read process of getting LSFG to work in a native Vulkan environment by PancakeTAS here\
https://github.com/PancakeTAS/lsfg-vk/wiki/Porting-LSFG-to-native-Vulkan#figuring-out-how-lsfg-works
- Need Lossless Scaling downloaded on Steam and access to Lossless.dll, file can reside anywhere.

#### Decky plugin or Manual install

#### Decky plug-in
- If you wish to install the Decky plugin you can grab it here\
https://github.com/xXJSONDeruloXx/decky-lsfg-vk or from Decky store\
https://github.com/SteamDeckHomebrew/decky-plugin-store

#### Manual install 
- Manual install for lsfg-vk pre build binaries can be found in\
https://github.com/PancakeTAS/lsfg-vk/releases
- the code was recently updated to v2 and there have been some major changes\
Installation is quite straight‑forward

- download the prebuild lsfg-vk 2.xx and extract
  - copy liblsfg-vk-layer.so to\
/home/deck/.local/lib/liblsfg-vk-layer.so
  - copy VkLayer_LSFGVK_frame_generation.json to\
/home/deck/.local/share/vulkan/implicit_layer.d/VkLayer_LSFGVK_frame_generation.json\
edit VkLayer_LSFGVK_frame_generation.json and change line 7 to correct directory path of liblsfg-vk-layer.so
  - copy lsfg-vk-ui to\
/home/deck/.local/bin/lsfg-vk-ui
  - copy lsfg-vk-cli to\
/home/deck/.local/bin/lsfg-vk-cli
- restart terminal and run ./lsfg-vk-ui
- adjust settings and point to the correct path of Lossless.dll
- your primary adjustment is Multiplier, keep between 2-3
- create a new profile in lsfg-vk-ui adjust settings and close and remember the name
- on your steam launch options add add the profile name to LSFGVK_PROFILE=name of profile
- start the game on steam and you should see a increase in fps

*if you are in steam deck you might need to use DISABLE_VK_LAYER_VALVE_steam_fossilize_1=1 to make lsfg-vk active

---
[← Back to top](#top)
## Testing and Debug

#### Logging to the console

If you are using nested desktop mode in steam deck and want to use gamescope but also need game boot info, launch gamescope
in a seperate terminal and use DISPLAY=:0 %command% as launch option in steam  
[use echo $DISPLAY to find the correct output]

#### MANGOHUD

Using mangohud in gamescope require you to use --mangoapp flag, you can pass the config file by using
MANGOHUD_CONFIGFILE=/home/deck/.config/MangoHud/mangoapp.conf

#### Process monitoring

```bash
# Find gamescope processes
ps -eo pid,comm,cmd | grep -iE 'gamescope'

# Use pstree to get an idea of the tree
pstree

# Use strings to check environment variables for the found process
# (replace [] with PID obtained from ps command above)
sudo strings /proc/[PID]/environ | sort | uniq > ~/Desktop/gamescope-wl_envs.txt
```

#### gamescopectl (must have an active gamescope session to utilize)

```bash
# Dump debug info about the backend state
gamescopectl backend_info

# More accurate way to limit rate than -r flag but more work involved
gamescopectl debug_set_fps_limit

# Not sure if this will work
gamescopectl paint_steam_overlay_plane

# Cleanly shutdown gamescope
gamescopectl shutdown

# Enable or disable adaptive sync
gamescopectl adaptive_sync 1
```

**Adaptive sync options:**
- `adaptive_sync`: Whether or not adaptive sync is enabled if available
- `adaptive_sync_ignore_overlay`: Whether or not to ignore overlay planes for pushing commits with adaptive sync
- `adaptive_sync_overlay_cycles`: Number of vblank cycles to ignore overlay repaints before forcing a commit with adaptive sync

### Debug Tips

Keep `VK_LOADER_DEBUG=all` enabled temporarily to verify the layer stack. Once everything works smoothly, remove it from your wrapper to reduce log spam.

```bash
# Can pull FPS and controller data with this method
--stats-path=/home/deck/gamescope_stat.txt
```

PROTON_LOG=1 display proton based game info on a file created in your home directory.

---

## How MESA_VK_WSI_PRESENT_MODE Works with Gamescope

```
Game → (Proton/DXVK) → Mesa Vulkan WSI → Gamescope WSI Layer → Gamescope Compositor → Display
```

**Available modes:**
- `MESA_VK_WSI_PRESENT_MODE=immediate` - Forces immediate presentation (lowest latency)
- `--immediate-flips` - Reduces latency in gamescope
- `MESA_VK_WSI_PRESENT_MODE=mailbox` - Triple buffering, smooth without tearing
- `vk_xwayland_wait_ready=false` - Reduces wait time
- `ENABLE_GAMESCOPE_WSI=0` - This disables gamescope's WSI layer and uses SDL backend instead - may improve performance in some games

### Test Configurations

#### Test 1: Immediate + Immediate (Lowest Latency)

```bash
LD_PRELOAD="" MESA_VK_WSI_PRESENT_MODE=immediate gamescope -W 1920 -H 1080 -r 60 --immediate-flips --backend wayland --mangoapp -- env /home/deck/lsfg-vk %command%
```

- Use `-r 60` since 20 FPS × 3 = 60 FPS target
- Both game output and gamescope presentation are immediate
- **Expect:** Lowest input lag, possible tearing

#### Test 2: Mailbox + Immediate (Balanced)

```bash
LD_PRELOAD="" MESA_VK_WSI_PRESENT_MODE=mailbox vk_xwayland_wait_ready=false gamescope -W 1920 -H 1080 -r 60 --immediate-flips --backend wayland --mangoapp -- env /home/deck/lsfg-vk %command%
```

- Smooth game→gamescope handoff, fast gamescope→display
- **Expect:** Good balance of smoothness and responsiveness

#### Test 3: Relaxed Inside Gamescope (Smooth Priority)

```bash
LD_PRELOAD="" MESA_VK_WSI_PRESENT_MODE=immediate gamescope -W 1920 -H 1080 -r 60 -f --backend wayland --mangoapp -- env MESA_VK_WSI_PRESENT_MODE=relaxed /home/deck/lsfg-vk %command%
```

- Immediate outside gamescope, relaxed inside
- `-f` enables fullscreen (removes `--immediate-flips`)
- **Expect:** Smoothest framing, slight latency increase

### Current Call

```bash
LD_PRELOAD="" SteamDeck=0 ENABLE_GAMESCOPE_WSI=1 DISABLE_VK_LAYER_VALVE_steam_overlay_1=1
MESA_VK_WSI_PRESENT_MODE=immediate gamescope --rt --immediate-flips -W 1920 -H 1080 -r 120 -f --mangoapp -- %command%

**Note:** `-f` might invalidate `--immediate-flips`
```

---
### Environment Variables and Flags
<details>
<summary>!!NOT TESTED!!</summary>

```bash
# WSI Reported better framelimiting with this enabled/
export ENABLE_GAMESCOPE_WSI=0/
# Enable support for xwayland isolation per-game in Steam/
export STEAM_MULTIPLE_XWAYLANDS=1
# sdl video driver
export SDL_VIDEODRIVER=x11
# hdr support in steam
#export STEAM_GAMESCOPE_HDR_SUPPORTED=1
# Show VRR controls in Steam
#export STEAM_GAMESCOPE_VRR_SUPPORTED=1
# Enable Mangoapp
#export STEAM_MANGOAPP_PRESETS_SUPPORTED=1
#export STEAM_USE_MANGOAPP=1
#export STEAM_DISABLE_MANGOAPP_ATOM_WORKAROUND=1
# Enable horizontal mangoapp bar
#export STEAM_MANGOAPP_HORIZONTAL_SUPPORTED=1
# Scaling support
export STEAM_GAMESCOPE_FANCY_SCALING_SUPPORT=1
# Color management support
export STEAM_GAMESCOPE_COLOR_MANAGED=1
export STEAM_GAMESCOPE_VIRTUAL_WHITE=1
# Fix intel color corruption. might come with some performance degradation
#export INTEL_DEBUG=norbc
#export mesa_glthread=true
# Set input method modules for Qt/GTK that will show the Steam keyboard
#export QT_IM_MODULE=steam
#export GTK_IM_MODULE=Steam
# Enable volume key management via steam for this session
#export STEAM_ENABLE_VOLUME_HANDLER=1
# Disable automatic audio device switching in steam, handled by wireplumber
export STEAM_DISABLE_AUDIO_DEVICE_SWITCHING=1
# Force Qt applications to run under xwayland
export QT_QPA_PLATFORM=xcb
# Don't wait for buffers to idle on the client side before sending them
export vk_xwayland_wait_ready=false
# Some environment variables by default (taken from Deck session)
export SDL_VIDEO_MINIMIZE_ON_FOCUS_LOSS=0
# nv12 color fix
export GAMESCOPE_NV12_COLORSPACE=k_EStreamColorspace_BT601
# To expose vram info from radv
export WINEDLLOVERRIDES=dxgi=n
# radv stuff? requires more configuration?
#export STEAM_USE_DYNAMIC_VRS=1
# Workaround older versions of vkd3d-proton
#export VKD3D_SWAPCHAIN_LATENCY_FRAMES=3
# Have SteamRT's xdg-open send http:// and https:// URLs to Steam
#export SRT_URLOPEN_PREFER_STEAM=1
# To play nice with the short term callback-based limiter for now
export GAMESCOPE_LIMITER_FILE=$(mktemp /tmp/gamescope-limiter.XXXXXXXX)
# We have the Mesa integration for the fifo-based dynamic fps-limiter
#export STEAM_GAMESCOPE_DYNAMIC_FPSLIMITER=1
# We have NIS support
export STEAM_GAMESCOPE_NIS_SUPPORTED=1
# Let steam know it can unmount drives without superuser privileges
#export STEAM_ALLOW_DRIVE_UNMOUNT=1
### Tearing ###
# Temporary crutch until dummy plane interactions / etc are figured out
#export GAMESCOPE_DISABLE_ASYNC_FLIPS=1
# Support for gamescope tearing with GAMESCOPE_ALLOW_TEARING atom
export GAMESCOPE_ALLOW_TEARING=1
export STEAM_GAMESCOPE_HAS_TEARING_SUPPORT=1
# Enable tearing controls in steam
export STEAM_GAMESCOPE_TEARING_SUPPORTED=1
# nvidia low latency
export __GL_MaxFramesAllowed=1
```
</details>

---
[← Back to top](#top)
### Github Gamescope Discussions
#### Wayland backend: add host-to-client clipboard sync
  - https://github.com/ValveSoftware/gamescope/pull/2062
#### Issues with the WSI layer, hdr, and steam overlay / input failing to hook 
  - https://github.com/ValveSoftware/gamescope/issues/1225
#### Stuttery input, but perfect frametiming
  - https://github.com/ValveSoftware/gamescope/issues/995
#### gamescope having CAP_SYS_NICE break the steam overlay in nested mode
  - https://github.com/ValveSoftware/gamescope/issues/107
#### Huge lag spikes while moving my cursor in Path of Exile 2
  - https://github.com/ValveSoftware/gamescope/issues/2039

---

## Glossary
### Core Terms
- **DRM/KMS** - Direct Rendering Manager / Kernel Mode Setting - Linux subsystem for controlling display hardware
- **WSI** - Window System Integration - Layer that connects Vulkan to display systems (Wayland/X11)
- **Xwayland** - X11 compatibility layer that runs on top of Wayland
- **PTY** - Pseudo-TTY - Virtual terminal used by terminal emulators
- **TTY** - TeleTypewriter - Virtual console you access with Ctrl+Alt+F1-F12

### Gamescope Terms
- **Nested Mode** - Gamescope runs as a window inside your desktop (uses `--backend wayland/sdl`)
- **Embedded Mode** - Gamescope takes direct control of display hardware (uses `-e` or `--backend drm`)
- **Microcompositor** - A lightweight compositor running inside another compositor

### Performance Terms
- **CAP_SYS_NICE** - Linux capability allowing processes to set high scheduling priority
- **VSync** - Vertical Synchronization - Syncs frame delivery to display refresh rate
- **Immediate Flips** - Present frames immediately without waiting for VSync
- **Frame Pacing** - Consistent delivery of frames at regular intervals

---
[← Back to top](#top)
## Resources:
### Official Documentation
- https://deepwiki.com/ValveSoftware/gamescope
### Credits & Acknowledgments
- Examples and explanations developed with assistance from Claude (Anthropic)
- Quite possibly the most complete source of information on gamescope, from render to build system https://deepwiki.com/ValveSoftware/gamescope
- Steam Tinker Launch - Great tool to streamline steam to gamescope integration\
https://github.com/sonic2kk/steamtinkerlaunch/wiki/Steam-Deck
- ScopeBuddy - A manager script to make gamescope easier to use on desktop\
https://github.com/OpenGamingCollective/ScopeBuddy
- GE-Proton - Proton compatibility layer by GloriousEggroll using bleeding-edge Proton\
Experimental Wine with alot of updates\
https://github.com/GloriousEggroll/proton-ge-custom
- Proton-EM is a Development Oriented Compatibility tool for Steam Play based on Wine and additional\
components by Etaash-mathamsetty -- [fsr4 and wayland patches]\
https://github.com/Etaash-mathamsetty/Proton?tab=readme-ov-file#runtime-config-options
- wine-tkg-git - Wine-tkg is a build-system aiming at easier custom wine and proton builds creation\
https://github.com/Frogging-Family/wine-tkg-git
- Goverlay is an easy graphical interface to configure MangoHud, vkBasalt, and OptiScaler\
https://github.com/benjamimgois/goverlay
- lsfg-vk - Lossless Scaling Frame Generation on Linux\
https://github.com/PancakeTAS/lsfg-vk
- Decky LSFG-VK - Decky plugin to streamline installation and usage of lsfg-vk\
https://github.com/xXJSONDeruloXx/decky-lsfg-vk
- GPUVis - GPU Trace Visualizer\
https://github.com/mikesart/gpuvis
- vktracer - Cross-platform cross-vendor Vulkan profiler\
https://www.vktracer.com/
- CryoUtilities - Steam Deck Utility that can tune swap and storage mount by CryoByte33\
https://github.com/CryoByte33/steam-deck-utilities
- Conty - Easy to use unprivileged Linux container packed into a single portable executable\
https://github.com/Kron4ek/Conty?tab=readme-ov-file#conty
- NonSteamLaunchers-On-Steam-Deck - Automatic installation of the most popular launchers\
for your Steam Deck and Steam Machine on SteamOS\
https://github.com/moraroy/NonSteamLaunchers-On-Steam-Deck?tab=readme-ov-file#nonsteamlaunchers-
- CachyOS - Arch-based distribution\
https://github.com/cachyos
- Bazzite is a custom Fedora Atomic image\
https://github.com/ublue-os/bazzite?tab=readme-ov-file#about--features
- FEX: Emulate x86 Programs on ARM64 - A fast usermode x86 and x86-64 emulator for Arm64 Linux\
https://github.com/FEX-Emu/FEX?tab=readme-ov-file#fex-emulate-x86-programs-on-arm64

---
