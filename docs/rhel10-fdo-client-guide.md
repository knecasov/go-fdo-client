# Installing and configuring the FIDO Device Onboard (FDO) client on RHEL 10

> **Important:** FDO is only supported on a RHEL 10 system in image mode (`bootc`).

## Overview

FIDO Device Onboard (FDO) enables zero-touch device onboarding and secure device identity management for edge deployments. The `go-fdo-client` is a client implementation of the FDO specification in Go. It provides the following commands:

- **`device-init`** — provision the initial device credential into a newly manufactured device (DI protocol)
- **`onboard`** — device queries the rendezvous server for owner contact information (TO1 protocol), then the device and owner authenticate each other and transfer credentials (TO2 protocol)
- **`print`** — print device credential from blob or TPM
- **`help`** — help about any command

> **Note:** The commands below use example values. Replace them with values appropriate for your environment. For all available options, run `go-fdo-client <command> --help`.

## Prerequisites

- RHEL 10 system in image mode (`bootc`)
- FDO servers deployed and accessible (manufacturer, rendezvous, owner). See the [FDO server documentation](TBD) for server deployment.

## Installing the FDO client

In image mode, the operating system is immutable, so packages must be installed via the `Containerfile` used to build the container image:

```dockerfile
FROM registry.redhat.io/rhel10/rhel-bootc:10.2

RUN dnf install -y go-fdo-client && dnf clean all
```

Build the container image:

```console
$ podman build -t quay.io/example/fdo-client-bootc:latest -f Containerfile .
```

Push the container image to the registry:

```console
$ podman push quay.io/example/fdo-client-bootc:latest
```

The image is then referenced in the Kickstart `ostreecontainer` directive to [deploy the image](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_image_mode_for_rhel_to_build_deploy_and_manage_operating_systems/deploying-the-rhel-bootc-images) on the target device.

## Provisioning the device credential

To provision the initial device credential, add the following to your Kickstart file in the `%post` section, after the `bootc switch` command:

```
go-fdo-client --blob /boot/device_credential --debug \
  device-init http://192.168.1.100:8038 \
  --device-info my-device --key ec256
```

> **Note:** Before onboarding can succeed, the ownership voucher (OV) must be transferred from the manufacturer to the owner server, and the owner server must upload the OV to the rendezvous server (TO0 protocol). These are server-side operations. See the [FDO server documentation](TBD) for details.

## Onboarding the device

After the Kickstart installation completes and the device reboots, run the onboarding command:

```console
# go-fdo-client --blob /boot/device_credential onboard \
    --key ec256 --kex ECDH256 --debug
```

On success, the output contains:

```
FIDO Device Onboard Complete
```

> **Note:** The `onboard` command implements an infinite retry loop — it continues attempting TO1 and TO2 protocols until successful or manually interrupted (`Ctrl+C`).
