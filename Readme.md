
eip: <TBD>
title: Standard Factory Interface for Predictable Wallet Creation
author: 
status: Draft
type: Standards Track
category: Interface



## Abstract

Define a minimal, interoperable factory interface for deploying wallet contracts that:

1. Standardizes creation calls.
2. Exposes deterministic address prediction (`CREATE2`).
3. Provides metadata helpers so tooling can discover wallet instances and implementations.

## Motivation

Tooling (wallet explorers, analytics, on-chain safe bootstrappers, UX libraries) needs a predictable, minimal interface for factories that produce wallet instances.  
Without a standard, each factory implements ad-hoc functions and events, increasing friction and fragility for integrators.  

This EIP standardizes a small set of functions and events that solve the common needs while remaining implementation-agnostic.

## Rationale

- `createWallet` is intentionally minimal: `implementation` address, `initData`, and `salt` cover the most common deployment patterns (proxy, minimal proxy, native wallet bytecode).
- `predictWalletAddress` explicitly enables deterministic tooling and wallets-to-be-created workflows (social recovery precomputation, ENS linking, pre-funding).
- `WalletCreated` event is required so indexers and explorers can discover wallets created by the factory.
- `getImplementation` is optional but recommended for proxies; it lets tooling map wallets to their logic contracts.
- `factoryVersion` gives tooling a compact handshake to handle different factory behaviors.

## Backwards Compatibility

This is an interface standard.  
Existing factories remain functional. Implementers can adopt this interface in addition to their custom APIs.  
Tools should use `supportsInterface` checks (ERC-165) or attempt static calls and fall back gracefully.

## Test Cases

1. Deploy a factory that uses `CREATE2`.
2. Call `predictWalletAddress` and confirm `createWallet` results in the same address.
3. Indexer: Listen for `WalletCreated`, verify `getImplementation` returns the expected logic address for proxies.
4. Call `factoryVersion` and assert semantic version format.

## Reference Implementation (Pseudo-Solidity)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MinimalWalletFactory {
    event WalletCreated(address indexed wallet, address indexed creator, address implementation);

    function createWallet(
        address implementation,
        bytes calldata initData,
        bytes32 salt
    ) external returns (address wallet) {
        bytes memory bytecode = abi.encodePacked(
            hex"3d602d80600a3d3981f3", // EIP-1167 minimal proxy
            abi.encode(implementation),
            initData
        );
        assembly {
            wallet := create2(0, add(bytecode, 0x20), mload(bytecode), salt)
        }
        emit WalletCreated(wallet, msg.sender, implementation);
    }

    function predictWalletAddress(
        address implementation,
        bytes calldata initData,
        bytes32 salt
    ) external view returns (address predicted) {
        bytes memory bytecode = abi.encodePacked(
            hex"3d602d80600a3d3981f3",
            abi.encode(implementation),
            initData
        );
        bytes32 hash = keccak256(
            abi.encodePacked(bytes1(0xff), address(this), salt, keccak256(bytecode))
        );
        predicted = address(uint160(uint256(hash)));
    }

    function factoryVersion() external pure returns (string memory) {
        return "1.0.0";
    }
}
Security Considerations
predictWalletAddress must not leak secrets; it should compute deterministic addresses only from public inputs.

The factory must document whether creator in the event is msg.sender (recommended) or tx.origin.

initData may call arbitrary initialization logic; callers must ensure safe constructors and init code to avoid accidental ownership transfers.

Implementation Notes
Implementers should optionally add ERC-165 support to advertise this interface ID.

For proxies, using EIP-1167 (minimal proxy) with metadata in creation initData is a common pattern.

If the factory supports both CREATE and CREATE2, document behavior (e.g., salt == 0 ⇒ non-deterministic).a
