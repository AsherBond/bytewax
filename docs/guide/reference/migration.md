(xref-migration)=
# Migration Guide

This guide can help you upgrade code through breaking changes from one
Bytewax version to the next. For a detailed list of all changes, see
the
[CHANGELOG](https://github.com/bytewax/bytewax/blob/main/CHANGELOG.md).

## From v0.19 to v0.20

### Windowing Components Renamed

Windowing operators have been moved from `bytewax.operators.window`
to {py:obj}`bytewax.operators.windowing`.

In addition, `ClockConfig` classes have had the `Config` suffix
dropped from them, and `WindowConfig`s have been renamed to
`Windower`s. Other than the name changes, the functionality
is unchanged.

Before:

```python
import bytewax.operators.window as win
from bytewax.operators.window import EventClockConfig, TumblingWindow
```

After:

```{testcode}
import bytewax.operators.windowing as win
from bytewax.operators.windowing import EventClock, TumblingWindower
```

### Windowing Operator Output

Windowing operators now return a set of three streams bundled in a
{py:obj}`~bytewax.operators.windowing.WindowOut` object:

1. `down` stream - window IDs and the resulting output from that operator.

2. `late` stream - items which were late and not assigned or processed in a window, but labeled with the window ID they would have been included in.

3. `meta` stream - window IDs and the final {py:obj}`~bytewax.operators.windowing.WindowMetadata` describing the open and close times of that window.

Items in all three window output streams are now labeled with the unique `int`
window ID they were assigned to facilitate joining the data later to derive more
complex context about the resulting windows.

To recreate the exact downstream items that window operators emitted in v0.19,
you'll now need to {py:obj}`~bytewax.operators.join` the `down` stream with the
`meta` stream on key and window ID, then remove the window ID. This transformation
only works with windowing operators that emit a single item downstream.

Before:

```python
from datetime import datetime, timedelta, timezone

from bytewax.dataflow import Dataflow
import bytewax.operators as op
import bytewax.operators.window as win
from bytewax.operators.window import SystemClockConfig, TumblingWindow
from bytewax.testing import TestingSource

events = [
    {"user": "a", "val": 1},
    {"user": "a", "val": 1},
    {"user": "b", "val": 1},
]

flow = Dataflow("count")
inp = op.input("inp", flow, TestingSource(events))
counts = win.count_window(
    "count",
    inp,
    SystemClock(),
    TumblingWindow(
        length=timedelta(seconds=2), align_to=datetime(2023, 1, 1, tzinfo=timezone.utc)
    ),
    lambda x: x["user"],
)
op.inspect("inspect", counts)
# ("user", (WindowMetadata(...), count_per_window))
```

After:

```{testcode}
from datetime import datetime, timedelta, timezone
from typing import Tuple, TypeVar

import bytewax.operators as op
import bytewax.operators.windowing as win
from bytewax.dataflow import Dataflow
from bytewax.operators.windowing import SystemClock, TumblingWindower, WindowMetadata
from bytewax.testing import TestingSource

events = [
    {"user": "a", "val": 1},
    {"user": "a", "val": 1},
    {"user": "b", "val": 1},
]

flow = Dataflow("count")
inp = op.input("inp", flow, TestingSource(events))
count_out = win.count_window(
    "count",
    inp,
    SystemClock(),
    TumblingWindower(
        length=timedelta(seconds=2), align_to=datetime(2023, 1, 1, tzinfo=timezone.utc)
    ),
    lambda x: x["user"],
)
# Returning just the counts per window: ('user', (window_id, count_per_window))
op.inspect("check_down", count_out.down)
X = TypeVar("X")


def rekey_by_window(key_id_obj: Tuple[str, Tuple[int, X]]) -> Tuple[str, X]:
    key, id_obj = key_id_obj
    win_id, obj = id_obj
    return (f"{key}-{win_id}", obj)


keyed_metadata = op.map("rekey_meta", count_out.meta, rekey_by_window)
keyed_counts = op.map("rekey_counts", count_out.down, rekey_by_window)
joined_counts = op.join("check_joined", keyed_metadata, keyed_counts)


def unkey_join_rows(
    rekey_row: Tuple[str, Tuple[WindowMetadata, int]],
) -> Tuple[str, Tuple[WindowMetadata, int]]:
    rekey, row = rekey_row
    key, _win_id = rekey.rsplit("-", maxsplit=1)
    return (key, row)


# Returning the old output ('user', (WindowMetadata(..), count_per_window))
cleaned_joined_counts = op.map("unkey_joined", joined_counts, unkey_join_rows)
op.inspect("check_joined_counts", cleaned_joined_counts)
```

If your original dataflow ignored the {py:obj}`~bytewax.operators.windowing.WindowMetadata`, you can skip doing the above step and instead use `down` directly without joining and drop the window ID.

### Fold Window Merges

{py:obj}`~bytewax.operators.windowing.fold_window` now requires a `merger` callback that takes two fully formed accumulators and combines them into one. The `merger` function
will be called with when the windower determines that two
windows must be merged. This most commonly happens when using the {py:obj}`~bytewax.operators.windowing.SessionWindower` and a new, out-of-order item bridges a gap.

Before:

```python
from datetime import datetime, timedelta, timezone
from typing import List

import bytewax.operators as op
import bytewax.operators.window as win
from bytewax.dataflow import Dataflow
from bytewax.operators.window import EventClockConfig, SessionWindow
from bytewax.testing import TestingSource

align_to = datetime(2022, 1, 1, tzinfo=timezone.utc)
events = [
    {"user": "a", "val": 1, "time": align_to + timedelta(seconds=1)},
    {"user": "a", "val": 2, "time": align_to + timedelta(seconds=3)},
    {"user": "a", "val": 3, "time": align_to + timedelta(seconds=7)},
]

flow = Dataflow("merge_session")
inp = op.input("inp", flow, TestingSource(events)).then(
    op.key_on, "key_all", lambda _: "ALL"
)


def ts_getter(event: dict) -> datetime:
    return event["time"]


def add(acc: List[str], event: dict[str, str]) -> List[str]:
    acc.append(event["val"])
    return acc


counts = win.fold_window(
    "count",
    inp,
    EventClockConfig(ts_getter, wait_for_system_duration=timedelta(seconds=0)),
    SessionWindow(gap=timedelta(seconds=3)),
    list,
    add,
)
op.inspect("inspect", counts.down)
```

After:

```{testcode}
from datetime import datetime, timedelta, timezone
from typing import List

import bytewax.operators as op
import bytewax.operators.windowing as win
from bytewax.dataflow import Dataflow
from bytewax.operators.windowing import EventClock, SessionWindower
from bytewax.testing import TestingSource

align_to = datetime(2022, 1, 1, tzinfo=timezone.utc)
events = [
    {"user": "a", "val": 1, "time": align_to + timedelta(seconds=1)},
    {"user": "a", "val": 2, "time": align_to + timedelta(seconds=3)},
    {"user": "a", "val": 3, "time": align_to + timedelta(seconds=7)},
]

flow = Dataflow("merge_session")
inp = op.input("inp", flow, TestingSource(events)).then(
    op.key_on, "key_all", lambda _: "ALL"
)


def ts_getter(event: dict) -> datetime:
    return event["time"]


def add(acc: List[str], event: dict[str, str]) -> List[str]:
    acc.append(event["val"])
    return acc


counts = win.fold_window(
    "count",
    inp,
    EventClock(ts_getter, wait_for_system_duration=timedelta(seconds=0)),
    SessionWindower(gap=timedelta(seconds=3)),
    list,
    add,
    list.__add__,
)
op.inspect("inspect", counts.down)
```

### Join Modes

To specify a running {py:obj}`~bytewax.operators.join`, now use `mode="running"` instead of `running=True`. To specify a product {py:obj}`~bytewax.operators.windowing.join_window`, use `mode="product"` instead of `product=True`. Both these operators now have more modes to choose from; see {py:obj}`bytewax.operators.JoinInsertMode` and {py:obj}`bytewax.operators.JoinEmitMode`.

Before:

```python
flow = Dataflow("running_join")

names_l = [
    {"user_id": 123, "name": "Bee"},
    {"user_id": 456, "name": "Hive"},
]

names = op.input("names", flow, TestingSource(names_l))


emails_l = [
    {"user_id": 123, "email": "bee@bytewax.io"},
    {"user_id": 456, "email": "hive@bytewax.io"},
    {"user_id": 123, "email": "queen@bytewax.io"},
]

emails = op.input("emails", flow, TestingSource(emails_l))


keyed_names = op.map("key_names", names, lambda x: (str(x["user_id"]), x["name"]))
keyed_emails = op.map("key_emails", emails, lambda x: (str(x["user_id"]), x["email"]))
joined = op.join("join", keyed_names, keyed_emails, running=True)
op.inspect("insp", joined)
```

After:

```{testcode}
from datetime import datetime, timedelta, timezone

import bytewax.operators as op
import bytewax.operators.windowing as win
from bytewax.dataflow import Dataflow
from bytewax.operators.windowing import EventClock, TumblingWindower
from bytewax.testing import TestingSource
from typing import Tuple, Any

align_to = datetime(2022, 1, 1, tzinfo=timezone.utc)
events = [
    {"user": "a", "val": 1, "timestamp": align_to + timedelta(seconds=0)},
    {"user": "a", "val": 1, "timestamp": align_to + timedelta(seconds=1)},
    {"user": "b", "val": 1, "timestamp": align_to + timedelta(seconds=3)},
]

flow = Dataflow("count")
inp = op.input("inp", flow, TestingSource(events))
clock = EventClock(lambda x: x["timestamp"], wait_for_system_duration=timedelta(seconds=0))
counts = win.count_window(
    "count",
    inp,
    clock,
    TumblingWindower(
        length=timedelta(seconds=2), align_to=datetime(2023, 1, 1, tzinfo=timezone.utc)
    ),
    lambda x: x["user"],
)
# Returning just the counts per window: ('user', (window_id, count_per_window))
op.inspect("new_output", counts.down)

def join_window_id(window_out: Tuple[str, Tuple[int, Any]]) -> Tuple[str, Any]:
    (user_id, (window_id, count)) = window_out
    return (f"{user_id}:{window_id}", count)

# Re-key each stream by user-id:window-id
keyed_metadata = op.map("k_md", counts.meta, join_window_id)
keyed_counts = op.map("k_count", counts.down, join_window_id)
# Join the output of the window and the metadata stream on user-id:window-id
joined_meta = op.join("joined_output", keyed_metadata, keyed_counts)
user_output = op.map("reformat-out", joined_meta, lambda x: (x[0].split(":")[0], x[1]))
op.inspect("old_output", user_output)
```

### Removal of `join_named` and `join_window_named`

The `join_named` and `join_window_named` operators have been removed as
they can't be made to support fully type checked dataflows.

The same functionality is still available but with a slightly
differently shaped API via {py:obj}`~bytewax.operators.join` or {py:obj}`~bytewax.operators.windowing.join_window`:

Before:

```python
import bytewax.operators as op
from bytewax.dataflow import Dataflow
from bytewax.testing import TestingSource

flow = Dataflow("join_eg")
names_l = [
    {"user_id": 123, "name": "Bee"},
    {"user_id": 456, "name": "Hive"},
]
names = op.input("names", flow, TestingSource(names_l))
emails_l = [
    {"user_id": 123, "email": "bee@bytewax.io"},
    {"user_id": 456, "email": "hive@bytewax.io"},
    {"user_id": 123, "email": "queen@bytewax.io"},
]
emails = op.input("emails", flow, TestingSource(emails_l))
keyed_names = op.map("key_names", names, lambda x: (str(x["user_id"]), x["name"]))
keyed_emails = op.map("key_emails", emails, lambda x: (str(x["user_id"]), x["email"]))
joined = op.join_named("join", name=keyed_names, email=keyed_emails)
op.inspect("check_join", joined)
```

After:

```{testcode}
from dataclasses import dataclass
from typing import Dict, Tuple

import bytewax.operators as op
from bytewax.dataflow import Dataflow
from bytewax.testing import TestingSource

flow = Dataflow("join_eg")
names_l = [
    {"user_id": 123, "name": "Bee"},
    {"user_id": 456, "name": "Hive"},
]
emails_l = [
    {"user_id": 123, "email": "bee@bytewax.io"},
    {"user_id": 456, "email": "hive@bytewax.io"},
    {"user_id": 123, "email": "queen@bytewax.io"},
]


def id_field(field_name: str, x: Dict) -> Tuple[str, str]:
    return (str(x["user_id"]), x[field_name])


names = op.input("names", flow, TestingSource(names_l))
emails = op.input("emails", flow, TestingSource(emails_l))
keyed_names = op.map("key_names", names, lambda x: id_field("name", x))
keyed_emails = op.map("key_emails", emails, lambda x: id_field("email", x))
joined = op.join("join", keyed_names, keyed_emails)
op.inspect("join_returns_tuples", joined)


@dataclass
class User:
    user_id: int
    name: str
    email: str


def as_user(id_row: Tuple[str, Tuple[str, str]]) -> User:
    user_id, row = id_row
    # Unpack the tuple values in the id_row, and create the completed User
    name, email = row
    return User(int(user_id), name, email)


users = op.map("as_dict", joined, as_user)
op.inspect("inspect", users)
```

## From v0.18 to v0.19

### Removal of the `builder` argument from `stateful_map`

The `builder` argument has been removed from
{py:obj}`~bytewax.operators.stateful_map`. The initial state value is
always `None` and you can call your previous builder by hand in the
`mapper` function.

Before:

```python
import bytewax.operators as op
from bytewax.dataflow import Dataflow
from bytewax.testing import TestingSource


flow = Dataflow("builder_eg")
keyed_amounts = op.input(
    "inp",
    flow,
    TestingSource(
        [
            ("KEY", 1.0),
            ("KEY", 6.0),
            ("KEY", 2.0),
        ]
    ),
)


def running_builder():
    return []


def calc_running_mean(values, new_value):
    values.append(new_value)
    while len(values) > 3:
        values.pop(0)

    running_mean = sum(values) / len(values)
    return (values, running_mean)


running_means = op.stateful_map(
    "running_mean", keyed_amounts, running_builder, calc_running_mean
)
```

After:

```{testcode}
import bytewax.operators as op
from bytewax.dataflow import Dataflow
from bytewax.testing import TestingSource


flow = Dataflow("builder_eg")
keyed_amounts = op.input("inp", flow, TestingSource([
    ("KEY", 1.0),
    ("KEY", 6.0),
    ("KEY", 2.0),
]))


def calc_running_mean(values, new_value):
    # On the initial value for this key, instead of the operator calling the
    # builder for you, you can call it yourself when the state is un-initalized.

    if values is None:
        values = []

    values.append(new_value)
    while len(values) > 3:
        values.pop(0)

    running_mean = sum(values) / len(values)
    return (values, running_mean)


running_means = op.stateful_map("running_mean", keyed_amounts, calc_running_mean)
```

### Connector API Now Contains Step ID

{py:obj}`bytewax.inputs.FixedPartitionedSource.build_part`,
{py:obj}`bytewax.inputs.DynamicSource.build`,
{py:obj}`bytewax.outputs.FixedPartitionedSink.build_part` and
{py:obj}`bytewax.outputs.DynamicSink.build` now take an additional
`step_id` argument. This argument can be used as a label when creating
custom Python metrics.

Before:

```python
from bytewax.inputs import DynamicSource


class PeriodicSource(DynamicSource):
    def build(self, now: datetime, worker_index: int, worker_count: int):
        pass
```

After:

```{testcode}
from bytewax.inputs import DynamicSource


class PeriodicSource(DynamicSource):
    def build(self, step_id: str, worker_index: int, worker_count: int):
        pass
```

### `datetime` Arguments Removed for Performance

{py:obj}`bytewax.inputs.FixedPartitionedSource.build_part`,
{py:obj}`bytewax.inputs.DynamicSource.build` and
`bytewax.operators.UnaryLogic.on_item` no longer take a `now:
datetime` argument.
{py:obj}`bytewax.inputs.StatefulSourcePartition.next_batch`,
{py:obj}`bytewax.inputs.StatelessSourcePartition.next_batch`, and
`bytewax.operators.UnaryLogic.on_notify` no longer take a `sched:
datetime` argument. Generating these values resulted in significant
overhead, even for the majority of sources and stateful operators that
never used them.

If you need the current time, you still can manually get the current
time:

```{testcode}
from datetime import datetime, timezone

now = datetime.now(timezone.utc)
```

If you need the previously scheduled awake time, store it in an
instance variable before returning it from
`~bytewax.operators.UnaryLogic.notify_at`. Your design probably
already has that stored in an instance variable.

### Standardization on Confluent's Kafka Serialization Interface

Bytewax's bespoke Kafka schema registry and serialization interface
has been removed in favor of using
{py:obj}`confluent_kafka.schema_registry.SchemaRegistryClient` and
{py:obj}`confluent_kafka.serialization.Deserializer`s and
{py:obj}`confluent_kafka.serialization.Serializer`s directly. This now
gives you direct control over all of the configuration options for
serialization and supports the full range of use cases. Bytewax's
Kafka serialization operators in
{py:obj}`bytewax.connectors.kafka.operators` now take these Confluent
types.

#### With Confluent Schema Registry

If you are using Confluent's schema registry (with it's magic byte
prefix), you can pass serializers like
{py:obj}`confluent_kafka.schema_registry.avro.AvroDeserializer`
directly to our operators. See Confluent's documentation for all the
options here.

