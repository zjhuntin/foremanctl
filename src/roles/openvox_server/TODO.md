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

## What's NOT working yet (needs fixes)

### 1. Auth.conf — environment_classes API access
The proxy gets 403 when trying to import classes because `/puppet/v3/environment_classes`
is denied by default in puppetserver's auth.conf. Need to mount a custom auth.conf that
allows the proxy to access class listing APIs.

**In non-container (installer):** The puppet module configures auth.conf automatically
via `puppet::server::config`.

**Fix:** Add auth.conf template to the role with rules allowing the proxy host to
access environment and class APIs.

### 2. SELinux label handling
The container requires `--security-opt label=disable` because puppetserver tries to
chmod SSL and log directories on startup, which conflicts with SELinux labels on
bind-mounted volumes.

**In non-container (installer):** Not an issue — puppetserver owns the filesystem directly.

**Fix options:**
- Use named volumes instead of bind mounts for SSL/CA dirs
- Set proper SELinux contexts with `semanage fcontext`
- Or accept `label=disable` as foremanctl's pattern (other containers use `:Z` label)

### 3. ENC (External Node Classifier) integration
Puppetserver needs to call Foreman's ENC endpoint to get node classifications.
This requires:
- An ENC script in the container that calls Foreman's API
- puppet.conf `external_nodes` pointing to that script
- `node_terminus = exec` in puppet.conf

**In non-container (installer):** The `puppet::server::enc` class installs the ENC
script and configures puppet.conf. The `puppetserver_foreman` module installs the
ENC script at `/etc/puppetlabs/puppet/node.rb`.

**Fix:** Mount the `puppetserver_foreman` ENC script into the container, or package
it in the image. Configure puppet.conf template with `external_nodes` and `node_terminus`.

### 4. Report processor — puppet reports to Foreman
Puppetserver needs to forward reports to Foreman so puppet runs appear in the UI.
This requires:
- The `foreman` report processor installed in the server
- `reports = foreman` in puppet.conf
- Foreman URL and SSL credentials configured in `/etc/puppetlabs/puppet/foreman.yaml`

**In non-container (installer):** The `puppetserver_foreman` module installs the
report processor and configures foreman.yaml.

**Fix:** Include the `puppetserver_foreman` gem/scripts in the container image and
mount a foreman.yaml config secret.

### 5. Facts upload — puppet facts to Foreman
The ENC script can be configured to upload facts to Foreman on each puppet run.
This is controlled by `enc_upload_facts` in the foreman ENC config.

**In non-container (installer):** Configured via `puppetserver_foreman` module.

**Fix:** Part of the ENC integration (item 3).

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
| ENC integration | `puppet::server::enc` + `puppetserver_foreman` | Not implemented | TODO |
| Report forwarding | `puppet::server::foreman` + report processor | Not implemented | TODO |
| Facts upload | ENC config `enc_upload_facts` | Not implemented | TODO |
| Auth.conf | `puppet::server::config` | Not implemented (403 on class import) | TODO |
| Autosign | `--puppet-autosign true` | puppet.conf `autosign = true` | Working |
| Hammer CLI | `--enable-foreman-cli-puppet` | Install in foreman container | Working (manual step) |
| Environment management | File-based environments | Bind-mount /var/lib/openvox/code | Working |
| Module import via proxy | auth.conf + environment_classes API | Blocked by auth.conf | TODO |

## Priority order for remaining work

1. **Auth.conf template** — unblocks class import, highest impact
2. **ENC integration** — makes puppet useful (nodes get classifications from Foreman)
3. **Report forwarding** — puppet runs visible in Foreman UI
4. **Puppet CA proxy** — cert management from Foreman UI
5. **Container image publishing** — move from localhost build to quay.io
6. **SELinux handling** — proper volume labeling instead of label=disable
