---
title: OSCAL import and export
description: Import OSCAL JSON through the CLI or HTTP API and understand the limits of typed exports.
sidebar_position: 12
---

# OSCAL import and export

Operators and developers use the Continuous Compliance Framework (CCF) API to exchange OSCAL JSON documents with other tools. This guide covers interchange behavior verified against API `v0.21.1` and the next-release source. For released behavior, select the `0.21` version of this guide.

## How it works

The CLI connects directly to the configured database. HTTP import uses the running API and its authorization policy. Both decode JSON into OSCAL `1.1.3` types and persist relational records; neither is a general XML/YAML converter or a lossless document archive.

| OSCAL JSON root key | CLI import | HTTP import | CLI export type |
|---|---|---|---|
| `assessment-plan` | Yes | Yes | `ap` |
| `assessment-results` | Yes | Yes | Unsupported |
| `catalog` | Yes | Yes | `catalog` |
| `component-definition` | Yes | Yes | Unsupported |
| `plan-of-action-and-milestones` | Yes | Yes | Unsupported |
| `profile` | Yes | Yes | `profile` |
| `system-security-plan` | Yes | Yes | `ssp` |

Use one model wrapper per input file. CLI import creates documents for new UUIDs and updates existing documents additively: removed associations and zero-value fields are not deleted or cleared. It synchronizes profile controls. HTTP import uses `FirstOrCreate`: an existing UUID is not merged or updated. An existing SSP explicitly reports that nothing was written while still returning `success: true`.

Document interchange does not publish an SSP capability or create a downstream subscription. For offerings, inherited responsibilities and drift, use [SSP sharing](./ssp-sharing.md). For model concepts, see OSCAL (general documentation) and risks, findings and observations (general documentation).

## Before you start

