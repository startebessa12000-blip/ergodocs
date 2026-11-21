---
tags:
  - fleet-sdk
  - mock-chain
  - testing
  - vitest
  - typescript
  - ergoscript
  - tutorial
owner: docs
last_reviewed: 2026-11-21
---

# Testing Contracts with MockChain

`@fleet-sdk/mock-chain` provides an in-memory Ergo blockchain simulator for testing ErgoScript smart contracts and transaction builders locally without running a full node or connecting to testnet. Combined with Vitest, you can write fast unit and adversarial tests for contract spending conditions, register validation, and timelocks.

## Prerequisites

Install the Fleet SDK packages and Vitest in a TypeScript project:

```bash
npm install --save-dev @fleet-sdk/core @fleet-sdk/compiler @fleet-sdk/mock-chain vitest
```

## Example Contract

Consider a timelocked vault contract where:

- The **owner** can withdraw funds at any time.
- A **beneficiary** can claim the funds only after a target block height (`deadline`).

```scala
{
  val isOwner = ownerPk
  val isBeneficiaryAfterDeadline = beneficiaryPk && (HEIGHT >= deadline)
  sigmaProp(isOwner || isBeneficiaryAfterDeadline)
}
```

## Writing Unit and Adversarial Tests

Create a test file `vault.test.ts` that compiles the contract with `@fleet-sdk/compiler`, sets up funded parties on `MockChain`, and verifies both valid and invalid spending attempts.

```typescript
import { describe, it, expect, beforeEach } from "vitest";
import { compile } from "@fleet-sdk/compiler";
import { OutputBuilder, TransactionBuilder } from "@fleet-sdk/core";
import { MockChain } from "@fleet-sdk/mock-chain";

const VAULT_SCRIPT = `
{
  val isOwner = ownerPk
  val isBeneficiaryAfterDeadline = beneficiaryPk && (HEIGHT >= deadline)
  sigmaProp(isOwner || isBeneficiaryAfterDeadline)
}
`;

describe("Timelocked Vault Contract", () => {
  let chain: MockChain;

  beforeEach(() => {
    chain = new MockChain({ height: 100_000 });
  });

  it("allows the owner to withdraw before the deadline", () => {
    const owner = chain.newParty("owner");
    const beneficiary = chain.newParty("beneficiary");
    owner.addBalance({ nanoergs: 10_000_000_000n });

    const ergoTree = compile(VAULT_SCRIPT, {
      map: {
        ownerPk: owner.key.publicKey,
        beneficiaryPk: beneficiary.key.publicKey,
        deadline: 100_500
      }
    });

    const lockTx = new TransactionBuilder(chain.height)
      .from(owner.utxos)
      .to(new OutputBuilder(5_000_000_000n, ergoTree))
      .sendChangeTo(owner.address)
      .payMinFee()
      .build();

    expect(chain.execute(lockTx, { signers: [owner] })).toBe(true);

    const contractBox = chain.utxos
      .toArray()
      .find((b) => b.ergoTree === ergoTree.toHex())!;

    const unlockTx = new TransactionBuilder(chain.height)
      .from(contractBox)
      .to(new OutputBuilder(4_998_900_000n, owner.address))
      .payMinFee()
      .build();

    expect(chain.execute(unlockTx, { signers: [owner] })).toBe(true);
  });

  it("rejects beneficiary withdrawal before the deadline", () => {
    const owner = chain.newParty("owner");
    const beneficiary = chain.newParty("beneficiary");
    owner.addBalance({ nanoergs: 10_000_000_000n });

    const ergoTree = compile(VAULT_SCRIPT, {
      map: {
        ownerPk: owner.key.publicKey,
        beneficiaryPk: beneficiary.key.publicKey,
        deadline: 100_500
      }
    });

    const lockTx = new TransactionBuilder(chain.height)
      .from(owner.utxos)
      .to(new OutputBuilder(5_000_000_000n, ergoTree))
      .sendChangeTo(owner.address)
      .payMinFee()
      .build();

    expect(chain.execute(lockTx, { signers: [owner] })).toBe(true);

    const contractBox = chain.utxos
      .toArray()
      .find((b) => b.ergoTree === ergoTree.toHex())!;

    const earlyClaimTx = new TransactionBuilder(chain.height)
      .from(contractBox)
      .to(new OutputBuilder(4_998_900_000n, beneficiary.address))
      .payMinFee()
      .build();

    expect(chain.execute(earlyClaimTx, { signers: [beneficiary] })).toBe(false);
  });

  it("allows beneficiary withdrawal at or after the deadline", () => {
    const owner = chain.newParty("owner");
    const beneficiary = chain.newParty("beneficiary");
    owner.addBalance({ nanoergs: 10_000_000_000n });

    const ergoTree = compile(VAULT_SCRIPT, {
      map: {
        ownerPk: owner.key.publicKey,
        beneficiaryPk: beneficiary.key.publicKey,
        deadline: 100_500
      }
    });

    const lockTx = new TransactionBuilder(chain.height)
      .from(owner.utxos)
      .to(new OutputBuilder(5_000_000_000n, ergoTree))
      .sendChangeTo(owner.address)
      .payMinFee()
      .build();

    expect(chain.execute(lockTx, { signers: [owner] })).toBe(true);

    chain.jumpTo(100_500);

    const contractBox = chain.utxos
      .toArray()
      .find((b) => b.ergoTree === ergoTree.toHex())!;

    const claimTx = new TransactionBuilder(chain.height)
      .from(contractBox)
      .to(new OutputBuilder(4_998_900_000n, beneficiary.address))
      .payMinFee()
      .build();

    expect(chain.execute(claimTx, { signers: [beneficiary] })).toBe(true);
  });
});
```

## Running the Tests

```bash
npx vitest run
```

## Related Documentation

- [Fleet SDK](fleet.md)
- [Fleet SDK Recipes](fleet-sdk-recipes.md)
- [ErgoScript](ergoscript.md)
- [ErgoTree Specification](ergotree.md)
