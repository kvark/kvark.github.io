---
layout: post
title:  "NixOS on Framework laptop"
date:   2021-10-17 11:11:11 -0500
categories: linux framework
---

The time has come to replace the work laptop (thanks, Mozilla!). Here is an adventure of setting up NixOS on a Framework laptop. I'm writing this on it right now :)

## Base Point

![previous laptops](/resource/previous-laptops.jpg)

### Main - X1

The main one I had for 3 years was **Thinkpad X1 Extreme** (1st generation). It's a heavy and relatively big laptop with Intel H-series CPU and NVidia 1050 GPU. The reasoning was that it's got to be mostly stationary, and having a bigger screen, as well as more horsepower, would help to work more efficiently. Gecko compile times are generally long. I couldn't be more wrong...

Most of my work wasn't compiling Gecko. Besides, I can always switch to issue triage when I had to wait 30-60 minutes for it to build, so compile times wasn't a real issue. In exchange I lost an ability to move it around, I got subpar battery life, and the usage experience was poor due to constant fan noise and heat coming from it. These properties were important pre-covid, when I commuted on a train, and in-covid, when I had to move around the house.

On the software side, I had Solus Budgie Linux. It's a very nice distribution and desktop environment! However, Mozilla requires LUKS full-disk encryption, and having it setup conflicted with NVidia's proprietary driver. Basically, if I auto-updated the system, it wouldn't boot. I had to tweak the EFI configuration by hands a bit. Maybe I was doing it wrong, but it wasn't difficult, just slightly annoying.

