# ShelfSense learning map

## The loop

1. Choose one narrow behavior.
2. Write the input/output contract.
3. Implement the simplest version.
4. Test normal and bad inputs.
5. Run it on real examples.
6. Measure quality and inspect failures.
7. Explain what changed and why.
8. Only then expand the system.

## Product pipeline

```text
image
  -> validated bytes
  -> decoded pixels
  -> detected boxes
  -> identified SKU or unknown
  -> counts
  -> inventory events
  -> API/dashboard
```

## Concepts

- Detection: where are the objects?
- Identification: what product is each object?
- Aggregation: how many of each SKU?
- Observation: what did this image show at a point in time?
- Event: what action-worthy condition follows from the observation?
- Abstention: the system says “unknown” instead of guessing.

## First success criterion

Given a fixture image, the program returns a schema-valid observation containing boxes, SKU counts, unknown count, model version, and latency. It handles invalid images safely and has tests for the domain behavior.