Before:

```python
from bytewax.connectors.kafka import operators as kop
from bytewax.connectors.kafka.registry import ConfluentSchemaRegistry

flow = Dataflow("confluent_sr_eg")
kinp = kop.input("inp", flow, brokers=["broker.test"], topics=["eg_topic"])

sr_conf = {"url": "http://registry.test/", "basic.auth.user.info": "user:pass"}
registry = ConfluentSchemaRegistry(SchemaRegistryClient(sr_conf))

key_de = AvroDeserializer(client)
val_de = AvroDeserializer(client)

# Deserialize both key and value
msgs = kop.deserialize("de", kinp.oks, key_deserializer=key_de, val_deserializer=val_de)
```

After:

```{testcode}
from bytewax.dataflow import Dataflow
from bytewax.connectors.kafka import operators as kop

from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroDeserializer

flow = Dataflow("confluent_sr_eg")
kinp = kop.input("inp", flow, brokers=["broker.test"], topics=["eg_topic"])

# Confluent's SchemaRegistryClient
client = SchemaRegistryClient(
    {"url": "http://registry.test/", "basic.auth.user.info": "user:pass"}
)
key_de = AvroDeserializer(client)
val_de = AvroDeserializer(client)

# Deserialize both key and value
msgs = kop.deserialize("de", kinp.oks, key_deserializer=key_de, val_deserializer=val_de)
```


