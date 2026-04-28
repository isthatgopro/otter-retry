# otter-retry

A retry-with-backoff prototype for Otter, with a small PIS demo.

## Included

- `DESIGN.md`: design proposal
- `otter/`: revised Otter code
- `pis/`: demo setup and output

## Demo

The demo shows a task failing twice with temporary `503` errors, being retried with backoff, then succeeding on the third attempt. Validation then succeeds, and the manifest records the retry history.

## Links

- [Design notes](./DESIGN.md)
- [Demo output](https://github.com/isthatgopro/pis/blob/main/OUTPUT.md)
