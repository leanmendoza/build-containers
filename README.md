# Godot engine build containers

This repository contains the Dockerfiles for the official Godot engine builds.
These containers should help you build Godot for all platforms supported on
any machine that can run Docker containers.

The in-container build scripts are in a separate repository:
https://github.com/godotengine/godot-build-scripts


## Introduction

These scripts build a number of containers which are then used to build final
Godot tools, templates and server packages for several platforms.

Once these containers are built, they can be used to compile different Godot
versions without the need of recreating them.

The `upload.sh` file is meant to be used by Godot Release Team and is not
documented here.


## Requirements

These containers have been tested under Fedora 34 and Ubuntu 18.04 (others may work too).

The tool used to build and manage the containers is `podman`.

See the Host OS section below for further information on how to setup your host OS before start.


## Usage

The 'build.sh' script included is used to build the containers themselves.

Run the command using:

    ./build.sh 3.x mono-6.12.0.147

Note that this will also download that Mono branch (2020-02) from Mono repository.
That branch corresponds to the given Mono version (6.12.0.147) as per
https://www.mono-project.com/docs/about-mono/versioning/#mono-source-versioning .

More details can be found in the Godot https://github.com/godotengine/godot-mono-builds
repository (but you don't need this repository, as in this case Mono is built
inside the containers)

The above will generate images using the tag '3.x-mono-6.12.0.147'. This is convenient
since as of today, this branch can be used to compile every 3.x version or
your custom modifications.

### Selecting which images to build

If you don't need to build all versions or you want to try with a single target OS first,
you can comment out the corresponding lines from the script:

    $podman_build_mono -t godot-linux:${img_version} -f Dockerfile.linux . 2>&1 | tee logs/linux.log
    $podman_build_mono -t godot-windows:${img_version} -f Dockerfile.windows . 2>&1 | tee logs/windows.log
    $podman_build_mono -t godot-javascript:${img_version} -f Dockerfile.javascript . 2>&1 | tee logs/javascript.log
    $podman_build_mono -t godot-android:${img_version} -f Dockerfile.android . 2>&1 | tee logs/android.log
    ...

Note: The MSVC image (used for UWP builds) does not work currently.

## Host OS preparation

### Podman Fedora image

To be extra-sure that you are building with the same base container image as the official
builds, you can use:

    podman pull registry.fedoraproject.org/fedora@sha256:sha256:8b01cffca564ca914d5d3c8dc8c6eca12a755ee4d1d898e22e83ad7128fae256
    podman image tag registry.fedoraproject.org/fedora@abec9a7a7dc6 fedora:34

### Fedora 34 Host

Fedora 34 default configuration is able to build the containers. Ensure the tools
are installed:

    sudo dnf -y install podman

### Ubuntu 18.04 Host

Note: Using a Ubuntu 18.04 host is not recommended, we strongly recommend to use
Fedora hosts to guarantee compatibility with what the official buildsystem uses.

Install `podman` (as per https://podman.io/getting-started/installation). On
Ubuntu 18.04, podman 2.2.1 was used successfully:

    . /etc/os-release
    echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/ /" | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
    curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/xUbuntu_${VERSION_ID}/Release.key | sudo apt-key add -
    sudo apt-get update
    sudo apt-get -y upgrade
    sudo apt-get -y install podman
    # (Ubuntu 18.04) Restart dbus for rootless podman
    systemctl --user restart dbus

Modify your system default ulimit to support more open file handlers.
Add this at the end of your /etc/sysctl.conf file:

    fs.file-max = 65536

Then reboot or run:

    sudo sysctl -p

Install Python3 dataclasses:

    pip3 install dataclasses

Install wine64, binfmt_misc, and configure it:

    sudo apt install wine64 wine64-preloader binfmt-support

    sudo bash -c "echo -1 > /proc/sys/fs/binfmt_misc/wine"  # It's ok this command fails, eg. if you don't have wine binfmt
    sudo bash -c 'echo ":windows:M::MZ::/usr/bin/wine:" > /proc/sys/fs/binfmt_misc/register'
    sudo bash -c 'echo ":windowsPE:M::PE::/usr/bin/wine:" > /proc/sys/fs/binfmt_misc/register'

This `binfmt` configuration **is not persistent**, you need to do it after a reboot in order to build the containers.

(Note that this may break previous .exe binfmt support through `run-detectors`.)


## Appendix: Image sizes

These are the expected container image sizes, so you can plan your disk usage in advance:

    REPOSITORY                                       TAG                    SIZE
    localhost/godot-fedora                           3.x-mono-6.12.0.147  642 MB
    localhost/godot-export                           3.x-mono-6.12.0.147  1.11 GB
    localhost/godot-mono                             3.x-mono-6.12.0.147  1.54 GB
    localhost/godot-mono-glue                        3.x-mono-6.12.0.147  1.78 GB
    localhost/godot-linux                            3.x-mono-6.12.0.147  3.56 GB
    localhost/godot-windows                          3.x-mono-6.12.0.147  3.46 GB
    localhost/godot-javascript                       3.x-mono-6.12.0.147  3.8 GB
    localhost/godot-android                          3.x-mono-6.12.0.147  19.6 GB
    localhost/godot-osx                              3.x-mono-6.12.0.147  5.85 GB
    localhost/godot-ios                              3.x-mono-6.12.0.147  7.08 GB

In addition to this, generating containers will also require some host disk space
(up to 30 GB) for the downloaded Mono sources and dependencies (Xcode, MSVC).
