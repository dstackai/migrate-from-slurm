# 14 Reservations

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

## dstack

TBA

