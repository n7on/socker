# Socker
Socker is a Docker implementation in bash using flat images instead of `overlay` filesystem. Making it useful to inspect images, create base images, flatten existing images etc. Image root filesystem is downloaded and extracted to `~/.socker/images/<user>/<repo>/<tag>` where it can be further manipulated and uploaded again.

## Prerequisites
* bash
* jq
* curl

## Install
```bash
curl -sO https://raw.githubusercontent.com/n7on/socker/main/socker && sudo mv socker /usr/sbin/ && sudo chmod 755 /usr/sbin/socker
```


## Examples

### Download Image
`socker` requests Docker registry API's to fetch the manifest, downloads the layers and extracts it to the fileystem in correct order under `~/.socker/images/<docker-hub-username>/<repo>/<tag>`. Making usable in order to explore it further.  

Ex. Download alpine:latest to `~/.socker/images` 

``` bash

# note: in the main Docker Registry, all official images are part of the "library" "user".
socker pull library/alpine:latest

```

### Upload Image
You probably would want to first do `socker pull` to download some other image to work on, and move that to `~/.socker/images/<username>/`

`socker push` expects following environment variables to be exported before running.

```bash
export DOCKER_USERNAME=<your username>
export DOCKER_PASSWORD=<your password>

```

Ex. Move Nginx image/filesystem so it can be altered and uploaded to your own Docker Repo.
```bash
mkdir -p ~/.socker/images/<username>
mv ~/.socker/images/library/nginx ~/.socker/images/<username>/<repo>

```

Ex. Upload `~/.socker/images/<username>/alpine/latest` to `<username>/alpine:latest`  
``` bash

# note: in the main Docker registry, all official images are part of the "library" repository.
socker put <username>/alpine:latest

```


### Create own base Image
New base images can be created by copying what we need to `/.socker/images/<username>/<repo>/<tag>/` path and run `socker push <username>/<repo>:<tag>`. Which upload the manifest, configuration file and a single layer (as a tarball) to Docker repository using it's API's. Executables in Linux usually have dependencies to shared objects (dynamic libraries), so we need to add them as well. With that in mind, we would create an Image with only `ls` and `bash`, and upload it to our own Docker Repo, as a base-image:

```bash
# 1. create folders
path=~/.socker/images/<username>/<repo>/<tag>/
mkdir -p $path/{bin,lib}

cd $path

# 2. copy "ls" and "bash"
cp $(which bash) $(which ls) ./bin

# print shared object dependencies
ldd ./bin/ls

# 3. copy shared objects to our "lib"

# selinux
cp /lib/x86_64-linux-gnu/libselinux.so.1 ./lib
# glibc
cp /lib/x86_64-linux-gnu/libc.so.6 ./lib
# pcre
cp /lib/x86_64-linux-gnu/libpcre2-8.so.0 ./lib
cp /lib64/ld-linux-x86-64.so.2 ./lib
# above would need to be in lib64, so we just create that as a link.
ln -s lib lib64 

# What about linux-vdso.so.1? That is actually a kernel module loaded from memory.

# Now do same thing for "bash". Only 1 extra object to be added
ldd ./bin/ls
cp /lib/x86_64-linux-gnu/libtinfo.so.6 ./lib

# 4. Optional: run it in chroot, to test that it works.
# sudo chroot ./test bash

# 5. upload to Docker Registry using it's API's
# Before following exports is needed:
# 
# export DOCKER_USERNAME=<docker-hub-username>
# export DOCKER_PASSWORD=<docker-hub-password>
#
socker push <docker-hub-username>/<repository>:<tag>

```


### Run
`socker` uses `unshare` to create namespaces uts, mount, pid and ipc. And initializes the process by doing following:

* Directory above the root filesystem is mounted to itself. 
* oldroot directory is created, which is needed by `pivot_root`.
* `pivot_root` is used to change root to new root filesystem.
* `proc` filesystem is mounted to `/proc`.
* `sys` filesystem is mounted to `/sys`.
* `dev` filesystem is mounted to `/dev`.
* oldroot is unmounted and removed.
* hostname is changed to image name.
* The bash process is __replaced__ with the intended process, which is part of the new root filesystem. This is needed because the process started need to be PID 1. 

To run sh inside alpine image, as a container, we could do like this:
```bash

socker run library/alpine:latest sh
```

### Use with WSL2
We could use the image root filesystem in WSL2, importing the filesystem as a tarball in WSL2 will create a new WSL2 distribution. 

> Docker Desktop for Windows can be used with WSL2, which will create two distributions called docker-desktop & docker-desktop-data. The one where Docker Runtime runs is docker-desktop.

Ex. Use image with WSL2

``` powershell
# First create some folders in Windows
$Path = c:\WSLDistros\alpine
New-Item -ItemType Directory -Path $Path

```
And copy tarball to that folder  
```bash
# Linux

# download
socker pull library/alpine:latest

# pack
tar -czvf alpine.tar.gz -C ~/.socker/library/alpine/latest .

# copy
cp alpine.tar.gz /mnt/c/WSLDistros/
```
And import it into WSL2.
``` powershell
# Windows again

wsl.exe --import "alpine" C:\WSLDistros\alpine\ C:\WSLDistros\alpine.tar.gz
```
