## The Plan 10 source packages collection

This repository contains the Plan 10 source packages collection to build binary packages for the Plan 10 Linux distribution.

The included `ppk-src` script will fetch and compile the sources, and install its
files into a `fake destdir` to generate ppk binary packages that can be installed
or queried through the `ppk install(1)` and `ppk search(1)` utilities, respectively.

### Requirements

- GNU bash
- ppk >= 0.55
- curl(1) - required by `ppk-src update-check`
- flock(1) - util-linux
- install(1) - coreutils
- other common POSIX utilities included by default in almost all UNIX systems.

`ppk-src` requires a utility to chroot and bind mount existing directories
into a `masterdir` that is used as its main `chroot` directory. `ppk-src` supports
multiple utilities to accomplish this task:

 - `bwrap` - bubblewrap, see https://github.com/projectatomic/bubblewrap.
 - `ethereal` - only useful for one-shot containers, i.e docker (used with travis).
 - `ppk-uunshare(1)` - ppk utility that uses `user_namespaces(7)` (part of ppk, default).
 - `ppk-uchroot(1)` - ppk utility that uses `namespaces` and must be `setgid` (part of ppk).
 - `proot(1)` - utility that implements chroot/bind mounts in user space, see https://proot-me.github.io/.

> NOTE: you don't need to be `root` to use `ppk-src`, use your preferred chroot style as explained below.

#### ppk-uunshare(1)

This utility requires these Linux kernel options:

- CONFIG\_NAMESPACES
- CONFIG\_IPC\_NS
- CONFIG\_UTS\_NS
- CONFIG\_USER\_NS

This is the default method, and if your system does not support any of the required kernel
options it will fail with `EINVAL (Invalid argument)`.

#### ppk-uchroot(1)

This utility requires these Linux kernel options:

- CONFIG\_NAMESPACES
- CONFIG\_IPC\_NS
- CONFIG\_PID\_NS
- CONFIG\_UTS\_NS

Your user must be added to a special group to be able to use `ppk-uchroot(1)` and the
executable must be `setgid`:

    # chown root:<group> ppk-uchroot
    # chmod 4750 ppk-uchroot
    # usermod -a -G <group> <user>

> NOTE: by default in void you shouldn't do this manually, your user must be a member of
the `ppkbuilder` group.

To enable it:

    $ cd plan10-pkg
    $ echo PPK_CHROOT_CMD=uchroot >> etc/conf

If for some reason it's erroring out as `ERROR clone (Operation not permitted)`, check that
your user is a member of the required `group` and that `ppk-uchroot(1)` utility has the
proper permissions and owner/group as explained above.

#### proot(1)

The `proot(1)` utility implements chroot and bind mounts support completely in user space,
and can be used if your Linux kernel does not have support for namespaces. See https://proot-me.github.io/.
for more information.

To enable it:

    $ cd plan10-pkg
    $ echo PPK_CHROOT_CMD=proot >> etc/conf

### Quick setup in Plan 10

Clone the `plan10-pkg` git repository, install the bootstrap packages:

```
$ git clone git://github.com/plantenos/plan10-pkg.git
$ cd plan10-pkg
$ ./ppk-src binary-bootstrap
```

Type:

     $ ./ppk-src -h

to see all available targets/options and start building any available package
in the `srcpkgs` directory.

### Install the bootstrap packages

The `bootstrap` packages are a set of packages required to build any available source package in a container. There are two methods to install the `bootstrap`:

 - `bootstrap`: all bootstrap packages will be built from scratch; additional utilities are required in the
host system to allow building the `base-chroot` package: binutils, gcc, perl, texinfo, etc.

 - `binary-bootstrap`: the bootstrap binary packages are downloaded via ppk repositories.

If you don't want to waste your time building everything from scratch probably it's better to use `binary-bootstrap`.

### Configuration

The `etc/defaults.conf` file contains the possible settings that can be overridden
through the `etc/conf` configuration file for the `ppk-src` utility; if that file
does not exist, will try to read configuration settings from `~/.ppk-src.conf`.

