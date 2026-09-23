# CIS NGINX Hardening with Ansible

An Ansible-based security-hardening project for an NGINX web server, developed against the **CIS NGINX Benchmark v3.0.0** in an AWS laboratory environment.

The project assessed all **44 benchmark recommendations** and implemented **30 controls**. The remaining recommendations were documented as already compliant through audit, not applicable to the architecture, laboratory exceptions, or Level 2 controls that were not selected.

> This is an academic laboratory implementation, not a production-ready security baseline. Review all variables, certificates, network settings, and control applicability before using it in another environment.

## Project Results

| Assessment outcome | Number |
|---|---:|
| Implemented | 30 |
| Compliant by audit, default, or design | 5 |
| Not applicable | 4 |
| Laboratory exceptions | 2 |
| Level 2 controls not selected | 3 |
| **Total assessed** | **44** |

## Project Highlights

- Automated NGINX hardening through a reusable Ansible role
- Separate AWS control and managed nodes
- Dedicated, locked, non-login NGINX worker account
- Restricted ownership and permissions for configuration and TLS files
- Authorized virtual hosts and a catch-all default server
- HTTPS redirection, TLS 1.3, HSTS, and protected private keys
- JSON access logging and security-focused error logging
- Request timeouts, body-size limits, method restrictions, and rate limiting
- Protection against hidden-file access and information disclosure
- Browser security headers and custom error responses
- Control-specific Ansible tags and collected validation evidence

## Technologies

- AWS EC2
- Ubuntu Server 26.04 LTS
- NGINX 1.28.3
- Ansible
- OpenSSL 3.5.5
- Bash
- Git and GitHub

The original environment used one Ansible control node and one managed NGINX server. The Section 1 tasks require NGINX 1.28.0 or later.

## Repository Structure

```text
.
├── README.md
├── ansible.cfg
├── site.yml
├── inventory/
│   └── hosts.yml
├── audit/
│   ├── baseline-audit.sh
│   └── post-hardening-audit.sh
├── evidence/
│   ├── before/
│   └── after/
└── roles/
    └── nginx_cis/
        ├── defaults/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        ├── tasks/
        │   ├── main.yml
        │   ├── deploy_site.yml
        │   ├── section_1_installation.yml
        │   ├── section_2_configuration.yml
        │   ├── section_3_logging.yml
        │   ├── section_4_tls.yml
        │   └── section_5_request_controls.yml
        └── templates/
            ├── cis-connection-timeouts.conf.j2
            ├── cis-default-server.conf.j2
            ├── cis-deny-hidden.conf.j2
            ├── cis-logging.conf.j2
            ├── cis-request-controls.conf.j2
            ├── cis-security-headers.conf.j2
            ├── enterprise-site-tls.conf.j2
            ├── enterprise-site.conf.j2
            └── hardened-site.conf.j2
```

The repository also retains earlier site-template versions created during development. `hardened-site.conf.j2` is the cumulative hardened site configuration.

## Control Areas

| Section | Focus | Examples |
|---|---|---|
| 1 | Installation and updates | Version validation, package-source review, current NGINX package |
| 2 | Basic configuration | Service account, permissions, authorized sites, default-server handling, custom errors and timeouts |
| 3 | Logging | JSON access logs, protected log files and informational error logging |
| 4 | Encryption | HTTPS redirection, protected TLS files, TLS 1.3 and HSTS |
| 5 | Request controls | Method restrictions, request-size limits, rate limiting, hidden-file protection and security headers |

Controls are tagged by section and, where applicable, by recommendation number. For example, `cis_section_4` runs the TLS section, while `cis_4_1_8` targets the HSTS-related deployment tasks.

## Prerequisites

- An Ansible control system
- A supported Ubuntu or Debian NGINX target
- SSH access from the control system to the managed server
- A remote account with `sudo` access
- Python available on the managed server
- Port 22 permitted between the control and managed nodes
- Ports 80 and 443 permitted for web testing

The role uses Ubuntu/Debian paths and the `apt` package manager. Other Linux distributions require modifications.

## Configuration

Clone the repository:

```bash
git clone https://github.com/Kundanlyl/nginx-cis-hardening-ansible.git
cd nginx-cis-hardening-ansible
```

Edit `inventory/hosts.yml` with the address and SSH user for the managed NGINX server.

Review the variables in:

```text
roles/nginx_cis/defaults/main.yml
```

At minimum, replace the laboratory-specific values:

- `nginx_cis_server_names`
- `nginx_cis_primary_name`
- `nginx_cis_private_ip`
- Certificate and private-key paths
- Timeout values
- Rate-limit settings
- Maximum request-body size

Confirm connectivity:

```bash
ansible nginx -m ping
```

Check the playbook syntax:

```bash
ansible-playbook --syntax-check site.yml
```

## Running the Playbook

Apply the complete hardening role:

```bash
ansible-playbook site.yml
```

Run one benchmark section:

```bash
ansible-playbook site.yml --tags cis_section_4
```

Run a control-specific tag:

```bash
ansible-playbook site.yml --tags cis_4_1_8
```

List the available tasks and tags:

```bash
ansible-playbook site.yml --list-tasks --list-tags
```

The role creates section-specific backups of `/etc/nginx` before major configuration stages. The handler validates the NGINX configuration before reloading the service.

## Validation

Successful Ansible output confirms that the tasks completed, but the resulting server state should also be tested independently.

Validate the NGINX configuration:

```bash
ansible nginx -b -m command -a "nginx -t"
```

Check the service:

```bash
ansible nginx -b -m command -a "systemctl is-active nginx"
```

Inspect listening ports:

```bash
ansible nginx -b -m shell -a "ss -lntp | grep -E ':(80|443)[[:space:]]'"
```

Inspect active timeout directives:

```bash
ansible nginx -b -m shell -a "nginx -T 2>/dev/null | grep -E '^[[:space:]]*(client_body_timeout|client_header_timeout|keepalive_timeout|send_timeout)[[:space:]]'"
```

Test the HTTPS response and security headers:

```bash
curl -skI --resolve web01.corp.example:443:<SERVER_IP> https://web01.corp.example/
```

Test TLS 1.3:

```bash
openssl s_client -connect <SERVER_IP>:443 -servername web01.corp.example -tls1_3
```

The `evidence/before` and `evidence/after` directories contain selected output captured during the original laboratory deployment.

## Laboratory Limitations

The original deployment generated a self-signed certificate for an internal `.local` hostname. This encrypted laboratory traffic but did not provide the automatic identity trust of a certificate issued by a public or organizational certificate authority.

A production deployment should use trusted certificate management and separately implement or evaluate:

- Centralized monitoring
- Vulnerability scanning
- Operating-system hardening
- Secure secret management
- Backups and recovery
- Automated certificate renewal
- Application-level security testing

The repository uses example hostnames and private IP addresses for laboratory configuration. Replace them with values appropriate for the deployment environment.

## Security Notice

Do not commit:

- SSH or TLS private keys
- AWS credentials
- Passwords or access tokens
- Ansible Vault password files
- Sensitive inventory information

The included `.gitignore` excludes common sensitive file types, but changes should still be reviewed before every commit.

## Disclaimer

This project is an independent educational implementation informed by the CIS NGINX Benchmark v3.0.0. It is not affiliated with or endorsed by the Center for Internet Security (CIS), and the benchmark document itself is not redistributed in this repository.

CIS recommendations require environment-specific assessment. Running this role does not by itself guarantee compliance or a secure production system.
