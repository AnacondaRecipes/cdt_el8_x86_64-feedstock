# Rocky Linux 8 CDTs

## Building the CDTs

There is a script, `conda-build-all` which iterates through the CDTs
building them in order:

``` bash
$ ./cdt_el8_x86_64/conda-build-all [-c {sysroot-staging-channel}]
...
```

>[!TIP]
> The recipes skip anything not `linux-64` so you will want to run
> such a (Docker) instance.

>[!TIP]
> The recipes depend on the linux sysroot using GLIBC 2.28 so you'll
> want to build that first if it is not already published.

``` bash
conda build linux-sysroot-feedstock
```

## How To Extend The Set of CDTs

Unlike most feedstocks, the CDTs are lots of subdirectories each with
its own `meta.yaml` and `build.sh` -- there is no `recipe` directory
_per se_, each `foo-el8-x86_64` directory is equivalent to another
`recipe` directory.  `conda build` knows what to do with it all.

The basic structure of these files is very simple, you can start by
cut'n'pasting an existing entry noting that `build.sh` is _nearly_
identical in all subdirectories, you'll probably only need to edit
`meta.yaml`:

``` bash
mkdir foo-el-x86_64
cp libxcrypt-el8-x86_64/* foo-el-x86_64
```

### build.sh

`build.sh` should only be setting up some `lib` / `usr/lib64` symlinks
(slightly dubiously!) and copying files from `binary` (where we
downloaded and extracted the RPM into).

To patch some things up, there may be some additional `rm`s in
individual `build.sh` scripts -- so watch out if you're copying a
random one.

### meta.yaml

#### package

Change `name` and `version`!

#### source.url

You would think that most of the entities that we want are in
`BaseOS`,
eg. `https://download.rockylinux.org/vault/rocky/8.9/BaseOS/x86_64/os/Packages`
but some of them might be in `AppStream`,
eg. `https://raw.repo.almalinux.org/vault/8.9/AppStream/x86_64/os/Packages`
and possibly elsewhere.

We obviously want to use the latest version at the time.

##### Source RPM

Be warned, the source RPM URL, although commented out currently, may
well have a different **name** from the package URL.

#### requirements

The basic section looks like:

``` yaml
requirements:
  build:
    - sysroot_linux-64 2.28.*
  host:
  run:
    - sysroot_linux-64 2.28.*
    - {runtime-dependencies}
```

where `2.28` is our chosen GLIBC version and `{runtime-dependencies}`
gets a bit more interesting.

These are either going to be:

- regular conda dependencies, particularly where we've been building
  out some core dependencies ourselves (so we are less dependent on
  CDTs)

- other CDTs where the dependency information looks like

``` yaml
    - libcap-ng-el8-x86_64 >=0.7.11 *_0
```

and `0.7.11` is the version number of the CDT we are building (have a
look in `libcap-ng-el8-x86_64/meta.yaml`!) and `*_0` (which should
become `*_{{ build_number }}`!) marries this recipe up with the same
build number across all of these CDTs -- should we find the need to
revisit them.

##### Which Dependencies Are Required?

In order to know what dependencies a given CDT has we, essentially,
need to figure out what libraries it uses in turn.

We can query the RPMs `rpm -qp [options] $RPM`.  If you have "built"
the conda package, the source RPM will be available in
`.../conda-bld/src_cache`:

