---
layout: post
title: "Registering a (Rust) program as a browser on Windows"
date: 2021-03-13 17:43:00 -0800
comments: true
categories: Rust
---

Registering as a potential web browser in modern Windows versions is mostly just about configuring a handful of registry keys, and there's nothing particularly tricky about doing so in Rust. Exactly what you need to configure (and why) can be hard to divine, and so I'll walk through the various registry keys that are expected, why, and show sample Rust code for setting them. In addition, there's a small call to `SHChangeNotify` that's required to refresh the OS' notion of available browsers, and that requires a snippet of unsafe Rust code. 

I figured out all this while writing an app that pretends to be a browser and forwards URLs to the appropriate browser based on pattern matching. I named it [bichrome], and you can find a [full example of URL registration in bichrome's src/windows.rs][bichrome-registration], or keep reading to learn why these parts are needed and how to adapt it to your program!

<!-- more -->

You'll need to pick a couple of arbitrary identifiers. There's a "canonical name" for your browser (for [bichrome] I chose `bichrome.exe`) and a program ID for your URL associations (for [bichrome] I chose `bichromeHTML`), and these will be referred to as `CANONICAL_NAME` and `PROGID` in the samples. You'll also need `DISPLAY_NAME` and `DESCRIPTION`, which are simply user-facing names & descriptions for your application.

```rust
const CANONICAL_NAME: &str = "bichrome.exe";
const PROGID: &str = "bichromeHTML";
const DISPLAY_NAME: &str = "bichrome";
const DESCRIPTION: &str = "Pick the right Chrome profile for each URL";
```

The samples will use the [const_format] crate for its [`concatcp!`][concatcp] macro, the [winreg] crate to write to the registry, and [windows-rs] to call `SHChangeNotify`.

## ProgID registration

Registering the ProgID we've picked lets us use the ProgID in file & URL associations, so that Windows knows what commands to invoke when a particular file or URL gets handled by our ProgID.  The registration happens by configuring `SOFTWARE\CLASSES\<PROGID>` and it's subkeys `DefaultIcon` and `shell\open\command`. The latter is what gets invoked when our ProgID is asked to handle something.

```rust
use std::env::current_exe;

// Find the current executable's name, which we'll use to register
let exe_path = current_exe()?;
let exe_name = exe_path
    .file_name()
    .map(|s| s.to_str())
    .flatten()
    .unwrap_or_default()
    .to_owned();
let exe_path = exe_path.to_str().unwrap_or_default().to_owned();

// We're assuming that the registration will use the icon resource of our EXE
let icon_path = format!("\"{}\",0", exe_path);
// This is the command that will get invoked
let open_command = format!("\"{}\" \"%1\"", exe_path);

// We write to the local user's registry, so that it does not require administrative
// privileges to update.
let hkcu = RegKey::predef(HKEY_CURRENT_USER);

const PROGID_CLASS_PATH: &str = concatcp!(r"SOFTWARE\Classes\", PROGID);
let (progid_class, _) = hkcu.create_subkey(PROGID_CLASS_PATH)?;
progid_class.set_value("", &DISPLAY_NAME)?;

let (progid_class_defaulticon, _) = progid_class.create_subkey("DefaultIcon")?;
progid_class_defaulticon.set_value("", &icon_path)?;

let (progid_class_shell_open_command, _) = progid_class.create_subkey(r"shell\open\command")?;
progid_class_shell_open_command.set_value("", &open_command)?;
```

## Configuring the Default Programs

Default Programs is the user flow that Windows 10 uses to let the user configure defaults for their browser, email client, et cetera. You can read about it in detail [on the MSDN page][defaultprograms].

To become a valid browser, we need to register some information about our application, let the OS know we're a browser-type application, tell it which URLs and file types we're capable of handling, and let it know how to launch, register, and unregister us.

First, we set up the high level information -- display name, our icon, and name & description:
```rust
const DPROG_PATH: &str = concatcp!(r"SOFTWARE\Clients\StartMenuInternet\", CANONICAL_NAME);
let (dprog, _) = hkcu.create_subkey(DPROG_PATH)?;
dprog.set_value("", &DISPLAY_NAME)?;
dprog.set_value("LocalizedString", &DISPLAY_NAME)?;

let (dprog_defaulticon, _) = dprog.create_subkey("DefaultIcon")?;
dprog_defaulticon.set_value("", &icon_path)?;

let (dprog_capabilites, _) = dprog.create_subkey("Capabilities")?;
dprog_capabilites.set_value("ApplicationName", &DISPLAY_NAME)?;
dprog_capabilites.set_value("ApplicationIcon", &icon_path)?;
dprog_capabilites.set_value("ApplicationDescription", &DESCRIPTION)?;
```

Then we tell the OS we have a [StartMenuInternet capability][startmenu-capability], and then link this browser's URL & file associations to the program ID we chose (and registered above).

```rust
let (dprog_capabilities_startmenu, _) = dprog_capabilites.create_subkey("Startmenu")?;
dprog_capabilities_startmenu.set_value("StartMenuInternet", &CANONICAL_NAME)?;

// Register for various URL protocols that our target browsers might support.
let (dprog_capabilities_urlassociations, _) =
    dprog_capabilites.create_subkey("URLAssociations")?;
for protocol in &["ftp", "http", "https", "webcal"] {
    dprog_capabilities_urlassociations.set_value(protocol, &PROGID)?;
}

// Register for various file types, so that we'll be invoked for file:// URLs for these types (e.g.
// by `cargo doc --open`.)
let (dprog_capabilities_fileassociations, _) =
    dprog_capabilites.create_subkey("FileAssociations")?;
for filetype in &[
    ".htm", ".html", ".pdf", ".shtml", ".svg", ".webp", ".xht", ".xhtml",
] {
    dprog_capabilities_fileassociations.set_value(filetype, &PROGID)?;
}
```

To finalize the Default Programs configuration, we set up the [installation information][installation-information] which allows the OS to register & unregister us. You can read about the expectations for e.g. the reinstall command [on this MSDN page][reinstall-expectations].

```rust
// Set up reinstallation and show/hide icon commands (https://docs.microsoft.com/en-us/windows/win32/shell/reg-middleware-apps#registering-installation-information)
let (dprog_installinfo, _) = dprog.create_subkey("InstallInfo")?;
dprog_installinfo.set_value("ReinstallCommand", &format!("\"{}\" register", exe_path))?;
dprog_installinfo.set_value("HideIconsCommand", &format!("\"{}\" hide-icons", exe_path))?;
dprog_installinfo.set_value("ShowIconsCommand", &format!("\"{}\" show-icons", exe_path))?;

// Only update IconsVisible if it hasn't been set already
if let Err(_) = dprog_installinfo.get_value::<u32, _>("IconsVisible") {
    dprog_installinfo.set_value("IconsVisible", &1u32)?;
}

let (dprog_shell_open_command, _) = dprog.create_subkey(r"shell\open\command")?;
dprog_shell_open_command.set_value("", &open_command)?;
```

## Tying it all together

Finally, we just need to create a [Registered Application][registered-application] that maps our application name to the Default Programs capabilities we listed, and set up an app path for convenience sake -- so that `myprogram.exe` can resolve to our program without needing to modify the `PATH`.

    
```rust
// Set up a registered application for our Default Programs capabilities (https://docs.microsoft.com/en-us/windows/win32/shell/default-programs#registeredapplications)
let (registered_applications, _) =
    hkcu.create_subkey(r"SOFTWARE\RegisteredApplications")?;
let dprog_capabilities_path = format!(r"{}\Capabilities", DPROG_PATH);
registered_applications.set_value(DISPLAY_NAME, &dprog_capabilities_path)?;

// Application Registration (https://docs.microsoft.com/en-us/windows/win32/shell/app-registration)
let appreg_path = format!(r"SOFTWARE\Microsoft\Windows\CurrentVersion\App Paths\{}", exe_name);
let (appreg, _) = hkcu.create_subkey(APPREG_PATH)?;
// This is used to resolve "myprogram.exe" -> full path, if needed.
appreg.set_value("", &exe_path)?;
// UseUrl indicates that we don't need the shell to download a file for us -- we can handle direct
// HTTP URLs.
appreg.set_value("UseUrl", &1u32)?;
```

## Letting the OS know we're good to go

As one last step, we [notify the OS that the available URL handlers have changed][notify-assoc], so that it will refresh the list of browsers, and if this is our first registration, automatically prompt the user if they'd like to change their default.

```rust
mod windows_bindings {
    ::windows::include_bindings!();
}

use windows_bindings::windows::win32::shell::{SHChangeNotify, SHCNE_ID, SHCNF_FLAGS};

// Notify the shell about the updated URL associations. (https://docs.microsoft.com/en-us/windows/win32/shell/default-programs#becoming-the-default-browser)
unsafe {
    SHChangeNotify(
        SHCNE_ID::SHCNE_ASSOCCHANGED,
        SHCNF_FLAGS::SHCNF_DWORD | SHCNF_FLAGS::SHCNF_FLUSH,
        std::ptr::null_mut(),
        std::ptr::null_mut(),
    );
}
```

Setting up windows-rs is a little bit heavier weight than most crates -- you can find documentation on [the crates.io page for the windows crate][windows-rs], but a minimal example is shown below. First, add the following to your `Cargo.toml`:
```toml
[dependencies]
windows = "0.4.0"

[build-dependencies]
windows = "0.4.0"
```

Then you'll also need to specify a `build.rs` script that generates the appropriate bindings for you:
```rust
fn main() {
  windows::build!(windows::win32::shell::*);
}
```

## Being suave about it

By default Rust applications use the "console" subsystem on Windows -- this means you can more or less always write to the console and have it show up somewhere. The downside of this is that if the application is run outside of a console window, e.g. by double-clicking the EXE or when invoked as a browser by the OS, you'll see a console window pop up. Luckily, since Rust 1.18, it's quite trivial to switch to the Windows subsystem. Just add the following to the top of your `main.rs`:

```rust
#![windows_subsystem = "windows"]
```

## In closing

If you're looking to just drop this in to your own project, I would suggest you look at [the full example in bichrome's src/windows.rs][bichrome-registration], which has all the various parts in two methods (`register_urlhandler` and `refresh_shell`). 