If you want to customize default `CFLAGS`, `CXXFLAGS` and `LDFLAGS`, don't override
those defined in `etc/defaults.conf`, set them on `etc/conf` instead i.e:

    $ echo 'PPK_CFLAGS="your flags here"' >> etc/conf
    $ echo 'PPK_LDFLAGS="your flags here"' >> etc/conf

Native and cross compiler/linker flags are set per architecture in `common/build-profiles`
and `common/cross-profiles` respectively. Ideally those settings are good enough by default,
and there's no need to set your own unless you know what you are doing.

#### Virtual packages

The `etc/defaults.virtual` file contains the default replacements for virtual packages,
used as dependencies in the source packages tree.

If you want to customize those replacements, copy `etc/defaults.virtual` to `etc/virtual`
and edit it accordingly to your needs.

### Directory hierarchy

The following directory hierarchy is used with a default configuration file:

         /plan10-pkg
            |- common
            |- etc
            |- srcpkgs
            |  |- ppk
            |     |- template
            |
            |- hostdir
            |  |- binpkgs ...
            |  |- ccache ...
            |  |- distcc-<arch> ...
            |  |- repocache ...
            |  |- sources ...
            |
            |- masterdir
            |  |- builddir -> ...
            |  |- destdir -> ...
            |  |- host -> bind mounted from <hostdir>
            |  |- plan10-pkg -> bind mounted from <plan10-pkg>


The description of these directories is as follows:

 - `masterdir`: master directory to be used as rootfs to build/install packages.
 - `builddir`: to unpack package source tarballs and where packages are built.
 - `destdir`: to install packages, aka **fake destdir**.
 - `hostdir/ccache`: to store ccache data if the `PPK_CCACHE` option is enabled.
 - `hostdir/distcc-<arch>`: to store distcc data if the `PPK_DISTCC` option is enabled.
 - `hostdir/repocache`: to store binary packages from remote repositories.
 - `hostdir/sources`: to store package sources.
 - `hostdir/binpkgs`: local repository to store generated binary packages.

### Building packages

The simplest form of building package is accomplished by running the `pkg` target in `ppk-src`:

```
$ cd plan10-pkg
$ ./ppk-src pkg <pkgname>
```

When the package and its required dependencies are built, the binary packages will be created
and registered in the default local repository at `hostdir/binpkgs`; the path to this local repository can be added to 
any ppk configuration file (see ppk.d(5)) or by explicitly appending them via cmdline, i.e:

    $ ppk install --repository=hostdir/binpkgs ...
    $ ppk query --repository=hostdir/binpkgs ...

By default **ppk-src** will try to resolve package dependencies in this order:

 - If a dependency exists in the local repository, use it (`hostdir/binpkgs`).
 - If a dependency exists in a remote repository, use it.
 - If a dependency exists in a source package, use it.

It is possible to avoid using remote repositories completely by using the `-N` flag.

> The default local repository may contain multiple *sub-repositories*: `debug`, `multilib`, etc.

### Package build options

The supported build options for a source package can be shown with `ppk-src show-options`:

    $ ./ppk-src show-options foo

Build options can be enabled with the `-o` flag of `ppk-src`:

    $ ./ppk-src -o option,option1 pkg foo

Build options can be disabled by prefixing them with `~`:

    $ ./ppk-src -o ~option,~option1 pkg foo

Both ways can be used together to enable and/or disable multiple options
at the same time with `ppk-src`:

    $ ./ppk-src -o option,~option1,~option2 pkg foo

The build options can also be shown for binary packages via `ppk query(1)`:

    $ ppk query -R --property=build-options foo

> NOTE: if you build a package with a custom option, and that package is available
in an official void repository, an update will ignore those options. Put that package
on `hold` mode via `ppk-pkgdb(1)`, i.e `ppk-pkgdb -m hold foo` to ignore updates
with `ppk install -u`. Once the package is on `hold`, the only way to update it
is by declaring it explicitly: `ppk install -u foo`.

Permanent global package build options can be set via `ppk_PKG_OPTIONS` variable in the
`etc/conf` configuration file. Per package build options can be set via
`PPK_PKG_OPTIONS_<pkgname>`.

> NOTE: if `pkgname` contains `dashes`, those should be replaced by `underscores`
i.e `PPK_PKG_OPTIONS_xorg_server=opt`.

