## Challenge

In order to provably execute deterministic binaries in an untrusted environment, (ie. proving untampered execution of `smi` executable that captures NVIDIA card metrics),
one construction we could look at is the use of a SNARK.

Using a circuit-based cryptographic library, executing an external binary and capturing its output directly within a circuit isn't feasible. 
Zero-Knowledge Proof (ZKP) circuits, like those created with `snarkjs`, are designed to perform deterministic, fixed computations defined by arithmetic circuits. 
They are not capable of interacting with the external environment, executing binaries, or performing operations like checksum verification outside the predetermined circuit.

However, we can achieve the desired effect by structuring out problem differently:

1. **Pre-process the Binary and Capture Output**: 
   - Before creating the proof, you execute the binary externally in a trusted environment.
   - Capture the output and compute a checksum (e.g., SHA-256) of the binary.

2. **Create a ZKP Circuit to Verify**:
   - The circuit can take the checksum, the binary's hash, and the captured output as inputs.
   - The circuit then verifies that the provided binary hash matches the expected one and that the output corresponds to the expected result for the given binary and inputs.

### Steps to Implement:

1. **External Execution and Checksum Calculation**:
   - Execute the binary.
   - Compute the checksum of the binary.
   - Capture the output of the binary.

2. **Circuit Design**:
   - Create a ZKP circuit that verifies the checksum.
   - Ensure the circuit also takes as input the precomputed output and verifies its validity.

3. **Proof Generation**:
   - Generate a proof that asserts the correctness of the checksum and the output.

4. **Verification**:
   - The verifier checks the proof, ensuring that the binary was correct (not tampered with) and that the output matches what it should be for that binary.

### Pseudocode for Implementation:

```js
const { createHash } = require('crypto');
const { execSync } = require('child_process');
const snarkjs = require('snarkjs');

// 1. Compute checksum of the binary
const binaryPath = "path/to/binary";
const binaryData = require('fs').readFileSync(binaryPath);
const checksum = createHash('sha256').update(binaryData).digest('hex');

// 2. Execute the binary and capture output
const output = execSync(`${binaryPath} --args`).toString().trim();

// 3. Design circuit to verify checksum and output
// This will be done in Circom

// circuit.circom
// ...
// component main = ... {
//     // Inputs: binary checksum, expected output, actual output
//     signal input expectedChecksum;
//     signal input actualChecksum;
//     signal input expectedOutput;
//     signal input actualOutput;
    
//     // Logic to compare checksums and outputs
//     expectedChecksum === actualChecksum;
//     expectedOutput === actualOutput;
// };

// 4. Generate the proof using snarkjs
// snarkjs plonk prove circuit_final.zkey witness.wtns proof.json public.json

// 5. Verify the proof
// snarkjs plonk verify verification_key.json public.json proof.json

// 5a. Verify the proof with a smart contract
// Generate proof
// snarkjs zkey export solidityverifier circuit_final.zkey verifier.sol
// Generate calldata
// snarkjs zkey export soliditycalldata public.json proof.json
```

### Example

To prove the output of executing the `/usr/bin/nvidia-smi` command using a Zero-Knowledge Proof (ZKP) approach, 
you would need to follow a structured approach. Since you cannot directly execute a command within a ZKP circuit, 
the process involves capturing the output externally, hashing the output, and then proving the validity of that 
hash and other related data within a ZKP circuit.

Here's how you can do it:

### 1. **Execute `/usr/bin/nvidia-smi` and Capture Output**

First, you execute the command and capture its output.

```javascript
const { execSync } = require('child_process');
const { createHash } = require('crypto');

// Execute the command and capture the output
const output = execSync('/usr/bin/nvidia-smi').toString().trim();

// Compute the hash of the output
const outputHash = createHash('sha256').update(output).digest('hex');

console.log("Output:", output);
console.log("Output Hash:", outputHash);
```

### 1b. **Calculate the Checksum of the Binary**

First, compute the checksum of the `/usr/bin/nvidia-smi` binary using a cryptographic hash function (e.g., SHA-256).

