---
date: "2026-10-13T22:58:04+09:00"
title: "Forwarding Celery task events to Socket.IO in a FastAPI app"
tags: ["Python", "Celery", "FastAPI", "Socket.IO", "RabbitMQ"]
---

In a FastAPI app that hands long-running jobs, such as training a model, to [Celery](https://docs.celeryq.dev/) workers, clients usually want to know how their task is doing: when it starts, how far along it is, and when it succeeds or fails. One way to do this is to build a feedback system of one's own, with a dedicated queue in RabbitMQ that workers publish to and the API consumes. Celery already has such a mechanism, though: [task events](https://docs.celeryq.dev/en/stable/userguide/monitoring.html#events), which workers send through the broker and which anyone connected to it can receive. This article shows how to emit custom progress events from a task and forward all task events to clients with [Socket.IO](https://python-socketio.readthedocs.io/).

## The worker

Workers only send task events (`task-started`, `task-succeeded`, `task-failed`, etc.) when `worker_send_task_events` is enabled:

```python
# main.py (worker)
import time

from celery import Celery
from config import RABBITMQ_URL, MONGODB_URI, TASK_QUEUE, TASK_NAME

app = Celery(
    'tasks',
    broker=RABBITMQ_URL,
    backend=MONGODB_URI,
    task_default_queue=TASK_QUEUE,
    broker_connection_retry_on_startup=True,
    result_persistent=True,
    result_extended=True,
    task_track_started=True,
    # Task state and results are read from the result backend: never expire them
    # (otherwise celery beat's backend_cleanup would delete them after one day).
    # 0 means "never"; None would be ignored here and fall back to the 1-day default
    result_expires=0,
    # Task started / succeeded / failed events
    # and the progress events of progressReporter,
    worker_send_task_events=True,
)
```

On top of the built-in events, a bound task can send events of its own with `self.send_event`. Here, the task reports its progress through a small `report` function:

```python
@app.task(bind=True, name=TASK_NAME)
def train_model(self, **kwargs):

    report = progressReporter(self)

    # Task code

    report("training", {"epoch": 22})

    # Task code
```

`progressReporter` is a wrapper around `task.send_event` that sends a `task-progress` event with a `stage` and some `data`:

```python
# main.py (worker)

# After failing to send a progress event (e.g. RabbitMQ unreachable, each attempt then takes ~0.6 s), seconds during which progress isn't sent, so as not to slow the task down
PROGRESS_PAUSE_AFTER_FAILURE = 10

def progressReporter(task):
    """report(stage, data) for the training code: sends a task-progress event.
    Never raises: progress is informational and must not fail the task"""
    pausedUntil = 0.0

    def report(stage, data=None):
        nonlocal pausedUntil
        if time.monotonic() < pausedUntil:
            return
        try:
            task.send_event(
                "task-progress",
                retry=False,
                stage=stage,
                data={} if data is None else data
            )
        except Exception as e:
            print(f"Could not send progress event {stage}: {e}")
            pausedUntil = time.monotonic() + PROGRESS_PAUSE_AFTER_FAILURE

    return report
```

Progress is only informational, so the wrapper is careful never to get in the way of the task itself. It catches every exception, so that a broker problem doesn't fail the task. It also sends with `retry=False` and, after a failure, stops sending for a few seconds: when RabbitMQ is unreachable, each attempt takes a while, and a training loop that reports often would otherwise slow down noticeably.

## The FastAPI app

On the API side, a Celery app connected to the same broker and result backend is used to receive the events. It doesn't run tasks, so it's called `publisher` here:

```python
# celeryPublisher.py
from celery import Celery
from config import RABBITMQ_URL, MONGODB_URI

publisher = Celery(
    __name__,
    broker=RABBITMQ_URL,
    backend=MONGODB_URI,
    result_extended=True,
    task_track_started=True,
    result_persistent=True,
    # Task state and results are read from the result backend: never expire them
    # (otherwise celery beat's backend_cleanup would delete them after one day).
    # 0 means "never"; None would be ignored here and fall back to the 1-day default
    result_expires=0,
)
```

The Socket.IO server is an `AsyncServer` in ASGI mode:

```python
# socketioServer.py
import socketio

sio = socketio.AsyncServer(async_mode='asgi', cors_allowed_origins='*')
```

The FastAPI app is wrapped in a `socketio.ASGIApp`, which handles Socket.IO traffic and passes everything else, lifespan events included, to FastAPI. The task event receiver is started in the app's lifespan handler:

```python
# main.py (API)
from contextlib import asynccontextmanager

import socketio
from fastapi import FastAPI

from taskEvents import startTaskEventReceiver
from socketioServer import sio

@asynccontextmanager
async def lifespan(app: FastAPI):
    startTaskEventReceiver()
    yield

app = FastAPI(lifespan=lifespan)

app = socketio.ASGIApp(sio, app)
```

## Receiving the events

Celery's event receiver is built on kombu and is blocking, so it can't run on the app's event loop. Instead, it runs in its own daemon thread, which is given the running event loop so that it can schedule Socket.IO emits on it:

```python
# taskEvents.py
import asyncio
import threading
import time

from celeryPublisher import publisher
from socketioServer import sio

def startTaskEventReceiver():
    thread = threading.Thread(
        target=receiveTaskEvents,
        args=(asyncio.get_running_loop(),),
        name="task-events",
        daemon=True,
    )
    thread.start()
    return thread
```

The thread itself connects to the broker and captures every event with a catch-all `"*"` handler. `sio.emit` is a coroutine, so it's handed to the event loop with `asyncio.run_coroutine_threadsafe`. If the connection is lost, for instance while RabbitMQ restarts, the receiver waits a few seconds and connects again:

```python
# taskEvents.py

RECONNECT_DELAY = 5

def receiveTaskEvents(loop):
    # Blocking (kombu): runs in its own thread, and emits on the app's event loop

    # Wrapper for Socket.io
    def emit(name, data):
        asyncio.run_coroutine_threadsafe(sio.emit(name, data), loop)

    # Forward to Socket.io
    def forward(event):
        try:
            forwarded = socketEvent(event)
            if forwarded:
                emit(*forwarded)
        except Exception as e:
            # One bad event must not stop the receiver
            print(f"Could not forward task event {event.get('type')}: {e}")

    while True:
        try:
            with publisher.connection_for_read() as connection:
                connection.ensure_connection(max_retries=1)
                receiver = publisher.events.Receiver(
                    connection,
                    handlers={"*": forward}
                )
                print("Receiving Celery task events")
                receiver.capture(limit=None, timeout=None, wakeup=True)
        except Exception as e:
            # E.g. RabbitMQ restarting: events sent until receiver is back are lost
            print(f"Celery task events receiver disconnected, reconnecting: {e}")
            time.sleep(RECONNECT_DELAY)
```

Finally, `socketEvent` decides which events are forwarded and what clients receive. Worker events such as heartbeats are dropped, and task events are emitted under their own name, with the task ID, the progress stage and the progress data:

```python
# taskEvents.py
def socketEvent(event):
    # (Socket.IO event name, payload) for a Celery event, or None if not forwarded.
    # Custom events can arrive before task-started: each event is mapped on its own
    eventType = event.get("type", "")
    data = event.get("data") or {}
    taskId = event.get("uuid")
    stage = event.get("stage")

    if not eventType.startswith("task-") or not taskId:
        return None

    payload = {"taskId": taskId, "stage": stage, **data}
    return eventType, payload
```

Clients therefore receive Socket.IO events named after the Celery events, including:

- `task-started`
- `task-progress`
- `task-succeeded`
- `task-failed`

## Limitations

Celery events aren't stored: events sent while the receiver is disconnected, for example while RabbitMQ restarts, are lost. That's acceptable here because they are only notifications. The state and result of each task are kept in the result backend, which is configured never to expire them, so that's where the authoritative state is read from.
