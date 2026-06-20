# Session 16 — Jobs & CronJobs

## What You Will Learn

Jobs run finite tasks to completion, making them ideal for batch processing, data migration, and one-off administrative operations. CronJobs extend this capability by scheduling Jobs to run repeatedly at specified intervals. Unlike Deployments, which keep Pods running indefinitely, Jobs and CronJobs are designed to start, complete, and terminate cleanly.

---

## Core Concepts

- A **Job** runs one or more Pods until a specified number of completions succeed, then stops—making it the right abstraction for batch workloads rather than long-running services
- `completions` defines how many successful Pod executions the Job needs before it is considered finished, enabling parallelized batch processing when combined with `parallelism`
- `parallelism` controls how many Pods run concurrently, allowing a Job to distribute work across multiple nodes and complete faster without creating individual Pod specs manually
- `backoffLimit` limits how many times Kubernetes retries a failed Pod before marking the entire Job as failed, preventing infinite retry loops on fundamentally broken workloads
- `restartPolicy: OnFailure` is **required** for Jobs because `Always` is incompatible with a controller that expects Pods to terminate; `Never` is also valid for one-shot tasks that should not retry
- A **CronJob** wraps a Job template and schedules it using standard cron syntax, automating recurring tasks like backups, report generation, or certificate renewal without external schedulers
- CronJobs do **not guarantee execution**; if the cluster is unavailable at the scheduled time, the run is skipped, so they are best suited for idempotent tasks that tolerate occasional misses

---

## YAML Walkthrough

### job.yaml

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: countdown
spec:
  completions: 3
  parallelism: 1
  backoffLimit: 2
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: counter
        image: busybox:1.36
        command: ['sh', '-c', 'for i in 3 2 1; do echo $i; sleep 1; done']
```

| Field | Meaning |
|-------|---------|
| `spec.completions` | Job succeeds after this many pods finish successfully |
| `spec.parallelism` | Number of pods that run concurrently |
| `spec.backoffLimit` | Max retries before Job is marked failed |
| `spec.template.spec.restartPolicy: OnFailure` | Restart container if it exits non-zero (required for Jobs) |
| `spec.template.spec.containers[].command` | The actual work the Job performs |

### cronjob.yaml

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: hello
            image: busybox:1.36
            command: ['sh', '-c', 'echo "Hello from CronJob at $(date)"']
```

| Field | Meaning |
|-------|---------|
| `spec.schedule` | Cron expression: `分 时 日 月 周` (`*/1 * * * *` = every minute) |
| `spec.jobTemplate` | Defines the Job to create (nested under CronJob) |
| `spec.jobTemplate.spec.template` | Pod template for each Job run |

---

## Step-by-Step Implementation

```bash
# 1. Apply the Job
cd sessions/16-jobs
kubectl apply -f job.yaml

# 2. Watch the Job complete all 3 iterations
kubectl get jobs -w
# Expected: 3 completions, STATUS = Complete

# 3. View the Job's logs
kubectl logs job/countdown
# Expected: 3, 2, 1 printed for each of the 3 completions

# 4. Apply the CronJob
kubectl apply -f cronjob.yaml

# 5. Verify the CronJob schedule and check for active jobs
kubectl get cronjob hello-cron
# Expected: Schedule = */1 * * * *, ACTIVE = 0 or 1

# 6. Wait for the first Job to run (wait ~60 seconds)
sleep 65 && kubectl get jobs -l app=hello-cron
# Expected: A new job appears, then completes

# 7. List all active Jobs (including those spawned by CronJob)
kubectl get jobs
```

---

## Verification Commands

| Command | What It Proves |
|---------|----------------|
| `kubectl get jobs` | Jobs exist and show completion status |
| `kubectl get jobs -w` | Watch mode: see completions increment to 3/3 |
| `kubectl logs job/countdown` | Job output is captured in logs |
| `kubectl get cronjob` | CronJob exists with correct schedule |
| `kubectl get jobs -l app=hello-cron` | CronJob-spawned jobs are labeled correctly |
| `kubectl describe cronjob hello-cron` | Last scheduled time, next run time |

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Job stuck in `Active` | Pods failing repeatedly, hitting backoffLimit | Check `kubectl describe job <name>` for events |
| `restartPolicy: Always` error | Jobs only support `OnFailure` or `Never` | Change to `OnFailure` or `Never` |
| CronJob not creating Jobs | CronJob controller not running | Check `kubectl get pods -n kube-system` |
| CronJob missed schedule | Cluster was down during the scheduled time | CronJob has no SLI guarantee for missed runs |

---

## Key Takeaways

1. Jobs run to completion (N times via `completions`), then stop, making them ideal for batch and administrative tasks
2. `restartPolicy: Always` is invalid for Jobs; you must use `OnFailure` or `Never`
3. CronJobs use a nested `jobTemplate` to define the Job spec and schedule it with standard cron syntax
4. `parallelism` allows Jobs to process work concurrently across multiple Pods without manual coordination
5. The `backoffLimit` prevents infinite retries on broken workloads by capping how many times a failed Pod is restarted
6. CronJobs are best suited for idempotent tasks because missed schedules are not automatically replayed when the cluster recovers

---

## Cleanup

```bash
kubectl delete -f job.yaml -f cronjob.yaml

# Note: Deleting a CronJob does NOT delete its spawned Jobs.
# To clean up leftover Jobs created by the CronJob:
kubectl delete jobs -l app=hello-cron

# Or wait for Jobs to complete and be garbage collected
```