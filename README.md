---
title: Lab - Manage your own Debian packages
author: Sylvain Reynaud
date : 21/02/2022
geometry: margin=1in
---

In this lab, you will learn to:

* create your own Debian package,
* setup your own Debian repository,
* sign your package and your repository.

The lab uses a simple application called 'audioboard' for the example.

# Create a Debian package

Goal: create a Debian package for the application 'audioboard'.

## Setup of the repository structure

### The plan

Audioboard is a small application that plays a mp3 file:

* it uses mpg123 to play the mp3 file -> it is a dependency of the package,
* it brings a terminal interface -> a simple bash script,
* it brings a mp3 file named 'audio.mp3' -> it is a data file of the package.

The debian package will:

* Install mpg123 package if needed,
* Setup the bash script `audioboard` in `/usr/bin/`,
* Setup `audio.mp3` in `usr/local/audioboard/`

Then we will build the package.

### Setup

Let's describe the directory structure:

* A folder named `audioboard` will be the root folder,
* Inside this `audioboard` folder there is a `DEBIAN` folder which contains the `control` file,
* All files that need to be copied onto the final machine are in the `audioboard` folder, the tree structure will be merge with the one on the machine.

Create the folder structure:

```sh
export PACKAGE_NAME=audioboard

mkdir ${PACKAGE_NAME}

mkdir -p ${PACKAGE_NAME}/DEBIAN
mkdir -p ${PACKAGE_NAME}/usr/local/${PACKAGE_NAME}
mkdir -p ${PACKAGE_NAME}/usr/bin/
```

Create a simple `control` file:

```sh
echo "Package: ${PACKAGE_NAME}
Version: 0.0.1
Maintainer: Me Me <me@me.local>
Depends: mpg123
Architecture: amd64
Description: A program that play a sound
Section: utils
Priority: extra
" \
> ${PACKAGE_NAME}/DEBIAN/control
```

