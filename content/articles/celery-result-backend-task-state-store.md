---
date: "2026-10-14T07:58:09+09:00"
title: "Using the Celery result backend as the primary task state store"
tags: ["Python", "Celery", "MongoDB", "RabbitMQ"]
---

When an application hands long-running jobs to [Celery](https://docs.celeryq.dev/) workers, it's tempting to keep a copy of each task's state in the application's own documents and update it as the task progresses. That copy can easily drift from what Celery itself knows. An alternative is to make Celery's result backend, here the `celery_taskmeta` collection in MongoDB, the single source of truth for tasks, with application documents keeping at most a task ID. This article goes through what that takes in practice, both on the API side and in the workers.

## 1. Same configuration on both sides

The producer and the workers must use the same result backend, and the backend must store the task's details: without `result_extended`, kwargs aren't stored, and tasks can't be found by their arguments.

```python
app = Celery(
    __name__,
    broker=RABBITMQ_URL,
    backend=MONGODB_URI,      # same database for producer and all workers
    result_extended=True,     # store name, args, kwargs, worker, queue
    task_track_started=True,  # record STARTED, not only the final state
    result_persistent=True,
    result_expires=0,         # never delete records (0 = never, None = 1-day default)
)
```

The code below relies on internals of the MongoDB backend, so Celery should be pinned to the same version everywhere:

```
celery == 5.x.y
```

## 2. Record queued tasks yourself

Celery only stores a task once a worker starts it. Until then, its ID reads as `PENDING`, just like an ID that doesn't exist at all. To tell a queued task apart from an unknown one, the producer records the task itself before sending it. The record must include the task's details: with `result_extended`, a record without them can't be read back.

```python
from types import SimpleNamespace
from celery import states

def sendTask(name, queue, kwargs):
    taskId = str(uuid.uuid4())
    request = SimpleNamespace(task=name, args=[], kwargs=kwargs, hostname=None,
                              retries=0, delivery_info={"routing_key": queue}, chord=None)
    # Before sending: a worker picking it up immediately must not have its STARTED overwritten
    publisher.backend.store_result(taskId, None, states.PENDING, request=request)
    try:
        publisher.send_task(name, kwargs=kwargs, queue=queue, task_id=taskId)
    except Exception as error:
        publisher.backend.store_result(taskId, error, states.FAILURE, request=request)
        raise
    return taskId
```

The `PENDING` record is written before the task is sent rather than after: otherwise, a worker could pick the task up and record `STARTED` before the producer writes `PENDING` over it.

## 3. Read state through Celery

The state and result of a task are read through Celery's own API rather than from the collection:

```python
from celery.result import AsyncResult

result = AsyncResult(taskId, app=publisher)
result.state    # PENDING, STARTED, SUCCESS, FAILURE, REVOKED: return them unchanged
result.result   # SUCCESS: return value. FAILURE: the exception
```

Application documents only reference the task:

```python
pipelineCollection.update_one({"_id": pipelineId},
    {"$set": {"model.trainingTask": {"id": taskId, "submittedAt": datetime.now(timezone.utc)}}})
```

## 4. Find tasks by their arguments: query the collection

Celery's API can't search tasks by their arguments, for example to check whether a pipeline already has a task running. For that, `celery_taskmeta` is queried directly. Since its records never expire, it should be indexed:

```python
UNFINISHED = [states.PENDING, states.STARTED]

celeryTaskCollection.create_index([("kwargs.pipelineId", 1), ("status", 1)])

def busy(pipelineId):
    return celeryTaskCollection.count_documents(
        {"kwargs.pipelineId": pipelineId, "status": {"$in": UNFINISHED}}) > 0
```

These queries are the part most exposed to a change in Celery's storage format, so it's best to keep all of them in a single module.

## 5. Stopping: revoke and record REVOKED

`revoke` alone records nothing if the worker running the task is dead, so the producer records `REVOKED` itself. The MongoDB backend replaces the whole record, so the task's details have to be passed again:

```python
def stopTask(taskId):
    result = AsyncResult(taskId, app=publisher)
    if result.state not in UNFINISHED:
        return False  # don't overwrite a final state
    publisher.control.revoke(taskId, terminate=True, signal="SIGKILL")
    request = SimpleNamespace(task=result.name, args=result.args, kwargs=result.kwargs,
                              hostname=result.worker, retries=result.retries,
                              delivery_info={"routing_key": result.queue}, chord=None)
    publisher.backend.mark_as_revoked(taskId, request=request)
    return True
```

A task revoked while still queued can still run later. Workers keep revoked IDs in memory only, so they are lost on restart, and they expire after 3 hours by default. To avoid this, workers can persist them, with `celery worker --statedb=/data/worker.state`, and `CELERY_WORKER_REVOKE_EXPIRES` can be raised. Another option is to have workers check the result backend for `REVOKED` in `task_prerun`, which runs before the worker records `STARTED`.

## 6. Lost tasks: nothing ends them for you

A worker killed in the middle of a task, for instance during a deployment, because it ran out of memory or because its pod was evicted, leaves its task `STARTED` forever. Nothing in Celery fixes this, so the application has to check periodically that each `STARTED` task's own worker is still running it, and stop the task after a few missed checks:

```python
records = list(celeryTaskCollection.find({"status": states.STARTED}))
workers = list({r["worker"] for r in records if r.get("worker")})
replies = publisher.control.inspect(destination=workers, timeout=5.0) \
    .query_task(*[r["_id"] for r in records]) or {}
running = {taskId for tasks in replies.values()
           for taskId, (state, _) in tasks.items() if state == "active"}
# Not in `running` for N consecutive checks: stopTask(...)
```

For this to work, each worker pod needs a unique name. Celery's default, `celery@<hostname>`, provides one, so workers shouldn't be given a fixed `--hostname`.

## 7. Progress: update_state, staying STARTED

The result backend can also hold a task's progress. A bound task updates its own record with `update_state`, and the API reads the progress back as `AsyncResult(taskId).info`. Each call writes the whole record, so it should be used sparingly, for example when the stage changes rather than on every batch, and a failure to store progress shouldn't fail the task:

```python
self.update_state(state=states.STARTED, meta={"stage": "training", "data": {"epoch": 22}})
```

- The [Celery documentation](https://docs.celeryq.dev/en/stable/userguide/tasks.html#custom-states) uses a custom state, such as `PROGRESS`, for this. That works too, but the custom state then has to be added to `UNFINISHED` and to the lost-task query. Staying `STARTED` changes nothing else.
- `update_state` passes the task's request to the backend, so the name, kwargs and worker are written again and queries on `kwargs.pipelineId` keep working (checked in Celery 5.6.3).
- When a worker starts a task, it stores `{"pid": ..., "hostname": ...}` in `result`. Until the first progress update, `info` holds that instead of progress.
- With the default JSON result serializer, `result` is stored as a string, so progress can only be read through `AsyncResult`, not queried in the collection.
- A progress update made after `stopTask` recorded `REVOKED`, but before the worker received the revoke, records `STARTED` again. The lost-task check eventually ends the task again.

## Limitations

- This only covers tasks run by Celery: tasks run elsewhere, for example by Argo Workflows, need another store.
- `celery_taskmeta` grows forever, kwargs included, which may contain things like user claims. A retention policy has to be decided.
- There's no model for ownership, history or auditing. If one is needed, a thin collection written at submit time (task ID, target, owner) can be added, while still keeping the state itself in Celery.
