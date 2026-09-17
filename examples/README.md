# CooL SDK Examples

This directory contains standalone, self-contained project examples demonstrating how to use the CooL SDK.

### Running an Example
To run any of the examples, navigate inside its specific folder and execute the following commands:
```bash
npm install # Links the local build of cool-nwc
npm start
```

### Available Projects

- **[basic/](basic)**: The fastest 30-second integration. Shows how to record a single piece of evidence and verify it.
- **[verification/](verification)**: Demonstrates how to write evidence to a file, verify it in a separate process, and observe a tampered copy fail validation.
- **[express/](express)**: Shows how to generate evidence for incoming HTTP requests to a model (handled off the critical response path).
- **[agent/](agent)**: Features an AI agent generating a mathematically verifiable audit trail of its decisions and tool calls.
- **[dstack/](dstack)**: Demonstrates how to bind generated evidence to a hardware-attested Phala dstack enclave.

*Note: The examples are configured with `"cool-nwc": "file:../.."` in their package.json, meaning they run against the local build in this repository. When you use this in your own project, the dependency will simply be `"cool-nwc": "^3.0.0"`.*
