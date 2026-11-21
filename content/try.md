+++
title = "Try it in five minutes"
description = "Run stafeto from a release archive on your own machine, play Doom, or launch the public image on AWS."
template = "page.html"
+++

stafeto 0.6.0 is experimental. Keep nothing secret on it. There are two ways to try it: a release archive that runs in QEMU on your machine, and a public image for AWS EC2 Graviton.

## On your machine

You need QEMU. On macOS, `brew install qemu` installs it.

1. Download one of two archives from the [0.6.0 release](https://github.com/stafeto/stafeto/releases/tag/v0.6.0).
2. Unpack it and run `./run.sh`.

| Archive | What it holds |
|---|---|
| `stafeto-0.6.0-aarch64-virt-base.tar.xz` | The shell, the package tool and the base packages. |
| `stafeto-0.6.0-aarch64-virt-full.tar.xz` | Every package, including the network programs and Doom. It needs 1024M of memory, which its `run.sh` passes. |

```sh
tar xf stafeto-0.6.0-aarch64-virt-full.tar.xz
cd stafeto-0.6.0-aarch64-virt-full
./run.sh              # QEMU with emulation, any host
./run.sh --cpus 4     # four CPUs
./run.sh --hvf        # hardware virtualization on a Mac with Apple silicon
```

You get a BusyBox `ash` shell with line editing. `bash`, `nano`, `htop`, `top` and `ncdu` are in the image. `htop` needs `TERM=vt100`. Type `exit` or press Ctrl-A X to end QEMU. Files live in RAM, so changes are gone after a restart.

### Play Doom

The full image holds Doom and the Freedoom data. `run.sh` opens a QEMU window for the screen and keeps the console in the terminal. At the `ash` prompt type:

```sh
doom -iwad /games/freedoom1.wad
```

The game opens in the window, which takes the keys while it is in focus (click the window). Arrows move and turn, Ctrl fires, Space opens doors, Shift runs, Alt strafes, and Esc opens the menu. There is no sound, network or saved game. On a Mac with Apple silicon, add `--hvf` for a much faster run.

## On AWS

The public image of 0.6.0 is the AMI `ami-04d65173ce457da3c` in `eu-central-1` (name `stafeto-0.6.0`, arm64, UEFI). It holds:

- nginx on ports 80 and 443 (TLS 1.3, HTTP/2). The certificate is self-signed and made at every boot, so use `curl -k`.
- `nginx-fast` on port 8080, which runs on a user-space FreeBSD network stack (F-Stack) over the ENA device.
- Dropbear SSH on port 22. The boot script reads the public key of the key pair of the instance and logs in `root` with it. An instance launched with no key pair has no SSH.
- `pkg`, the package tool, with the index of the series 0.6: `pkg update`, `pkg search` and `pkg install htop` work on the instance. The package store lives in memory, so installed packages are gone after a reboot.
- A landing page on the three web ports that says what stafeto is and what to try.

To launch it:

1. Use the region `eu-central-1` and the instance type `t4g.small`, the only type tested. The image boots in UEFI mode and needs the ENA network device, which every t4g instance has.
2. Give the instance a key pair and a public IPv4 address.
3. Use a security group whose inbound rules allow TCP 22, 80, 443 and 8080 from your own address (a /32) and nothing else.
4. Keep the second volume that the image maps as `/dev/sdf` (see below).
5. Log in with `ssh root@ADDRESS` and the private key of the pair, or open `http://ADDRESS/`.
6. Read the console with the EC2 system log. The lines `ena: link up`, `net: lease <address>`, `web: nginx on 80 and 443`, `fast: nginx-fast on 8080` and `ssh: host key SHA256:...` show that the services are up. The SSH fingerprint on the console is the one to compare at your first login.

With the AWS CLI the launch looks like this (replace the placeholders):

```sh
aws ec2 run-instances \
  --region eu-central-1 \
  --image-id ami-04d65173ce457da3c \
  --instance-type t4g.small \
  --key-name YOUR-KEY-PAIR \
  --security-group-ids sg-XXXXXXXX \
  --associate-public-ip-address
```

The AMI id comes from the notes of 0.6.0, and the steps above are those of the notes of 0.5.1.

### The data volume

The image maps a second volume, `/dev/sdf`, 2 GiB of gp3, made at each launch from a snapshot of the image. It holds a littlefs file system. The SSH host key made at the first boot is saved there and put back at every later boot, so the fingerprint stays the same after a stop and a start. POSIX programs see the volume as `/data`, and their files there survive reboots.

- **Cost.** The volume belongs to the account that launches the instance: about 0.19 USD a month in `eu-central-1` (approximate), billed as an EBS volume, and deleted with the instance.
- **Without it.** Launch with the mapping `DeviceName=/dev/sdf,NoDevice=`. The host key is then new at every boot, and the SSH client warns about it.
- **Larger.** Give the launch a mapping of the same device with a larger `VolumeSize`, for example `DeviceName=/dev/sdf,Ebs={VolumeSize=4,VolumeType=gp3,DeleteOnTermination=true}`. The partitions keep their size, so the added space stays unused.

The instance itself is billed to your account at the normal EC2 price of a `t4g.small`. Stop or terminate it when you are done. A stop and a start work. A plain `aws ec2 reboot-instances` is ignored by the image, and a stop turns into a forced stop after a few minutes.

## Where to read more

The release notes of [0.6.0](https://github.com/stafeto/stafeto/releases/tag/v0.6.0) and the repository documentation (`docs/releases/0.5.1.md` for the AWS image, `docs/doom.md` for Doom) have the details and the known limits.
