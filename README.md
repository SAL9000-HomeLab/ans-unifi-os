# ans-unifi-os

Installs and upgrades [UniFi OS Server](https://blog.ui.com/article/introducing-unifi-os-server), Ubiquiti's
self-hosted UniFi platform (the Network application, Identity Hub, Site Magic and the rest of UniFi OS), on an Ubuntu
VM with Ubiquiti's official installer. It replaces the old self-hosted UniFi Network controller.

| Role | What it does |
| --- | --- |
| [unifi_os_server](roles/unifi_os_server) | Installs the installer's dependencies (Podman, rootless networking), downloads the pinned installer and checks its SHA-256, runs it, and makes sure the services run and the web UI answers |

## How UniFi OS Server runs

The installer sets up UniFi OS as a rootless Podman container, `uosserver`, owned by the system user `uosserver`, and
two systemd services: `uosserver` (the container) and `uosserver-updater` (applies updates started from the UI, as
root). Everything UniFi OS keeps (MongoDB, the Network application's data, its own backups, certificates) is in
Podman volumes under `/home/uosserver`; the installer's settings are in `/var/lib/uosserver/server.conf`. The CLI is
`uosserver` (`uosserver version`, `status`, `stop`, `start`, `shell`).

It publishes these ports on the VM:

| Port | Protocol | Used for |
| --- | --- | --- |
| 11443 | TCP | Web UI and API (UniFi OS's 443; `unifi_os_server_web_port`) |
| 8080 | TCP | Device inform (adoption and management) |
| 3478 | UDP | STUN |
| 10003 | UDP | Device discovery (layer 2: devices on the same network find it without being told) |
| 8444, 8880–8882 | TCP | Hotspot portal |
| 6789 | TCP | Mobile speed test |
| 5514 | UDP | Remote syslog from devices |
| 5671, 9543, 28082 | TCP | AMQPS, Identity Hub, and other UniFi OS services |

## Requirements

- An Ubuntu 24.04 (or later) x86_64 VM, reached over SSH by a user with sudo. 4 GB of memory and 2 cores at least,
  and 64 GB of disk: the container image is about 3 GB, and MongoDB and backups grow with the number of devices.
- For devices to find it on their own, put the VM on the devices' network (layer 2 discovery on UDP 10003);
  otherwise point them at it (see [Adopting devices](#adopting-devices)).
- Outbound HTTPS from the VM to `fw-download.ubnt.com` (the installer).
- ansible-core 2.16 or later; the role uses `ansible.builtin` only.

## Inventory

[inventory/hosts.yml](inventory/hosts.yml) shows the shape: one group, `unifi_os_server`, which
[site.yml](site.yml) targets with `become: true`.

## Variables

All in [roles/unifi_os_server/defaults/main.yml](roles/unifi_os_server/defaults/main.yml):

| Variable | Default | Purpose |
| --- | --- | --- |
| `unifi_os_server_version` | `5.1.42` | The release to install. A host with an older one is upgraded; a newer one (updated from the UI) is left alone. |
| `unifi_os_server_installer_url` | 5.1.42's linux-x64 installer | Change with the version (see [Upgrading](#upgrading)). |
| `unifi_os_server_installer_sha256` | 5.1.42's checksum | Checked before the installer runs. |
| `unifi_os_server_web_port` | `11443` | Web UI port, set when the role installs or upgrades. |
| `unifi_os_server_packages` | Podman, uidmap, passt, slirp4netns, iptables, ca-certificates | The installer's dependencies. |
| `unifi_os_server_download_dir` | `/var/cache/unifi-os-server` | Where the installer (about 800 MB) is downloaded; deleted after installing. |
| `unifi_os_server_wait_timeout` | `900` | Seconds to wait for the web UI to answer. |

## Deploying

1. **Create the VM** with proxmox-deploy, from the Ubuntu 24.04 template, on the devices' network:

   ```yaml
   provision_vms:
     - name: uos-01
       template_vmid: 9004                # the vm-templates Ubuntu 24.04 template
       net_bridge: vnet10                 # the devices' network
       cores: 2
       memory: 4096
       disk_size: 64G
       tags: [unifi]
   ```

2. **Add an AWX/Ascender job template** for this repository's `site.yml`, with an inventory that has the VM in the
   group `unifi_os_server` and the machine credential proxmox-deploy's VMs accept, and run it. The first run takes 10
   to 15 minutes (the download and UniFi OS's first start).
3. **Set it up** at `https://<vm>:11443` with UniFi OS's setup wizard (it names the console and creates the owner
   account; signing in with a UI account adds remote access through unifi.ui.com). Keep the credentials in your
   password manager.
4. **Adopt the devices** (below).

## Adopting devices

- **On the same network as the VM:** new or factory-reset devices show up under **UniFi Devices** as ready to adopt.
- **On another network,** or still adopted by an old controller: factory-reset the device, then SSH to it
  (`ubnt`/`ubnt` after a reset) and point it at the VM: `set-inform http://<vm>:8080/inform`. It appears as pending
  adoption; adopt it in the UI. Run `set-inform` again if it doesn't appear within a minute. A DNS record `unifi`
  in the devices' search domain, pointing at the VM, does the same for new devices without SSH.
- **From an old UniFi Network controller:** its backup (`.unf`) can be restored in the Network application (**Settings
  → System → Backups**), then the devices are pointed at the new server as above.

## Upgrading

Ubiquiti publishes releases through its firmware API. The latest release-channel version, installer URL and SHA-256:

```shell
curl -fsSL "https://fw-update.ubnt.com/api/firmware-latest?filter=eq~~product~~unifi-os-server&filter=eq~~platform~~linux-x64&filter=eq~~channel~~release" |
  jq -r '._embedded.firmware[0] | .version, ._links.data.href, .sha256_checksum'
```

Read the release notes, set the three variables (in a pull request, or as job extra vars), and run the job: the
installer upgrades in place and keeps the data. Take a VM snapshot first.

Updating from the UniFi OS UI works too (the `uosserver-updater` service applies it); the role then reports the newer
version and leaves it. Pin the new version afterwards so a rebuilt VM gets the same one.

## Backups

The Network application makes its own backups (**Settings → System → Backups**, scheduled automatic backups), kept
under `/home/uosserver` on the VM. Download one now and then, or back up the whole VM (Proxmox Backup Server or
snapshots): the installer's data is all on its disk.

## Troubleshooting

```shell
sudo uosserver status
sudo systemctl status uosserver uosserver-updater
sudo -u uosserver podman logs --tail 50 uosserver
```

- **"After the installer, UniFi OS Server reports '' instead of …":** the installer failed (it exits 0 anyway); the
  message ends with its last lines. Re-run the job; the installer replaces a half-done install.
- **The web UI doesn't answer within the timeout:** the first start can take several minutes on a small VM. Check
  `sudo uosserver status` and the container's log; raise `unifi_os_server_wait_timeout`.
- **`Failed to open() /dev/net/tun`:** the rootless network (pasta) needs `/dev/net/tun`, which a VM has. It fails in
  some containers and LXC; use a VM.
- **Devices don't show up for adoption:** they're on another network, or still adopted elsewhere; see
  [Adopting devices](#adopting-devices). UDP 10003 and TCP 8080 must reach the VM.

## Development and CI

**Ansible CI** (yamllint, syntax check, ansible-lint) and **Linting Validation** (markdownlint, linkspector, yamllint)
run on pull requests.

```shell
yamllint -f parsable .
ansible-playbook -i inventory/hosts.yml --syntax-check site.yml
ansible-lint
npx markdownlint-cli2 "**/*.md"
```

Add a line under `## [Unreleased]` in [CHANGELOG.md](CHANGELOG.md) with each change.