Having this driver caused issues in development as well. The plan was to be able to test software on both Intel and NV GPUs. However, the display is connected to NVidia, so getting Intel GPU to rendering on screen failed - [mesa#4688](https://gitlab.freedesktop.org/mesa/mesa/-/issues/4688) - on both Vulkan and GLES (under Linux).

Finally, on Windows, Lenovo Vantage is horrifying. I didn't ask for it, and I never want to see it again. Same goes to Windows intrusive advertizing and manipulation practices.

### Secondary - MBP and T495s

As a result, I mostly used the secondary work laptop - **MacBook Pro 2016**. It's a wonderful machine! It's slick, silent, very portable. Got me really appreciate Apple's tech and the vertically integrated stack. It sleeps and wakes up quickly without issues.

The only issue with it is keyboard, which has known issues on that generation of Mac books. Oh, and touch bar sometimes bugs out, which means you can't Escape!. Power-wise, it has a dual-core Intel CPU with Iris 550 GPU. It's sufficient for general use, even for driving an external monitor, but it doesn't excel at building software.

In addition, I have a personal **Thinkpad T495s**. It's a decent portable AMD 3500U machine, useful for GPU testing and development on Linux when I don't want to open the *X1*. 4 CPU cores are better than *MBP* at compiling, but single-core performance is noticeably worse, so linking performance suffered. It still has a driver issue with waking up, where sometimes the screen wouldn't turn on (related to AMD drivers, possibly), and I have to reboot it. So it's hard to consider it for any serious work.

## Framework laptop

![framework package](/resource/framework-package.jpg) ![framework inside](/resource/framework-inside.jpg)

Here comes the Framework DYI edition, delivered as a bag of parts that you need to assemble before using. Packaging is great, and there is a neat screwdriver to open all the locks. They took extra care about the process - it's impossible to lose any bolts, which happened to me with other laptops.

The *body* seems solid and well thought through, even if not as sturdy as *MBP*'s aluminum. It's also a bit thicker, but slightly lighter. It stands on pads, and fan exhaust goes down. The fan itself can spin up when doing anything, but overall it's silent (although, not as good as *MBP*).

I appreciate the Core i7-1185G7 CPU, which is 4-core U-series but can boost the clocks temporarily. I did a bit of compile speed comparison by building [wgpu](https://github.com/gfx-rs/wgpu) (using roughly the same Rust, around v1.55):
  - *Framework* took 27s (!)
  - *T495s* took 57s, so 2x slower
  - *MBP* took 75s, so 2.5x slower, but building slightly less code (only Metal backend)
  - *X1* took 120s, so 4.5x slower... totally unexpected

*Keyboard* feels as a mix between Thinkpads and MacBooks. I dare to say the mix combines the best parts of those. I only wish the up/down arrow buttons were normal size, like on Thinkpads. Touchpad is fairly large and precise, but not at the level of *MBP*. Hardware switches for camera and microphone are sweet.

Continuing comparison with *MBP*, the bezels are smaller, and the footprint of the laptop seems smaller, making it more portable. That's also because of *3:2 screen* ratio, which I find very comfortable for code editing and browsing.

In terms of *power consumption*, I think there is a lot to gain here. Even with "deep" sleep mode, I see that the battery discharges a bit after night (which isn't the case for *MBP*). I expect it to improve with BIOS and Linux kernel updates. In particular, it needs to cap the charging levels, like Apple hardware does since forever. On the positive side, it's stable: I can put it to sleep and wake up without issues. The wake-up is long though, 10-15 seconds, which I hope to be able to fix some day.

## NixOS

![nixos logo](/resource/207px-Home-nixos-logo.png) ![framework logo](/resource/framework-logo.jpg)

### Promise

As an OS-enthusiast, I tried working under many different operating systems. Was a big fan of BeOS and Haiku, poked into BSDs and QNX, started with DOS, and of course went through all sorts of Windows and Linux. In Unix-based systems, there was a common problem of updating: some config files got changed, links get broken, removing software is never without a trace (same for Windows, but not necessarily macOS). So after prolonged use, system breaks and needs a refresh.

This is the problem NixOS solves. There is no "install a package" command. Everything is immutable, and you just ask for things you need, and when you need them. So there is still a possibility of broken dependencies, but at any point you can just use a previous configuration without penalty or risk. All the system configuration is declared in one place. You can find mine [on github](https://github.com/kvark/dotfiles/tree/d2df6365b33c08d92801f8eb2b60dd518069bb54/nix).

### Setup

I started with [NixOS on the Framework](https://grahamc.com/blog/nixos-on-framework) post, and it still largely applies, was just incomplete for me. So the first step is disabling Secure Boot in BIOS.

Following the official installation guide got me into trouble because of [nixpkgs#141557](https://github.com/NixOS/nixpkgs/pull/141557). It assumes Root but doesn't explain how to get there. As a result, after first installation half of the folders where owned by my user, and another half by `root`, which was totally unexpected by things, and nothing worked (e.g. unable to get into graphical login). So eventually I started over, armed with `sudo -i`, and life was good.

#### WiFi

Just downloading NixOS on a USB drive and trying to install on Framework is a dead end. First problem is Intel Wi-Fi 6E AX210, which isn't supported by the current stable kernel. The blog post advises to set `boot.kernelPackages = pkgs.linuxPackages_latest;` to solve this, but rebuilding the system (with `nixos-rebuild`) requires Internet access, which needs WiFi. Hooking up a USB-Ethernet dongle also failed, because it turned out to be Realtek for me, and Linux didn't like it...

The solution I found was booting from the NixOS latest (unstable) image, which only comes with "minimal" (no GUI). It's annoying to use the console at this stage, but it's not yet a solution. Kernel recognizes WiFi module, but `wpa_supplicant`
 fails to connect, see [a similar thread](https://forum.manjaro.org/t/i-dont-want-my-wifi-to-stop-working/34818) in Manjaro forums. To workaround, I have to reboot, hit "e" in grub, and add `net.ifnames=0` to the kernel parameters, then Ctrl+X. Now, finally, I'm able to enable WiFi with this elaborate command:
 ```bash
 wpa_supplicant -B -i wlan0 -c <(wpa_passphrase 'SSID' 'password')
 
 ```

#### Sleep

Power consumption on Linux is always an adventure. The first step was enabling "deep" sleep using this configuration line:
```nix
boot.kernelParams = [ "mem_sleep_default=deep" ];

```
The [next BIOS update](https://knowledgebase.frame.work/en_us/framework-laptop-bios-releases-S1dMQt6F) promises improved usage for the sound card. There is also quite a few entries on Framework forums about user configurations, which I haven't gotten to.

#### Desktop Environment

The screen has 2256x1504 resolution, which looks rather poor on Gnome's default 2x scaling. Reading [about it more](https://wiki.archlinux.org/title/HiDPI) leads to a horrifying realization that the only way to get fractional scaling on Gnome is via super-sampling. That's definitely undesirable, so I started looking elsewhere.

Apparently, KDE Plasma5 does the scaling right. Setting it to 150% gives pleasuring look. Overall, I find KDE's approach with a taskbar much more reasonable than what Gnome forces on me. It's the paradigm shift from "tell me what you need" (modern Gnome, also Google) to "show me what you have" (KDE and others application menu, also your file manager).

#### Input

There is an issue with touchpad on KDE: setting two-finger tap as right click (like in macOS) is currently malfunctioning, see (kde#408116)[https://bugs.kde.org/show_bug.cgi?id=408116]. It's quite annoying, but not NixOS fault.

## Conclusion

![final photo](/resource/new-laptop.jpg)

Overall, I find the laptop sufficiently usable and portable, and quite fast.
It's definitely better than either of *X1* or *T495s*, and mostly better than *MBP*.

Setting up NixOS on it is a bit rough, but once you get there, the system feels rock stable.
I'm starting to feel productive now, and excited to fully migrate to it for development!

Clearly, there are still issues, and the story is to be continued. Bluetooth doesn't work - see [kernel#213829](https://bugzilla.kernel.org/show_bug.cgi?id=213829). Power consumption can be better. I'm hoping this post will help others to get up to speed. Please feel free to use the parts of [my config](https://github.com/kvark/dotfiles).
