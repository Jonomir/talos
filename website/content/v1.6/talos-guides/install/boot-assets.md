---
title: "Boot Assets"
description: "Creating customized Talos boot assets, disk images, ISO and installer images."
---

Talos Linux provides a set of pre-built images on the [release page](https://github.com/siderolabs/talos/releases/{{< release >}}), but these images
can be customized further for a specific use case:

* adding [system extensions]({{< relref "../configuration/system-extensions" >}})
* updating [kernel command line arguments]({{< relref "../../reference/kernel" >}})
* using custom `META` contents, e.g. for [metal network configuration]({{< relref "../../advanced/metal-network-configuration" >}})
* generating [SecureBoot]({{< relref "../install/bare-metal-platforms/secureboot" >}}) images signed with a custom key

There are two ways to generate Talos boot assets:

* using [Image Factory]({{< relref "#image-factory" >}}) service (recommended)
* manually using [imager]({{< relref "#imager" >}}) container image (advanced)

Image Factory is easier to use, but it only produces images for official Talos Linux releases and official Talos Linux system extensions.
The `imager` container can be used to generate images from `main` branch, with local changes, or with custom system extensions.

## Image Factory

[Image Factory]({{< relref "../../learn-more/image-factory" >}}) is a service that generates Talos boot assets on-demand.
Image Factory allows to generate boot assets for the official Talos Linux releases and official Talos Linux system extensions.

The main concept of the Image Factory is a *schematic* which defines the customization of the boot asset.
Once the schematic is configured, Image Factory can be used to pull various Talos Linux images, ISOs, installer images, PXE booting bare-metal machines across different architectures,
versions of Talos and platforms.

Sidero Labs maintains a public Image Factory instance at [https://factory.talos.dev](https://factory.talos.dev).
Image Factory provides a simple [UI](https://factory.talos.dev) to prepare schematics and retrieve asset links.

### Example: Bare-metal with Image Factory

Let's assume we want to boot Talos on a bare-metal machine with Intel CPU and add a `gvisor` container runtime to the image.
Also we want to disable predictable network interface names with `net.ifnames=0` kernel argument.

First, let's create the schematic file `bare-metal.yaml`:

```yaml
# bare-metal.yaml
customization:
  extraKernelArgs:
    - net.ifnames=0
  systemExtensions:
    officialExtensions:
      - siderolabs/gvisor
      - siderolabs/intel-ucode
```

> The schematic doesn't contain system extension versions, Image Factory will pick the correct version matching Talos Linux release.

And now we can upload the schematic to the Image Factory to retrieve its ID:

```shell
$ curl -X POST --data-binary @bare-metal.yaml https://factory.talos.dev/schematics
{"id":"b8e8fbbe1b520989e6c52c8dc8303070cb42095997e76e812fa8892393e1d176"}
```

The returned schematic ID `b8e8fbbe1b520989e6c52c8dc8303070cb42095997e76e812fa8892393e1d176` we will use to generate the boot assets.

> The schematic ID is based on the schematic contents, so uploading the same schematic will return the same ID.

Now we have two options to boot our bare-metal machine:

* using ISO image: https://factory.talos.dev/image/b8e8fbbe1b520989e6c52c8dc8303070cb42095997e76e812fa8892393e1d176/{{< release >}}/metal-amd64.iso (download it and burn to a CD/DVD or USB stick)
* PXE booting via iPXE script:  https://factory.talos.dev/pxe/b8e8fbbe1b520989e6c52c8dc8303070cb42095997e76e812fa8892393e1d176/{{< release >}}/metal-amd64

> The Image Factory URL contains both schematic ID and Talos version, and both can be changed to generate different boot assets.

Once the bare-metal machine is booted up for the first time, it will require Talos Linux `installer` image to be installed on the disk.
The `installer` image will be produced by the Image Factory as well:

```yaml
# Talos machine configuration patch
machine:
  install:
    image: factory.talos.dev/installer/b8e8fbbe1b520989e6c52c8dc8303070cb42095997e76e812fa8892393e1d176:{{< release >}}
```

Once installed, the machine can be upgraded to a new version of Talos by referencing new installer image:

```shell
talosctl upgrade --image factory.talos.dev/installer/b8e8fbbe1b520989e6c52c8dc8303070cb42095997e76e812fa8892393e1d176:<new_version>
```

Same way upgrade process can be used to transition to a new set of system extensions: generate new schematic with the new set of system extensions, and upgrade the machine to the new schematic ID:

```shell
talosctl upgrade --image factory.talos.dev/installer/<new_schematic_id>:{{< release >}}
```

### Example: AWS with Image Factory

Talos Linux is installed on AWS from a disk image (AWS AMI), so only a single boot asset is required.
Let's assume we want to boot Talos on AWS with `gvisor` container runtime system extension.

First, let's create the schematic file `aws.yaml`:

```yaml
# aws.yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/gvisor
```

And now we can upload the schematic to the Image Factory to retrieve its ID:

```shell
$ curl -X POST --data-binary @aws.yaml https://factory.talos.dev/schematics
{"id":"d9ff89777e246792e7642abd3220a616afb4e49822382e4213a2e528ab826fe5"}
```

The returned schematic ID `d9ff89777e246792e7642abd3220a616afb4e49822382e4213a2e528ab826fe5` we will use to generate the boot assets.

Now we can download the AWS disk image from the Image Factory:

```shell
curl -LO https://factory.talos.dev/image/d9ff89777e246792e7642abd3220a616afb4e49822382e4213a2e528ab826fe5/{{< release >}}/aws-amd64.raw.xz
```

Now the `aws-amd64.raw.xz` file contains the customized Talos AWS disk image which can be uploaded as an AMI to the [AWS]({{< relref "../install/cloud-platforms/aws" >}}).

Once the AWS VM is created from the AMI, it can be upgraded to a different Talos version or a different schematic using `talosctl upgrade`:

```shell
# upgrade to a new Talos version
talosctl upgrade --image factory.talos.dev/installer/d9ff89777e246792e7642abd3220a616afb4e49822382e4213a2e528ab826fe5:<new_version>
# upgrade to a new schematic
talosctl upgrade --image factory.talos.dev/installer/<new_schematic_id>:{{< release >}}
```

## Imager

A custom disk image, boot asset can be generated by using the Talos Linux `imager` container: `ghcr.io/siderolabs/imager:{{<  release >}}`.
The `imager` container image can be checked by [verifying its signature]({{< relref "../../advanced/verifying-images" >}}).

The generation process can be run with a simple `docker run` command:

```shell
docker run --rm -t -v $PWD/_out:/secureboot:ro -v $PWD/_out:/out -v /dev:/dev --privileged ghcr.io/siderolabs/imager:{{< release >}} <image-kind> [optional: customization]
```

A quick guide to the flags used for `docker run`:

* `--rm` flag removes the container after the run (as it's not going to be used anymore)
* `-t` attaches a terminal for colorized output, it can be removed if used in scripts
* `-v $PWD/_out:/secureboot:ro` mounts the SecureBoot keys into the container (can be skipped if not generating SecureBoot image)
* `-v $PWD/_out:/out` mounts the output directory (where the generated image will be placed) into the container
* `-v /dev:/dev --privileged` is required to generate disk images (loop devices are used), but not required for ISOs, installer container images

The `<image-kind>` argument to the `imager` defines the base profile to be used for the image generation.
There are several built-in profiles:

* `iso` builds a Talos ISO image (see [ISO]({{< relref "../install/bare-metal-platforms/iso" >}}))
* `secureboot-iso` builds a Talos ISO image with SecureBoot (see [SecureBoot]({{< relref "../install/bare-metal-platforms/secureboot" >}}))
* `metal` builds a generic disk image for bare-metal machines
* `secureboot-metal` builds a generic disk image for bare-metal machines with SecureBoot
* `secureboot-installer` builds an installer container image with SecureBoot (see [SecureBoot]({{< relref "../install/bare-metal-platforms/secureboot" >}}))
* `aws`, `gcp`, `azure`, etc. builds a disk image for a specific Talos platform

The base profile can be customized with the additional flags to the imager:

* `--arch` specifies the architecture of the image to be generated (default: host architecture)
* `--meta` allows to set initial `META` values
* `--extra-kernel-arg` allows to customize the kernel command line arguments.
  Default kernel arg can be removed by prefixing the argument with a `-`.
  For example `-console` removes all `console=<value>` arguments, whereas `-console=tty0` removes the `console=tty0` default argument.
* `--system-extension-image` allows to install a system extension into the image

### Extension Image Reference

While Image Factory automatically resolves the extension name into a matching container image for a specific version of Talos, `imager` requires the full explicit container image reference.
The `imager` also allows to install custom extensions which are not part of the official Talos Linux system extensions.

To get the official Talos Linux system extension container image reference matching a Talos release, use the [following command](https://github.com/siderolabs/extensions?tab=readme-ov-file#installing-extensions):

```shell
crane export ghcr.io/siderolabs/extensions:{{< release >}} | tar x -O image-digests | grep EXTENSION-NAME
```

> Note: this command is using [crane](https://github.com/google/go-containerregistry/blob/main/cmd/crane/README.md) tool, but any other tool which allows
> to export the image contents can be used.

For each Talos release, the `ghcr.io/siderolabs/extensions:VERSION` image contains a pinned reference to each system extension container image.

### Example: Bare-metal with Imager

Let's assume we want to boot Talos on a bare-metal machine with Intel CPU and add a `gvisor` container runtime to the image.
Also we want to disable predictable network interface names with `net.ifnames=0` kernel argument and replace the Talos default `console` arguments and add a custom `console` arg.

First, let's lookup extension images for Intel CPU microcode updates and `gvisor` container runtime in the [extensions repository](https://github.com/siderolabs/extensions):

```shell
$ crane export ghcr.io/siderolabs/extensions:{{< release >}} | tar x -O image-digests | grep -E 'gvisor|intel-ucode'
ghcr.io/siderolabs/gvisor:20231214.0-{{< release >}}@sha256:548b2b121611424f6b1b6cfb72a1669421ffaf2f1560911c324a546c7cee655e
ghcr.io/siderolabs/intel-ucode:20231114@sha256:ea564094402b12a51045173c7523f276180d16af9c38755a894cf355d72c249d
```

Now we can generate the ISO image with the following command:

```shell
$ docker run --rm -t -v $PWD/_out:/out ghcr.io/siderolabs/imager:{{< release >}} iso --system-extension-image ghcr.io/siderolabs/gvisor:20231214.0-{{< release >}}@sha256:548b2b121611424f6b1b6cfb72a1669421ffaf2f1560911c324a546c7cee655e --system-extension-image ghcr.io/siderolabs/intel-ucode:20231114@sha256:ea564094402b12a51045173c7523f276180d16af9c38755a894cf355d72c249d --extra-kernel-arg net.ifnames=0 --extra-kernel-arg=-console --extra-kernel-arg=console=ttyS1
profile ready:
arch: amd64
platform: metal
secureboot: false
version: {{< release >}}
customization:
  extraKernelArgs:
    - net.ifnames=0
input:
  kernel:
    path: /usr/install/amd64/vmlinuz
  initramfs:
    path: /usr/install/amd64/initramfs.xz
  baseInstaller:
    imageRef: ghcr.io/siderolabs/installer:{{< release >}}
  systemExtensions:
    - imageRef: ghcr.io/siderolabs/gvisor:20231214.0-{{< release >}}@sha256:548b2b121611424f6b1b6cfb72a1669421ffaf2f1560911c324a546c7cee655e
    - imageRef: ghcr.io/siderolabs/intel-ucode:20231114@sha256:ea564094402b12a51045173c7523f276180d16af9c38755a894cf355d72c249d
output:
  kind: iso
  outFormat: raw
initramfs ready
kernel command line: talos.platform=metal console=ttyS1 init_on_alloc=1 slab_nomerge pti=on consoleblank=0 nvme_core.io_timeout=4294967295 printk.devkmsg=on ima_template=ima-ng ima_appraise=fix ima_hash=sha512 net.ifnames=0
ISO ready
output asset path: /out/metal-amd64.iso
```

Now the `_out/metal-amd64.iso` contains the customized Talos ISO image.

If the machine is going to be booted using PXE, we can instead generate kernel and initramfs images:

```shell
docker run --rm -t -v $PWD/_out:/out ghcr.io/siderolabs/imager:{{< release >}} iso --output-kind kernel
docker run --rm -t -v $PWD/_out:/out ghcr.io/siderolabs/imager:{{< release >}} iso --output-kind initramfs --system-extension-image ghcr.io/siderolabs/gvisor:20231214.0-{{< release >}}@sha256:548b2b121611424f6b1b6cfb72a1669421ffaf2f1560911c324a546c7cee655e --system-extension-image ghcr.io/siderolabs/intel-ucode:20231114@sha256:ea564094402b12a51045173c7523f276180d16af9c38755a894cf355d72c249d
```

Now the `_out/kernel-amd64` and `_out/initramfs-amd64` contain the customized Talos kernel and initramfs images.

> Note: the extra kernel args are not used now, as they are set via the PXE boot process, and can't be embedded into the kernel or initramfs.

As the next step, we should generate a custom `installer` image which contains all required system extensions (kernel args can't be specified with the installer image, but they are set in the machine configuration):

```shell
$ docker run --rm -t -v $PWD/_out:/out ghcr.io/siderolabs/imager:{{< release >}} installer --system-extension-image ghcr.io/siderolabs/gvisor:20231214.0-{{< release >}}@sha256:548b2b121611424f6b1b6cfb72a1669421ffaf2f1560911c324a546c7cee655e --system-extension-image ghcr.io/siderolabs/intel-ucode:20231114@sha256:ea564094402b12a51045173c7523f276180d16af9c38755a894cf355d72c249d
...
output asset path: /out/metal-amd64-installer.tar
```

The `installer` container image should be pushed to the container registry:

```shell
crane push _out/metal-amd64-installer.tar ghcr.io/<username></username>/installer:{{< release >}}
```

Now we can use the customized `installer` image to install Talos on the bare-metal machine.

When it's time to upgrade a machine, a new `installer` image can be generated using the new version of `imager`, and updating the system extension images to the matching versions.
The custom `installer` image can now be used to upgrade Talos machine.

### Example: AWS with Imager

Talos is installed on AWS from a disk image (AWS AMI), so only a single boot asset is required.

Let's assume we want to boot Talos on AWS with `gvisor` container runtime system extension.

First, let's lookup extension images for the `gvisor` container runtime in the [extensions repository](https://github.com/siderolabs/extensions):

```shell
$ crane export ghcr.io/siderolabs/extensions:{{< release >}} | tar x -O image-digests | grep gvisor
ghcr.io/siderolabs/gvisor:20231214.0-{{< release >}}@sha256:548b2b121611424f6b1b6cfb72a1669421ffaf2f1560911c324a546c7cee655e
```

Next, let's generate AWS disk image with that system extension:

```shell
$ docker run --rm -t -v $PWD/_out:/out -v /dev:/dev --privileged ghcr.io/siderolabs/imager:{{< release >}} aws --system-extension-image ghcr.io/siderolabs/gvisor:20231214.0-{{< release >}}@sha256:548b2b121611424f6b1b6cfb72a1669421ffaf2f1560911c324a546c7cee655e
...
output asset path: /out/aws-amd64.raw
compression done: /out/aws-amd64.raw.xz
```

Now the `_out/aws-amd64.raw.xz` contains the customized Talos AWS disk image which can be uploaded as an AMI to the [AWS]({{< relref "../install/cloud-platforms/aws" >}}).

If the AWS machine is later going to be upgraded to a new version of Talos (or a new set of system extensions), generate a customized `installer` image following the steps above, and upgrade Talos to that `installer` image.