```javascript
const { execSync } = require('child_process');
const { createHash } = require('crypto');
const fs = require('fs');

// Compute the checksum of the binary
const binaryPath = '/usr/bin/nvidia-smi';
const binaryData = fs.readFileSync(binaryPath);
const binaryChecksum = createHash('sha256').update(binaryData).digest('hex');

// Execute the command and capture the output
const output = execSync(binaryPath).toString().trim();

// Compute the hash of the output
const outputHash = createHash('sha256').update(output).digest('hex');

console.log("Binary Checksum:", binaryChecksum);
console.log("Output:", output);
console.log("Output Hash:", outputHash);
```


### 2. **Design the ZKP Circuit**

You would create a ZKP circuit using a tool like Circom that verifies the hash of the output. The circuit will take the expected hash as input and compare it against the hash of the provided output.

#### Example Circuit (Pseudo-Circom Code)

```circom
pragma circom 2.0.0;

template ChecksumVerifier(expectedChecksum) {
    signal input actualChecksum;

    // Compare the actual checksum with the expected one
    component isEqual = IsEqual();
    isEqual.in[0] <== expectedChecksum;
    isEqual.in[1] <== actualChecksum;
    
    // Output must be 1 (true) if checksums match
    isEqual.out === 1;
}

template OutputHashVerifier(expectedOutputHash) {
    signal input actualOutputHash;

    // Compare the actual output hash with the expected one
    component isEqual = IsEqual();
    isEqual.in[0] <== expectedOutputHash;
    isEqual.in[1] <== actualOutputHash;
    
    // Output must be 1 (true) if hashes match
    isEqual.out === 1;
}

component main {
    signal input actualBinaryChecksum;
    signal input actualOutputHash;

    // Expected values for the binary checksum and output hash
    signal const expectedBinaryChecksum = <expected_binary_checksum>;
    signal const expectedOutputHash = <expected_output_hash>;

    // Instantiate the checksum verifier
    component binaryVerifier = ChecksumVerifier(expectedBinaryChecksum);
    binaryVerifier.actualChecksum <== actualBinaryChecksum;

    // Instantiate the output hash verifier
    component outputVerifier = OutputHashVerifier(expectedOutputHash);
    outputVerifier.actualOutputHash <== actualOutputHash;
}
```

### 3. **Proof Generation**

1. **Generate the inputs**: Prepare the inputs for the ZKP circuit, which include the checksum of the binary and the hash of the output.

```json
{
  "actualBinaryChecksum": "<binary_checksum>",
  "actualOutputHash": "<output_hash>"
}
```

2. **Compile and Prove**: Compile the circuit and generate the proof using `snarkjs`:

```bash
circom circuit.circom --r1cs --wasm --sym
snarkjs groth16 setup circuit.r1cs powersOfTau28_hez_final_10.ptau circuit_0000.zkey
snarkjs groth16 prove circuit_final.zkey witness.wtns proof.json public.json
```

### 4. **Verification**

Once the proof is generated, the verifier checks the proof using the public inputs, which are the expected binary checksum and output hash:

```bash
snarkjs groth16 verify verification_key.json public.json proof.json
```

### Summary

- **Binary Checksum Calculation**: Compute the checksum of the binary and include it in the proof.
- **Output Hash Calculation**: Capture and hash the output of the binary execution.
- **ZKP Circuit Design**: Extend the circuit to verify both the binary checksum and the output hash.
- **Proof Generation**: Generate a proof that both the checksum of the binary and the output hash match the expected values.
- **Verification**: The verifier confirms the proof against the expected checksum and output hash.

### Caveats

- **Deterministic Output**: Ensure that the output of `/usr/bin/nvidia-smi` is deterministic for your inputs, as the ZKP requires the output to be consistent.
- **Trusted Environment**: The execution of the command must be done in a trusted environment since the ZKP circuit itself cannot ensure the integrity of the command execution process.

This process allows you to prove that the output of executing a command like `/usr/bin/nvidia-smi` was valid and has not been tampered with, without revealing the actual output.

### Considerations:

- **Security**: The external execution environment must be trusted, as the circuit cannot enforce security for the external binary's execution.
- **Determinism**: Ensure that the output of the binary is deterministic, as ZKP circuits require consistent results for verification.
