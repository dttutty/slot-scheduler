# SchedLang Design Notes

This document defines a more systematic design for `SchedLang`, the experimental scheduling DSL that sits in front of `slot-scheduler`.

The current implementation is intentionally small. It already supports:

- `pool`
- `policy`
- `experiment`
- `matrix`

That is enough for a first compiler prototype, but it is not yet a complete language for expressing rich scheduling intent.

The goal of this document is to define:

1. the core object model of the language
2. the boundary between resource capability, task requirement, and scheduling preference
3. a plausible syntax direction for a richer next version

## Design Goal

`SchedLang` is meant to express scheduling intent for heterogeneous research compute, especially mixed environments with:

- local workers
- SSH-accessible servers
- Slurm-managed nodes
- preemptible cloud capacity such as EC2 Spot Instances
- shared hosts with policy limits
- tasks with different GPU and placement needs

The runtime should remain simple. The language exists so that users can declare what they need and why, while the compiler and scheduler decide where a task can go.

## Three Semantic Layers

The language should separate three different kinds of information.

| Layer | Meaning | Typical examples |
| --- | --- | --- |
| Resource capability | What the infrastructure has | `backend = ssh`, `provider = "runpod"`, `gpu_count = 8`, `gpu_mem_gb = 80`, `tags = ["shared", "a100"]` |
| Task requirement | What a task must have | `gpu_count >= 4`, `backend in ["slurm", "ssh"]`, `provider in ["runpod", "vast"]`, `host_tags contains "a100"` |
| Scheduling preference | What the scheduler should try first | `prefer local`, `avoid shared`, `spread across hosts`, `pack small jobs` |

This separation is important because the same phrase can mean very different things.

For example:

- `"sun is shared"` is a resource fact
- `"this task must not run on shared hosts"` is a task requirement
- `"prefer non-shared hosts, but fall back to shared hosts if needed"` is a scheduling preference

For preemptible resources, the same separation still matters:

- `"this pool is Spot-backed"` is a resource fact
- `"this task tolerates preemption"` is a task requirement or capability declaration
- `"prefer On-Demand, but spill to Spot"` is a scheduling preference

## Lifecycle and Preemption Semantics

In addition to placement, a practical scheduling language also needs to describe lifecycle behavior.

This becomes especially important for:

- EC2 Spot Instances
- preemptible VMs in other clouds
- low-priority or revocable cluster queues

For these resources, the question is not only "where can this task run?" but also:

- what should happen when the resource is about to disappear
- whether the task can recover
- where recoverable state should be written
- whether the task should restart or resume

This suggests a fourth conceptual dimension:

| Lifecycle concern | Meaning | Typical examples |
| --- | --- | --- |
| Preemption semantics | What can interrupt the resource | `market = spot`, `preemptible = true`, `rebalance_signal = true` |
| Drain semantics | What to do before the resource disappears | `stop_accepting_new_work`, `checkpoint-and-exit`, `flush-metrics` |
| Recovery semantics | How the task continues elsewhere | `resume-from-checkpoint`, `restart-from-scratch`, `requeue-on-preemption` |

This layer cuts across the previous three semantic layers, because it involves:

- resource facts
- task capabilities
- scheduler and runtime policy

## Core Object Model

The language should revolve around three groups of objects.

### Resource-side objects

#### `HostCapability`

Describes a machine as a scheduling target.

Suggested fields:

- `name`
- `backend`
- `provider`
- `address` or `alias`
- `gpu_count`
- `gpu_mem_gb`
- `cpu_count`
- `ram_gb`
- `interconnect`
- `tags`
- `shared`
- `labels`
- `market`
- `preemptible`
- `interruption_behavior`
- `rebalance_signal`

Example meaning:

- `sun` is an SSH host with 8 GPUs
- `moon` is an SSH host with 2 GPUs
- `gpu1-003` is a Slurm-reachable single-GPU node
- a RunPod worker can still use `backend = "ssh"` while setting `provider = "runpod"`
- an EC2 worker pool might be `market = "spot"` and `preemptible = true`

