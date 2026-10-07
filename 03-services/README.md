# Services and High Availability

## Objective
Stand up the lab's management/services plane — Zabbix monitoring, NTP, centralized syslog, and TACACS+ AAA — on a dedicated host, separate from the emulated network devices in PNETLab.

## Environment
- **Host:** `Control_VM` — a Rocky Linux 10.2 ("Red Quartz") VM running under **VMware ESXi** on a second physical machine ("Laptop 2"), alongside a Windows Server 2019 VM (`VM01`, see below). Not bare metal, not inside the same VMware Workstation host that runs PNETLab.
- **Network:** `Control_VM` and the PNETLab host currently sit on the same **flat `192.168.1.0/24` LAN** (gateway `192.168.1.1`). The originally planned isolated management VLAN 99 (`172.16.0.0/24`) does not physically exist yet — no VLAN trunking or ESXi port groups are configured for it. `Control_VM`'s static address (`192.168.1.231`) was deliberately chosen to match its existing DHCP lease rather than jump to the paper-design address, to avoid breaking connectivity. VLAN 99 remains a real future task.
- **Role on this host:** Zabbix server + web UI + database (all as Docker containers), plus planned NTP/syslog/TACACS+/automation containers.

> **History:** an earlier attempt built this natively (`dnf`-installed Zabbix, MariaDB, firewalld, SELinux) directly on bare-metal Rocky Linux 9/10. That install was lost when the host's disk was corrupted and the machine was reimaged with ESXi. Everything below describes the current, actually-running Docker-based rebuild.

## Host setup — `Control_VM`

**Non-root admin account** (SSH access, not `root` directly):
```bash
useradd -m admin
passwd admin
usermod -aG wheel admin        # grants sudo via the wheel group
```

**Disable direct root SSH login** (confirmed via `sudo grep -n "^PermitRootLogin" /etc/ssh/sshd_config` first):
```bash
sudo sed -i 's/^#PermitRootLogin prohibit-password/PermitRootLogin no/' /etc/ssh/sshd_config
sudo systemctl restart sshd
```
Verified from a fresh terminal: `ssh root@<ip>` refused, `ssh admin@<ip>` + `sudo` still works.

**VMware Tools** (guest integration — reported IP, clean shutdown from the ESXi UI):
```bash
sudo dnf install -y open-vm-tools
sudo systemctl enable --now vmtoolsd
```

