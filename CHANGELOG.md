# Changelog

All notable changes to this project are documented here. Releases are cut by
pushing a `vX.Y.Z` tag; the release workflow publishes the matching section.

## [Unreleased]

- Added: role `unifi_os_server`: installs UniFi OS Server 5.1.42 on Ubuntu 24.04 with Ubiquiti's installer (Podman
  and rootless networking, checksum-verified download, non-interactive install, version check), upgrades an older
  install, leaves a newer one, and waits for the web UI; `site.yml` for the group `unifi_os_server`.