#### `SlotCapability`

Describes an allocatable unit inside a host.

Suggested fields:

- `name`
- `host`
- `gpu_indices`
- `exclusive`
- `capacity`
- `run_root`
- `workdir`
- `tags`

Today, `slot-scheduler` effectively uses one active job per slot. In a future version, `capacity` could allow multiple small jobs per slot when explicitly enabled.

#### `Pool`

A reusable named set of resource-side selectors or task-side defaults.

Pools are useful when many tasks share the same scheduling envelope, for example:

- all SSH machines in one lab
- all hosts with A100 GPUs
- all shared hosts that should be capped

### Task-side objects

#### `TaskTemplate`

Describes a family of tasks before expansion.

Suggested fields:

- `name`
- `run`
- `env`
- `cwd`
- `matrix`
- `retries`
- `requirements`
- `preferences`
- `checkpoint`
- `resume_policy`
- `preemption_policy`

This is close to the current `experiment` block.

#### `TaskInstance`

A concrete job after matrix expansion and template substitution.

Suggested fields:

- `name`
- `resolved_command`
- `resolved_env`
- `requirements`
- `preferences`
- `retry_budget`

The runtime should schedule `TaskInstance`s, not templates.

#### `RequirementSet`

The hard constraints a task must satisfy.

Suggested fields:

- `backend`
- `host`
- `provider`
- `pool`
- `slot`
- `gpu_count`
- `gpu_mem_gb`
- `cpu_count`
- `ram_gb`
- `required_tags`
- `required_labels`
- `co_located`
- `same_host`

If a requirement is not satisfied, the task is unschedulable for that placement.

#### `CheckpointPolicy`

Describes how a task persists recoverable state.

Suggested fields:

- `path`
- `storage`
- `interval`
- `on_rebalance`
- `on_interruption`
- `finalize_artifacts`

The main design goal is to avoid last-minute rescue logic.

In other words, the task should usually write important state continuously to durable storage, instead of relying on emergency file collection during a two-minute preemption window.

#### `ResumePolicy`

Describes how a task should recover after interruption.

Suggested fields:

- `mode`
- `checkpoint_selector`
- `max_resume_age`
- `requeue_on_preemption`

Typical modes:

- `restart`
- `resume`
- `resume-if-possible`

### Lifecycle-side objects

#### `PreemptionPolicy`

Describes what the runtime should do when a task is running on preemptible capacity.

Suggested fields:

- `tolerates_preemption`
- `preferred_market`
- `fallback_market`
- `on_rebalance`
- `on_interruption`
- `max_preemptions`

This policy answers questions such as:

- may this task run on Spot at all
- should the scheduler prefer Spot or avoid it
- should the runtime checkpoint and requeue, or fail immediately
- should the task be retried on Spot again, or escalated to On-Demand

#### `DrainPolicy`

Describes the sequence of actions during graceful shutdown of an at-risk worker.

Suggested fields:

- `stop_new_work`
- `checkpoint_timeout_sec`
- `flush_logs`
- `upload_small_artifacts`
- `graceful_exit`

This object is intentionally separate from `CheckpointPolicy`.

`CheckpointPolicy` describes what state matters.
`DrainPolicy` describes the operational order in which shutdown should happen.

### Scheduler-side objects

#### `HostPolicy`

Operational limits attached to resources.

Examples:

- `max_active_slots = 4`
- `max_active_fraction = 0.5`
- `reserved_for = ["large_jobs"]`

This already exists in the current runtime in a minimal form.

#### `PreferenceSet`

Soft constraints that rank valid placements.

Examples:

- prefer hosts with `a100`
- prefer `runpod` over other marketplace providers
- avoid `shared`
- prefer local or low-latency hosts
- spread replicas across hosts
- pack small jobs onto a subset of machines

