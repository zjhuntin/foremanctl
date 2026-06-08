# OpenVox Server Role — Parity TODO

## What this role does today

- Builds/pulls an OpenVox Server container image (CentOS Stream 9 + openvox-server RPM)
- Imports foremanctl's CA into puppetserver via `puppetserver ca import` (shared trust)
- Deploys the openvox-server as a podman quadlet with foremanctl's CA
- Adds puppet proxy feature to foreman-proxy (settings.d templates + cert mounts)
- Adds `puppet` feature definition to foremanctl features.yaml

## What's working

- [x] Container builds and starts with foremanctl's CA
- [x] Puppet proxy feature registered in Foreman
- [x] Proxy can list puppet environments from openvox
- [x] Shared CA trust — foremanctl-signed certs trusted by openvox
- [x] Auth.conf — environment_classes API access (template added)
- [x] ENC script — mounted from puppetserver_foreman (enc.rb)
- [x] Report processor — mounted from puppetserver_foreman (report.rb)
- [x] foreman.yaml — configured with Foreman URL + SSL certs
- [x] Facts upload — enabled via foreman.yaml `:facts: true`

## What's NOT working yet (needs fixes)

### 1. SELinux label handling
The container requires `--security-opt label=disable` because puppetserver tries to
chmod SSL and log directories on startup, which conflicts with SELinux labels on
bind-mounted volumes.

**In non-container (installer):** Not an issue — puppetserver owns the filesystem directly.

**Fix options:**
- Use named volumes instead of bind mounts for SSL/CA dirs
- Set proper SELinux contexts with `semanage fcontext`
- Or accept `label=disable` as foremanctl's pattern (other containers use `:Z` label)

### 6. Puppet CA proxy feature
The foreman-proxy can also proxy puppetserver CA operations (sign/revoke certs
from Foreman UI). This requires:
- `puppetca` feature enabled in foreman-proxy settings
- Proxy has access to puppetserver CA API

**In non-container (installer):** `--foreman-proxy-puppetca true` enables this.

**Fix:** Add puppetca.yml.j2 template and feature task. The proxy already has
the puppetca config files in the image.

### 7. Container image publishing
Currently the openvox-server image is built locally on the VM. For a real
deployment:
- The Containerfile should be in `theforeman/foreman-oci-images`
- Published to quay.io alongside foreman, foreman-proxy, pulp, candlepin
- Referenced in foremanctl's `images.yml`

### 8. Hammer CLI puppet plugin in container
The hammer role tries to install `hammer-cli-plugin-foreman_puppet` as an RPM
but it doesn't exist as a standalone package. The gem is installed in the
foreman container via `rubygem(hammer_cli_foreman_puppet)`.

**Fix:** The hammer role needs to handle containerized hammer differently —
install hammer CLI plugins inside the foreman container, not on the host.

## Parity checklist vs foreman-installer

| Feature | Installer | foremanctl | Status |
|---------|-----------|-----------|--------|
| Puppetserver running | `--enable-puppet --puppet-server true` | openvox_server role | Working |
| Foreman puppet plugin | `--enable-foreman-plugin-puppet` | `foreman_puppet` in FOREMAN_ENABLED_PLUGINS | Working (needs nightly-full image) |
| Puppet proxy feature | `--foreman-proxy-puppet true` | puppet.yml.j2 + feature task | Working |
| Puppet CA proxy | `--foreman-proxy-puppetca true` | Not implemented | TODO |
| Shared CA trust | Installer manages Puppet CA | `puppetserver ca import` with foremanctl CA | Working |
| ENC integration | `puppet::server::enc` + `puppetserver_foreman` | enc.rb mounted, puppet.conf configured | Implemented |
| Report forwarding | `puppet::server::foreman` + report processor | report.rb mounted as foreman report processor | Implemented |
| Facts upload | ENC config `enc_upload_facts` | foreman.yaml `:facts: true` | Implemented |
| Auth.conf | `puppet::server::config` | auth.conf.j2 with environment_classes rule | Implemented |
| Autosign | `--puppet-autosign true` | puppet.conf `autosign = true` | Working |
| Hammer CLI | `--enable-foreman-cli-puppet` | Install in foreman container | Working (manual step) |
| Environment management | File-based environments | Bind-mount /var/lib/openvox/code | Working |
| Module import via proxy | auth.conf + environment_classes API | auth.conf allows, proxy configured | Implemented |

## Priority order for remaining work

1. **Auth.conf template** — unblocks class import, highest impact
2. **ENC integration** — makes puppet useful (nodes get classifications from Foreman)
3. **Report forwarding** — puppet runs visible in Foreman UI
4. **Puppet CA proxy** — cert management from Foreman UI
5. **Container image publishing** — move from localhost build to quay.io
6. **SELinux handling** — proper volume labeling instead of label=disable
