# 12 Authentication and security

## Slurm

### User authentication

Slurm relies on the underlying operating system for user authentication. Users must have valid accounts on the cluster's login nodes and compute nodes.

- **OS-level authentication**: Slurm relies on the operating system's authentication mechanism. Users must have valid Unix accounts on the cluster. If the OS uses PAM, LDAP, or Kerberos, Slurm inherits that authentication automatically. Slurm does not implement its own authentication system.
- **LDAP/Kerberos**: If the cluster uses LDAP or Kerberos for centralized authentication, users authenticate through these systems. Slurm inherits the authentication mechanism configured at the OS level.
- **User identity**: Slurm identifies users by their Unix UID/GID. The `SLURM_JOB_USER` environment variable contains the username of the job submitter.

### Login nodes

Login nodes serve as the entry point for users to interact with the cluster.

- **Purpose**: Login nodes are where users submit jobs (`sbatch`), query the queue (`squeue`), and access shared filesystems. They are **not** for running compute workloads.
- **SSH access**: When `salloc` creates an allocation, users can SSH directly to allocated nodes listed in `SLURM_NODELIST`.
- **Example: accessing allocated nodes** (from login node after salloc):
  ```bash
  # Allocate resources interactively
  salloc --nodes=2 --time=1:00:00
  
  # SSH to first allocated node
  ssh $SLURM_NODELIST
  
  # Or use srun to run commands on allocated nodes
  srun hostname
  ```

### Network security and ports

Slurm daemons communicate over specific network ports that must be accessible within the cluster.

- **Port requirements**: Slurm daemons communicate over specific network ports. The controller listens for client connections and node communication, compute node daemons listen for controller instructions, and the database daemon listens for controller connections. These ports must be open between the controller and compute nodes. Login nodes need access to the controller port.

### MUNGE authentication

MUNGE (MUNGE Uid 'N' Gid Emporium) provides secure authentication between Slurm daemons.

- **Token-based communication**: MUNGE generates cryptographic tokens that authenticate messages between `slurmctld`, `slurmd`, and `slurmdbd`. This prevents unauthorized nodes from joining the cluster.
- **Shared secret**: All nodes must share the same MUNGE key. The key must be identical across all nodes in the cluster.
- **Key distribution**: The MUNGE key must be securely distributed to all nodes. It should have restricted permissions (readable only by the `munge` user).
- **Example: MUNGE key location** (on all nodes):
  ```bash
  # MUNGE key typically located at
  /etc/munge/munge.key
  
  # Permissions should be restricted
  chmod 400 /etc/munge/munge.key
  chown munge:munge /etc/munge/munge.key
  ```
- **Token expiration**: MUNGE tokens have a limited lifetime (configurable), requiring periodic re-authentication.
- **Token refresh**: MUNGE automatically refreshes tokens before expiration. Daemons maintain authenticated sessions by periodically exchanging new tokens. If a token expires, the daemon requests a new one transparently.

### User impersonation

Slurm runs user jobs with the identity of the submitting user, not as root or a service account.

- **Mechanism**: `slurmstepd` runs as root (or a configured service user). When launching a job, it uses `setuid()` to switch to the job owner's UID/GID. The user's process runs with their own permissions, not root privileges.
- **Security isolation**: This ensures that users cannot access other users' files or processes. Each job runs with the permissions of its owner.
- **Environment variables**: The job inherits the user's environment, including `HOME`, `USER`, and other standard Unix variables.
- **File permissions**: Jobs can only write to directories where the user has write permissions.

### Security mechanisms

- **Job submission authentication**: When users run `sbatch`, `squeue`, or other Slurm commands, they authenticate to `slurmctld` using their Unix UID/GID. The controller verifies the user's identity and checks association limits before accepting the job submission.
- **User limits**: Slurm's association limits (`MaxJobs`, `MaxSubmitJobs`) configured in the database prevent users from overwhelming the scheduler or exhausting resources.
- **Security audit logging**: Slurm logs authentication events, job submissions, and administrative actions.

## dstack

### User authentication and projects

dstack uses a token-based authentication system with project-based isolation.

- **Projects**: Projects enable isolation of different teams and their resources. Each project can configure its own backends, gateways, and control which users have access to it. Projects are managed via the UI or REST API.
- **Users**: Users are created and managed within the dstack server. Each user can be assigned to one or more projects with specific roles.
- **User tokens**: When a user is created, they are issued a token. This token is used for authentication when logging into the control plane UI and when using the CLI or API. The token can be found on the user account settings page and can be refreshed if needed.
- **Project roles**: Users can be assigned different roles within a project:
  - **Admin** – Can manage project settings, including backends, gateways, and members.
  - **Manager** – Can manage project members but cannot configure backends and gateways.
  - **User** – Can manage project resources including runs, fleets, and volumes.