#### With Redpanda Schema Registry

If you are using Redpanda's schema registry or another setup for which
the serialized form does not use Confluent's wire format, we provide
compatible (de)serializer classes in
{py:obj}`bytewax.connectors.kafka.serde`.

Before:

```python
from bytewax.connectors.kafka import operators as kop
from bytewax.connectors.kafka.registry import RedpandaSchemaRegistry, SchemaRef

REDPANDA_REGISTRY_URL = "http://localhost:8080/schema-registry"

registry = RedpandaSchemaRegistry(REDPANDA_REGISTRY_URL)
key_de = registry.deserializer(SchemaRef("sensor-key"))
val_de = registry.deserializer(SchemaRef("sensor-value"))

msgs = kop.deserialize("de", kinp.oks, key_deserializer=key_de, val_deserializer=val_de)
```

After:

% Can't run this test because it calls out to the schema registry.

```python
from bytewax.connectors.kafka import operators as kop
from bytewax.connectors.kafka.serde import PlainAvroDeserializer

from confluent_kafka.schema_registry import SchemaRegistryClient

REDPANDA_REGISTRY_URL = "http://localhost:8080/schema-registry"

client = SchemaRegistryClient({"url": REDPANDA_REGISTRY_URL})

# Use plain Avro instead of Confluent's prefixed-Avro wire format.
# We need to specify the schema in the deserializer too here.
key_schema = client.get_latest_version("sensor-key").schema
key_de = PlainAvroDeserializer(schema=key_schema)

val_schema = client.get_latest_version("sensor-value").schema
val_de = PlainAvroDeserializer(schema=val_schema)

# Deserialize both key and value
msgs = kop.deserialize("de", kinp.oks, key_deserializer=key_de, val_deserializer=val_de)
```


