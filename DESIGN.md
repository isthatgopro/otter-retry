# DESIGN.md

## Proposal: two-tier retry backoff model to support Otter
1. **Task-local resilience stays, but remains narrow**
   Some tasks already know how to deal with their own protocol-specific issues. For example, a task that talks to one API may already know how to retry a small HTTP request safely. That kind of logic can stay where it is useful.

2. **Otter owns the outer retry contract**
   Otter should provide the final, uniform retry behavior for task attempts.
   Including:
   1. bounded retry attempts
   2. backoff between attempts
   3. selective retry based on error type
   4. consistent logging
   5. manifest provenance for every attempt
   6. support for failures in both `run()` and `validate()`

--> In this design, local retries are tactical, while Otter retries are authoritative.

---

## How it works

Otter is intentionally designed to be small. Its current strengths are that tasks are declared in YAML, execution is straightforward and provenance is captured in the manifest. A retry design should feel like a natural extension of those ideas rather than a separate subsystem bolted on from a larger workflow platform.

### 1. Retry scope

Retries should apply to a **task attempt**, not to the whole step.

That means if one task fails, Otter retries that task only. It does not restart the whole step and it does not repeat tasks that have already succeeded.


### 2. Retry points

Otter should retry when either of these phases fails: 1. `run()` and 2. `validate()`. This is important because a task may complete its main work successfully and still fail during validation.

### 3. Retry policy

A retry policy should be configurable, small and explicit.

Example:
```yaml
steps:
  so:
    - name: copy sequence ontology
      source: https://raw.githubusercontent.com/The-Sequence-Ontology/SO-Ontologies/master/Ontology_Files/so.json
      destination: input/so/so.json
      retry:      ## <--- Started from here
          max_attempts: 3
          initial_delay_seconds: 1
          backoff: exponential
          max_delay_seconds: 30
          jitter: true
```

### 4. Selective retry

Retries should be selective rather than universal.

- Otter should retry only failures that are likely to be transient. 

    Examples:

        1. temporary HTTP transport errors
        2. timeouts
        3. connection resets
        4. temporary 5xx responses
        5. maybe 429 rate limit responses

- Otter should usually not retry failures that are likely deterministic. 

    Examples:

        1. invalid configuration
        2. schema mismatch
        3. malformed input that will not fix itself
        4. missing required fields
        5. permanent 404 style input mistakes

- Implement a small helper method to differentiate between retryable and non-retryable failures, i.e. `RetryableTaskError` / `NonRetryableTaskError`/ `task.is_retryable(error)`.


### 5. Backoff

I would use exponential backoff with optional jitter.

- Reason:

        1. constant retries are easy to create but less friendly to remote systems
        2. exponential backoff is simple and common
        3. jitter reduces retry storms if multiple workers fail at once

--> calculate the next delay centrally rather than pushing this responsibility into each task.

### 6. Logging

Otter should log every attempt in a consistent way.

- For each attempt, I would record:

        1. task name
        2. phase (run or validate)
        3. attempt number
        4. max attempts
        5. exception type
        6. short exception message
        7. delay before next retry
        8. final outcome

--> make retries visible without forcing every task author to build their own logging story.

### 7. Manifest provenance

Retries should also be recorded in the manifest, not only in console logs.

- I would add an attempt history structure to each task entry, something like:

    ``` json
    {
      "task": "copy sequence ontology",
      "result": "success",
      "attempts": [
        {
          "phase": "run",
          "attempt": 1,
          "result": "failed",
          "error_type": "HTTPError",
          "error_message": "503 Service Unavailable",
          "retryable": true,
          "next_delay_seconds": 1
        },
        {
          "phase": "run",
          "attempt": 2,
          "result": "failed",
          "error_type": "HTTPError",
          "error_message": "503 Service Unavailable",
          "retryable": true,
          "next_delay_seconds": 2
        },
        {
          "phase": "run",
          "attempt": 3,
          "result": "success"
        },
        {
          "phase": "validate",
          "attempt": 1,
          "result": "success"
        }
      ]
    }
    ```

---

## Comparison with other frameworks

