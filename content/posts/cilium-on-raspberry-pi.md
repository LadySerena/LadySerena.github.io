{
"title": "Cilium on Raspberry Pi",
"date": "2022-02-15T10:44:48-06:00",
"draft": false,
"description": "Cilium for CNI for a Raspberry Pi Kubernetes Cluster",
"catagories": [
"Kubernetes"
],
"tags": [
"Kubernetes",
"Homelab"
],
"series": [
"Kubernetes at Home"
]

}

## Intro

Hello everyone and today I'm going through the steps I took to get cilium up and running on my raspberry pis. This post
is going to go through the sysctls I had to enable as well as enabling the appropriate kernel modules that aren't part
of the stock kernel package. Without further ado let's get into it!

So why did I get into cilium for CNI when a lot of people really like calico or flannel? I hang out on Twitter and a lot
of people were really amped about cilium and how powerful eBPF (Extended Berkeley Packet Filter) is and you can write
programs to do really cool observability things. My coworker proceeded to say a string of words that I couldn't parse
because I'm not all that familiar with packet filter programs, but I knew that if cool people say this is important tech
then I should at least check it out.

I chose cilium for CNI, but I needed a home for my Kubernetes cluster. I priced out a GKE cluster, and noticed that for
the VMs it would cost me about $300 a month...That was a bit expensive for me. I instead bought 3 Raspberry Pis and got
some 256GB SSDs and the USB3 to SATA connectors. I think I had a one time cost of about $500 (still a chunk of change
but one time purchase vs ongoing expense is more affordable).

## Image Building

Now that I have hardware in hand, I need to grab a Linux distro. I'm lazy and don't want to configure everything by hand
so let's make our own images! I was pretty familiar with [Packer][packer-site] and there's a [builder for ARM]
[packer-builder-arm], but I had some issues with it on newer Packer versions, but it was helpful. I read the source code
to learn how to mount `.img` files. It uses [losetup][losetup-man-page] which creates loop devices (fake block devices).

I found a [blog post][custom-image-post] on building custom Arch Linux ARM images, in the end I used ubuntu server 21.10
but this was helpful for me understanding how to build my own images.

After cobbling together the sources to build my own image I wrote this script to build my own Raspberry Pi images
[link to script][image-build-script]. Let's step through what the script is doing. The linked script is not as annotated
as the one in this post, but I wanted to annotate this one since I don't want to give readers a script and not explain
what is happening. You may have to scroll to see the full script comments.

```shell
#!/usr/bin/env bash
# tells the shell which interpreter to use in this case it is bash
set -exo pipefail 
#print which commands are being ran for debugging and have the script fail if a 
# command fails

wget https://cdimage.ubuntu.com/releases/21.10/release/ubuntu-21.10-preinstalled-server-arm64+raspi.img.xz
echo "126f940d3b270a6c1fc5a183ac8a3d193805fead4f517296a7df9d3e7d691a03 *ubuntu-21.10-preinstalled-server-arm64+raspi.img.xz" | shasum -a 256 --check

# decompress the image file since we can't manipulate compressed files
xz -dk ubuntu-21.10-preinstalled-server-arm64+raspi.img.xz 

# mount the image file onto a loop device
sudo losetup -Pf ubuntu-21.10-preinstalled-server-arm64+raspi.img

# I had to expand the root partition by 2GB to contain all the packages I was using
sudo truncate -c -s +2048M ubuntu-21.10-preinstalled-server-arm64+raspi.img
# parted gives machine parsable output on the partition table and I then get the array to get the new ending sector 
# before expanding the partition
IN=$(sudo parted /dev/loop0 print -m -s | tail -n 1)

# I had to disable this part of shellcheck (great linter btw) since I did want to split the output
# shellcheck disable=SC2206
arrIN=(${IN//:/ })

# the third item in the array contained the new ending sector of the root partition
sudo parted /dev/loop0 resizepart 2 "${arrIN[2]}" -s
# forces a check and repair of the partition (it was required to get the resize2fs command to work)
sudo e2fsck -p -f /dev/loop0p2
# finally resize the filesystem to use the newly added space
sudo resize2fs /dev/loop0p2

# setup the mount points for the image
sudo mount /dev/loop0p2 /mnt/
sudo mount /dev/loop0p1 /mnt/boot/firmware
```

## Breadcrumbs to Get Cilium Working

## Install Kubernetes

[packer-site]: https://www.packer.io/

[packer-builder-arm]: https://github.com/mkaczanowski/packer-builder-arm

[custom-image-post]: https://disconnected.systems/blog/raspberry-pi-archlinuxarm-setup/

[losetup-man-page]: https://manpages.debian.org/bullseye/mount/losetup.8.en.html

[image-build-script]: https://github.com/LadySerena/tf-platform/blob/0caefa59da44cc664793d795364b2db740ad612e/scripts/ubuntu-21-10.bash