# ansible-role-pingtunnel

Ansible role for deploying [pingtunnel server and clients](https://github.com/esrrhs/pingtunnel) as systemd services. Single pingtunnel server can handle multiple pingtunnel clients.

<img src="https://github.com/foi/ansible-role-pingtunnel/actions/workflows/ci.yml/badge.svg?branch=main">

Compatibility
--------------

python: 3.12, 3.13, 3.14
ansible: 12, 13

Install
--------------

`ansible-galaxy role install foi.pingtunnel`

or add it in `requirements.yml`

```
roles:
  - name: foi.pingtunnel
```

and run `ansible-galaxy install -r requirements.yml`

Role Variables
--------------
```yml
pingtunnel_install_url: https://github.com/esrrhs/pingtunnel/releases/download/master-2f75aea1924d47f975aeddf7527302622149b2ea/pingtunnel_linux_amd64.zip
pingtunnel_install_tmp_path: "/tmp/pingtunnel.zip"
pingtunnel_server_service_name: pingtunnel-server
# pingtunnel_server_options:
#   - -key 982
#   - -nolog 1
pingtunnel_server_options: []
pingtunnel_server_install: true
# pingtunnel_clients:
#   pingtunnel-client:
#     options:
#       - -key 982
#       - -l :11000
#       - -s 1.1.1.1
#       - -t 1.1.1.1:11000
#       - -nolog 1
pingtunnel_clients: {}
pingtunnel_user: pingtunnel
pingtunnel_default_unit_options:
  Wants: network-online.target
  After: network-online.target
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
  StartLimitBurst: 30
  StartLimitIntervalSec: 30
  AmbientCapabilities: CAP_NET_RAW
  CapabilityBoundingSet: CAP_NET_RAW
```

Example Playbook
----------------
```yml
# inventory
[pingtunnel]
server
client
# playbook.yml
- hosts: pingtunnel
  roles:
    - role: foi.pingtunnel
  become: true
# host_vars/server.yml
pingtunnel_server_options:
  - "-key 987"
  - -nolog 1
# host_vars/client.yml
pingtunnel_server_install: false
pingtunnel_clients:
  pt-client:
    options:
      - "-key 987"
      - -l :11000
      - -s 1.1.1.1
      - -t 1.1.1.1:11000
      - -nolog 1
```
License
-------

MIT
