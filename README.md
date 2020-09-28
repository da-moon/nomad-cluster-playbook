# nomad-cluster-playbook

<p align="center">
  <a href="https://gitpod.io#https://github.com/da-moon/nginx-lua">
    <img src="https://img.shields.io/badge/open%20in-gitpod-blue?logo=gitpod" alt="Open In GitPod">
  </a>
  <img src="https://img.shields.io/github/languages/code-size/da-moon/nginx-lua" alt="GitHub code size in bytes">
  <img src="https://img.shields.io/github/commit-activity/w/da-moon/nginx-lua" alt="GitHub commit activity">
  <img src="https://img.shields.io/github/last-commit/da-moon/nginx-lua/master" alt="GitHub last commit">
</p>

Cloud agnostic Ansible playbooks used to deploy nomad cluster.

## commands

- these playbooks only work when ansible host and remotes are both `debian` based linux distribution and cpu architecture must be `amd64`.
- make sure `jq` and `sshpass` are installed before running make targets.
- `ansible-generate` was used for generating directory structure. i.e

```bash
ansible-generate -i production staging pre-staging -r 00-ansible-controller -p . -a
```

- generate staging and production `ansible-nomad` encryption masterkeys and store them in `STAGING_VAULT_PASSWORD_FILE` and `VAULT_PASSWORD_FILE` files (located in `vars.mk`). you can use the following snippet to generate a 16 byte long hex string.

```bash
# => with GNU coreutils 'head'
head -c16 </dev/urandom|xxd -p -u
# => with hexdump
hexdump -n 16 -e '4/4 "%08X" 1 "\n"' /dev/random
# => with openssl : 16 bytes (32 hex symbols)
openssl rand -hex 16
```

this master-key is used to encrypt/decrypt secrets and artifacts such as certificates.

- launch bare-bone lxd containers in case you want to run playbook against local pre-staging environment with the following command

  - you can customize lxd base image by changing `LXC_IMAGE` variable in `vars.mk`. default image is `debian/buster`
  - the following make target generates two set of nodes, `server` and `client`, each set with container count of `CONTAINER_COUNT` which you can customize number of containers by changing it. It is recommended to have at least 3 nodes in each set.
  - if you have not installed `lxd`, install `snap` package manager and run `contrib/lxd-init/lxd` script. refer to [`lxd-init-readme`](contrib/lxd-init/README) for more information.

```bash
make -j`nproc` init
```

> `[NOTE]` increase `DELAY` value in `vars.mk` in case container ip was not available when corresponding make target is running.

> `[NOTE]` this make target installs necessary packages for a debian based environment. in case you change the image to a non-debian based distribution, this target will not work.

> `[NOTE]` while it is possible to have a nomad agent act both as client and server, it is not recommended to do so. As long as long as all nodes resources, no problem arises but in case a job saturates one of the resources (e.g cpu, storage) on a node and cause it to crash or be unable to do work in a timely manner, the other nodes would eventually consider that job as "lost" and reschedule it. The job would then cause its new host to become unresponsive and you would no longer have a functioning Nomad cluster.

> `[NOTE]` Another issue that might lead to making server unresponsive is the growth of Nomad server memory,which can happen for a variety of reasons over time, isn't accounted for when making scheduling decisions since Nomad isn't managing itself and associated resource constraints. in case one is using systemd for running nomad, something like the following snippet in the service can limit resource consumption.

```service
[Service]
# ....
MemoryAccounting=true
MemoryHigh=1024K
MemoryMax=4096K
# ....
```

- run ansible and provision pre-staging environment by running the following command

```bash
make pre-staging
```

> `[NOTE]` this make target automatically updates pre-staging hosts so no need to manually update `inventories/pre-staging/hosts` every time you create/launch pre-staging containers
  
- remove all pre-staging lxd containers

```bash
make -j`nproc` lxd-clean
```

- decrypt gossip encryption key (e.g for prestaging)

```bash
ansible localhost -m debug -a var="gossip_encryption_key" -e "@inventories/pre-staging/group_vars/gossip_encryption_key.yml" --vault-password-file ~/.vault_pass.txt
```

## TODO

- [] fix cewrts
- [] nomad client
- [] bootstrap acl policy
- [] add vault integration
- [] add consul integration