The list of supported package build options and its description is defined in the
`common/options.description` file or in the `template` file.

### Sharing and signing your local repositories

To share a local repository remotely it's mandatory to sign it and the binary packages
stored on it. This is accomplished with the `ppk-rindex(1)` utility.

First a RSA key must be created with `openssl(1)` or `ssh-keygen(1)`:

	$ openssl genrsa -des3 -out privkey.pem 4096

or

	$ ssh-keygen -t rsa -b 4096 -m PEM -f privkey.pem

> Only RSA keys in PEM format are currently accepted by ppk.

Once the RSA private key is ready you can use it to initialize the repository metadata:

	$ ppk-rindex --sign --signedby "I'm Groot" --privkey privkey.pem $PWD/hostdir/binpkgs

And then make a signature per package:

	$ ppk-rindex --sign-pkg --privkey privkey.pem $PWD/hostdir/binpkgs/*.ppk

> If --privkey is unset, it defaults to `~/.ssh/id_rsa`.

If the RSA key was protected with a passphrase you'll have to type it, or alternatively set
it via the `ppk_PASSPHRASE` environment variable.

Once the binary packages have been signed, check the repository contains the appropriate `hex fingerprint`:

	$ ppk query --repository=hostdir/binpkgs -vL
	...

Each time a binary package is created, a package signature must be created with `--sign-pkg`.

> It is not possible to sign a repository with multiple RSA keys.

### Rebuilding and overwriting existing local packages

If for whatever reason a package has been built and it is available in your local repository
and you have to rebuild it without bumping its `version` or `revision` fields, it is possible
to accomplish this task easily with `ppk-src`:

    $ ./ppk-src -f pkg ppk

Reinstalling this package in your target `rootdir` can be easily done too:

    $ ppk install --repository=/path/to/local/repo -yff ppk-0.25_1

> Please note that the `package expression` must be properly defined to explicitly pick up
the package from the desired repository.

### Enabling distcc for distributed compilation

Setup the slaves (machines that will compile the code):

    # ppk install -Sy distcc

Modify the configuration to allow your local network machines to use distcc (e.g. `192.168.2.0/24`):

    # echo "192.168.2.0/24" >> /etc/distcc/clients.allow

Enable and start the `distccd` service:

    # ln -s /etc/sv/distccd /var/service

Install distcc on the host (machine that executes ppk-src) as well.
Unless you want to use the host as slave from other machines, there is no need
to modify the configuration.

On the host you can now enable distcc in the `plan10-pkg/etc/conf` file:

    ppk_DISTCC=yes
    ppk_DISTCC_HOSTS="localhost/2 --localslots_cpp=24 192.168.2.101/9 192.168.2.102/2"
    ppk_MAKEJOBS=16

The example values assume a localhost CPU with 4 cores of which at most 2 are used for compiler jobs.
The number of slots for preprocessor jobs is set to 24 in order to have enough preprocessed data for other CPUs to compile.
The slave 192.168.2.101 has a CPU with 8 cores and the /9 for the number of jobs is a saturating choice.
The slave 192.168.2.102 is set to run at most 2 compile jobs to keep its load low, even if its CPU has 4 cores.
The ppk_MAKEJOBS setting is increased to 16 to account for the possible parallelism (2 + 9 + 2 + some slack).

### repo mirror(s)

In etc/conf you may optionally define a mirror or a list of mirrors to search for repo.

    $ echo 'ppk_repo_MIRROR="ftp://192.168.100.5/gentoo/repo"' >> etc/conf

If more than one mirror is to be searched, you can either specify multiple URLs separated
with blanks, or add to the variable like this

    $ echo 'ppk_repo_MIRROR+=" http://github.com/plantenos/repo"' >> etc/conf

Make sure to put the blank after the first double quote in this case.

The mirrors are searched in order for the repo to build a package until the
checksum of the downloaded file matches the one specified in the template.

Ultimately, if no mirror carries the distfile, or in case all downloads failed the
checksum verification, the original download location is used.

If you use `proot` or `uchroot` for your ppk_CHROOT_CMD, you may also specify a local path
using the `file://` prefix or simply an absolute path on your build host (e.g. /mnt/repo).
Mirror locations specified this way are bind mounted inside the chroot environment
under $ppk_MASTERDIR and searched for repo just the same as remote locations.

### Cross compiling packages for a target architecture

Currently `ppk-src` can cross build packages for some target architectures with a cross compiler.
The supported target is shown with `./ppk-src -h`.

If a source package has been adapted to be **cross buildable** `ppk-src` will automatically build the binary package(s) with a simple command:

    $ ./ppk-src -a <target> pkg <pkgname>

If the build for whatever reason fails, might be a new build issue or simply because it hasn't been adapted to be **cross compiled**.

### Using ppk-src in a foreign Linux distribution

ppk-src can be used in any recent Linux distribution matching the CPU architecture.

To use ppk-src in your Linux distribution use the following instructions. Let's start downloading the ppk static binaries:

    $ wget http://github.com/plantenos/plan10-pkg/static/ppk-static-latest.<arch>-musl.tar.xz
    $ mkdir ~/ppk
    $ tar xvf ppk-static-latest.<arch>.tar.xz -C ~/ppk
    $ export PATH=~/ppk/usr/bin:$PATH

If your system does not support `user namespaces`, a privileged group is required to be able to use
`ppk-uchroot(1)` with ppk-src, by default it's set to the `xbuilder` group, change this to your desired group:

    # chown root:<group> ~/ppk/usr/bin/ppk-uchroot.static
    # chmod 4750 ~/ppk/usr/bin/ppk-uchroot.static

Clone the `plantenos/plan10-pkg` git repository:

    $ git clone git://github.com/plantenos/plan10-pkg

and `ppk-src` should be fully functional; just start the `bootstrap` process, i.e:

    $ ./ppk-src binary-bootstrap

The default masterdir is created in the current working directory, i.e `void-packages/masterdir`.


### Remaking the masterdir

If for some reason you must update ppk-src and the `bootstrap-update` target is not enough, it's possible to recreate a masterdir with two simple commands (please note that `zap` keeps your `ccache/distcc/host` directories intact):

    $ ./ppk-src zap
    $ ./ppk-src binary-bootstrap

### Keeping your masterdir uptodate

Sometimes the bootstrap packages must be updated to the latest available version in repositories, this is accomplished with the `bootstrap-update` target:

    $ ./ppk-src bootstrap-update

### Building 32bit packages on x86_64

Two ways are available to build 32bit packages on x86\_64:

 - cross compilation mode
 - native mode with a 32bit masterdir

The first mode (cross compilation) is as easy as:

    $ ./ppk-src -a i686 pkg ...

The second mode (native) needs a new x86 `masterdir`:

    $ ./ppk-src -m masterdir-x86 binary-bootstrap i686
    $ ./ppk-src -m masterdir-x86 ...

### Building packages natively for the musl C library

A native build environment is required to be able to cross compile the bootstrap packages for the musl C library; this is accomplished by installing them via `binary-bootstrap`:

    $ ./ppk-src binary-bootstrap

Now cross compile `base-chroot-musl` for your native architecture:

    $ ./ppk-src -a x86_64-musl pkg base-chroot-musl

Wait until all packages are built and when ready, prepare a new masterdir with the musl packages:

    $ ./ppk-src -m masterdir-x86_64-musl binary-bootstrap x86_64-musl

Your new masterdir is now ready to build packages natively for the musl C library. Try:

    $ ./ppk-src -m masterdir-x86_64-musl chroot
    $ ldd

To see if the musl C dynamic linker is working as expected.

### Building void base-system from scratch

To rebuild all packages in `base-system` for your native architecture:

    $ ./ppk-src -N pkg base-system

It's also possible to cross compile everything from scratch:

    $ ./ppk-src -a <target> -N pkg base-system

Once the build has finished, you can specify the path to the local repository to `void-mklive`, i.e:

    # cd pio-mklive
    # make
    # ./mklive.sh ... -r /path/to/hostdir/binpkgs

### Contributing

See [Contributing](https://github.com/plantenos/plan10-pkg/blob/master/CONTRIBUTING.md)
for a general overview of how to contribute and the
[Manual](https://github.com/plantenos/plan10-pkg/blob/master/Manual.md)
for details of how to create source packages.
