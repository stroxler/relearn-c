# Setting up virtual machines with QEMU

This is an essential part of learning low-level
programming at this point, for a number of reasons:
- First, I'm on an M1 Mac now, but many of the resources
  assume x86 linux
- Second, at least a handful of compiler type books target
  alternative architectures like RISC-V or MIPS
- Third, I'd like the option of going through some OS course
  material, and without a full emulator there's no good way
  to do this. So learning QEMU is good even if I don't really
  need it.
  
  
# My first working setup on an M1: Alpine linux

First step is to install `qemu`, I did it with
`nix-env -iA nixpkgs.qemu`.

Then, we need to create our image by:
```bash
# Creating a .qemu-data folder
mkdir -p ~/.qemu-data

# Downloading an .iso boot image
curl https://dl-cdn.alpinelinux.org/alpine/v3.18/releases/x86_64/alpine-virt-3.18.0-x86_64.iso -o ~/.qemu-data/alpine-virt-3.18.0-x86_64.iso

# Creating a disk image for qemu
qemu-img create -f qcow2 ~/.qemu-data/alpine.qcow2 8G

# Booting from the iso pointed at the disk, which initializes the disk
qemu-system-x86_64 \
  -m 512 -nic user -nographic \
  -boot d -cdrom alpine-virt-3.18.0-x86_64.iso \
  -drive file=alpine.qcow2,media=disk

# ^ In this boot session we have to install alpine by:
#
# - logging in as root (no password)
# - running the setup-alpine command
# - for the most part, just using the defaults
# - set the password for root (it doesn't need to be secure)
# - use mirror 18 (cmu)
# - set up a user stroxler
# - install the system as a 'sys' image to `sda`
```

Next, verify that we can boot normally by running:
```
qemu-system-x86_64 \
  -m 512 -nic user -nographic \
  -drive file=alpine.qcow2,media=disk \
  -net nic,model=virtio \
  -net user,hostfwd=tcp::10022-:22
```

Once in there, bootstrap `nix` for the `stroxler` user:
```bash
apk add curl xz bash

# Install single-user nix for stroxler
mkdir -m 0755 /nix && chown stroxler /nix # (as root!)
su stroxler
sh <(curl -L https://nixos.org/nix/install) --no-daemon
```

and also make sure that ssh works by running adding these lines
to `/etc/ssh/sshd_config` (use `vi`, the only editor in alpine):
```bash
PermitRootLogin yes
PermitEmptyPasswords yes
```
and restart with `/etc/init.d/sshd restart`.

Then, still as `stroxler`, run
```
mkdir ~/.ssh
echo '<key>' > ~/.ssh/authorized_keys
```
where `<key>` is the contents of your host machine's
`~/.ssh/id_rsa.pub`.

If the `stroxler` user had issues, you may need to modify
`/etc/shadow`; in my case I accidentally created a locked user
with a password of `!` which I had to change to `*`.

Now test logging in:
```
ssh localhost -p 10022
```
This should work without a password!

If you need to debug you can log in with a password as root:
```
ssh root@localhost -p 10022
```


