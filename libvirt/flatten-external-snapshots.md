# Flattening a Libvirt/QEMU VM with External Snapshots

## Why I wrote this

I use a lot of VMs, and I take a lot of snapshots. Over time, some of my VMs end up with long external snapshot chains, which makes the VM storage messy and harder to manage.

This guide is my workflow for flattening a VM’s current state into a single clean `.qcow2` image.

The goal is simple: keep the VM exactly as it is now, but remove the huge backing chain so the VM boots from one normal disk image again.

This is especially useful if a single VM has started to look like this:


```text
debian-dev.qcow2
debian-dev.snapshot1
debian-dev.snapshot2
debian-dev.snapshot3
debian-dev.snapshot4
debian-dev.snapshot5
debian-dev.snapshot6
debian-dev.snapshot7
debian-dev.snapshot8
debian-dev.snapshot9
```

And you want to turn it into something clean like this:

```text
debian-dev-flattened.qcow2
```

## Tested on

Example VM used in this guide:

```text
debian-dev
```

This guide assumes you are using:

* QEMU/KVM
* libvirt
* `virsh`
* `qemu-img`
* virt-manager, optionally

**If you are using virt-manager on Linux, this is likely your stack.**

---

## Before you start

Shut down the VM first.

```bash
sudo virsh shutdown debian-dev
```

Make sure it's shut down.

```bash
sudo virsh domstate debian-dev
```

You want it to output:

```text
shut off
```

---

## Check the VM's current active disk

To check the active disk image your VM is currently using:

```bash
sudo virsh domblklist debian-dev --details
```

The `vda` source is the active disk image. If your VM is currently using an external snapshot, this should be the top snapshot you flatten from.

---

## Check the current backing chain

In this example, the active top snapshot is:

```text
/var/lib/libvirt/images/debian-dev.snapshot9
```

Check its backing chain:

```bash
sudo qemu-img info --backing-chain /var/lib/libvirt/images/debian-dev.snapshot9
```

You should see the chain of backing files listed.

That means the current VM state is not contained in one file. It depends on the whole stack.

---

## Flatten the VM disk

Now convert the top snapshot into one standalone `.qcow2` image:

```bash
sudo qemu-img convert -p \
  -f qcow2 \
  -O qcow2 \
  /var/lib/libvirt/images/debian-dev.snapshot9 \
  /var/lib/libvirt/images/debian-dev-flattened.qcow2
```
*Replace the file paths with your correct corresponding top snapshot and the directory/name of the new disk image you want to create*

This creates a new disk image:

```text
/var/lib/libvirt/images/debian-dev-flattened.qcow2
```

That new image should contain the current state of the VM without needing the old backing chain.

---

## Verify the new flattened image

Check the backing chain of the new image:

```bash
sudo qemu-img info --backing-chain /var/lib/libvirt/images/debian-dev-flattened.qcow2
```

You want to see no backing chain.

In other words, you should **not** see anything like this:

```text
Backing file: ...
```

If there is no backing file listed, the image is flattened.

---

## Point the VM at the new flattened disk

Now edit the VM definition:

```bash
sudo EDITOR=nvim virsh edit debian-dev
```

Find the disk section that points to the old top snapshot:

```xml
<disk type='file' device='disk'>
  ...
  <source file='/var/lib/libvirt/images/debian-dev.snapshot9'/>
  <target dev='vda' bus='virtio'/>
  ...
</disk>
```

Change it so it points to the new flattened image:

```xml
<disk type='file' device='disk'>
  ...
  <source file='/var/lib/libvirt/images/debian-dev-flattened.qcow2'/>
  <target dev='vda' bus='virtio'/>
  ...
</disk>
```

Save and exit.

---

## Test the VM

Start the VM normally.

It should boot from the new flattened disk image and behave exactly like it did before.

Do not clean up old snapshot files until you are confident the flattened image works.

---

## Delete the old snapshot disk files

Once the VM boots correctly from the new flattened disk image, the old backing-chain files are no longer needed.

Shut the VM down again before deleting the old disk files:

```bash
sudo virsh shutdown debian-dev
```

Make sure it is shut down:

```bash
sudo virsh domstate debian-dev
```

You want it to output:

```text
shut off
```

First, confirm the VM is pointed at the flattened image:

```bash
sudo virsh domblklist debian-dev --details
```

You should see the disk source pointing to:

```text
/var/lib/libvirt/images/debian-dev-flattened.qcow2
```

Now list the old snapshot files:

```bash
sudo bash -c 'ls -lh /var/lib/libvirt/images/debian-dev.snapshot*'
```

If that only shows the old snapshot files you want to remove, delete them:

```bash
sudo bash -c 'rm -Iv /var/lib/libvirt/images/debian-dev.snapshot*'
```

Enter 'y' to proceed with deletion.

Now remove the old original/base disk image separately:

```bash
sudo rm -Iv /var/lib/libvirt/images/debian-dev.qcow2
```

After that, check what remains:

```bash
sudo ls -lh /var/lib/libvirt/images/ | grep -i "debian-dev"
```

You should now only see the new flattened disk image:

```text
debian-dev-flattened.qcow2
```

---

## Cleanup: remove stale snapshot metadata from libvirt/virt-manager

At this point, the VM may work fine, but virt-manager might still show the old snapshots.

That is because libvirt can still have snapshot metadata registered, even though you have manually moved the VM to a new flattened disk.

Check the snapshot tree:

```bash
sudo virsh snapshot-list debian-dev --tree
```

You might see something like this:

```text
snapshot1
  |
  +- snapshot2
      |
      +- snapshot3
          |
          +- snapshot4
              |
              +- snapshot5
                  |
                  +- snapshot6
                      |
                      +- snapshot7
                          |
                          +- snapshot8
                              |
                              +- snapshot9
```

Not clean.

We fix this by deleting the youngest child snapshot metadata first, then moving upward:

```text
snapshot9 -> snapshot8 -> snapshot7 -> ...
```

This matters because libvirt snapshots are parent/child metadata. If you delete them in the wrong order, libvirt may complain because the parent still has children.

## Delete the snapshot metadata backward

This command lists the snapshot names, reverses the order with `tac`, then deletes the metadata entries one by one:

```bash
sudo virsh snapshot-list --name debian-dev | sed '/^$/d' | tac | while read -r snapshot; do
  sudo virsh snapshot-delete debian-dev "$snapshot" --metadata
done
```

---

## Restart virt-manager

Fully close virt-manager so the GUI refreshes:

```bash
pkill virt-manager
```

Then check the snapshot tree again:

```bash
sudo virsh snapshot-list debian-dev --tree
```

It should now return an empty line.

Open virt-manager again, and the stale snapshots should be gone from the GUI.

You can now start making snapshots through the GUI or CLI again on a clean slate.
