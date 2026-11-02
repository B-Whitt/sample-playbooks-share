# EDA activation log purge examples

These playbooks queue log purges through `POST /api/eda/v1/logs/purge/`.
They support **Bearer token** or **Basic Auth** — each playbook auto-detects
which method to use based on the variables you supply. The API requires a
system administrator account and returns HTTP `202` with a job ID when it
queues the purge. A queued job is not a completed purge.

| Playbook | Scope |
| --- | --- |
| [Global Purge with retention-duration](global_purge_with_retention_duration.yml) | Logs older than the retention duration across all processes, including logs without a process and earlier audit markers. |
| [Purge a specific Activation by Name](purge_activation_by_name.yml) | Logs older than the retention duration for activations matching the name. |
| [Purge a specific Activation by ID](purge_activation_by_id.yml) | Logs older than the retention duration for the activation ID. |

## Authentication

Each playbook supports two authentication methods. Provide **one** of the
following:

| Method | Variables | Notes |
| --- | --- | --- |
| **Bearer token** | `aap_token` | OAuth2 or Personal Access Token. Set this and omit `aap_username`/`aap_password`. |
| **Basic Auth** | `aap_username` + `aap_password` | System administrator credentials. Used when `aap_token` is not set. |

A pre-flight `assert` task fails the playbook early if neither method is
provided. See [sample-eda-purge-vars.yml](sample-eda-purge-vars.yml) for an
example vars file showing both options.

## Inputs

All three playbooks use these variables:

| Variable | Purpose | Default |
| --- | --- | --- |
| `aap_hostname` | Base URL of the AAP instance, without a trailing slash. Replace the example value before running. | `https://example.com` |
| `aap_token` | OAuth2 or Personal Access Token for Bearer auth. | Optional; commented out |
| `aap_username` | System administrator username for Basic Auth. | `admin` |
| `aap_password` | Password for that account. Supply through a protected credential or encrypted vars file. | `password123` |
| `aap_validate_certs` | Whether to verify the server's TLS certificate. | `true` |
| `retention_duration` | Positive ISO 8601 duration. Only logs older than this duration are selected. For example, `P30D` means 30 days and `PT5M` means five minutes. | `P30D` |

The name playbook uses `activation_names`, and the ID playbook uses
`activation_ids`. Both default to example placeholders; replace them
with existing activations before running. The API rejects unknown names or
IDs. A name may match more than one activation; use the ID playbook when you
need to select exactly one activation.

## Run

Supply the connection and credentials through your AAP job template or an
encrypted Ansible Vault vars file. For a command-line run:

**Using Basic Auth:**

```sh
ansible-playbook global_purge_with_retention_duration.yml --ask-vault-pass -e @/secure/path/eda-purge-vars.yml

ansible-playbook purge_activation_by_name.yml -e @/secure/path/eda-purge-vars.yml -e '{"activation_names": ["my-activation"]}'

ansible-playbook purge_activation_by_id.yml -e @/secure/path/eda-purge-vars.yml -e '{"activation_ids": [42]}'
```

**Using a Bearer token:**

```sh
ansible-playbook global_purge_with_retention_duration.yml -e @/secure/path/eda-purge-vars.yml -e aap_token="YOUR_TOKEN"

ansible-playbook purge_activation_by_name.yml -e @/secure/path/eda-purge-vars.yml -e aap_token="YOUR_TOKEN" -e '{"activation_names": ["my-activation"]}'

ansible-playbook purge_activation_by_id.yml -e @/secure/path/eda-purge-vars.yml -e aap_token="YOUR_TOKEN" -e '{"activation_ids": [42]}'
```

Set `retention_duration` in the vars file or pass it as an extra variable to
override the 30-day example. Review the scope and cutoff before submitting a
purge. Each request sets `audit_trail: true`, which creates one new audit marker
for each process whose logs were deleted. The global purge also removes older
audit markers that fall before the cutoff.