For additional informations about `control` check out the [documention](https://www.debian.org/doc/debian-policy/ch-controlfields.html#source-package-control-files-debian-control)

Copy your own `audio.mp3` in `/usr/local/${PACKAGE_NAME}`:

```sh
cp audio.mp3 ${PACKAGE_NAME}/usr/local/${PACKAGE_NAME}/audio.mp3
```

Copy your `audioboard` in `/usr/bin/`:

```sh
export BINAIRY_NAME=audioboard
# Place the binary/script at the right place
cp ${BINAIRY_NAME} ${PACKAGE_NAME}/usr/bin/${BINAIRY_NAME}
```

### Build the package

The default package name is `<package-name>_<version>-<release-number>_<architecture>.deb`

```sh
export VERSION=0.0.1
export RELEASE_NUMBER=1
export ARCH=amd64

export FINAL_PACKAGE_NAME=${PACKAGE_NAME}_${VERSION}-${RELEASE_NUMBER}_${ARCH}.deb

# apt-get install -y dpkg-dev
# if you are in the root directory do cd ..
dpkg-deb --build ${PACKAGE_NAME}

mv ${PACKAGE_NAME}.deb ${FINAL_PACKAGE_NAME}
```

You can check the package with `dpkg -I ${FINAL_PACKAGE_NAME}`.

The next step is setting up your repository in order to test your package.

# Setup a debian repository

Goal : setup a debian repository for the package `audioboard`, then install the package with apt.

Steps:

* Place your package in a directory (here `respository/`) exposed on the web ,
* Generate a compressed `Packages` file which contains the list of all available packages,
* Expose `respository/` with a web server,
* Add the repository to your `/etc/apt/sources.list.d` directory,
* Install the package with `apt`.

## Setup simple repository

**On the developer machine**

As root run `apt install dpkg-dev -y`, then:

```sh
mkdir repository
cp ${FINAL_PACKAGE_NAME} repository/
```

## Generate Packages file

**On the developer machine**

Do this at every updates:

```sh
cd repository
dpkg-scanpackages -m . | gzip -9c > Packages.gz
```lly :

`apt install reprepro`


## Expose the repository

**The server is supposed to be on another machine** but for the example we will use the same machine.

Expose the repository with a web server, here with a dockerized nginx on port 8000:

```sh
# In git root directory
cd nginx
docker-compose up -d

# Move the repository to the web server
mv ../repository/ ./src
```



## Add repository to /etc/apt/sources.list.d

**On the developer machine**

```sh
export REPO_HOST=http://localhost:8000/repository
# as root
echo "deb [trusted=yes] ${REPO_HOST} ./" > /etc/apt/sources.list.d/${PACKAGE_NAME}.list
```

## Install the package with `apt`

**On the developer machine**

```sh
# as root
apt update
apt install ${PACKAGE_NAME} -y
```

## Sign deb package

We previously used a flat repository. Here we will use a signed repository, which is a repository that contains the signature of the packages. First, we need to generate our gpg key.

Configure gpg by editing `~/.gnupg/gpg.conf`. Set the following parameters:

```
# Prioritize stronger algorithms for new keys.
default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAST5 BZIP2 ZLIB ZIP Uncompressed
# Use a stronger digest than the default SHA1 for certifications.
cert-digest-algo SHA512
```

Then generate a gpg key : `gpg --gen-key` and finally export the key :

```sh
export EMAIL=john@example.com
gpg --armor --export ${EMAIL} > ${EMAIL}.gpg.key
```

Your packages will be automatically signed as long as the name and email address in your package’s `control` file are the same as that of the GPG key you created.

Otherwise you will have to sign your packages manually :

```sh
apt install dpkg-sig
dpkg-sig --sign builder audioboard_0.0.1-1_amd64.deb
```

## Signed and non-flat repository

On the **repository machine**, create a distribution configuration in the repository directory:

```sh
mkdir conf
touch conf/distributions
```

```
Origin: <your-repository-host>
Label: apt repository
Codename: <osrelease>
Architectures: <arch>
Components: main
Description: My debian package repo
SignWith: yes
Pull: <osrelease>
```
In our case :

* `<your-repository-host>` : localhost

* `<osrelease>` : bullseye

* `<arch>` : amd64

Now, install `reprepro`, the tool that will create the complete structure (like specified on the [debian wiki](https://wiki.debian.org/DebianRepository/Format?action=show&redirect=RepositoryFormat)) inside the repository automatically :

`apt install reprepro`

Then, sign the repository :

`reprepro -Vb . -S utils includedeb bullseye $(pwd)/audioboard_0.0.1-1_amd64.deb`

Use `--ask-passphrase` is you have one on your gpg key.

`utils` is the Section of the package (set in the `control` file).

`bullseye` is the distribution name.

## Test the package

Add the gpg key to your keyring, then install the package !

```sh
export REPO_HOST=http://localhost:8000/repository
wget -O - ${REPO_HOST}/john@example.com.gpg.key | sudo apt-key add -
echo "deb ${REPO_HOST} bullseye main" > /etc/apt/sources.list.d/${PACKAGE_NAME}.list
apt update
apt install ${PACKAGE_NAME} -y
```

# Sources

* 18/02/2022 - [LinuxHint - Debian Package Creation HowTo](https://linuxhint.com/debian-package-creation-howto/)
* 18/02/2022 - [LinuxConfig - Easy way to create a Debian package and local package repository](https://linuxconfig.org/easy-way-to-create-a-debian-package-and-local-package-repository)
* 18/02/2022 - [Eeathly - Creating and hosting your own deb packages and apt repo](https://earthly.dev/blog/creating-and-hosting-your-own-deb-packages-and-apt-repo/#step-1-creating-a-deb-package)
* 18/02/2022 - [Debian.org - 5. Control files and their fields](https://www.debian.org/doc/debian-policy/ch-controlfields.html#source-package-control-files-debian-control)
* 18/02/2022 - [Debian.org - SetupWithReprepro](https://wiki.debian.org/DebianRepository/SetupWithReprepro)
* 18/02/2022 - [Jon Cowie's blog - Creating your own Signed APT Repository and Debian Packages](https://scotbofh.wordpress.com/2011/04/26/creating-your-own-signed-apt-repository-and-debian-packages/)