Please let me know if you have any questions or just to say that this was helpful to you! You can leave a comment, hit me up on [twitter (@jorgenpt)][twitter], or send me [a quick email][email].

[bichrome-registration]: https://github.com/jorgenpt/bichrome/blob/04e8a4476105501032121c05f487f592c6ca68ce/src/windows.rs#L53
[bichrome]: https://github.com/jorgenpt/bichrome
[concatcp]: https://docs.rs/const_format/0.2.13/const_format/macro.concatcp.html
[const_format]: https://crates.io/crates/const_format
[defaultprograms]: https://docs.microsoft.com/en-us/windows/win32/shell/default-programs
[email]: mailto:jorgenpt@gmail.com
[installation-information]: https://docs.microsoft.com/en-us/windows/win32/shell/reg-middleware-apps#registering-installation-information
[notify-assoc]: https://docs.microsoft.com/en-us/windows/win32/shell/default-programs#becoming-the-default-browser
[registered-application]: https://docs.microsoft.com/en-us/windows/win32/shell/default-programs#registeredapplications
[reinstall-expectations]: https://docs.microsoft.com/en-us/windows/win32/shell/reg-middleware-apps#the-reinstall-command
[startmenu-capability]: https://docs.microsoft.com/en-us/windows/win32/shell/default-programs#startmenu
[twitter]: https://twitter.com/jorgenpt
[windows-rs]: https://crates.io/crates/windows
[winreg]: https://crates.io/crates/winreg