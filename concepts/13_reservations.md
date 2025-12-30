# 13 Reservations

## Slurm

### Reservation types

Reservations allow administrators to reserve specific nodes or resources for exclusive use during a defined time period.

- **Advanced reservations**: One-time reservations for a specific start and end time.
- **Standing reservations**: Recurring reservations that repeat on a schedule using flags for weekly, daily, or hourly repetition in combination with `StartTime` and `EndTime`.
- **Node reservations**: Reserve specific nodes by name or by features (e.g., all nodes with a specific GPU model).
- **Partition reservations**: Reserve an entire partition for exclusive use.

### Reservation mechanics

Reservations interact with the scheduler to block resources from normal job allocation.

- **Resource blocking**: When a reservation is active, the specified nodes are marked as `RESERVED` and cannot be allocated to regular jobs. The scheduler treats reserved nodes as unavailable.
- **Scheduling interaction**: The scheduler accounts for reservations when calculating when jobs can start. Jobs that would overlap with a reservation remain `PENDING` until the reservation ends or is cancelled.
- **Preemption**: Reservations can preempt running jobs if configured. When a reservation starts, jobs running on reserved nodes may be cancelled or requeued.
- **Backfill impact**: Backfill scheduling respects reservations. The scheduler will not backfill jobs that would conflict with upcoming reservations.

### Reservation management

Reservations are created and managed using `scontrol` commands. Reservation flags include automatically replacing failed nodes, marking nodes for maintenance, recurring schedules (weekly, daily, hourly), and controlling job hold behavior.

### Reservation states and lifecycle

Reservations transition through states as they are created, activated, and completed.

- **INACTIVE**: Reservation is defined but not yet active (future start time).
- **ACTIVE**: Reservation is currently active and blocking resources.
- **COMPLETE**: Reservation has ended. Nodes return to normal scheduling.
- **Cancellation**: Administrators can cancel reservations before they start or while active. Cancelled reservations release nodes immediately.
- **Example: creating reservation** (from controller or login node, admin only):
  ```bash
  # Create one-time reservation for specific nodes
  scontrol create reservation ReservationName=maintenance \
    StartTime=2024-01-15T10:00:00 EndTime=2024-01-15T14:00:00 \
    Nodes=gpu-node[01-04] Users=admin
  
  # Create standing reservation (weekly)
  scontrol create reservation ReservationName=weekly-training \
    StartTime=2024-01-15T09:00:00 EndTime=2024-01-15T17:00:00 \
    Nodes=gpu-node[05-10] Flags=WEEKLY
  
  # Show reservation details
  scontrol show reservation maintenance
  
  # Cancel reservation
  scontrol delete reservation maintenance
  ```

## dstack

dstack does not support Slurm-style time-based reservations that block specific nodes or resources from normal job allocation during defined time periods. However, dstack supports cloud provider reservations for AWS and GCP, which are created and managed through the cloud provider's interface. 

### Cloud reservations

dstack can provision instances from these reservations by referencing a reservation ID. Unlike Slurm, dstack does not block resources from other jobs when a reservation is active—reservations only determine which instances can be provisioned, not which jobs can run.

To use a cloud reservation, specify the `reservation` parameter in your fleet or profile configuration:
- **AWS**: Use the capacity reservation ID (e.g., `cr-0123456789abcdef0`) for Capacity Reservations (billed at on-demand rates) or Capacity Blocks (time-based, discounted rates for ML workloads)
- **GCP**: Use the full reservation path (e.g., `projects/my-project/reservations/my-reservation`) or just the reservation name

**Example**:
```yaml
type: fleet
name: my-fleet
nodes: 2
placement: cluster
backends: [aws]
resources:
  gpu: A100:80GB:8
reservation: cr-0123456789abcdef0  # AWS capacity reservation ID
```

