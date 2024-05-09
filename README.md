# PeridioNerves

A sample repository demonstrating integration of the Peridio Daemon into a Nerves based Linux distribution.

## Peridio Integration

This is a sample nerves application that was generated uisng `mix nerves.new peridio_nerves`. The following additions were made to integrate the Peridio Daemon:

### Adding the dependency

Add the `peridiod` dependency to the mix file.

```elixir
{:peridiod, github: "peridio/peridiod", targets: @all_targets},
```

### Modifying the configuration

Add a file `config/peridio.conf` with the following:

```text
uboot_setenv(uboot-env, "peridio_certificate", "\${PERIDIO_CERTIFICATE}")
uboot_setenv(uboot-env, "peridio_private_key", "\${PERIDIO_PRIVATE_KEY}")
```

We will configure the build tooling to use this file when building the `fw` file from `mix firmware`. This file will allow us to insert the certificate
and private key that Peridio will use for the device identity into the U-Boot Environment. For production environments, it is advised to configure `peridiod`
to leverage an HSM to protect the device root identity.

Peridio Daemon will look for its device configuration from the filesystem. Lets add it to the rootfs overlay:

`rootfs_overlay/etc/peridiod/peridio.json`

```text
{
    "version": 1,
    "device_api": {
      "certificate_path": "/etc/peridiod/peridio-cert.pem",
      "url": "device.cremini.peridio.com",
      "verify": true
    },
    "fwup": {
      "devpath": "/dev/mmcblk0",
      "public_keys": []
    },
    "remote_shell": true,
    "node": {
      "key_pair_source": "uboot-env",
      "key_pair_config": {
        "private_key": "peridio_key",
        "certificate": "peridio_certificate"
      }
    }
  }
```

Reference the [Peridiod README.md](https://github.com/peridio/peridiod/blob/main/README.md) for more information on the key/value pairs you may need to modify in this file.

Add the certificate for the Peridio Cloud device endpoints:

`rootfs_overlay/etc/peridiod/peridio-cert.pem`

```text
-----BEGIN CERTIFICATE-----
MIICFzCCAb6gAwIBAgIJAK5kbGWgO+NmMAoGCCqGSM49BAMCMDMxEDAOBgNVBAoM
B1BlcmlkaW8xHzAdBgNVBAMMFlBlcmlkaW8gU2VydmVyIFJvb3QgQ0EwHhcNMjEx
MDIwMTEwMDAwWhcNMjYxMDIwMTIwMDAwWjAyMRAwDgYDVQQKDAdQZXJpZGlvMR4w
HAYDVQQDDBVQZXJpZGlvIERldmljZSBTZXJ2ZXIwWTATBgcqhkjOPQIBBggqhkjO
PQMBBwNCAAR6zwrEZYv4tR0Siq/Eh+e6Ahm0dgyvGAXy/q8BBjMfWbjjxjq0+NHZ
LCZLYTIY/Sz7aEcTD5oeOU3x0AhVGubuo4G7MIG4MAkGA1UdEwQCMAAwDgYDVR0P
AQH/BAQDAgWgMB0GA1UdJQQWMBQGCCsGAQUFBwMBBggrBgEFBQcDAjAdBgNVHQ4E
FgQUwz4hX7OpACnyAS1ddaQpUeqjqHYwHwYDVR0jBBgwFoAUw9oVi83LmK4JugYl
nHIH1DsQOQ0wPAYDVR0RBDUwM4IaZGV2aWNlLmNyZW1pbmkucGVyaWRpby5jb22C
FWRldmljZS5uZXJ2ZXMtaHViLm9yZzAKBggqhkjOPQQDAgNHADBEAiAEVbNmbHri
FcbjrNC3xfGmZ7JZ0t7MELbJaY5iq8CfkgIgC4LpKJjpyRylxkBtJOSpxtuUZ6qG
6LBQU3kzA1I8bnI=
-----END CERTIFICATE-----

```

Add the following keys to the `config.exs` file:

```elixir
config :nerves, :firmware,
  rootfs_overlay: "rootfs_overlay",
  provisioning: "config/peridio.conf"

config :nerves, :erlinit,
  env: "PERIDIO_CONFIG_FILE=/etc/peridiod/peridio.json"
```

You should now be ready to create firmware for your device. Set `MIX_TARGET` and run `mix firmware`

### Creating your PKI

You will need to create your own signing certificate chain to sign and create device identities for use with Peridio Cloud.
It is recommended that you follow the docs for [Creating X.509 Certificates with OpenSSL](https://docs.peridio.com/platform/guides/creating-x509-certificates-with-openssl)

On the last step when you create a certificate for an end entity, its recommended that you set the certificate common name to something you would like to use.
In the example, we configure the device to `CN=unique-end-entity-name`.

When you are finished and you have end entity private key and certificate pem files, you can use the following command to burn them to an inserted SD Card:

```bash
PERIDIO_CERTIFICATE=$(cat /path/to/device-cert.pem) PERIDIO_PRIVATE_KEY=$(cat /path/to/device-private-key.pem) fwup /path/to/app.fw
```

The path to the `fw` file will be printed to the console after running `mix firmware`.

### Using with Peridio Cloud

If you would like to use a device provisioned directly from this repository with your peridio account you will need to perform the following:

* [Create a new product](https://docs.peridio.com/cli/products-v2/create) named `peridio_nerves`.
* [Create a new device](https://docs.peridio.com/cli/devices/create) using the with the identifier from the previous section on configuring PKI.
* [Create a device certificate](https://docs.peridio.com/cli/device-certificates/create) for the device.

Your device should now be able to connect to Peridio.

### Updating the device

When you are ready to update the device, you can create a new firmware and upload it to peridio using the [create firmware](https://docs.peridio.com/cli/firmwares/create) command
in the CLI. You do not need to set `PERIDIO_CERTIFICATE` or `PERIDIO_PRIVATE_KEY` during this time as those values are persisted into the U-Boot environment and are not overwritten
on a device update, they are only needed the first time provisioning a device.

A device can be updated by [creating a deployment](https://docs.peridio.com/cli/deployments/create) and configuring it to distribute the new firmware to your device by matching their tags.
