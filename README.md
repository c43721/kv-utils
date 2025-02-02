# @deno/kv-utils

Utilities for working with Deno KV.

## Working with JSON

`Deno.Kv` stores are able to store values that are serializable using the
structured clone algorithm. The challenge is that supports a lot of types that
are not serializable using JSON. To work around this, you can use the the
utilities to convert values which can be safely serialized to JSON as well as
deserialize them back. This makes it possible to fully represent entries and
values in a browser, or communicate them between Deno processes.

The JSON utilities are:

- `entryMaybeToJSON` - Convert a `Deno.KvEntryMaybe` to JSON.
- `entryToJSON` - Convert a `Deno.KvEntry` to JSON.
- `keyToJSON` - Convert a `Deno.KvKey` to JSON.
- `keyPartToJSON` - Convert a `Deno.KvKeyPart` to JSON.
- `valueToJSON` - Convert a value which can be stored in Deno KV to JSON.
- `toEntry` - Convert a JSON object to a `Deno.KvEntry`.
- `toEntryMaybe` - Convert a JSON object to a `Deno.KvEntryMaybe`.
- `toKey` - Convert a JSON object to a `Deno.KvKey`.
- `toKeyPart` - Convert a JSON object to a `Deno.KvKeyPart`.
- `toValue` - Convert a JSON object to a value which can be stored in Deno KV.

### Examples

Taking a maybe entry from Deno.Kv and converting it to JSON and sending it as a
response:

```ts ignore
import { entryMaybeToJSON } from "@deno/kv-utils";

const db = await Deno.openKv();

Deno.serve(async (_req) => {
  const maybeEntry = await db.get(["a"]);
  const json = entryMaybeToJSON(maybeEntry);
  return Response.json(json);
});
```

Taking a value that was serialized to JSON in a browser and storing it in Deno
KV:

```ts ignore
import { toValue } from "@deno/kv-utils";

const db = await Deno.openKv();

Deno.serve(async (req) => {
  const json = await req.json();
  const value = toValue(json);
  await db.set(["a"], value);
  return new Response(null, { status: 204 });
});
```

## Estimating the size of a value

When working with Deno KV and there is a need to have transactions be
infallible, it is helpful to be able to estimate the size of a value before
storing it. This is because there are limits on the size of values that can be
stored in Deno KV, as well as the size of atomic operations.

Deno KV stores values by using the V8 serialization format, which converts
objects to a binary format and then that value is stored in the KV store.

The `estimateSize} function can be used to estimate the size of a value in
bytes. While it is not 100% accurate, it is 10x faster than using the V8
serialize function, which is not available in some environments.

### Example

```ts
import { estimateSize } from "@deno/kv-utils";
import { assertEquals } from "@std/assert";

const value = { a: new Map([[{ a: 1 }, { b: /234/ }]]), b: false };
assertEquals(estimateSize(value), 36);
```

## Importing and exporting entries

Deno KV stores can be exported and imported. This is useful for backing up and
restoring data, as well as for transferring data between different Deno
processes.

The import and export utilities are:

- `exportEntries` - Export entries from a Deno KV store as a stream or a
  response.
- `importEntries` - Import entries into a Deno KV store.

### Examples

Exporting entries from a Deno KV store and saving them to a file:

```ts
import { exportEntries } from "@deno/kv-utils";

const db = await Deno.openKv();
const file = await Deno.open("export.ndjson", { write: true, create: true });
for await (const chunk of exportEntries(db, { prefix: ["person"] })) {
  await file.write(chunk);
}
file.close();
db.close();
```

Exporting entries from a Deno KV store and sending them as a response:

```ts ignore
import { exportEntries } from "@deno/kv-utils";

const db = await Deno.openKv();
const server = Deno.serve((_req) =>
  exportEntries(
    db,
    { prefix: ["person"] },
    { type: "response" },
  )
);

await server.finished;
db.close();
```

Importing entries from a file and storing them in a Deno KV store:

```ts
import { importEntries } from "@deno/kv-utils";
import { assert } from "@std/assert";

const db = await Deno.openKv();
const file = await Deno.open("export.ndjson", { read: true });
const result = await importEntries(db, file.readable);
assert(result.errors === 0);
db.close();
```

# License

MIT