Unlike requirements, preferences do not make a placement illegal. They only affect ranking.

#### `PlacementDecision`

A scheduler output that ties together:

- one `TaskInstance`
- one candidate host/slot allocation
- an explanation for why this placement won

This becomes especially important once the language supports richer constraints.

## Hard Constraints vs Soft Preferences

The language should make this distinction explicit.

### Hard constraints

Hard constraints determine feasibility.

Examples:

- needs 4 GPUs
- needs at least 40 GB GPU memory
- must run on `ssh` or `slurm`
- must run on a host tagged `multi-gpu`

If no host satisfies these constraints, the scheduler should say so directly.

### Soft preferences

Soft preferences determine ranking among feasible placements.

Examples:

- prefer `a100`
- prefer local hosts
- avoid shared machines
- spread this sweep across hosts

If a preference cannot be satisfied, the task should still run on the best remaining legal placement.

## Spot and Other Preemptible Resources

For EC2 Spot Instances and similar capacity, the language should not treat interruption as an afterthought.

The preferred model is:

1. provision preemptible capacity through a separate pool or provisioner
2. register the resulting workers as runtime slots
3. watch for rebalance and interruption signals
4. drain the task
5. checkpoint durable state
6. requeue or resume elsewhere

This is usually better than trying to "collect files back" only after the preemption notice arrives.

The last-minute window is too small and too unreliable to be the primary durability mechanism.

Instead, the language should assume:

- important state is checkpointed continuously
- small final artifacts may still be uploaded during drain
- runtime recovery is primarily `checkpoint + requeue`, not emergency rsync

## Compilation Pipeline

A future `SchedLang` compiler should look conceptually like this:

1. Parse `.sched` source into an AST.
2. Normalize the AST into a typed IR with explicit resource objects, task objects, and policy objects.
3. Expand matrices into `TaskInstance`s.
4. Resolve reusable pools and defaults.
5. Build candidate placements by matching task requirements against resource capabilities.
6. Rank candidates using preference rules and host policies.
7. Attach lifecycle and recovery metadata to each compiled task.
8. Emit:
   - compiled `jobs.yaml`
   - optionally a derived `inventory.yaml`
   - optionally a placement plan or explanation report
   - optionally a drain/recovery plan for preemptible tasks

The current prototype implements only a smaller slice of this pipeline:

- parse -> AST-like document
- compile experiments into `jobs.yaml`
- compile host policies into derived `inventory.yaml`

It does not yet implement:

- typed resource declarations inside the DSL
- requirement solving beyond basic filters
- preference scoring
- explanation output
- preemption-aware recovery planning

## Mapping the Current Prototype

The current prototype maps to the larger design like this.

| Current construct | Rough role in the long-term model |
| --- | --- |
| `pool` | reusable task-side defaults such as `backends`, `required_tags`, or `slots` |
| `policy` | host-level operational caps |
| `experiment` | task template |
| `matrix` | task-instance expansion |

What is still missing is an explicit way to describe resource capability in the DSL itself.

Right now, resource capability mostly lives in `inventory.yaml`. That is fine for the short term, but a richer language likely needs a more direct resource model.

## Proposed Syntax Direction

The following syntax is a design sketch, not current implemented syntax.

### Resource declarations

```text
host sun {
  backend = "ssh"
  alias = "sun"
  gpu_count = 8
  gpu_mem_gb = 24
  cpu_count = 128
  tags = ["shared", "lab", "txstate"]
}

host moon {
  backend = "ssh"
  alias = "moon"
  gpu_count = 2
  gpu_mem_gb = 48
  tags = ["shared", "large-mem", "txstate"]
}
```

### Resource groups

```text
pool txstate_shared {
  hosts = ["sun", "moon", "gauss", "markov"]
  backends = ["ssh"]
}
```

### Operational policies

```text
policy shared_half {
  hosts = ["sun", "moon"]
  max_active_fraction = 0.5
}
```

### Preemptible cloud pools

