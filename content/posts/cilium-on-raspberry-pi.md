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
so let's make our own images! I was pretty familiar with [Packer][packer-site] and there's
a [builder for ARM][packer-builder-arm], but I had some issues with it on newer Packer versions, but it was helpful. I
read the source code to learn how to mount `.img` files. It uses [losetup][losetup-man-page] which creates loop
devices (fake block devices).

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

So the script at this point has downloaded the ubuntu server image and mounted the image to a loop device. Now it's time
to configure the image to do our important configuration before installing Kubernetes. The next part of the script 
is a large heredoc that is the script that actually does the configuration. 


maybe remove script since it's hella long?
```shell
#!/usr/bin/env bash
set -exo pipefail

# this function sets the relavant kernel arguments. Everything is default until the cgroup_enable argument.
# cgroup_enable=memory swapaccount=1 cgroup_memory=1 cgroup_enable=cpuset is required for Kubernetes
function kernel-nonsense() {
  cat <<'EOF' | tee /boot/firmware/cmdline.txt
dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=LABEL=writable rootfstype=ext4 elevator=deadline rootwait fixrtc quiet splash cgroup_enable=memory swapaccount=1 cgroup_memory=1 cgroup_enable=cpuset
EOF
}

# I wanted cloud-init to not create the default ubuntu user and add my public key
function cloud-init-fix() {
  cat <<'EOF' | tee /etc/cloud/cloud.cfg
# The top level settings are used as module
# and system configuration.
# A set of users which may be applied and/or used by various modules
# when a 'default' entry is found it will reference the 'default_user'
# from the distro configuration specified below
users:
  - name: kat
    gecos: my user
    groups: [ adm, audio, cdrom, dialout, dip, floppy, lxd, netdev, plugdev, sudo, video ]
    sudo: [ "ALL=(ALL) NOPASSWD:ALL" ]
    shell: /bin/bash
    ssh_authorized_keys:
      - ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIHRGGe84zs3TxJ8BTbsiVDAsctSf2JF5AS6g/5CyGD2l kat@local-pis
# If this is set, 'root' will not be able to ssh in and they
# will get a message to login instead as the default $user
disable_root: true
# This will cause the set+update hostname module to not operate (if true)
preserve_hostname: false
# If you use datasource_list array, keep array items in a single line.
# If you use multi line array, ds-identify script won't read array items.
# Example datasource config
# datasource:
#    Ec2:
#      metadata_urls: [ 'blah.com' ]
#      timeout: 5 # (defaults to 50 seconds)
#      max_wait: 10 # (defaults to 120 seconds)
# The modules that run in the 'init' stage
cloud_init_modules:
  - migrator
  - seed_random
  - bootcmd
  - write-files
  - growpart
  - resizefs
  - disk_setup
  - mounts
  - set_hostname
  - update_hostname
  - update_etc_hosts
  - ca-certs
  - rsyslog
  - users-groups
  - ssh
# The modules that run in the 'config' stage
cloud_config_modules:
  # Emit the cloud config ready event
  # this can be used by upstart jobs for 'start on cloud-config'.
  - emit_upstart
  - snap
  - ssh-import-id
  - locale
  - set-passwords
  - grub-dpkg
  - apt-pipelining
  - apt-configure
  - ubuntu-advantage
  - ntp
  - timezone
  - disable-ec2-metadata
  - runcmd
  - byobu
# The modules that run in the 'final' stage
cloud_final_modules:
  - package-update-upgrade-install
  - fan
  - landscape
  - lxd
  - ubuntu-drivers
  - write-files-deferred
  - puppet
  - chef
  - mcollective
  - salt-minion
  - reset_rmc
  - refresh_rmc_and_interface
  - rightscale_userdata
  - scripts-vendor
  - scripts-per-once
  - scripts-per-boot
  - scripts-per-instance
  - scripts-user
  - ssh-authkey-fingerprints
  - keys-to-console
  - install-hotplug
  - phone-home
  - final-message
  - power-state-change
# System and/or distro specific settings
# (not accessible to handlers/transforms)
system_info:
  # This will affect which distro class gets used
  distro: ubuntu
  # Default user name + that default users groups (if added/used)
  default_user:
    name: ubuntu
    lock_passwd: True
    gecos: Ubuntu
    groups: [adm, audio, cdrom, dialout, dip, floppy, lxd, netdev, plugdev, sudo, video]
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    shell: /bin/bash
  network:
    renderers: ['netplan', 'eni', 'sysconfig']
  # Automatically discover the best ntp_client
  ntp_client: auto
  # Other config here will be given to the distro class and/or path classes
  paths:
    cloud_dir: /var/lib/cloud/
    templates_dir: /etc/cloud/templates/
    upstart_dir: /etc/init/
  package_mirrors:
    - arches: [i386, amd64]
      failsafe:
        primary: http://archive.ubuntu.com/ubuntu
        security: http://security.ubuntu.com/ubuntu
      search:
        primary:
          - http://%(ec2_region)s.ec2.archive.ubuntu.com/ubuntu/
          - http://%(availability_zone)s.clouds.archive.ubuntu.com/ubuntu/
          - http://%(region)s.clouds.archive.ubuntu.com/ubuntu/
        security: []
    - arches: [arm64, armel, armhf]
      failsafe:
        primary: http://ports.ubuntu.com/ubuntu-ports
        security: http://ports.ubuntu.com/ubuntu-ports
      search:
        primary:
          - http://%(ec2_region)s.ec2.ports.ubuntu.com/ubuntu-ports/
          - http://%(availability_zone)s.clouds.ports.ubuntu.com/ubuntu-ports/
          - http://%(region)s.clouds.ports.ubuntu.com/ubuntu-ports/
        security: []
    - arches: [default]
      failsafe:
        primary: http://ports.ubuntu.com/ubuntu-ports
        security: http://ports.ubuntu.com/ubuntu-ports
  ssh_svcname: ssh
EOF
}

# This function sets up the kernel modules that are required for Kubernetes
# veth module is very important since it creates virtual ethernet devices.
# the net.bridget sysctls are required for Kubernetes.
function k8s-modules() {
  cat <<EOF | tee /etc/modules-load.d/k8s.conf
br_netfilter
veth
EOF
  cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
}

# This also sets up the same modules as k8s-modules() so this is probably unneeded
function containerd-modules() {
  cat <<EOF | tee /etc/modules-load.d/containerd.conf
  overlay
  br_netfilter
EOF
  cat <<EOF | tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
}

# This big heredoc is making containerd use systemd cgroup management
# use systemd driver instead of cgroupfs
function configure-containerd() {
  cat <<EOF | tee /etc/containerd/config.toml
version = 2
root = "/var/lib/containerd"
state = "/run/containerd"
plugin_dir = ""
disabled_plugins = []
required_plugins = []
oom_score = 0
[grpc]
  address = "/run/containerd/containerd.sock"
  tcp_address = ""
  tcp_tls_cert = ""
  tcp_tls_key = ""
  uid = 0
  gid = 0
  max_recv_message_size = 16777216
  max_send_message_size = 16777216
[ttrpc]
  address = ""
  uid = 0
  gid = 0
[debug]
  address = ""
  uid = 0
  gid = 0
  level = ""
[metrics]
  address = ""
  grpc_histogram = false
[cgroup]
  path = ""
[timeouts]
  "io.containerd.timeout.shim.cleanup" = "5s"
  "io.containerd.timeout.shim.load" = "5s"
  "io.containerd.timeout.shim.shutdown" = "3s"
  "io.containerd.timeout.task.state" = "2s"
[plugins]
  [plugins."io.containerd.gc.v1.scheduler"]
    pause_threshold = 0.02
    deletion_threshold = 0
    mutation_threshold = 100
    schedule_delay = "0s"
    startup_delay = "100ms"
  [plugins."io.containerd.grpc.v1.cri"]
    disable_tcp_service = true
    stream_server_address = "127.0.0.1"
    stream_server_port = "0"
    stream_idle_timeout = "4h0m0s"
    enable_selinux = false
    selinux_category_range = 1024
    sandbox_image = "k8s.gcr.io/pause:3.2"
    stats_collect_period = 10
    systemd_cgroup = false
    enable_tls_streaming = false
    max_container_log_line_size = 16384
    disable_cgroup = false
    disable_apparmor = false
    restrict_oom_score_adj = false
    max_concurrent_downloads = 3
    disable_proc_mount = false
    unset_seccomp_profile = ""
    tolerate_missing_hugetlb_controller = true
    disable_hugetlb_controller = true
    ignore_image_defined_volumes = false
    [plugins."io.containerd.grpc.v1.cri".containerd]
      snapshotter = "overlayfs"
      default_runtime_name = "runc"
      no_pivot = false
      disable_snapshot_annotations = true
      discard_unpacked_layers = false
      [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
        runtime_type = ""
        runtime_engine = ""
        runtime_root = ""
        privileged_without_host_devices = false
        base_runtime_spec = ""
      [plugins."io.containerd.grpc.v1.cri".containerd.untrusted_workload_runtime]
        runtime_type = ""
        runtime_engine = ""
        runtime_root = ""
        privileged_without_host_devices = false
        base_runtime_spec = ""
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
          runtime_type = "io.containerd.runc.v2"
          runtime_engine = ""
          runtime_root = ""
          privileged_without_host_devices = false
          base_runtime_spec = ""
          [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
    [plugins."io.containerd.grpc.v1.cri".cni]
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
      max_conf_num = 1
      conf_template = ""
    [plugins."io.containerd.grpc.v1.cri".registry]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]
    [plugins."io.containerd.grpc.v1.cri".image_decryption]
      key_model = ""
    [plugins."io.containerd.grpc.v1.cri".x509_key_pair_streaming]
      tls_cert_file = ""
      tls_key_file = ""
  [plugins."io.containerd.internal.v1.opt"]
    path = "/opt/containerd"
  [plugins."io.containerd.internal.v1.restart"]
    interval = "10s"
  [plugins."io.containerd.metadata.v1.bolt"]
    content_sharing_policy = "shared"
  [plugins."io.containerd.monitor.v1.cgroups"]
    no_prometheus = false
  [plugins."io.containerd.runtime.v1.linux"]
    shim = "containerd-shim"
    runtime = "runc"
    runtime_root = ""
    no_shim = false
    shim_debug = false
  [plugins."io.containerd.runtime.v2.task"]
    platforms = ["linux/arm64/v8"]
  [plugins."io.containerd.service.v1.diff-service"]
    default = ["walking"]
  [plugins."io.containerd.snapshotter.v1.devmapper"]
    root_path = ""
    pool_name = ""
    base_image_size = ""
    async_remove = false
EOF
}

# since ubuntu 21.10 ships with 248.3 and 
function cilium-sysctl() {
#  https://github.com/cilium/cilium/issues/10645
  echo 'net.ipv4.conf.lxc*.rp_filter = 0' > /etc/sysctl.d/99-override_cilium_rp_filter.conf
}
k8s-modules
containerd-modules
apt-get update
apt-get install -y openssh-server ca-certificates curl lsb-release wget gnupg sudo lm-sensors perl htop crudini bat \
  apt-transport-https nftables linux-modules-extra-raspi
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
apt-get update
apt-get install -y containerd.io
configure-containerd
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
apt-get remove -y unattended-upgrades snapd
cloud-init-fix
curl https://baltocdn.com/helm/signing.asc | apt-key add -
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | tee /etc/apt/sources.list.d/helm-stable-debian.list
apt-get update
sudo apt-get install helm -y
cilium-sysctl
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-arm64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-arm64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-arm64.tar.gz /usr/local/bin
rm cilium-linux-arm64.tar.gz{,.sha256sum}
```

## Breadcrumbs to Get Cilium Working

## Install Kubernetes

[packer-site]: https://www.packer.io/

[packer-builder-arm]: https://github.com/mkaczanowski/packer-builder-arm

[custom-image-post]: https://disconnected.systems/blog/raspberry-pi-archlinuxarm-setup/

[losetup-man-page]: https://manpages.ubuntu.com/manpages/impish/en/man8/losetup.8.html

[parted-man-page]: https://manpages.ubuntu.com/manpages/impish/en/man8/parted.8.html

[image-build-script]: https://github.com/LadySerena/tf-platform/blob/0caefa59da44cc664793d795364b2db740ad612e/scripts/ubuntu-21-10.bash