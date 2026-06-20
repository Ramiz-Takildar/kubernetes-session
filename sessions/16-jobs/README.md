# Session 16 — Jobs & CronJobs

## What You Will Learn

Jobs run one-time tasks to completion. CronJobs schedule Jobs to run repeatedly at defined times. Unlike Deployments, Jobs are meant to terminate after their work is done.

---

## Core Concepts

- **Job**: Runs a Pod until N completions succeed (not continuously running)
- `completions`: How many Pods must complete successfully for the Job to finish
- `parallelism`: How many Pods run concurrently
- `backoffLimit`: How many times to retry a failed Pod before marking the Job as failed
- **CronJob**: Schedules Jobs based on cron syntax (`分 时 日 月 周`)
- CronJobs use `jobTemplate.spec.template.spec` for the Job's pod spec
- Jobs do not restart automatically — they fail and optionally retry

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

1. Jobs run to completion (N times via `completions`), then stop
2. `restartPolicy: Always` is invalid for Jobs — use `OnFailure` or `Never`
3. CronJobs use a nested `jobTemplate` to define the Job spec
4. CronJob `schedule` follows standard cron syntax: `分 时 日 月 周`
5. CronJobs do not guarantee execution — they are "at least once"

---

## Cleanup

```bash
kubectl delete -f job.yaml -f cronjob.yaml

# Note: Deleting a CronJob does NOT delete its spawned Jobs.
# To clean up leftover Jobs created by the CronJob:
kubectl delete jobs -l app=hello-cron

# Or wait for Jobs to complete and be garbage collected
```