## From v0.17 to v0.18

### Non-Linear Dataflows

With the addition of non-linear dataflows, the API for constructing
dataflows has changed. Operators are now stand-alone functions that
can take and return streams.

 All operators, not just stateful ones, now require a `step_id`; it
 should be a `"snake_case"` description of the semantic purpose of
 that dataflow step.

 Also instantiating the dataflow itself now takes a "dataflow ID" so
 you can disambiguate different dataflows in the metrics.

Before:

```python
from bytewax.dataflow import Dataflow
from bytewax.testing import TestingSource
from bytewax.connectors.stdio import StdOutput

flow = Dataflow()
flow.input("inp", TestingInput(range(10)))
flow.map(lambda x: x + 1)
flow.output("out", StdOutput())
```

After:

```python
import bytewax.operators as op

from bytewax.dataflow import Dataflow
from bytewax.testing import TestingSource
from bytewax.connectors.stdio import StdOutSink

flow = Dataflow("basic")
# `op.input` takes the place of `flow.input` and takes a `Dataflow`
# object as the first parameter.
# `op.input` returns a `Stream[int]` in this example:
stream = op.input("inp", flow, TestingSource(range(10)))
# `op.map` takes a `Stream` as it's second argument, and
# returns a new `Stream[int]` as it's return value.
add_one_stream = op.map("add_one", stream, lambda x: x + 1)
# `op.output` takes a stream as it's second argument, but
# does not return a new `Stream`.
op.output("out", add_one_stream, StdOutSink())
```

### Kafka/RedPanda Input

{py:obj}`~bytewax.connectors.kafka.KafkaSource` has been updated.
{py:obj}`~bytewax.connectors.kafka.KafkaSource` now returns a stream
of {py:obj}`~bytewax.connectors.kafka.KafkaSourceMessage` dataclasses,
and a stream of errors rather than a stream of (key, value) tuples.

You can still use {py:obj}`~bytewax.connectors.kafka.KafkaSource` and
{py:obj}`~bytewax.connectors.kafka.KafkaSink` directly, or use the
custom operators in {py:obj}`bytewax.connectors.kafka` to construct an
input source.

Before:

```python
from bytewax.connectors.kafka import KafkaInput, KafkaOutput
from bytewax.connectors.stdio import StdOutput
from bytewax.dataflow import Dataflow

flow = Dataflow()
flow.input("inp", KafkaInput(["localhost:9092"], ["input_topic"]))
flow.output(
    "out",
    KafkaOutput(
        brokers=["localhost:9092"],
        topic="output_topic",
    ),
)
```

After:

```python
from typing import Tuple, Optional

from bytewax import operators as op
from bytewax.connectors.kafka import operators as kop
from bytewax.connectors.kafka import KafkaSinkMessage
from bytewax.dataflow import Dataflow

flow = Dataflow("kafka_in_out")
kinp = kop.input("inp", flow, brokers=["localhost:19092"], topics=["in_topic"])
in_msgs = op.map("get_k_v", kinp.oks, lambda msg: (msg.key, msg.value))


def wrap_msg(k_v):
    k, v = k_v
    return KafkaSinkMessage(k, v)


out_msgs = op.map("wrap_k_v", in_msgs, wrap_msg)
kop.output("out1", out_msgs, brokers=["localhost:19092"], topic="out_topic")
```

### Renaming

We have renamed the IO classes to better match their semantics and the
way they are talked about in the documentation. Their functionality
are unchanged. You should be able to search and replace for the old
names and have them work correctly.

| Old | New |
|-----|-----|
| `Input` | `Source` |
| `PartitionedInput` | `FixedPartitionedSource` |
| `DynamicInput` | `DynamicSource` |
| `StatefulSource` | `StatefulSourcePartition` |
| `StatelessSource` | `StatelessSourcePartition` |
| `SimplePollingInput` | `SimplePollingSource` |
| `Output` | `Sink` |
| `PartitionedOutput` | `FixedPartitionedSink` |
| `DynamicOutput` | `DynamicSink` |
| `StatefulSink` | `StatefulSinkPartition` |
| `StatelessSink` | `StatelessSinkPartition` |

