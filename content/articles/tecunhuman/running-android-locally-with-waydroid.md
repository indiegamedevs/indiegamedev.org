---
title: "Running Android Locally with Waydroid"
date: 2026-10-05
articles_tags: ["android", "godot", "linux", "testing", "waydroid"]
image: "/images/articles/tecunhuman/waydroid.png"
image_alt: "Waydroid logo"
image_blur: false
summary: "How TecunHuman uses Waydroid and a nested Weston session to run and test Godot Android builds locally on Linux."
---

I recently needed a convenient way to test an Android build of one of my Godot projects without continually copying the APK to a phone.

For quick development builds, I wanted something that could run locally on my Linux desktop and still behave like Android. I ended up using [Waydroid](https://waydro.id/), which runs a full Android system in a Linux container.

My desktop setup needed one extra piece: Waydroid expects a Wayland session, so I used [Weston](https://wayland.pages.freedesktop.org/weston/), a small Wayland compositor, in a window. That gave Waydroid the Wayland display it needed without requiring me to change my normal desktop session.

Here is the complete workflow I used to install Waydroid, start Android inside Weston, install a Godot APK, and remove an older build when necessary.

## Installing Waydroid and Weston

These commands are for Ubuntu, Debian, and related distributions. The [official Waydroid installation guide](https://docs.waydro.id/usage/install-on-desktops) includes the current repository setup instructions and directions for other Linux distributions.

After adding the Waydroid repository, I installed Waydroid and Weston:

```bash
sudo apt install waydroid weston -y
```

Waydroid downloads and configures its Android image during initialization. I chose the GAPPS image because I wanted Google apps and services available:

```bash
sudo waydroid init -s GAPPS
```

GAPPS is optional. If a project does not depend on Google Play services, the default vanilla image may be enough:

```bash
sudo waydroid init
```

Choose the image you want the first time you initialize Waydroid. Replacing an existing image is a reset operation and may remove the Android data already stored in the container.

I then enabled and started the container service:

```bash
sudo systemctl enable --now waydroid-container
```

## Giving Waydroid a Wayland Display

Because I was not already running the desktop in a Wayland session, I launched Weston as a nested compositor:

```bash
weston --xwayland
```

This opens a Weston desktop in its own window. Keep that process running while using Waydroid.

Weston prints the name of the Wayland display it creates. On my machine it was `wayland-1`, so I launched the full Android interface with:

```bash
WAYLAND_DISPLAY=wayland-1 waydroid show-full-ui
```

The socket name is not guaranteed to be `wayland-1`. If that command cannot connect to the display, check the Weston terminal output or list the available sockets:

```bash
ls "$XDG_RUNTIME_DIR"/wayland-*
```

Then use the appropriate name as `WAYLAND_DISPLAY`.

Waydroid can also be started in two explicit steps. This is useful when diagnosing whether a problem is in the Android session or only in the UI:

```bash
waydroid session start
WAYLAND_DISPLAY=wayland-1 waydroid show-full-ui
```

The session command should be run as the normal desktop user, not with `sudo`.

## Choosing a Useful Window Size

For testing a phone layout, I can give the nested Weston window a portrait-oriented size:

```bash
weston --xwayland --width=720 --height=1280
```

I also experimented with Waydroid's Android display properties:

```bash
waydroid prop set persist.waydroid.width 720
```

Waydroid properties generally take effect after restarting the session:

```bash
waydroid session stop
waydroid session start
```

Setting the Weston window size was the simpler option for my workflow because it made the entire nested desktop resemble a phone display.

## Installing a Godot APK

Once Android was running, I installed my exported Godot project directly from the terminal:

```bash
waydroid app install ./quetzalcoatl.apk
```

For more information when an installation failed, I enabled verbose output and sent the details to the terminal:

```bash
waydroid --details-to-stdout -v app install ./quetzalcoatl.apk
```

That version is especially useful during development because a normal install command may not show enough information to explain the problem.

To confirm that the application was installed, I listed the Waydroid applications and filtered the result:

```bash
waydroid app list | grep -i quetzalcoatl
```

If I know the Android package name, I can launch it directly:

```bash
waydroid app launch com.example.quetzalcoatl
```

The application also appears in Waydroid's Android launcher.

One important limitation is that the APK must support the architecture used by the Waydroid image. On a typical x86_64 Linux computer, an APK containing only ARM native libraries will not run without an additional translation layer. For Godot projects, I make sure the Android export includes an architecture compatible with the environment I am testing.

## Removing an Older Build

During development, Android may refuse an update if the new APK was signed with a different key, uses an incompatible version, or otherwise conflicts with the installed package.

Waydroid provides an app removal command when I know the package name:

```bash
waydroid app remove com.example.quetzalcoatl
```

When I needed to work directly through Android's package manager, I used:

```bash
sudo waydroid shell pm uninstall com.example.quetzalcoatl
```

After uninstalling the old package, I could install the new APK again:

```bash
waydroid --details-to-stdout -v app install ./quetzalcoatl.apk
```

The value passed to `remove`, `launch`, or `pm uninstall` is the Android package ID, not the APK filename or the human-readable application name. In a Godot project, this corresponds to the package identifier configured in the Android export preset.

## A Few Useful Diagnostic Commands

These were the commands I found most useful when something was not working:

```bash
sudo systemctl status waydroid-container
waydroid app list
waydroid log
sudo waydroid shell
```

The first checks the Linux container service. `waydroid app list` confirms what Android thinks is installed, `waydroid log` shows Waydroid's log output, and the shell provides direct access to Android commands such as `pm`.

If an application checks specifically for a Wi-Fi connection, Waydroid can also make selected packages see its network connection as Wi-Fi. To apply that behavior to every package:

```bash
waydroid prop set persist.waydroid.fake_wifi '*'
```

That is not needed for most applications, so I would only enable it when a particular app requires it.

## My Everyday Workflow

After the initial installation, my normal testing loop is short:

1. Start Weston with `weston --xwayland`.
2. Open Android with `WAYLAND_DISPLAY=wayland-1 waydroid show-full-ui`.
3. Export the Android APK from Godot.
4. Install it with `waydroid --details-to-stdout -v app install ./quetzalcoatl.apk`.
5. Launch the app and test it.
6. Remove the installed package if Android will not accept the next development build.

Waydroid does not replace testing on real Android hardware. Physical devices still matter for touch input, sensors, performance, graphics-driver differences, screen cutouts, and device-specific behavior.

What it does provide is a very fast local feedback loop. For testing whether a build installs, starts, connects to services, and handles an Android-sized display, I can stay on the development machine and repeat the process as often as necessary.

That has made Android testing feel much closer to running an ordinary desktop build, while still exercising the actual Android export of the project.

[Originally published on Patreon](https://www.patreon.com/tecunhuman/posts/running-android-171525467)