| Framework | Retry Scope | Selective Retry | Backoff Controls | What I would Borrow|
| -------- | -------- | -------- |--------  |--------  |
| [Airflow](https://github.com/apache/airflow)     | task instance     | mostly coarse-grained task config     | retries, retry_delay, exponential backoff, max retry delay | task-level outer retry|
| [Celery](https://github.com/celery/celery)     | task execution     | exception-based autoretry     | max_retries, retry_backoff, retry_jitter| explicit retryable exception patterns|
| [Dagster](https://github.com/dagster-io/dagster)     | op or job     | policy-based or manual retry request     | max_retries, delay, backoff, jitter | clean policy object and optional manual control |
| [Prefect](https://github.com/prefecthq/prefect)     | task inside a flow     | custom retry condition function     | retries, retry delay, exponential backoff helper| dynamic retry decision based on error|

---

## Alternatives considered
- **Alternative 1: No framework retry, only task-local retries**
--> The behavior might become inconsistent. Some tasks may be resilient while others fail immediately. Also, a task may retry in `run()` but not in `validate()`, which creates surprising behavior.

- **Alternative 2: Otter retries everything on any exception**
    --> Too blunt. Not all failures are transient. Retrying configuration mistakes or schema mismatches wastes time and can hide the real issue.

- **Alternative 3: Remove all local resilience and move everything into Otter**

    --> Some tasks genuinely know more than the framework about what is safe to retry. A task interacting with a specific API may need request-level resilience that is more precise than a generic framework wrapper.


## Open questions and risks
1. **How should retryable errors be identified?**  
    --> Typed exceptions are clean, but they require task authors to use them correctly. A classifier helper is flexible, but it may be less explicit.

2. **Should retries repeat both `run()` and `validate()` after a validation failure?**
    --> My default answer is yes: retrying the current failing phase keeps semantics easy to reason about. The downside is extra work if `run()` is expensive.

3. **How much retry configuration should be exposed in YAML?**
    --> Too little and the feature becomes rigid. Too much and Otter becomes harder to understand. I would start with only a few fields such as `max_attempts`, `initial_delay_seconds`, `backoff`, `max_delay_seconds`, and `jitter`.

4. **Retry Amplification**
    --> If task-local retries and Otter retries are both large, request volume can grow quickly.
This is why local retries should remain short and Otter retries should be bounded.

5. **Worker slot blocking**
   --> If a worker sleeps during backoff, that worker is temporarily unavailable for other tasks. This is simple to implement, but it may reduce throughput when many tasks fail at once. For an MVP, I think this tradeoff is acceptable, but it should be called out.

6. **Idempotency**
   --> Retries are safest when tasks are idempotent or mostly idempotent.
Some tasks may need extra care if repeating `run()` can partially overwrite state.

7. **How should local retries appear in logs and provenance?**
   If the outer Otter retry is the authoritative layer, then Otter should record outer attempts in the manifest. Local retries can still appear in task logs, but they probably should not be promoted to manifest-level attempts unless we want much more detailed provenance.

---


## Proposed Implementation
``` text
1. a worker runs the current task phase
2. if that phase succeeds, Otter continues as usual
3. if that phase fails, Otter decides whether the failure is retryable
4. if it is retryable and there are attempts left, Otter waits and retries the same phase
5. if it is not retryable, or the retry limit is reached, the task fails and the result is written into the manifest
```


- For the first version, I would keep the retry unit small and easy to understand: Otter retries the **current failing phase**, either `run()` or `validate()`, rather than restarting the whole step.

The sequence below shows where the retry logic would live. The coordinator still hands work to workers as usual, but the worker becomes responsible for classifying failures, waiting between attempts, and recording retry history in the manifest.

``` mermaid
sequenceDiagram
    participant C as Coordinator
    participant W as Worker
    participant T as Task
    participant M as Manifest

    C->>W: enqueue task in RUNNING or VALIDATING state
    W->>M: mark phase started if first attempt
    loop attempts <= max_attempts
        W->>T: execute current phase
        alt success
            W->>M: append success attempt
            W->>M: mark phase finished
            W-->>C: return task
        else failure
            W->>W: classify exception
            W->>M: append failed attempt
            alt retryable and attempts remain
                W->>W: sleep with backoff + jitter
            else terminal failure
                W->>M: set failure_reason/result
                W-->>C: return failed task
            end
        end
    end
```

## Demo

A small prototype, demo setup, and sample outputs are described in [OUTPUT.md](https://github.com/isthatgopro/pis/blob/main/OUTPUT.md).