We have also updated the names of the built-in connectors to match
this new naming scheme.

| Old | New |
|-----|-----|
| `FileInput` | `FileSource` |
| `FileOutput` | `FileSink` |
| `DirInput` | `DirSource` |
| `DirOutput` | `DirSink` |
| `CSVInput` | `CSVSource` |
| `StdOutput` | `StdOutSink` |
| `KafkaInput` | `KafkaSource` |
| `KafkaOutput` | `KafkaSink` |
| `TestingInput` | `TestingSource` |
| `TestingOutput` | `TestingSink` |

### Current Time Convenience

In addition to the name changes, we have also added a `datetime` argument to
`build{_part,}` with the current time to allow you to easily
set up a next awake time. You can ignore this parameter if you are not scheduling
awake times.

We have also added a `datetime` argument to `next_batch`, which contains the
scheduled awake time for that source. Since awake times are scheduled, but not
guaranteed to fire at the precise time specified, you can use this parameter
to account for any difference.


### `SimplePollingSource` moved

{py:obj}`~bytewax.inputs.SimplePollingSource` has been moved from
`bytewax.connectors.periodic` to {py:obj}`bytewax.inputs`. You'll need
to change your imports if you are using that class.

Before:

```python
from bytewax.connectors.periodic import SimplePollingSource
```

After:

```python
from bytewax.inputs import SimplePollingSource
```

### Window Metadata

Window operators now emit
{py:obj}`~bytewax.operators.windowing.WindowMetadata` objects
downstream. These objects can be used to introspect the open_time and
close_time of windows. This changes the output type of windowing
operators from a stream of: `(key, values)` to a stream of `(key,
(metadata, values))`.

### Recovery flags

The default values for `snapshot-interval` and `backup-interval` have been removed
when running a dataflow with recovery enabled.

Previously, the defaults values were to create a snapshot every 10 seconds and
keep a day's worth of old snapshots. This means your recovery DB would max out at a size on disk
theoretically thousands of times bigger than your in-memory state.

See <project:#xref-recovery> for how to appropriately pick these
values for your deployment.

### Batch -> Collect

In v0.18, we've renamed the `batch` operator to
{py:obj}`~bytewax.operators.collect` so as to not be confused with
runtime batching. Behavior is unchanged.

## From v0.16 to v0.17

Bytewax v0.17 introduces major changes to the way that recovery works
in order to support rescaling. In v0.17, the number of workers in a
cluster can now be changed by stopping the dataflow execution and
specifying a different number of workers on resume.

In addition to rescaling, we've changed the Bytewax inputs and outputs
API to support batching which has yielded significant performance
improvements.

In this article, we'll go over the updates we've made to our API and
explain the changes you'll need to make to your dataflows to upgrade
to v0.17.

### Recovery changes

In v0.17, recovery has been changed to support rescaling the number of
workers in a dataflow. You now pre-create a fixed number of SQLite
recovery DB files before running the dataflow.

SQLite recovery DBs created with versions of Bytewax prior to v0.17
are not compatible with this release.

#### Creating recovery partitions

Creating recovery stores has been moved to a separate step from
running your dataflow.

Recovery partitions must be pre-initialized before running the
dataflow initially. This is done by executing this module:

``` bash
$ python -m bytewax.recovery db_dir/ 4
```


This will create a set of partitions:

``` bash
$ ls db_dir/
part-0.sqlite3
part-1.sqlite3
part-2.sqlite3
part-3.sqlite3
```

Once the recovery partition files have been created, they must be
placed in locations that are accessible to the workers. The cluster
has a whole must have access to all partitions, but any given worker
need not have access to any partition in particular (or any at
all). It is ok if a given partition is accesible by multiple workers;
only one worker will use it.

If you are not running in a cluster environment but on a single
machine, placing all the partitions in a single local filesystem
directory is fine.

Although the partition init script will not create these, partitions
after execution may consist of multiple files:

```
$ ls db_dir/
part-0.sqlite3
part-0.sqlite3-shm
part-0.sqlite3-wal
part-1.sqlite3
part-2.sqlite3
part-3.sqlite3
part-3.sqlite3-shm
part-3.sqlite3-wal
```
You must remember to move the files with the prefix `part-*.` all together.

#### Choosing the number of recovery partitions

An important consideration when creating the initial number of
recovery partitions; When rescaling the number of workers, the number
of recovery partitions is currently fixed.

If you are scaling up the number of workers in your cluster to
increase the total throughput of your dataflow, the work of writing
recovery data to the recovery partitions will still be limited to the
initial number of recovery partitions. If you will likely scale up
your dataflow to accomodate increased demand, we recommend that that
you consider creating more recovery partitions than you will initially
need. Having multiple recovery partitions handled by a single worker
is fine.

### Epoch interval -> Snapshot interval

We've renamed the cli option of `epoch-interval` to
`snapshot-interval` to better describe its affect on dataflow
execution. The snapshot interval is the system time duration (in
seconds) to snapshot state for recovery.

Recovering a dataflow can only happen on the boundaries of the most
recently completed snapshot across all workers, but be aware that
making the `snapshot-interval` more frequent increases the amount of
recovery work and may impact performance.

### Backup interval, and backing up recovery partitions.

We've also introduced an additional parameter to running a dataflow:
`backup-interval`.

When running a Dataflow with recovery enabled, it is recommended to
back up your recovery partitions on a regular basis for disaster
recovery.

The `backup-interval` parameter is the length of time to wait before
"garbage collecting" older snapshots. This enables a dataflow to
successfully resume when backups of recovery partitions happen at
different times, which will be true in most distributed deployments.

This value should be set in excess of the interval you can guarantee
that all recovery partitions will be backed up to account for
transient failures. It defaults to one day.

If you attempt to resume from a set of recovery partitions for which
the oldest and youngest backups are more than the backup interval
apart, the resumed dataflow could have corrupted state.

### Input and Output changes

In v0.17, we have restructured input and output to support batching
for increased throughput.

If you have created custom input connectors, you'll need to update
them to use the new API.

### Input changes