```text
pool aws_spot_train {
  backend = "ec2-fleet"
  market = "spot"
  preemptible = true
  allocation_strategy = "price-capacity-optimized"
  rebalance_signal = true
}
```

### Task templates with explicit requirements and preferences

```text
experiment train_large {
  use_pool = "txstate_shared"

  requires {
    gpu_count >= 4
    backend in ["ssh", "slurm"]
    host_tags contains "multi-gpu"
  }

  prefers {
    host_tags contains "a100"
    avoid_host_tags = ["shared"]
    placement = "spread"
  }

  env {
    OMP_NUM_THREADS = "16"
  }

  checkpoint {
    path = "s3://my-bucket/checkpoints/${task}"
    interval = "5m"
    on_rebalance = "save"
    on_interruption = "save"
  }

  preemption {
    tolerates_preemption = true
    on_rebalance = "checkpoint-and-requeue"
    on_interruption = "checkpoint-and-exit"
  }

  resume {
    mode = "resume-if-possible"
    requeue_on_preemption = true
  }

  run = """
bash -lc "python train.py --model big"
"""
}
```

### Matrix expansion

```text
experiment vlmlp_grid {
  use_pool = "txstate_shared"

  matrix {
    dataset = ["ETTh2", "ETTm2"]
    pred_len = [96, 192, 336, 720]
    seed = [1, 2, 3]
  }

  requires {
    gpu_count >= 1
  }

  prefers {
    placement = "spread"
  }

  checkpoint {
    path = "s3://my-bucket/vlmlp/${dataset}/${pred_len}/${seed}"
    interval = "10m"
  }

  resume {
    mode = "resume"
    requeue_on_preemption = true
  }

  run = """
bash -lc "uv run python run_experiment.py ${dataset} ${pred_len} --seed ${seed}"
"""
}
```

## Suggested Near-term Language Features

If the language grows gradually, the next versions should probably come in this order.

### V0: current prototype

- `pool`
- `policy`
- `experiment`
- `matrix`
- compile to existing YAML runtime

### V1: typed requirements

- `requires { ... }`
- explicit resource fields like `gpu_count`, `gpu_mem_gb`, `backend`
- explicit host and slot selectors
- better validation of unschedulable tasks

### V2: preferences and explainability

- `prefers { ... }`
- ranking and tie-breaking rules
- `explain` output for placement and rejection
- richer pool composition

### V3: preemption-aware execution

- `checkpoint { ... }`
- `preemption { ... }`
- `resume { ... }`
- compile-time reports for `ready`, `unschedulable`, and `needs-multi-slot-runtime`
- drain and recovery plans for Spot-like resources

### V4: global strategy profiles

- `strategy = "greedy" | "spread" | "pack" | "fair-share"`
- reservation and anti-affinity
- limited multi-slot or multi-host placement semantics

## Explainability

A rich scheduling language should not only produce decisions. It should explain them.

Examples:

```text
task train_large is unschedulable:
- requires gpu_count >= 4
- requires host_tags contains "multi-gpu"
- no host in pool txstate_shared satisfies both constraints
```

```text
task vlmlp_ETTh2_96_s1 scheduled on sun-g2:
- satisfies backend = ssh
- satisfies gpu_count >= 1
- matches preferred tag txstate
- moon is deprioritized by shared-host policy
```

```text
task train_large on spot-worker-17 is draining:
- pool market = spot
- rebalance recommendation received
- checkpoint path = s3://my-bucket/checkpoints/train_large
- action = checkpoint-and-requeue
```

Without explanations, a more expressive language quickly becomes difficult to trust.

## Design Principle

The main principle is:

> describe intent in the DSL, keep execution simple in the runtime

`slot-scheduler` should continue to be a transparent runtime. `SchedLang` should provide the higher-level front-end that captures:

- what resources exist
- what tasks require
- what placements are preferred
- what operational policies must be respected

That separation keeps the system understandable while still allowing the language to grow.
