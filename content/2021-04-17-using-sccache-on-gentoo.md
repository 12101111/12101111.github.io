+++
title = "Using sccache on gentoo"
date = 2021-04-17
[taxonomies]
categories = ["Linux"]
tags = ["Rust","Linux"]
+++

Firest please fellow the page of [Sccache](https://wiki.gentoo.org/wiki/Sccache) on Gentoo wiki.

After the installation and configuration of sccache, you can try to compile some rust packages.
It may work fine for the first one, but later you may find some weird sandbox or permission error like "sccache: error : Failed to create temp dir ..." or "sccache: caused by: Permission denied (os error 13)"

Let's fix this issue.

<!-- more -->

## Reason

You may notice the Note in wiki: "Not like ccache, sccache can only read statistics from running instance. Those interested in statistics must manually spawn sccache server first". sccache is designed to work in a C/S model, the client execute the real compiler and submit the compiled object file and the server storage the object into local disk or cloud storage server.

> sccache works using a client-server model, where the server runs locally on the same machine as the client. The client-server model allows the server to be more efficient by keeping some state in memory. 
> 
> The sccache command will spawn a server process if one is not already running, or you can run sccache --start-server to start the background server process without performing any compilation.
> 
> You can run sccache --stop-server to terminate the server. It will also terminate after (by default) 10 minutes of inactivity.
> 
> Running sccache --show-stats will print a summary of cache statistics.

sccache will create some temp files while running, and the default location of those temp files is depend on the envirentment variable `TMPDIR`( this is the behavior of function `std::env::temp_dir` from rust std library ).

Since portage will run the ebuild in a sandboxed environment, the `TMPDIR` will set to `${T}` ( by default it's `${PORTDIR}/${CATEGORY}/${PN}/temp` ), and sccache server will inherit this envirentment variable. On next portage session, the `TMPDIR` is changed, but the sccache server still use the old directory, and it will throw some error about "Failed to create temp dir".

## Solution

1. install `app-portage/portage-bashrc-mv` from `::mv` overlay.
2. create file `/etc/portage/bashrc.d/90-stop-sccache.sh`:

```bash
#!/bin/bash
Stop_sccache_server() {
    if [[ ! -z $(pidof sccache) ]]; then
        sccache --stop-server >> /dev/null
    fi
}

Stop_sccache_server_preinst() {
    if [[ ! -z $(pidof sccache) ]]; then
        info=$(sccache --stop-server)
        einfo "${info}"
    fi
}

BashrcdPhase prepare Stop_sccache_server
BashrcdPhase preinst Stop_sccache_server_preinst
```

And that's it, the script will automatically stop previous sccache server, and print the statistics of this session.

```log
...
>>> Completed installing gnome-base/librsvg-2.50.4 into /tmp/portage/gnome-base/librsvg-2.50.4/image/

 * Final size of build directory: 977604 KiB (954.6 MiB)
 * Final size of installed tree:   41468 KiB ( 40.4 MiB)

Fixing .la files
   usr/lib/gdk-pixbuf-2.0/2.10.0/loaders/libpixbufloader-svg.la
strip: llvm-strip --strip-unneeded -N __gentoo_check_ldflags__ -R .comment -R .GCC.command.line -R .note.gnu.gold-version
   /usr/bin/rsvg-convert
   /usr/lib/gdk-pixbuf-2.0/2.10.0/loaders/libpixbufloader-svg.so
   /usr/lib/librsvg-2.so.2.47.0

>>> Installing (1 of 1) gnome-base/librsvg-2.50.4::gentoo
 * checking 45 files for package collisions
>>> Merging gnome-base/librsvg-2.50.4 to /
 * removing unneeded *.la files
 * Stopping sccache server...
 * Compile requests                    193
 * Compile requests executed           144
 * Cache hits                          144
 * Cache hits (Rust)                   144
 * Cache misses                          0
 * Cache timeouts                        0
 * Cache read errors                     0
 * Forced recaches                       0
 * Cache write errors                    0
 * Compilation failures                  0
 * Cache errors                          0
 * Non-cacheable compilations            0
 * Non-cacheable calls                  49
 * Non-compilation calls                 0
 * Unsupported compiler calls            0
 * Average cache write               0.000 s
 * Average cache read miss           0.000 s
 * Average cache read hit            0.000 s
 * Failed distributed compilations       0
 * 
 * Non-cacheable reasons:
 * crate-type                           47
 * -                                     2
 * 
 * Cache location                  Local disk: "/var/cache/sccache"
 * Cache size                          261 MiB
 * Max cache size                       10 GiB
--- /usr/
--- /usr/lib/
--- /usr/lib/girepository-1.0/
>>> /usr/lib/girepository-1.0/Rsvg-2.0.typelib
--- /usr/lib/pkgconfig/
>>> /usr/lib/pkgconfig/librsvg-2.0.pc
>>> /usr/lib/librsvg-2.so.2 -> librsvg-2.so.2.47.0
>>> /usr/lib/librsvg-2.so -> librsvg-2.so.2.47.0
--- /usr/lib/gdk-pixbuf-2.0/
--- /usr/lib/gdk-pixbuf-2.0/2.10.0/
--- /usr/lib/gdk-pixbuf-2.0/2.10.0/loaders/
>>> /usr/lib/gdk-pixbuf-2.0/2.10.0/loaders/libpixbufloader-svg.la
>>> /usr/lib/gdk-pixbuf-2.0/2.10.0/loaders/libpixbufloader-svg.so
...
 ```