The `list_parts` method has been updated to return a `List[str]` instead of
a `Set[str]`, and now should only reflect the available input partitions
that a given worker has access to. You no longer need to return the
complete set of partitions for all workers.

The `next` method of `StatefulSource` and `StatelessSource` has been
changed to `next_batch` and should to return a `List` of elements each
time it is called. If there are no elements to return, this method
should return the empty list.

#### Next awake

Input sources now have an optional `next_awake` method which you can
use to schedule when the next `next_batch` call should occur. You can
use this to "sleep" the input operator for a fixed amount of time
while you are waiting for more input.

The default behavior uses a simple heuristic to prevent a spin loop
when there is no input. Always use `next_awake` rather than using a
`time.sleep` in an input source.

See the `periodic_input.py` example in the examples directory for an
implementation that uses this functionality.

### Async input adapter

We've included a new `bytewax.inputs.batcher_async` to help you use
async Python libraries in Bytewax input sources. It lets you wrap an
async iterator and specify a maximum time and size to collect items
into a batch.

### Using Kafka for recovery is now removed

v0.17 removes the deprecated `KafkaRecoveryConfig` as a recovery store
to support the ability to rescale the number of workers.

### Support for additional platforms

Bytewax is now available for linux/aarch64 and linux/armv7.

## From 0.15 to 0.16

Bytewax v0.16 introduces major changes to the way that custom Bytewax
inputs are created, as well as improvements to windowing and
execution. In this article, we'll cover some of the updates we've made
to our API to make upgrading to v0.16 easier.

### Input changes

In v0.16, we have restructured input to be based around partitions in
order to support rescaling stateful dataflows in the future. We have
also dropped the `Config` suffix to our input classes. As an example,
`KafkaInputConfig` has been renamed to `KafkaInput` and has been moved
to `bytewax.connectors.kafka.KafkaInput`.

`ManualInputConfig` is now replaced by two base superclasses that can
be subclassed to create custom input sources. They are `DynamicInput`
and `PartitionedInput`.

#### `DynamicInput`

`DynamicInput` sources are input sources that support reading distinct
items from any number of workers concurrently.

`DynamicInput` sources do not support resume state and thus generally
do not provide delivery guarantees better than at-most-once.

### `PartitionedInput`

`PartitionedInput` sources can keep internal state on the current
position of each partition. If a partition can "rewind" and resume
reading from a previous position, they can provide delivery guarantees
of at-least-once or better.

`PartitionedInput` sources maintain the state of each source and
re-build that state during recovery.

### Deprecating Kafka Recovery

In order to better support rescaling Dataflows, the Kafka recovery
store option is being deprecated and will be removed from a future
release in favor of the SQLite recovery store.

### Capture -> Output

The `capture` operator has been renamed to `output` in v0.16 and is
now stateful, so requires a step ID:

In v0.15 and before:

``` python doctest:SKIP
flow.capture(StdOutputConfig())
```

In v0.16+:

``` python doctest:SKIP
from bytewax.dataflow import Dataflow
from bytewax.connectors.stdio import StdOutput

flow = Dataflow()
flow.output("out", StdOutput())
```

`ManualOutputConfig` is now replaced by two base superclasses that can
be subclassed to create custom output sinks. They are `DynamicOutput`
and `PartitionedOutput`.

### New entrypoint

In Bytewax v0.16, the way that Dataflows are run has been simplified,
and most execution functions have been removed.

Similar to other Python frameworks like Flask and FastAPI, Dataflows
can now be run using a Datfaflow import string in the
`<module_name>:<dataflow_variable_name_or_factory_function>` format.
As an example, given a file named `dataflow.py` with the following
contents:

``` python doctest:SKIP
from bytewax.dataflow import Dataflow
from bytewax.testing import TestingInput
from bytewax.connectors.stdio import StdOutput

flow = Dataflow()
flow.input("in", TestingInput(range(3)))
flow.output("out", StdOutput())
```

Since a Python file `dataflow.py` defines a module `dataflow`, the
Dataflow can be run with the following invocation:

``` bash
> python -m bytewax.run dataflow
0
1
2
```

By default, Bytewax looks for a variable in the given module named
`flow`, so we can eliminate the
`<dataflow_variable_name_or_factory_function>` part of our import
string.

Processes, workers, recovery stores and other options can be
configured with command line flags or environment variables. For the
full list of options see the `--help` command line flag:

``` bash
> python -m bytewax.run --help
usage: python -m bytewax.run [-h] [-p PROCESSES] [-w WORKERS_PER_PROCESS] [-i PROCESS_ID] [-a ADDRESSES] [--sqlite-directory SQLITE_DIRECTORY] [--epoch-interval EPOCH_INTERVAL] import_str

Run a bytewax dataflow

positional arguments:
  import_str            Dataflow import string in the format <module_name>:<dataflow_variable_or_factory> Example: src.dataflow:flow or src.dataflow:get_flow('string_argument')

options:
  -h, --help            show this help message and exit

Scaling:
  You should use either '-p' to spawn multiple processes on this same machine, or '-i/-a' to spawn a single process on different machines

  -p PROCESSES, --processes PROCESSES
                        Number of separate processes to run [env: BYTEWAX_PROCESSES]
  -w WORKERS_PER_PROCESS, --workers-per-process WORKERS_PER_PROCESS
                        Number of workers for each process [env: BYTEWAX_WORKERS_PER_PROCESS]
  -i PROCESS_ID, --process-id PROCESS_ID
                        Process id [env: BYTEWAX_PROCESS_ID]
  -a ADDRESSES, --addresses ADDRESSES
                        Addresses of other processes, separated by semicolumn: -a "localhost:2021;localhost:2022;localhost:2023" [env: BYTEWAX_ADDRESSES]

Recovery:
  --sqlite-directory SQLITE_DIRECTORY
                        Passing this argument enables sqlite recovery in the specified folder [env: BYTEWAX_SQLITE_DIRECTORY]
  --epoch-interval EPOCH_INTERVAL
                        Number of seconds between state snapshots [env: BYTEWAX_EPOCH_INTERVAL]
```

### Porting a simple example from 0.15 to 0.16

This is what a simple example looked like in `0.15`:

