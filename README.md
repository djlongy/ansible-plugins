# ansible-plugins

A collection of custom Ansible action plugins and supporting roles for operational visibility, audit logging, and CI/CD integration.

---

## Contents

- [`get_cli_args`](#get_cli_args) — Action plugin that captures playbook invocation metadata and git state at runtime
- [Audit Logging Framework](#audit-logging-framework) — Logger-agnostic audit framework built on `get_cli_args`

---

## get_cli_args

### What it does

An Ansible action plugin that exposes the `ansible-playbook` CLI arguments, Semaphore CI/CD variables, and the git state of the playbook's working tree to tasks within a running playbook.

**Use cases:**
- Log exactly how a playbook was invoked
- Detect modified or dirty files at runtime
- Extract Semaphore task metadata without hardcoding variables
- Build rich audit payloads for compliance or troubleshooting

### Location

```
action_plugins/get_cli_args/get_cli_args.py
```

### Returns

#### CLI keys

| Key | Type | Description |
|-----|------|-------------|
| `ansible_playbook_argv` | list | Cleaned CLI argument list (internal JSON extra-vars stripped) |
| `ansible_playbook_cmd` | string | Space-joined version of `ansible_playbook_argv` |
| `semaphore_vars` | dict \| null | Variables injected by Semaphore, or `null` if not run via Semaphore |

#### `git_status` keys

| Key | Type | Description |
|-----|------|-------------|
| `branch` | string | Current branch name, or `HEAD` if detached |
| `commit` | string | Full commit SHA |
| `commit_short` | string | 7-char short SHA |
| `tag` | string \| null | Nearest tag from `git describe --tags --always --dirty` |
| `modified` | list | Tracked files with unstaged modifications |
| `staged` | list | Files staged in the index |
| `untracked` | list | Untracked files |
| `deleted` | list | Tracked files deleted in the worktree |
| `is_clean` | bool | `true` if working tree and index are clean |
| `error` | string | Set if git is unavailable or directory is not a repo |

### Installation

Copy the plugin into your project's `action_plugins/` directory:

```bash
cp action_plugins/get_cli_args/get_cli_args.py your-project/action_plugins/get_cli_args.py
```

Or configure `ansible.cfg`:

```ini
[defaults]
action_plugins = /path/to/ansible-plugins/action_plugins:/usr/share/ansible/plugins/action
```

### Example

```yaml
- name: Capture invocation metadata
  get_cli_args:
  register: cli

- name: Show command
  ansible.builtin.debug:
    msg: "Invoked as: {{ cli.ansible_playbook_cmd }}"

- name: Warn on dirty working tree
  ansible.builtin.debug:
    msg: "Modified files at runtime: {{ cli.git_status.modified }}"
  when: not cli.git_status.is_clean

- name: Use Semaphore context
  ansible.builtin.debug:
    msg: "Semaphore task: {{ cli.semaphore_vars }}"
  when: cli.semaphore_vars is not none
```

### Sample output

```yaml
ansible_playbook_cmd: "ansible-playbook site.yml -e @vars/prod.yml"
ansible_playbook_argv:
  - ansible-playbook
  - site.yml
  - -e
  - "@vars/prod.yml"
semaphore_vars:
  task_details:
    id: 4881
    username: ci-bot
git_status:
  branch: main
  commit: a1b2c3d4e5f67890abcdef1234567890abcdef12
  commit_short: a1b2c3d
  tag: v1.2.0-4-ga1b2c3d
  modified:
    - roles/app/tasks/main.yml
  staged: []
  untracked: []
  deleted: []
  is_clean: false
```

### Requirements

- Python 3.6+
- Ansible 2.9+
- `git` available on the controller node

---

## Audit Logging Framework

A logger-agnostic framework that collects execution metadata from `get_cli_args` and routes it to one or more logging backends in a single structured JSON payload.

### Backends supported

| Backend | File |
|---------|------|
| Local file (JSONL) | `loggers/file.yml` |
| Syslog | `loggers/syslog.yml` |
| rsyslog (remote UDP) | `loggers/rsyslog.yml` |
| Fluentd (HTTP) | `loggers/fluentd.yml` |
| Elasticsearch | `loggers/elasticsearch.yml` |
| Splunk HEC | `loggers/splunk.yml` |
| AWS CloudWatch | `loggers/cloudwatch.yml` |

### Directory structure

```
roles/common/tasks/
├── audit_logging.yml
└── loggers/
    ├── cloudwatch.yml
    ├── elasticsearch.yml
    ├── file.yml
    ├── fluentd.yml
    ├── rsyslog.yml
    ├── splunk.yml
    └── syslog.yml
```

### Usage

Add to your playbook's `post_tasks`:

```yaml
post_tasks:
  - name: Write audit log
    ansible.builtin.include_role:
      name: common
      tasks_from: audit_logging.yml
    apply:
      tags: always
    tags: always
```

Configure backends via `audit_loggers`:

```yaml
vars:
  audit_loggers:
    - file
    - elasticsearch
```

### Configuration variables

| Logger | Variable | Default | Description |
|--------|----------|---------|-------------|
| cloudwatch | `cloudwatch_profile` | `default` | AWS CLI profile |
| cloudwatch | `cloudwatch_log_group` | `/ansible/audit` | Log group name |
| syslog | `syslog_facility` | `local0` | Syslog facility |
| syslog | `syslog_priority` | `info` | Syslog priority |
| fluentd | `fluentd_url` | `http://localhost:9880` | Fluentd HTTP endpoint |
| fluentd | `fluentd_tag` | `ansible.audit` | Fluentd tag |
| elasticsearch | `elasticsearch_url` | *required* | Elasticsearch URL |
| elasticsearch | `elasticsearch_index` | `ansible-audit` | Index name |
| elasticsearch | `elasticsearch_user` | — | Basic auth username |
| elasticsearch | `elasticsearch_password` | — | Basic auth password |
| elasticsearch | `elasticsearch_validate_certs` | `true` | Validate TLS certs |
| splunk | `splunk_hec_url` | *required* | Splunk HEC URL |
| splunk | `splunk_hec_token` | *required* | HEC token |
| splunk | `splunk_sourcetype` | `ansible:audit` | Sourcetype |
| splunk | `splunk_index` | `ansible` | Splunk index |
| file | `audit_log_file` | `/var/log/ansible/audit.jsonl` | Log file path |
| rsyslog | `rsyslog_host` | *required* | Remote syslog host |
| rsyslog | `rsyslog_port` | `514` | Remote syslog port (UDP) |

### Sample audit payload

```json
{
  "timestamp": "2025-01-15T03:42:00Z",
  "user": "deploy-bot",
  "control_node": "ci-runner-01",
  "playbook": "Deploy application",
  "roles": "common, nginx, app",
  "hosts": ["web01.example.com", "web02.example.com"],
  "status": "success",
  "ansible_command": "ansible-playbook site.yml -e @vars/prod.yml",
  "semaphore_vars": null,
  "run_tags": "all",
  "skip_tags": "",
  "git_branch": "main",
  "git_commit": "a1b2c3d4e5f67890abcdef1234567890abcdef12",
  "git_tag": "v1.2.0",
  "git_is_clean": true,
  "git_modified_files": [],
  "git_staged_files": [],
  "git_untracked_files": []
}
```

### Adding a new logger

1. Create `roles/common/tasks/loggers/mylogger.yml`
2. Use `audit_log_json` (string) or `audit_log_data` (dict)
3. Add to the selection block in `audit_logging.yml`:

```yaml
- name: Send to MyLogger
  ansible.builtin.include_tasks: loggers/mylogger.yml
  when: audit_loggers is defined and 'mylogger' in audit_loggers
```

### Examples

See [`examples/playbooks/`](examples/playbooks/) for ready-to-use examples:

- `audit_single_logger.yml` — minimal file-based logging
- `audit_multi_logger.yml` — Elasticsearch + syslog + file
- `audit_semaphore.yml` — Semaphore CI/CD with auto-extracted task metadata

---

## Security considerations

- Use `no_log: true` on the audit block if your playbooks handle sensitive data
- Configure log retention policies on your backend
- Restrict access to audit logs (RBAC, IAM policies)
- Use TLS for all remote logging backends
- Never commit credentials — use `ansible-vault` for passwords and tokens

---

## Requirements

- Python 3.6+
- Ansible 2.9+
- `git` on the controller node

---

## Changelog

### v1.0.1
- `get_cli_args`: added `git_status` (branch, commit, tag, modified/staged/untracked/deleted, `is_clean`)
- Audit framework: uses `git_status` natively — removed redundant async shell git tasks

### v1.0.0
- Initial release: `get_cli_args` action plugin
- Audit logging framework with 7 logger backends

---

## License

MIT
