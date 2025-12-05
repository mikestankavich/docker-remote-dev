# Roadmap

## Vision

A simple, multi-cloud tool to spin up a Docker development environment on a remote VM and connect to it seamlessly from your local machine. Simpler than Terraform/Pulumi for this specific use case, filling the gap left by Docker Machine's deprecation.

## Current State

~60-70% complete for AWS-only MVP. Core Terraform infrastructure exists but has bugs and isn't wired to local configuration.

### What Works
- Terraform EC2 module (security groups, EIP, Ubuntu AMI lookup)
- User data script installs Docker CE
- Ansible playbook for local SSH config + docker context
- Network module exists (commented out)

### Known Bugs
- `terraform/outputs.tf` - duplicate `public_dns` output
- `terraform/outputs.tf` - references `key_pair_name` without `var.` prefix
- `terraform/aws/ec2/main.tf:57` - instance type hardcoded to `t3.large` despite variable existing
- Ansible playbook has hardcoded IP and key file (not templated from Terraform outputs)
- `justfile` - `install` recipe is empty
- No destroy functionality

---

## Architecture Decisions

### Cloud-init over Bash User Data

**Decision:** Replace current bash user data script with declarative cloud-init YAML.

**Rationale:**
- Idempotent and declarative
- Installs Docker via APT sources (recommended) instead of get.docker.com script (discouraged for production)
- Native support across all target cloud providers
- Easy to add SSH hardening, version pinning, readiness signals
- Includes docker-compose-plugin by default

### Separation of Concerns

```
┌─────────────────────────────────────────────────────────────────┐
│                    REMOTE (cloud-init)                          │
│  • Docker CE + Compose plugin (via apt)                         │
│  • User setup (default user in docker group)                    │
│  • SSH hardening (optional)                                     │
│  • Readiness signal                                             │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ user_data
┌─────────────────────────────────────────────────────────────────┐
│                 TERRAFORM (per-provider modules)                │
│  • Compute instance                                             │
│  • SSH key management                                           │
│  • Firewall / Security Group                                    │
│  • Elastic/Floating IP                                          │
│  • Outputs: IP, hostname, key path                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ outputs
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    LOCAL (Ansible or shell)                     │
│  • ~/.ssh/config entry                                          │
│  • docker context create/use                                    │
│  • Wait for cloud-init completion                               │
└─────────────────────────────────────────────────────────────────┘
```

### Keep Ansible for Local Config

Local configuration (SSH config + docker context) stays in Ansible because:
- Already written and working
- Idempotent
- Handles edge cases well
- Only needs Terraform outputs wired in

---

## Multi-Cloud Strategy

### Target Providers (Priority Order)

| Priority | Provider | Rationale |
|----------|----------|-----------|
| 1 | **Hetzner** | Cheapest (€3.49/mo), solid TF provider, familiar |
| 2 | **Vultr** | Best affiliate ($35/conversion), $5/mo, simple API |
| 3 | **DigitalOcean** | Beginner-friendly, 10% recurring affiliate, excellent docs |
| 4 | **AWS** | Already exists, keep for enterprise users |

### Provider Comparison

| Provider | Entry Price | Affiliate | TF Provider | Notes |
|----------|-------------|-----------|-------------|-------|
| Hetzner | €3.49/mo | Credits only | `hetznercloud/hcloud` | EU-focused, US DCs limited |
| Vultr | $5/mo | $35/conversion | `vultr/vultr` | Good global coverage |
| DigitalOcean | $6/mo | 10% recurring/1yr | `digitalocean/digitalocean` | Best docs, most beginner-friendly |
| AWS EC2 | ~$8-15/mo | None | `hashicorp/aws` | Complex but familiar |

### Module Structure

```
terraform/
├── main.tf                    # Root - provider selection
├── variables.tf               # Common variables
├── outputs.tf                 # Common outputs
├── cloud-init/
│   └── docker-dev.yaml        # Shared cloud-init config
├── aws/
│   └── ec2/                   # Existing (needs fixes)
├── hetzner/
│   └── server/                # To create
├── vultr/
│   └── instance/              # To create
└── digitalocean/
    └── droplet/               # To create
```

### Per-Provider Resource Mapping

| Component | AWS | Hetzner | Vultr | DigitalOcean |
|-----------|-----|---------|-------|--------------|
| Compute | `aws_instance` | `hcloud_server` | `vultr_instance` | `digitalocean_droplet` |
| SSH Key | `aws_key_pair` | `hcloud_ssh_key` | `vultr_ssh_key` | `digitalocean_ssh_key` |
| Firewall | `aws_security_group` | `hcloud_firewall` | `vultr_firewall_group` | `digitalocean_firewall` |
| Floating IP | `aws_eip` | `hcloud_floating_ip` | `vultr_reserved_ip` | `digitalocean_floating_ip` |
| VPC/Network | `aws_vpc` | `hcloud_network` | `vultr_vpc` | `digitalocean_vpc` |