```python
import operator
import re

from datetime import timedelta, datetime

from bytewax.dataflow import Dataflow
from bytewax.inputs import ManualInputConfig
from bytewax.outputs import StdOutputConfig
from bytewax.execution import run_main
from bytewax.window import SystemClockConfig, TumblingWindowConfig


def input_builder(worker_index, worker_count, resume_state):
    state = None  # ignore recovery
    for line in open("wordcount.txt"):
        yield state, line


def lower(line):
    return line.lower()


def tokenize(line):
    return re.findall(r'[^\s!,.?":;0-9]+', line)


def initial_count(word):
    return word, 1


def add(count1, count2):
    return count1 + count2


clock_config = SystemClockConfig()
window_config = TumblingWindowConfig(length=timedelta(seconds=5))

flow = Dataflow()
flow.input("input", ManualInputConfig(input_builder))
flow.map(lower)
flow.flat_map(tokenize)
flow.map(initial_count)
flow.reduce_window("sum", clock_config, window_config, add)
flow.capture(StdOutputConfig())

run_main(flow)
```

To port the example to the `0.16` version we need to make a few
changes.

#### Imports

Let's start with the existing imports:

```python
from bytewax.inputs import ManualInputConfig
from bytewax.outputs import StdOutputConfig
from bytewax.execution import run_main
```

Becomes:
```python
from bytewax.connectors.files import FileInput
from bytewax.connectors.stdio import StdOutput
```

We removed `run_main` as it is now only used for unit testing
dataflows. Bytewax now has a built-in FileInput connector, which uses
the `PartitionedInput` connector superclass.

Since we are using a built-in connector to read from this file, we can
delete our `input_builder` function above.

#### Windowing

Most of the operators from `v0.15` are the same, but the `start_at`
parameter of windowing functions has been changed to `align_to`. The
`start_at` parameter incorrectly implied that there were no potential
windows before that time. What determines if an item is late for a
window is not the windowing definition, but the watermark of the
clock.

`SlidingWindow` and `TumblingWindow` now require the `align_to`
parameter. Previously, this was filled with the timestamp of the start
of the Dataflow, but must now be specified. Specifying this parameter
ensures that windows are consistent across Dataflow restarts, so make
sure that you don't change this parameter between executions.


The old `TumblingWindow` definition:

```python
clock_config = SystemClockConfig()
window_config = TumblingWindowConfig(length=timedelta(seconds=5))
```

becomes:

```python
clock_config = SystemClockConfig()
window_config = TumblingWindow(
    length=timedelta(seconds=5), align_to=datetime(2023, 1, 1, tzinfo=timezone.utc)
)
```

#### Output and execution

Similarly to the `input`, the output configuration is now stateful.
`capture` has been renamed to `output`, and now takes a name, as all
stateful operators do.

So we move from this:

```python
flow.capture(StdOutputConfig())
```

To this:

```python
flow.output("out", StdOutput())
```

#### Complete code

The complete code for the new simple example now looks like this:

```python
import operator
import re

from datetime import timedelta, datetime, timezone

from bytewax.dataflow import Dataflow
from bytewax.connectors.files import FileInput
from bytewax.connectors.stdio import StdOutput
from bytewax.window import SystemClockConfig, TumblingWindow


def lower(line):
    return line.lower()


def tokenize(line):
    return re.findall(r'[^\s!,.?":;0-9]+', line)


def initial_count(word):
    return word, 1


def add(count1, count2):
    return count1 + count2


clock_config = SystemClockConfig()
window_config = TumblingWindow(
    length=timedelta(seconds=5), align_to=datetime(2023, 1, 1, tzinfo=timezone.utc)
)

flow = Dataflow()
flow.input("inp", FileInput("examples/sample_data/wordcount.txt"))
flow.map(lower)
flow.flat_map(tokenize)
flow.map(initial_count)
flow.reduce_window("sum", clock_config, window_config, add)
flow.output("out", StdOutput())
```

Running our completed Dataflow looks like this, as we've named our
Dataflow variable `flow` in a file named `dataflow`:

``` bash
> python -m bytewax.run dataflow:flow
('whether', 1)
("'tis", 1)
('of', 2)
('opposing', 1)
...
```

We can even run our sample Dataflow on multiple workers to process the
file in parallel:

``` bash
> python -m bytewax.run dataflow:flow -p2
('whether', 1)
("'tis", 1)
('of', 2)
('opposing', 1)
...
```

In the background, Bytewax has spawned two processes, each of which is
processing a part of the file. To see this more clearly, you can start
each worker by hand:

In one terminal, run:

``` bash
> python -m bytewax.run dataflow:flow -i0 -a "localhost:2101;localhost:2102"
('not', 1)
('end', 1)
('opposing', 1)
('take', 1)
...
```

And in a second terminal, run:

``` bash
> python -m bytewax.run dataflow:flow -i1 -a "localhost:2101;localhost:2102"
('question', 1)
('the', 3)
('troubles', 1)
('fortune', 1)
...
```

## From 0.10 to 0.11

Bytewax 0.11 introduces major changes to the way that Bytewax
dataflows are structured, as well as improvements to recovery and
windowing. This document outlines the major changes between Bytewax
0.10 and 0.11.

### Input and epochs

