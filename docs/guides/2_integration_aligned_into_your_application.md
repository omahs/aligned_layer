# Integrating Aligned into your Application

Aligned can be integrated into your applications in a few simple steps to provide a way to verify ZK proofs generated inside your system.

This example shows a sample app that generates an SP1 proof that a user knows the answers to a quiz, then submits the proof to Aligned for verification. Finally, it includes a smart contract that verifies that a proof was verified in Aligned and mints an NFT.

You can find an example of the full flow of using Aligned in your app in the [ZKQuiz example](../../examples/zkquiz). 

## Steps

### Step 1 - Generate your ZK Proof

Generate your ZK proofs using any of the proving systems supported by Aligned.
For this example, we use the SP1 proving system. The current SP1 version used in Aligned is v1.0.8-testnet.

You can find an example of the quiz proof [program](../../examples/zkquiz/quiz/program/src/main.rs) as well as the [script](../../examples/zkquiz/quiz/script/src/main.rs) that generates it in the [ZKQuiz example](../../examples/zkquiz) directory.

### Step 2 - Write your smart contract

To check if a proof was verified in Aligned, you need to call to the Aligned ServiceManager contract inside your smart contract. 

Also, you will need a way to check that the proven program is your own.

The aligned CLI provides a way for you to get the verification key commitment without actually generating and submitting a proof.

You can do this by running the following command:

```bash
aligned get-vk-commitment --input <path_to_input_file>
```

The following is an example of how to call the `verifyBatchInclusionMethod` from the Aligned ServiceManager contract in your smart contract.

```solidity
contract YourContract {
    // Your contract variables ...
    address public alignedServiceManager;
    bytes32 public elfCommitment = <elf_commitment>;

    constructor(address _alignedServiceManager) {
        //... Your contract constructor ...
        alignedServiceManager = _alignedServiceManager;
    }
    
    // Your contract code ...
    
    function yourContractMethod(
        //... Your function variables, ...
        bytes32 proofCommitment,
        bytes32 pubInputCommitment,
        bytes32 provingSystemAuxDataCommitment,
        bytes20 proofGeneratorAddr,
        bytes32 batchMerkleRoot,
        bytes memory merkleProof,
        uint256 verificationDataBatchIndex
    ) {
        // ... Your function code
        
        require(elfCommitment == provingSystemAuxDataCommitment, "ELF does not match");
        
        (bool callWasSuccessful, bytes memory proofIsIncluded) = alignedServiceManager.staticcall(
            abi.encodeWithSignature(
                "verifyBatchInclusion(bytes32,bytes32,bytes32,bytes20,bytes32,bytes,uint256)",
                proofCommitment,
                pubInputCommitment,
                provingSystemAuxDataCommitment,
                proofGeneratorAddr,
                batchMerkleRoot,
                merkleProof,
                verificationDataBatchIndex
            )
        );

        require(callWasSuccessful, "static_call failed");
        
        bool proofIsIncludedBool = abi.decode(proofIsIncluded, (bool));
        require(proofIsIncludedBool, "proof not included in batch");
        
        // Your function code ...
    }
}
```

You can find the example of the smart contract that checks the proof was verified in Aligned in the [Quiz Verifier Contract](../../examples/zkquiz/contracts/src/VerifierContract.sol).

Note that the contract checks that the verification key commitment is the same as the program elf.

```solidity
require(elfCommitment == provingSystemAuxDataCommitment, "ELF does not match");
```

This contract also includes a static call to the Aligned ServiceManager contract to check if the proof was verified in Aligned.

```solidity
(bool callWasSuccessfull, bytes memory proofIsIncluded) = alignedServiceManager.staticcall(
    abi.encodeWithSignature(
        "verifyBatchInclusion(bytes32,bytes32,bytes32,bytes20,bytes32,bytes,uint256)",
        proofCommitment,
        pubInputCommitment,
        provingSystemAuxDataCommitment,
        proofGeneratorAddr,
        batchMerkleRoot,
        merkleProof,
        verificationDataBatchIndex
    )
);

require(callWasSuccessfull, "static_call failed");

bool proofIsIncludedBool = abi.decode(proofIsIncluded, (bool));
require(proofIsIncludedBool, "proof not included in batch");
```

### Step 3 - Submit and verify the proof to Aligned

First, generate the proof. For SP1, this means having the [script](../../examples/zkquiz/quiz/script/src/main.rs) generate the proof.

Then, submit the proof to Aligned for verification. This can be done either with the SDK or by using the Aligned CLI.

#### Using the SDK

To submit a proof using the SDK, you can use the `submit` function, and then you can use the `verify_proof_onchain` to check if the proof was correctly verified in Aligned.

The following code is an example of how to submit a proof using the SDK:

```rust
use aligned_sdk::sdk::submit;
use aligned_sdk::types::{ProvingSystemId, VerificationData};
use ethers::prelude::*;

const BATCHER_URL: &str = "wss://batcher.alignedlayer.com";
const ELF: &[u8] = include_bytes!("../../program/elf/riscv32im-succinct-zkvm-elf");

async fn submit_proof_to_aligned(
proof: Vec<u8>,
wallet: Wallet<SigningKey>
) -> Result<AlignedVerificationData, anyhow::Error> {
let verification_data = VerificationData {
    proving_system: ProvingSystemId::SP1,
    proof,
    proof_generator_addr: wallet.address(),
    vm_program_code: Some(ELF.to_vec()),
    verification_key: None,
    pub_input: None,
};

    submit(BATCHER_URL, &verification_data, wallet).await
        .map_err(|e| anyhow::anyhow!("Failed to submit proof: {:?}", e))
}

#[tokio::main]
async fn main() {
let wallet = // Initialize wallet
let proof = // Generate or obtain proof

    match submit_proof_to_aligned(proof, wallet).await {
        Ok(aligned_verification_data) => println!("Proof submitted successfully"),
        Err(err) => println!("Error: {:?}", err),
    }
}
```

The following code is an example of how to verify the proof was correctly verified in Aligned using the SDK:

```rust
use aligned_sdk::sdk::verify_proof_onchain;
use aligned_sdk::types::{AlignedVerificationData, Chain};
use ethers::prelude::*;
use tokio::time::{sleep, Duration};

async fn wait_for_proof_verification(
    aligned_verification_data: AlignedVerificationData,
    rpc_url: String,
) -> Result<(), anyhow::Error> {
    for _ in 0..10 {
        if verify_proof_onchain(aligned_verification_data.clone(), Chain::Holesky, rpc_url.as_str()).await.is_ok_and(|r| r) {
            println!("Proof verified successfully.");
            return Ok(());
        }
        println!("Proof not verified yet. Waiting 10 seconds before checking again...");
        sleep(Duration::from_secs(10)).await;
    }
    anyhow::bail!("Proof verification failed")
}

#[tokio::main]
async fn main() {
    let aligned_verification_data = // Obtain aligned verification data
    let rpc_url = "https://ethereum-holesky-rpc.publicnode.com".to_string();

    match wait_for_proof_verification(aligned_verification_data, rpc_url).await {
        Ok(_) => println!("Proof verified"),
        Err(err) => println!("Error: {:?}", err),
    }
}
```

You can find an example of the proof submission and verification in the [Quiz Program](../../examples/zkquiz/quiz/script/src/main.rs).

The example generates a proof, instantiate a wallet to submit the proof, and then submits the proof to Aligned for verification. It then waits for the proof to be verified in Aligned.

#### Using the CLI
You can find examples of how to submit a proof using the CLI in the [submitting proofs guide](0_submitting_proofs.md).