---

## Feature Roadmap

### Phase 1: Fix AWS & Establish Patterns (~3-5 hours)
- [ ] Fix `outputs.tf` bugs (duplicate output, missing var prefix)
- [ ] Make instance_type variable actually work
- [ ] Replace `userdata.tpl` with `cloud-init.yaml`
- [ ] Wire Terraform outputs to Ansible variables
- [ ] Complete justfile (install, destroy, status commands)
- [ ] Add destroy functionality
- [ ] Update README with working instructions

### Phase 2: Multi-Cloud (~12-15 hours)
- [ ] Create shared `cloud-init/docker-dev.yaml`
- [ ] Add Hetzner module (~4 hours)
- [ ] Add Vultr module (~4 hours)
- [ ] Add DigitalOcean module (~4 hours)
- [ ] Provider selection in justfile/main.tf
- [ ] Provider-specific documentation

### Phase 3: Mesh Networking (~5-8 hours)
- [ ] Tailscale support (easiest, most users)
- [ ] Netmaker support (self-hosted option)
- [ ] Auto-join to existing network
- [ ] Private IP connectivity (no public IP exposure option)

### Phase 4: Polish & Monetization (~3-5 hours)
- [ ] Affiliate links in README (Vultr, DigitalOcean)
- [ ] Cost comparison table in docs
- [ ] GitHub Actions CI (provision/destroy test instances)
- [ ] Shellcheck linting
- [ ] Release automation

### Phase 4b: Open Source Hygiene
- [ ] Branch protection rules (require PR for main)
- [ ] CONTRIBUTING.md (how to contribute, PR process, coding standards)
- [ ] CODE_OF_CONDUCT.md (Contributor Covenant or similar)
- [ ] SECURITY.md (vulnerability reporting process)
- [ ] Issue templates (bug report, feature request)
- [ ] PR template
- [ ] License headers in source files (MPL-2.0)

### Phase 5: Future Enhancements (backlog)
- [ ] Spot/preemptible instance support (cost savings)
- [ ] Instance stop/start (vs destroy) for cost management
- [ ] Pre-populated Docker images (faster startup)
- [ ] Volume persistence across destroy/create cycles
- [ ] rsync/mutagen for bind mount emulation
- [ ] VS Code Remote SSH auto-configuration
- [ ] Rust CLI replacement for justfile (stretch goal)

---

## Mesh Networking Notes

### Tailscale
- Simplest setup: `curl -fsSL https://tailscale.com/install.sh | sh`
- Can add to cloud-init runcmd
- Auth key via Terraform variable (pre-auth key from Tailscale admin)
- Enables: private IP access, no exposed ports, Magic DNS

### Netmaker
- Self-hosted alternative (you already run this on Vultr)
- Requires existing Netmaker server
- netclient install + join token
- More complex but no vendor dependency

---

## Competitive Landscape

### Why This Project Has Value
1. **Docker Machine is deprecated** - no official simple replacement
2. **Dev Containers/Codespaces aren't for everyone** - some want raw Docker on a VM
3. **Terraform/Pulumi are overkill** - too much complexity for "I just want Docker on a VM"
4. **Multi-cloud is rare** - most tools are single-provider

### Alternatives
- **Docker Machine** (legacy, declining)
- **Terraform + Packer** (heavyweight, requires HCL knowledge)
- **Dev Containers / Codespaces** (different paradigm, vendor lock-in)
- **Manual EC2 + user data** (error-prone, not reproducible)

---

## Resources

- [cloud-init Docker config examples](https://gist.github.com/syntaqx/9dd3ff11fb3d48b032c84f3e31af9163)
- [Hetzner Terraform Provider](https://registry.terraform.io/providers/hetznercloud/hcloud/latest/docs)
- [Vultr Terraform Provider](https://registry.terraform.io/providers/vultr/vultr/latest/docs)
- [DigitalOcean Terraform Provider](https://registry.terraform.io/providers/digitalocean/digitalocean/latest/docs)
- [Tailscale install docs](https://tailscale.com/download/linux)
- [Vultr Affiliate Program](https://www.vultr.com/company/affiliate/) - $35/conversion
- [DigitalOcean Affiliate Program](https://www.digitalocean.com/affiliates) - 10% recurring for 1 year
