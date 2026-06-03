# Example API payloads

These JSON files are **fully synthetic** — invented lab values for demos, tests, and documentation. They are not derived from and do not represent any real patient.

## Try locally

With the server running (`npm start`):

```bash
curl -s -X POST http://localhost:3000/calculate \
  -H "Content-Type: application/json" \
  -d @examples/moderate-depletion.request.json
```

```bash
curl -s -X POST http://localhost:3000/calculate \
  -H "Content-Type: application/json" \
  -d @examples/severe-depletion.request.json
```

Remove the `_comment` field before sending if your client rejects unknown keys (the calculator ignores it).