``` bash
$ rpm -qpl $RPM
...file listing...

$ rpm -qpi $RPM
...conda-like package info...

$ rpm -qp --provides pam-1.3.1-27.el8.x86_64_0a6d22f387.rpm
warning: pam-1.3.1-27.el8.x86_64_0a6d22f387.rpm: Header V4 RSA/SHA256 Signature, key ID 6d745a60: NOKEY
config(pam) = 1.3.1-27.el8
libpam.so.0()(64bit)
libpam.so.0(LIBPAM_1.0)(64bit)
libpam.so.0(LIBPAM_EXTENSION_1.0)(64bit)
...

$ rpm -qp --requires pam-1.3.1-27.el8.x86_64_0a6d22f387.rpm
warning: pam-1.3.1-27.el8.x86_64_0a6d22f387.rpm: Header V4 RSA/SHA256 Signature, key ID 6d745a60: NOKEY
...
ld-linux-x86-64.so.2()(64bit)
ld-linux-x86-64.so.2(GLIBC_2.3)(64bit)
libaudit.so.1()(64bit)
libc.so.6()(64bit)
libc.so.6(GLIBC_2.14)(64bit)
libc.so.6(GLIBC_2.15)(64bit)
libc.so.6(GLIBC_2.2.5)(64bit)
libc.so.6(GLIBC_2.27)(64bit)
libc.so.6(GLIBC_2.3)(64bit)
libc.so.6(GLIBC_2.3.4)(64bit)
libc.so.6(GLIBC_2.4)(64bit)
libc.so.6(GLIBC_2.7)(64bit)
libc.so.6(GLIBC_2.8)(64bit)
libc.so.6(GLIBC_2.9)(64bit)
libcrack.so.2()(64bit)
libcrypt.so.1()(64bit)
libcrypt.so.1(XCRYPT_2.0)(64bit)
libdb-5.3.so()(64bit)
libdl.so.2()(64bit)
libdl.so.2(GLIBC_2.2.5)(64bit)
libnsl.so.2()(64bit)
libnsl.so.2(LIBNSL_1.0)(64bit)
...
```

###### Where are those Dependencies?

First of all, query conda because we would prefer to use a regular
conda package over yet another CDT.

We can query whether any existing conda packages have a library
https://conda-metadata-app.streamlit.app/Search_by_file_path?path=lib%2Flibasound.so.2

Otherwise we can search for libraries online:
https://pkgs.org/search/?q=libasound.so.2 and then double check it is
available in https://download.rockylinux.org/vault/rocky/8.9.

###### Back to Basics

For a given shared library we can run `readelf -d foo.so | grep
NEEDED` and then discover where those libraries are, as above.

Technically, we should do that for any binaries too!

But wait!  I have an RPM, where is the shared library?  And doesn't an
RPM tell us about dependencies?

Well, the RPM's declaration of dependency is for the RPM eco-system
which is instructive but not necessarily fundamentally useful for us.

###### Cross-referencing Dependencies

Of course, one problem, here, is that we don't necessarily know if any
of our existing CDTs contains the library we are looking for.  So what
we want to do is iterate over all of the packages we create and
simultaneously record which libraries appear in which packages (CDTs)
and which libraries each of those libraries require in turn.

A further problem here, is that the success on the first run depends
on which order you probe the packages.  However, if you have recorded
the results of the first pass then on the second pass you should have
a complete set of library->package mappings and therefore you can
successfully match any needed libraries to packages and flag up any
missing ones.

Using such a script, `report-library-dependencies`, in the context of
a build we might:

``` bash
$ cd /path/to/aggregate
$ ./cdt_el8_x86_64/conda-build-all [-c {sysroot-staging-channel}]
...
$ cd /path/to/conda-bld/noarch
$ /path/to/report-library-dependencies
...
needed        by                   in
libawt.so     libawt_xawt.so       java-1.8.0-openjdk
libawt.so     libjawt.so           java-1.8.0-openjdk
libdb-5.3.so  pam_userdb.so        pam
libjava.so    libawt_xawt.so       java-1.8.0-openjdk
libjava.so    libjawt.so           java-1.8.0-openjdk
libjava.so    libjsoundalsa.so     java-1.8.0-openjdk
libjvm.so     libawt_xawt.so       java-1.8.0-openjdk
libjvm.so     libjawt.so           java-1.8.0-openjdk
libjvm.so     libjsoundalsa.so     java-1.8.0-openjdk
libnsl.so.2   pam_unix_acct.so     pam
libnsl.so.2   pam_unix_auth.so     pam
libnsl.so.2   pam_unix_passwd.so   pam
libnsl.so.2   pam_unix_session.so  pam
libnsl.so.2   pam_unix.so          pam
```

Here, we might debate about whether it matters that we are missing
these libraries.

##### -devel packages

If we are looking at some RPM, `foo`, and there is a corresponding
`foo-devel` RPM then we should be including both `foo-devel` and `foo`
RPM in our set of CDTs.