- Use the API binary for your deployment and a migrated database. Follow [API overview](./index.md#run-the-api) and the [configuration reference](./configuration.md) for database setup. The CLI reads the same environment and working-directory `.env` as the server; confirm it points to your intended database.
- HTTP requests need a user JWT. Import requires `import:execute`; the bundled Cedar `admin` role grants it, while `contributor` does not. The default `builtin` driver permits authenticated users on this resource. Retrieval needs the model's `read` permission, such as `catalog:read`. See [Authentication and permissions](./authentication-and-permissions.md).
- The examples use fictional data. Use a test database, `curl` and `jq`; supply `TOKEN` through your shell environment. CLI database access does not use the HTTP token or role checks.

No feature flag enables interchange. CLI import/export and HTTP import are available without extra configuration.

## Use it

### Prepare a fictional document

Save this as `fictional-catalog.json`. It is a small JSON Catalog with a stable UUID:

```json
{
  "catalog": {
    "uuid": "be2e5088-6cac-41e5-9ec6-825c28685527",
    "metadata": {
      "title": "Fictional lab catalog",
      "last-modified": "2026-01-01T00:00:00Z",
      "version": "1.0.0",
      "oscal-version": "1.1.3"
    },
    "controls": [{"id": "lab-1", "title": "Review fictional lab access"}]
  }
}
```

### Import a file or directory with the CLI

`--file` (`-f`) is required and repeatable. Each value is a file or directory path; directories are traversed recursively and dot-prefixed entries are skipped. The inherited `--debug` (`-d`) flag enables database debugging. There is no filename-extension filter: every other file is attempted as JSON.

```sh
api oscal import --file ./fictional-catalog.json
```

Prepare a directory containing additional fictional OSCAL JSON documents, then import it:

```sh
mkdir -p ./fictional-oscal
```

```sh
cp ./fictional-catalog.json ./fictional-oscal/catalog.json
```

```sh
api oscal import -f ./fictional-oscal
```

For separate inputs, repeat the flag: `api oscal import -f ./fictional-catalog.json -f ./fictional-oscal`. This reuses the catalog UUID and demonstrates additive updates rather than creating a second catalog.

The importer continues through other entries after file-open, persistence and recovered document-conversion errors, then exits nonzero when accumulated errors remain. It is not an atomic batch: earlier writes can persist. Unsupported JSON root keys are logged but do **not** themselves produce a failing exit status. An initial JSON decode error is logged; when no model was decoded, the fallback decoder returns malformed-JSON errors. Inspect logs and stored resources as well as the exit code; this command does not perform full OSCAL schema validation.

### Export one supported type with the CLI

| Flag | Purpose | Type | Default |
|---|---|---|---|
| `--output`, `-o` | Destination JSON file; creates or truncates it. | `string` | `oscal_export.json` |
| `--type`, `-t` | Required, case-sensitive model selector: `ap`, `catalog`, `profile`, `ssp`. | `string` | Required |

```sh
api oscal export --type catalog --output ./fictional-catalogs-export.json
```

```sh
api oscal export -t ssp -o ./fictional-ssps-export.json
```

Export selects **all stored records of that type**. There is no UUID, active-only, catalog-type, SSP or title filter. Output is an indented JSON array of model objects, without `catalog`/`system-security-plan` root wrappers or a `data` envelope. No matches produces `[]`. This is not directly accepted by import, which expects a wrapped object.

For example, extract only the fictional catalog and restore its wrapper:

```sh
jq -e '{catalog: (.[] | select(.uuid == "be2e5088-6cac-41e5-9ec6-825c28685527"))}' ./fictional-catalogs-export.json > ./fictional-catalog-wrapped.json
```

This changes the envelope only. Exporters preload specific associations and nesting depths; they do not guarantee every original field or nested object survives. Keep original source files and compare important associations before using exports for migration. An unknown type, database error, output creation error or encode error terminates the command with a fatal error; an already-created file can be empty or partial. File-close errors are logged.

### Import through HTTP

Set your API origin:

```sh
export API_URL=https://api.example.test
```

Send `POST /api/oscal/import` as `multipart/form-data`. Repeat the exact field name `files` for multiple files. Let `curl` set the multipart boundary; do not set a JSON content type:

```sh
curl --fail-with-body -sS "$API_URL/api/oscal/import" \
  -H "Authorization: Bearer $TOKEN" \
  -F 'files=@./fictional-catalog.json;type=application/json' \
  -F 'files=@./fictional-catalog-wrapped.json;type=application/json'
```

The response uses this shape (one new file illustrated):

```json
{
  "data": {
    "total_files": 1,
    "successful_count": 1,
    "failed_count": 0,
    "results": [{
      "filename": "fictional-catalog.json",
      "success": true,
      "message": "Successfully imported catalog",
      "type": "Catalog",
      "title": "Fictional lab catalog"
    }]
  }
}
```

Each file is processed independently. HTTP `200` means **at least one** file succeeded, even if others failed. HTTP `400` with the same counts/results envelope means all files failed. Always inspect `data.failed_count` and every `results[].success` and `message`. Malformed multipart data or no `files` returns `400` with an API error instead.

Invalid JSON becomes a per-file failure (`Failed to parse JSON`). An unknown root key becomes `Unsupported OSCAL document type`; an empty object cannot be detected. XML, YAML, an unwrapped model and a CLI-exported array are not supported import formats. Invalid UUIDs in relational conversion can panic and cause a server error instead of a structured per-file result; validate documents before submitting. This is not full schema validation. SSP import validates by-component implementation statuses at requirement and statement levels. Profile control synchronization can fail after the document was saved; a failure does not imply rollback.

### Retrieve a catalog through HTTP

For the fictional catalog, use the supported full-catalog route:

```sh
curl --fail-with-body -sS "$API_URL/api/oscal/catalogs/be2e5088-6cac-41e5-9ec6-825c28685527/full" \
  -H "Authorization: Bearer $TOKEN" -o ./fictional-catalog-response.json
```

It returns `200` with the Catalog in `data`, not an OSCAL document envelope. Wrap it for interchange:

```sh
jq -e '{catalog: .data}' ./fictional-catalog-response.json > ./fictional-http-catalog.json
```

The ordinary catalog list/get loads metadata; `/full` also loads selected group and control levels. It does not preload back matter, and deep trees can still be incomplete. This is a supported retrieval example, not a generic HTTP export endpoint. Use this version's **API reference** for other resources' get/full routes and response schemas. Invalid catalog UUIDs return `400`; unknown UUIDs return `404`.

## Troubleshooting

| Symptom | Verify and resolve |
|---|---|
| CLI cannot connect | Check the database settings and working directory used by this process, not only the running API's environment. |
| CLI returns success but nothing appears | Inspect logs for unsupported root keys. Supply one supported wrapper and valid JSON; verify the document UUID through its resource route. |
| Re-import does not remove old content | CLI updates are additive. HTTP does not update existing UUIDs. Use the resource update/delete operations appropriate to the intended change. |
| HTTP returns `401` or `403` | Refresh the user token and check `import:execute`. A contributor's permission to create individual catalogs does not grant bulk import. |
| HTTP returns `400` | Distinguish an invalid multipart request from per-file results. Use `files`, supported JSON wrappers and valid UUIDs; inspect each message. |
| Export cannot be imported | Select a model object from the array and wrap it with its root key. Verify retained associations before relying on the result. |
| Partial import or export | Inspect counts, logs and output. Batches are not atomic, and an export error may leave a truncated file. Preserve the originals. |

## Related

- [API overview](./index.md)
- [Configuration reference](./configuration.md)
- [Authentication and permissions](./authentication-and-permissions.md)
- [SSP sharing](./ssp-sharing.md)
- OSCAL models (general documentation)
- Risks, findings and observations (general documentation)
