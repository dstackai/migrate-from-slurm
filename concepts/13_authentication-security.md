# 13 Authentication and security

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

### Network security and ports

Slurm daemons communicate over specific network ports that must be accessible within the cluster.

- **Port requirements**: Slurm daemons communicate over specific network ports. The controller listens for client connections and node communication, compute node daemons listen for controller instructions, and the database daemon listens for controller connections. These ports must be open between the controller and compute nodes. Login nodes need access to the controller port.

### MUNGE authentication

MUNGE (MUNGE Uid 'N' Gid Emporium) provides secure authentication between Slurm daemons.

- **Token-based communication**: MUNGE generates cryptographic tokens that authenticate messages between `slurmctld`, `slurmd`, and `slurmdbd`. This prevents unauthorized nodes from joining the cluster.
- **Shared secret**: All nodes must share the same MUNGE key. The key must be identical across all nodes in the cluster.
- **Key distribution**: The MUNGE key must be securely distributed to all nodes. It should have restricted permissions (readable only by the `munge` user).
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

TBA