- **Global admins**: A user can be assigned a global admin role, which allows them to manage all projects and users. This can only be done by another global admin.

### Encryption

By default, dstack stores data in plaintext. To enforce encryption, you can configure encryption keys in the server configuration.

- **AES encryption**: dstack supports AES-256 encryption in GCM mode. To configure it, generate a random 32-byte key and specify it in the server configuration file (`~/.dstack/server/config.yml`).
- **Key rotation**: If multiple encryption keys are specified, the first is used for encryption, and all are tried for decryption. This enables key rotation by specifying a new encryption key while keeping old keys for decryption.
- **Secrets encryption**: By default, secrets are stored in plaintext in the database. When server encryption is configured, secrets are stored encrypted. The encryption configuration applies to all sensitive data stored by the server.

### SSH keys

dstack manages SSH keys at both the user and project levels to enable secure access to runs and instances.

- **User SSH keys**: Each user has an SSH key pair (public and private) that is automatically generated by the server. The user's SSH key is used when attaching to runs via `dstack attach` or `dstack apply`. By default, dstack downloads the user SSH key from the server, but you can override this via the `--ssh-identity` argument in both `dstack apply` and `dstack attach`. The user SSH key is only added to the host while the run is active. The CLI automatically downloads and manages user SSH keys, refreshing them periodically.
- **Project SSH keys**: Each project also has its own SSH key pair. The project SSH key is used by the server to establish SSH connections to provisioned instances and containers. This allows the server to manage instances without requiring users to share their personal SSH keys.
- **Automatic management**: dstack automatically generates, distributes, and manages SSH keys.

### Secrets

Secrets allow centralized management of sensitive values such as API keys and credentials.

- **Project-scoped**: Secrets are project-scoped and managed by project admins. They can be set, listed, retrieved, and deleted using the `dstack secret` CLI commands.
- **Usage in runs**: Secrets can be referenced in run configurations using the `${{ secrets.<secret_name> }}` syntax. Currently, secrets interpolation is supported in `env` and `registry_auth` properties, but not in `commands`.
- **Encryption**: By default, secrets are stored in plaintext in the database. When server encryption is configured, secrets are stored encrypted. The encryption ensures that sensitive values are protected at rest.

### Environment variables

Environment variables can be defined in run configurations to pass values to runs.

- **Defining in configuration**: Use the `env` property to define environment variables. You can either assign a value directly in the configuration or list only the variable name without a value.
- **Runtime values**: If you don't assign a value to an environment variable in the configuration, dstack requires the value to be passed via the CLI using `-e` or `--env` flags, or set in the current process environment. For example: `dstack apply -e HF_TOKEN=<token>`.
- **Not encrypted**: Unlike secrets, environment variables are not encrypted. They are passed as plaintext to the run. Use secrets for sensitive values that require encryption.
- **Example**:
  ```yaml
  type: task
  env:
    - HF_TOKEN
  commands:
    - pip install -U huggingface_hub
    - huggingface-cli download meta-llama/Llama-2-7b-hf
  ```

### Private Docker registry authentication

When pulling private Docker images from registries like Docker Hub, GitHub Container Registry, NVIDIA NGC, or other private registries, dstack needs credentials to authenticate with the registry.

- **Registry authentication**: Use the `registry_auth` property in run configurations to provide registry credentials. The `username` and `password` fields can reference secrets using the `${{ secrets.<secret_name> }}` syntax.
- **Example: NVIDIA NGC** (first set the secret, then reference it in the configuration):
  ```bash
  $ dstack secret set ngc_api_key <your-ngc-api-key>
  ```
  ```yaml
  type: task
  image: nvcr.io/nvidia/pytorch:24.01-py3
  registry_auth:
    username: $oauthtoken
    password: ${{ secrets.ngc_api_key }}
  commands:
    - python -c "import torch; print(torch.__version__)"
  ```

### Repo authentication

When using the `repos` feature to clone Git repositories in runs, dstack needs credentials to access private repositories.

- **Automatic detection**: dstack automatically tries to use your default Git credentials from `~/.ssh/config` or `~/.config/gh/hosts.yml`.
- **Custom credentials**: To provide custom Git credentials, use the `dstack init` command. You can set credentials with `--git-identity` (private SSH key) or `--token` (OAuth token). Run `dstack init` in the repo's directory, or pass the repo path or URL with `--repo`.
- **SSH keys for Git**: If using SSH keys for Git authentication, dstack uses the provided SSH key to clone private repositories within the container.
- **OAuth tokens for Git**: Alternatively, you can use OAuth tokens (e.g., GitHub personal access tokens) for Git authentication. This is useful when SSH keys are not available or preferred.

### SSO integration

SSO integration is only supported with dstack Sky and dstack Enterprise.

