I'll convert your aggregation query to ES|QL. The key insight is that your current query gets the latest agent version per host using `top_hits`, which in ES|QL we accomplish using `STATS ... BY host.hostname` with the appropriate aggregation.

Here's the ES|QL conversion in multiple formats:

## ES|QL Query

```esql
FROM *:metricbeat-*-mt-*
| WHERE @timestamp > NOW() - 7 days
  AND cloud.account.id IS NOT NULL
  AND agent.version IS NOT NULL
  AND agent.version != "8.19.4"
| STATS agent.version = VALUES(agent.version),
        cloud.account.name = VALUES(cloud.account.name),
        labels.app_code = VALUES(labels.app_code)
    BY host.hostname
| KEEP host.hostname, cloud.account.name, labels.app_code, agent.version
| SORT host.hostname ASC
| LIMIT 1000
```

## Alternative: Latest Version Per Host (closer to `top_hits` behavior)

If you want exactly one row per host with the most recent values (mirroring `top_hits` size:1 sorted by recency):

```esql
FROM *:metricbeat-*-mt-*
| WHERE @timestamp > NOW() - 7 days
  AND cloud.account.id IS NOT NULL
  AND agent.version IS NOT NULL
  AND agent.version != "8.19.4"
| KEEP @timestamp, host.hostname, cloud.account.name, labels.app_code, agent.version
| SORT @timestamp DESC
| STATS agent.version = TOP(agent.version, 1, "desc"),
        cloud.account.name = TOP(cloud.account.name, 1, "desc"),
        labels.app_code = TOP(labels.app_code, 1, "desc")
    BY host.hostname
| SORT host.hostname ASC
| LIMIT 1000
```

## Output Formats via `_query` API

ES|QL supports multiple output formats directly via the API. Use the `format` query parameter:

### CSV Format
```
POST /_query?format=csv
{
  "query": """
    FROM *:metricbeat-*-mt-*
    | WHERE @timestamp > NOW() - 7 days
      AND cloud.account.id IS NOT NULL
      AND agent.version IS NOT NULL
      AND agent.version != "8.19.4"
    | STATS agent.version = VALUES(agent.version),
            cloud.account.name = VALUES(cloud.account.name),
            labels.app_code = VALUES(labels.app_code)
        BY host.hostname
    | KEEP host.hostname, cloud.account.name, labels.app_code, agent.version
    | SORT host.hostname ASC
    | LIMIT 1000
  """
}
```

### Plain Text (Table) Format
```
POST /_query?format=txt
{
  "query": "..."
}
```

### TSV Format
```
POST /_query?format=tsv
```

### JSON Format (default)
```
POST /_query?format=json
```

### Other supported formats
- `yaml` — YAML output
- `smile` — binary JSON variant
- `cbor` — Concise Binary Object Representation
- `arrow` — Apache Arrow columnar format (great for large exports)

## Running from Kibana Dev Tools

In Kibana, you can run it like this:

```
POST /_query?format=csv
{
  "query": """
    FROM *:metricbeat-*-mt-*
    | WHERE @timestamp > NOW() - 7 days
      AND cloud.account.id IS NOT NULL
      AND agent.version IS NOT NULL
      AND agent.version != "8.19.4"
    | STATS agent.version = VALUES(agent.version),
            cloud.account.name = VALUES(cloud.account.name),
            labels.app_code = VALUES(labels.app_code)
        BY host.hostname
    | KEEP host.hostname, cloud.account.name, labels.app_code, agent.version
    | SORT host.hostname ASC
    | LIMIT 1000
  """,
  "columnar": false
}
```

## Quick notes on the conversion

A few things worth flagging given your CCS work:

1. **CCS pattern preserved** — `*:metricbeat-*-mt-*` works the same in ES|QL `FROM` clauses, so your remote cluster federation still applies.

2. **`VALUES()` vs `TOP()`** — `VALUES()` returns all distinct values per host (could be multiple if a host has reported different versions in the 7-day window). `TOP(field, 1, "desc")` gives you just one. For agent version inventory, `VALUES()` is often more useful because it surfaces hosts with inconsistent versions.

3. **`must_not` → `!=`** — The version exclusion translates cleanly. If you want to exclude multiple versions later, use `NOT IN ("8.19.4", "8.19.3")`.

4. **`exists` → `IS NOT NULL`** — Direct equivalent in ES|QL.

5. **`size: 1000` cap** — The `LIMIT 1000` mirrors your terms aggregation size. Bump it if you have more than 1000 hosts (ES|QL default max is 10,000 unless overridden).

If you'd like, I can also wrap this into your `es_csv_export.py` flow with the PIT/scroll fallback pattern you've been using for CCS exports.