Bytewax is built on top of the [Timely
Dataflow](https://github.com/TimelyDataflow/timely-dataflow)
framework. The idea of timestamps (which we refer to in Bytewax as
epochs) is central to Timely Dataflow's progress tracking mechanism.

Bytewax initially adopted an input model that included managing the
epochs at which input was introduced. The 0.11 version of Bytewax
removes the need to manage epochs directly.

Epochs continue to exist in Bytewax, but are now managed internally to
represent a unit of recovery. Bytewax dataflows that are configured
with recovery will shapshot their state after processing all items in
an epoch. In the event of recovery, Bytewax will resume a dataflow at
the last snapshotted state. The frequency of snapshotting can be
configured with an `EpochConfig`

### Recoverable input

Bytewax 0.11 will now allow you to recover the state of the input to
your dataflow.

Manually constructed input functions, like those used with
`ManualInputConfig` now take a third argument. If your dataflow is
interrupted, the third argument passed to your input function can be
used to reconstruct the state of your input at the last recovery
snapshot, provided you write your input logic accordingly. The
`input_builder` function must return a tuple of (resume_state, datum).

Bytewax's built-in input handlers, like `KafkaInputConfig` are also
recoverable. `KafkaInputConfig` will store information about consumer
offsets in the configured Bytewax recovery store. In the event of
recovery, `KafkaInputConfig` will start reading from the offsets that
were last committed to the recovery store.

### Stateful windowing

Version 0.11 also introduces stateful windowing operators, including a
new `fold_window` operator.

Previously, Bytewax included helper functions to manage windows in
terms of epochs. Now that Bytewax manages epochs internally, windowing
functions are now operators that appear as a processing step in a
dataflow. Dataflows can now have more than one windowing step.

Bytewax's stateful windowing operators are now built on top of its
recovery system, and their operations can be recovered in the event of
a failure. See the documentation on <project:#xref-recovery> for more
information.

### Output configurations

The 0.11 release of Bytewax adds some prepackaged output configuration
options for common use-cases:

`ManualOutputConfig` which calls a Python callback function for each
output item.

`StdOutputConfig` which prints each output item to stdout.

### Import path changes and removed entrypoints

In Bytewax 0.11, the overall Python module structure has changed, and
some execution entrypoints have been removed.

- `cluster_main`, `spawn_cluster`, and `run_main` have moved to
  `bytewax.execution`

- `Dataflow` has moved to `bytewax.dataflow`

- `run` and `run_cluster` have been removed

### Porting the Simple example from 0.10 to 0.11

This is what the `Simple example` looked like in `0.10`:

```python
import re

from bytewax import Dataflow, run


def file_input():
    for line in open("wordcount.txt"):
        yield 1, line


def lower(line):
    return line.lower()


def tokenize(line):
    return re.findall(r'[^\s!,.?":;0-9]+', line)


def initial_count(word):
    return word, 1


def add(count1, count2):
    return count1 + count2


flow = Dataflow()
flow.map(lower)
flow.flat_map(tokenize)
flow.map(initial_count)
flow.reduce_epoch(add)
flow.capture()


for epoch, item in run(flow, file_input()):
    print(item)
```

To port the example to the `0.11` version we need to make a few
changes.

#### Imports

Let's start with the existing imports:

```python
from bytewas import Dataflow, run
```

Becomes:
```python
from bytewax.dataflow import Dataflow
from bytewax.execution import run_main
```

We moved from `run` to `run_main` as the execution API has been
simplified, and we can now just use the `run_main` function to execute
our dataflow.

### Input

The way bytewax handles input changed with `0.11`. `input` is now a
proper operator on the Dataflow, and the function now takes 3
parameters: `worker_index`, `worker_count`, `resume_state`. This
allows us to distribute the input across workers, and to handle
recovery if we want to. We are not going to do that in this example,
so the change is minimal.

The input function goes from:

```python
def file_input():
    for line in open("wordcount.txt"):
        yield 1, line
```

to:

```python
def input_builder(worker_index, worker_count, resume_state):
    state = None  # ignore recovery
    for line in open("wordcount.txt"):
        yield state, line
```

So instead of manually yielding the `epoch` in the input function, we
can either ignore it (passing `None` as state), or handle the value to
implement recovery (see our docs on <project:#xref-recovery>).

Then we need to wrap the `input_builder` with `ManualInputConfig`,
give it a name ("file_input" here) and pass it to the `input` operator
(rather than the `run` function):

```python
from bytewax.inputs import ManualInputConfig


flow.input("file_input", ManualInputConfig(input_builder))
```

### Operators

Most of the operators are the same, but there is a notable change in
the flow: where we used `reduce_epoch` we are now using
`reduce_window`. Since the epochs concept is now considered an
internal detail in bytewax, we need to define a way to let the
`reduce` operator know when to close a specific window. Previously
this was done everytime the `epoch` changed, while now it can be
configured with a time window. We need two config objects to do this:

- `clock_config`

- `window_config`

The `clock_config` is used to tell the window-based operators what
reference clock to use, here we use the `SystemClockConfig` that just
uses the system's clock. The `window_config` is used to define the
time window we want to use. Here we'll use the `TumblingWindow` that
allows us to have tumbling windows defined by a length (`timedelta`),
and we configure it to have windows of 5 seconds each.

So the old `reduce_epoch`:
```python
flow.reduce_epoch(add)
```

becomes `reduce_window`:

```python
from bytewax.window import SystemClockConfig, TumblingWindow


clock_config = SystemClockConfig()
window_config = TumblingWindow(length=timedelta(seconds=5))
flow.reduce_window("sum", clock_config, window_config, add)
```

### Output and execution

Similarly to the `input`, the output configuration is now part of an
operator, `capture`. Rather than collecting the output in a python
iterator and then manually printing it, we can now configure the
`capture` operator to print to standard output.

Since all the input and output handling is now defined inside the
Dataflow, we don't need to pass this information to the execution
method.

So we move from this:

```python
flow.capture()

for epoch, item in run(flow, file_input()):
    print(item)
```

To this:

```python
from bytewax.outputs import StdOutputConfig


flow.capture(StdOutputConfig())

run_main(flow)
```

### Complete code

The complete code for the new simple example now looks like this:

```python
import operator
import re

from datetime import timedelta, datetime

from bytewax.dataflow import Dataflow
from bytewax.inputs import ManualInputConfig
from bytewax.outputs import StdOutputConfig
from bytewax.execution import run_main
from bytewax.window import SystemClockConfig, TumblingWindow


def input_builder(worker_index, worker_count, resume_state):
    state = None  # ignore recovery
    for line in open("wordcount.txt"):
        yield state, line


def lower(line):
    return line.lower()


def tokenize(line):
    return re.findall(r'[^\s!,.?":;0-9]+', line)


def initial_count(word):
    return word, 1


def add(count1, count2):
    return count1 + count2


clock_config = SystemClockConfig()
window_config = TumblingWindow(length=timedelta(seconds=5))

flow = Dataflow()
flow.input("input", ManualInputConfig(input_builder))
flow.map(lower)
flow.flat_map(tokenize)
flow.map(initial_count)
flow.reduce_window("sum", clock_config, window_config, add)
flow.capture(StdOutputConfig())

run_main(flow)
```
