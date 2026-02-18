ok got it built and installed. Maybe I could repackage that pre-built app so I wouldn't have to build anything next time. If I'm not mistaken the build-process allows for install on various platforms, but I'll always be using linux.. ?

Yeah — and you’re thinking about it the right way.

Right now you’ve got a locally-built binary installed into /usr, and next time you reinstall Mint (or move machines) you’d have to:

reinstall Nim

rebuild

reinstall

If you only care about Linux, you can absolutely “package the result” so future installs are basically one command.

The two sane options
Option 1: Make a .deb (best for your use case)

Since you’re on Mint/Ubuntu, making a .deb is the most natural:

you install it once with sudo dpkg -i taskopen_*.deb

you can uninstall cleanly with apt remove taskopen

you can keep it in your own little repo folder or just stash the .deb

This is what I’d recommend.

Option 2: Build a static-ish binary and stash it somewhere

This is the “just copy the binary” approach:

keep taskopen in ~/bin/ or /usr/local/bin/

copy it between machines

This works, but you lose clean package management and you still have to install scripts/manpages/etc manually if taskopen uses them.

A small reality check

You can’t truly make a “universal Linux binary” that works everywhere forever. Linux distros differ in:

glibc versions

libssl versions

paths, desktop opener commands, etc

But you can make a .deb that works great on Mint/Ubuntu (and likely Debian).

If you want: I can help you create a proper .deb

If you paste the output of:

make -n PREFIX=/usr install


(or show me the Makefile install section)

…I’ll tell you exactly what files it installs and I’ll give you a minimal debian/ packaging skeleton so you can build a .deb with dpkg-buildpackage.

That’ll get you a clean, reusable installer for your fork.

 djp  yoga  ~/dev/taskopen  master ◔  ls -al
total 564
drwxrwxr-x  7 djp djp   4096 Feb 17 20:26 .
drwxrwxr-x 42 djp djp   4096 Feb 17 20:04 ..
-rw-rw-r--  1 djp djp    326 Feb 17 20:04 AUTHORS
drwxrwxr-x  3 djp djp   4096 Feb 17 20:04 doc
drwxrwxr-x  3 djp djp   4096 Feb 17 20:04 examples
drwxrwxr-x  8 djp djp   4096 Feb 17 23:10 .git
-rw-rw-r--  1 djp djp  18092 Feb 17 20:04 LICENSE
-rw-rw-r--  1 djp djp   1946 Feb 17 20:04 Makefile
-rw-rw-r--  1 djp djp   2988 Feb 17 20:04 NEWS
-rw-rw-r--  1 djp djp   9585 Feb 17 20:04 README.md
drwxrwxr-x  4 djp djp   4096 Feb 17 20:04 scripts
drwxrwxr-x  2 djp djp   4096 Feb 17 20:04 src
-rwxrwxr-x  1 djp djp 496152 Feb 17 20:26 taskopen
-rw-rw-r--  1 djp djp    362 Feb 17 20:04 taskopen.nimble

 djp  yoga  ~/dev/taskopen  master ◔  make -n PREFIX=/usr install
gzip -c doc/man/taskopen.1 > doc/man/taskopen.1.gz
gzip -c doc/man/taskopenrc.5 > doc/man/taskopenrc.5.gz
mkdir -p /usr/bin
mkdir -p /usr/share/man/man1
mkdir -p /usr/share/man/man5
install -m 0755 taskopen /usr/bin/taskopen
install -m 0644 doc/man/taskopen.1.gz /usr/share/man/man1/taskopen.1.gz
install -m 0644 doc/man/taskopenrc.5.gz /usr/share/man/man5/taskopenrc.5.gz
mkdir -p /usr/share/taskopen/scripts/
cp -r scripts/* /usr/share/taskopen/scripts/
chmod -R 755 /usr/share/taskopen/scripts
mkdir -p /usr/share/taskopen/examples
cp -r examples/* /usr/share/taskopen/examples
chmod -R 755 /usr/share/taskopen/examples


Nice — this is very package-friendly. It installs exactly the kind of stuff a .deb wants:

/usr/bin/taskopen

manpages

/usr/share/taskopen/scripts/*

/usr/share/taskopen/examples/*

So yes: you can repackage this cleanly and never compile again.

Easiest path on Mint/Ubuntu: build a .deb using fpm

This avoids writing a full Debian packaging directory.

1) Install fpm
sudo apt update
sudo apt install ruby ruby-dev build-essential
sudo gem install --no-document fpm

2) Stage the install into a fake root

From your repo:

rm -rf pkgroot
mkdir -p pkgroot
make PREFIX=/usr DESTDIR="$PWD/pkgroot" install


Now your files will be in:

pkgroot/usr/bin/taskopen
pkgroot/usr/share/...

3) Build the .deb
fpm -s dir -t deb \
  -n taskopen \
  -v 1.0.2 \
  --iteration 1 \
  --license MIT \
  --maintainer "djp" \
  --description "taskwarrior task opener helper (fork of taskopen)" \
  -C pkgroot \
  .


That should produce something like:

taskopen_1.0.2-1_amd64.deb

Install it with:

sudo dpkg -i taskopen_*.deb


Uninstall cleanly:

sudo apt remove taskopen

One important tweak (so this works)

Your Makefile currently gzips manpages in-place during install:

gzip -c doc/man/taskopen.1 > doc/man/taskopen.1.gz


That’s fine, but when packaging you’ll want gzip available (it is on basically every system).

If you want the “proper Debian way”

You can also create a debian/ directory and use dpkg-buildpackage, but for a personal fork that you just want to reinstall easily, fpm is the sweet spot.

If you want, I can give you a tiny Makefile target like make deb that does the staging + fpm packaging in one command.

sure, I got 3msgs left, let's do it!

Perfect — let’s make this a one-command make deb situation.

Add this block to the bottom of your Makefile (or anywhere, but bottom is fine):

# ---------------------------
# Packaging (local .deb build)
# ---------------------------

PKGNAME   := taskopen
VERSION   := 1.0.2
ITERATION := 1
ARCH      := amd64
PREFIX    ?= /usr
DESTDIR   ?=

.PHONY: deb clean-deb

deb: clean taskopen
	@echo "==> staging install into ./pkgroot"
	rm -rf pkgroot
	mkdir -p pkgroot
	$(MAKE) PREFIX=$(PREFIX) DESTDIR=$$(pwd)/pkgroot install

	@echo "==> building .deb with fpm"
	fpm -s dir -t deb \
		-n $(PKGNAME) \
		-v $(VERSION) \
		--iteration $(ITERATION) \
		-a $(ARCH) \
		--license MIT \
		--maintainer "djp" \
		--description "taskwarrior task opener helper (fork of taskopen)" \
		-C pkgroot \
		.

	@echo "==> done"
	@ls -al *.deb

clean-deb:
	rm -rf pkgroot *.deb

Then install the packaging deps once
sudo apt update
sudo apt install ruby ruby-dev build-essential
sudo gem install --no-document fpm

Build your .deb

From the repo dir:

make deb

Install it
sudo dpkg -i taskopen_*.deb

2 small notes (so you don’t get bitten)

I hardcoded VERSION := 1.0.2 — bump that whenever you tag.

ARCH := amd64 is correct for your machine. If you ever build on ARM, change it.