**Static IP** (kept the VM's existing DHCP address rather than the planned-but-nonexistent VLAN 99 range):
```bash
sudo nmcli connection modify ens192 ipv4.addresses 192.168.1.231/24
sudo nmcli connection modify ens192 ipv4.gateway 192.168.1.1
sudo nmcli connection modify ens192 ipv4.dns 192.168.1.1
sudo nmcli connection modify ens192 ipv4.method manual
sudo nmcli connection up ens192
```

**Hostname:**
```bash
sudo hostnamectl set-hostname Control_VM
```
(Systemd derives a sanitized *static* hostname `ControlVM` from this, alongside the *pretty* hostname `Control_VM` — both appear in `hostnamectl` output. Harmless, just worth knowing if `ControlVM` shows up elsewhere.)

**Docker Engine** (Rocky doesn't ship Docker by default — only `podman`):
```bash
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker admin
newgrp docker
docker run hello-world   # verified clean end-to-end
```

## Zabbix stack — Docker Compose

Working directory on `Control_VM`: `~/services-stack/zabbix/`. The compose file and an example env template are mirrored into this repo as [`docker-compose.yml`](docker-compose.yml) and [`.env.example`](.env.example) — the real `.env` (with actual passwords) stays on the VM only and is never committed.

Built incrementally, one service at a time, verifying each with `docker compose logs <service>` before adding the next:

1. **`mysql-server`** (official `mysql:8.0` image, `utf8mb4`/`utf8mb4_bin` — Zabbix's documented requirement) — brought up and confirmed `ready for connections` before adding anything else.
2. **`zabbix-server`** (`zabbix/zabbix-server-mysql:alpine-7.0-latest`) — reaches the database by its Compose service name (`DB_SERVER_HOST: mysql-server`), not an IP; Docker Compose provides this as internal DNS automatically. Confirmed all worker threads started cleanly.
3. **`zabbix-web`** (`zabbix/zabbix-web-nginx-mysql:alpine-7.0-latest`) — reached the login page directly at `http://192.168.1.231/`, skipping any manual nginx config entirely (the official image auto-configures itself) — **this is the milestone the old native install never reached**, where it was stuck on an unconfigured-vhost 404.
4. **`zabbix-agent`** (`zabbix/zabbix-agent2:alpine-7.0-latest`, the modern Go-based agent) — added last.

### Post-deploy bug: self-monitoring showed "agent not available"
The "Zabbix server" host Zabbix auto-creates for itself defaults its agent interface to `127.0.0.1` — which is meaningless across container boundaries (each container has its own network namespace; `127.0.0.1` inside `zabbix-server` refers only to itself). Confirmed hands-on before changing anything:
```bash
docker exec -it zabbix-server sh -c "wget -qO- -T 2 http://127.0.0.1:10050 2>&1 || echo FAILED"
# -> connection refused
```
**Fix** (Zabbix UI → Data collection → Hosts → Zabbix server → Agent interface): changed "Connect to" from IP `127.0.0.1` to **DNS** `zabbix-agent` (the container's Compose service name). Confirmed fixed — dashboard went from "1 Not available" to "1 Available."

### Verified surviving a reboot
`docker compose stop` → clean VM shutdown (`sudo shutdown -h now`) → ESXi power-on → `docker compose up -d` (no service name, brings the whole stack back) restarted all containers correctly. The stack is not just "working once," it's durable across a restart cycle. Note: after a reboot, `docker compose ps` (no `-a`) can show nothing at first even though everything is fine — `restart: unless-stopped` deliberately does not auto-start a service that was manually `stop`ped before shutdown; `docker compose up -d` brings it back.

### NTP and syslog
Added as two more containers in the same stack:
5. **`ntp`** (`cturra/ntp`, with `cap_add: [SYS_TIME, SYS_NICE]` — this image needs just enough extra Linux capability to adjust the system clock, deliberately avoiding the much broader `--privileged` flag). Published on `123/udp`.
6. **`syslog`** (`balabit/syslog-ng:latest` — maintained by the actual creators of syslog-ng). Published on `514/udp` and `514/tcp` since network devices commonly use either; received logs persist in a named volume (`syslog_data`).

Both images were chosen after checking Docker Hub directly for pull counts and last-updated dates — unlike TACACS+ (see Status below), both had a clear, actively-maintained best choice.

## Lessons learned
- **Verify the actual OS version before trusting a documented package URL.** Rocky 10 ≠ the Rocky 9 the original plan assumed — caught by checking `repo.zabbix.com` directly rather than guessing.
- **A `GRANT` statement (or any "it ran without error") can silently not be what you think.** Confirm with a direct check (`SHOW GRANTS`, `docker compose logs`, re-reading the actual file) rather than trusting a clean exit code.
- **Container network namespaces are isolated.** `127.0.0.1` inside one container never means "the host" or "another container" — always reach sibling containers by their Compose service name.
- **Hand-typed commands produce a predictable class of typo**, not random errors: transposed/dropped letters (`rtestart`, `charater-set-server`, `depend_on`, `q0-` vs `qO-`). When something fails right after typing a command by hand, check the exact text first before assuming a deeper problem.
- **Docker image popularity doesn't guarantee maintenance.** Checked directly on Docker Hub before picking images: `cturra/ntp` and `balabit/syslog-ng` are both actively maintained (updated within the last day/months); the most-used TACACS+ images are both 5+ years stale — see Status below.

## Windows Server VM (`VM01`) — planned, not yet built
Decided role (build order: AD DS first, everything else depends on it):
1. **AD DS** — identity backend (TACACS+/RADIUS logins authenticate against real AD accounts)
2. **DNS** — internal name resolution
3. **DHCP** — dynamic addressing for VLAN 10/20, replacing static VPCS addressing
4. **NPS (RADIUS)** — backed by AD identity; complements the still-unresolved TACACS+ container situation

## Status
- [x] `Control_VM` provisioned: non-root admin+sudo, root SSH disabled, static IP, Docker installed
- [x] Zabbix stack (DB + server + web + agent) built, verified, survives a reboot cycle
- [x] Self-monitoring bug found and fixed (agent interface `127.0.0.1` → `zabbix-agent`)
- [x] NTP container (`cturra/ntp`) — deployed and healthy
- [x] Syslog container (`balabit/syslog-ng`) — deployed and healthy
- [ ] TACACS+ container — **no good prebuilt image exists** (best options are 5 years stale); likely path is building `dchidell/docker-tacacs`'s Dockerfile from source rather than pulling a stale image, or deferring in favor of Windows NPS/RADIUS for AAA coverage
- [ ] Automation tooling container (Netmiko/Jinja2) — not started
- [ ] Windows Server VM (`VM01`) — role decided, not built
