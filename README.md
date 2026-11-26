# ansible-role-pingtunnel

Deploy pingtunnel on Linux with systemd. Each entry in `pingtunnel_configs`
becomes `/etc/pingtunnel/<name>.json` and `pingtunnel@<name>.service`.

The config is a native YAML dictionary serialized with `to_nice_json` by
`ansible.builtin.copy`. There is no Jinja template for JSON. The only template
is the shared systemd unit, `pingtunnel@.service`.

Install
-------

```sh
ansible-galaxy role install foi.pingtunnel
```

or add it in `requirements.yml`

```
roles:
  - name: foi.pingtunnel
```

and run `ansible-galaxy install -r requirements.yml`

Example
-------

```yaml
# playbook.yml
- hosts: pingtunnel
  become: true
  roles:
    - foi.pingtunnel

# host_vars/server.yml
pingtunnel_configs:
  server:
    type: server
    password: "{{ vault_pingtunnel_password }}"
    nolog: true
    loglevel: warn

# host_vars/client.yml
pingtunnel_configs:
  fin:
    type: client
    password: "{{ vault_pingtunnel_password }}"
    listen: 127.0.0.1:28006
    server: 203.0.113.10
    target: 127.0.0.1:28006
    udp: true
    nolog: true
    loglevel: warn
```

This creates `pingtunnel@server.service` on the server and
`pingtunnel@fin.service` on the client.

For TCP forwarding, omit `udp` or set it to `false`. For SOCKS5 use
`socks5: true`, `listen` and `server`; `target` is then optional. Native config
fields are defined in upstream [config.go](https://github.com/esrrhs/pingtunnel/blob/3.1/config.go).

Role variables
--------------

Defaults:

```yaml
pingtunnel_version: '3.1'
pingtunnel_architecture: "{{ {'x86_64': 'amd64', 'aarch64': 'arm64', 'armv7l': 'arm', 'armv6l': 'arm', 'i386': '386', 'i686': '386'}.get(ansible_facts['architecture'], ansible_facts['architecture']) }}"
pingtunnel_install_url: "https://github.com/esrrhs/pingtunnel/releases/download/{{ pingtunnel_version }}/pingtunnel_linux_{{ pingtunnel_architecture }}.zip"
pingtunnel_tmp_path: /var/cache/pingtunnel
pingtunnel_install_tmp_path: "{{ pingtunnel_tmp_path }}/pingtunnel-{{ pingtunnel_version }}-linux-{{ pingtunnel_architecture }}.zip"
pingtunnel_extract_path: "{{ pingtunnel_tmp_path }}/{{ pingtunnel_version }}-{{ pingtunnel_architecture }}"
pingtunnel_binary_path: /usr/local/bin/pingtunnel
# Official GitHub release asset digests. Override the checksum for custom builds.
pingtunnel_release_checksums:
  '3.1':
    amd64: sha256:1688cbf062f9173c7810292d0bea80c86839dbe0a8afbbe7711d0c2b38888332
    arm64: sha256:acab6a364aa0ef5155dda97e2462b57ec91022fb58843cb5a93f893f70488c30
    arm: sha256:018de24c56197f9e3735ede2d00293e586541e62698201dc0398c9070e08a4fd
    '386': sha256:199f4a172f3c93e7d3b016506dd46fa5d065ab9bf8bea3d35206b297f9077b87
pingtunnel_install_checksum: "{{ pingtunnel_release_checksums.get(pingtunnel_version, {}).get(pingtunnel_architecture, '') }}"
pingtunnel_config_path: /etc/pingtunnel
# Each key is a config basename and a systemd instance: server -> pingtunnel@server.
# Each value is a native pingtunnel 3.1 JSON configuration (use Vault for passwords).
pingtunnel_configs: {}
pingtunnel_user: pingtunnel
pingtunnel_default_unit_options:
  Wants: network-online.target
  After: network-online.target
  StartLimitBurst: 30
  StartLimitIntervalSec: 30
pingtunnel_default_service_options:
  User: "{{ pingtunnel_user }}"
  Restart: always
  TimeoutStopSec: 5s
  LimitNOFILE: 1048576
  ProtectSystem: strict
  ProtectHome: tmpfs
  PrivateTmp: true
  PrivateDevices: true
  ProtectKernelTunables: true
  ProtectKernelModules: true
  ProtectKernelLogs: true
  ProtectControlGroups: true
  MemoryDenyWriteExecute: true
  LockPersonality: true
  RestrictRealtime: true
  ProtectClock: true
  RestartSec: 10
  NoNewPrivileges: true
  AmbientCapabilities: CAP_NET_RAW
  CapabilityBoundingSet: CAP_NET_RAW
```

`pingtunnel_configs` defaults to an empty mapping: the binary and shared unit
are installed, with no instance enabled. `pingtunnel_config_path` changes the
JSON directory. Override `pingtunnel_architecture` or `pingtunnel_install_url`
for custom builds; provide the corresponding `pingtunnel_install_checksum`.
Official v3.1 SHA256 digests are included for Linux amd64, arm64, arm and 386.

Archives and extracted files are cached by version and architecture. Installation
atomically replaces the executable when its content changes, including when an
executable already exists. Binary or unit changes restart all configured
instances; a config-only change restarts only that instance. Unchanged runs also
ensure configured instances are enabled and started.

`pingtunnel_unit_options` and `pingtunnel_service_options` override individual
keys in their respective default dictionaries. The service runs as `pingtunnel`
with `CAP_NET_RAW`; root execution is not required. Configured instances are
enabled and started automatically.

Managing instances
------------------

Deleting a key from `pingtunnel_configs` does not remove an existing JSON file
or disable its instance. To retire one explicitly:

```sh
systemctl disable --now pingtunnel@fin.service
rm /etc/pingtunnel/fin.json
```

Testing
-------

```sh
molecule test
```

License
-------

MIT
