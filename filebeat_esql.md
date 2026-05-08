Here's the ES|QL conversion for your filebeat query. The structure mirrors the metricbeat one, with two version exclusions and the added `cloud.account.id` field.

## ES|QL Query

```esql
FROM *:filebeat-*-mt-*
| WHERE @timestamp > NOW() - 7 days
  AND cloud.account.id IS NOT NULL
  AND agent.version IS NOT NULL
  AND NOT (agent.version IN ("8.19.5", "8.19.4"))
| STATS agent.version = VALUES(agent.version),
        cloud.account.id = VALUES(cloud.account.id),
        cloud.account.name = VALUES(cloud.account.name),
        labels.app_code = VALUES(labels.app_code)
    BY host.hostname
| KEEP host.hostname, cloud.account.id, cloud.account.name, labels.app_code, agent.version
| SORT host.hostname ASC
| LIMIT 1000
```

## Alternative: Latest Record Per Host (mirrors `top_hits` size:1)

If you want exactly one row per host based on the most recent document (closer to the original `top_hits` behavior):

```esql
FROM *:filebeat-*-mt-*
| WHERE @timestamp > NOW() - 7 days
  AND cloud.account.id IS NOT NULL
  AND agent.version IS NOT NULL
  AND NOT (agent.version IN ("8.19.5", "8.19.4"))
| KEEP @timestamp, host.hostname, cloud.account.id, cloud.account.name, labels.app_code, agent.version
| SORT @timestamp DESC
| STATS agent.version = TOP(agent.version, 1, "desc"),
        cloud.account.id = TOP(cloud.account.id, 1, "desc"),
        cloud.account.name = TOP(cloud.account.name, 1, "desc"),
        labels.app_code = TOP(labels.app_code, 1, "desc")
    BY host.hostname
| SORT host.hostname ASC
| LIMIT 1000
```

## Output Formats via `_query` API

### CSV Format
```
POST /_query?format=csv
{
  "query": """
    FROM *:filebeat-*-mt-*
    | WHERE @timestamp > NOW() - 7 days
      AND cloud.account.id IS NOT NULL
      AND agent.version IS NOT NULL
      AND NOT (agent.version IN ("8.19.5", "8.19.4"))
    | STATS agent.version = VALUES(agent.version),
            cloud.account.id = VALUES(cloud.account.id),
            cloud.account.name = VALUES(cloud.account.name),
            labels.app_code = VALUES(labels.app_code)
        BY host.hostname
    | KEEP host.hostname, cloud.account.id, cloud.account.name, labels.app_code, agent.version
    | SORT host.hostname ASC
    | LIMIT 1000
  """
}
```

### Plain Text (Table) Format
```
POST /_query?format=txt
{
  "query": "<same query as above>"
}
```

### TSV Format
```
POST /_query?format=tsv
{
  "query": "<same query as above>"
}
```

### Other supported formats
- `json` — default structured output
- `yaml` — YAML output
- `smile` — binary JSON variant
- `cbor` — Concise Binary Object Representation
- `arrow` — Apache Arrow columnar format (efficient for large exports)

## Kibana Dev Tools (Full Example)

```
POST /_query?format=csv
{
  "query": """
    FROM *:filebeat-*-mt-*
    | WHERE @timestamp > NOW() - 7 days
      AND cloud.account.id IS NOT NULL
      AND agent.version IS NOT NULL
      AND NOT (agent.version IN ("8.19.5", "8.19.4"))
    | STATS agent.version = VALUES(agent.version),
            cloud.account.id = VALUES(cloud.account.id),
            cloud.account.name = VALUES(cloud.account.name),
            labels.app_code = VALUES(labels.app_code)
        BY host.hostname
    | KEEP host.hostname, cloud.account.id, cloud.account.name, labels.app_code, agent.version
    | SORT host.hostname ASC
    | LIMIT 1000
  """,
  "columnar": false
}
```

## Notes on this conversion

A few items specific to this query versus the metricbeat one:

1. **Multiple version exclusion** — The original used a nested `bool/should` with two `match_phrase` clauses. `NOT (agent.version IN ("8.19.5", "8.19.4"))` is the cleanest equivalent and easy to extend when more versions get excluded over time (e.g., as you roll out 8.19.6+).

2. **`cloud.account.id` added to output** — Unlike the metricbeat version, this query keeps `cloud.account.id` in the result set. Since you're already filtering on `cloud.account.id IS NOT NULL`, every row will have a value.

3. **Multi-value awareness** — `VALUES()` will return arrays if a host has reported under multiple `cloud.account.id` values in 7 days (rare, but possible during account migrations). If you want strictly scalar output, use the `TOP(...)` variant.

4. **CCS scope** — `*:filebeat-*-mt-*` will hit all remote clusters matching your multitenant pattern, same as the original `_search` call.

Want me to fold both this and the metricbeat query into a single Python wrapper (similar to `es_csv_export.py`) that runs them across your cluster set from `es_clusters.json` and writes timestamped CSVs per cluster?
