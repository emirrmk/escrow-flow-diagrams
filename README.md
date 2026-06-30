# Governed On-Chain Accounting and Escrow Layer Flow Diagrams

This document outlines the architecture and the 4 main operational flows of the **governed (custodial/governed) on-chain accounting and escrow layer** implemented on the **Ripple EVM Sidechain** using the **ERC-3525 Semi-Fungible Token** standard.

---

## Architectural Specifications & Token Design

To support fine-grained financial logic and deterministic escrow states, the token attributes of the ERC-3525 standard are mapped as follows:

1. **ERC-3525 Slot (`uint256`):** Represents the unique service configuration, mapped as `serviceId(type, startDate, endDate, defaultDate)`. These parameters are packed or hashed (e.g., via `keccak256`) into a single `uint256` slot to classify tokens sharing the same business rule structure.
   - **Service Token Slot:** `Slot 1` — Represents the provider's collateral parameters.
   - **Obligation Token Slot:** `Slot 2` — Represents the client's payment obligation parameters.

2. **ERC-3525 Token ID (`uint256`):** Represents the identity/credentials of the token owner. Since the ERC-3525 `tokenId` is a `uint256` value, it stores the **Owner's Public Key** (or its Ethereum address derivation, which fits comfortably within the 256-bit unsigned integer limits).

3. **ERC-3525 Value:** Directly represents the exact locked **RLUSD (Ripple USD)** amount. Financial operations like value transfer (`transferValue`) or value burning (`burnValue`) trigger the underlying ERC-20 RLUSD transfers between client/provider wallets and the escrow pool.

4. **Gated Off-Chain Orchestration:** Direct wallet-to-wallet transfers via standard ERC-3525 methods are blocked. All business logic rules (vesting schedules, matching, market fragmentation) are computed off-chain, and validated state updates are pushed to the chain using the **Centralized Algorithm (signed by the Admin Key)**.

---

## 1. Job Initialization (Atomic Mint & Lock)

When a match is established off-chain, the Client and the Provider sign the transaction/allowance with their crypto wallets. The escrow contract executes the atomic lock of RLUSD and mints the corresponding ERC-3525 tokens. No manual admin approval is required.

```mermaid
%%{init: {'theme': 'neutral', 'themeVariables': { 'primaryColor': '#f9fafb', 'edgeLabelBackground':'#ffffff', 'clusterBkg':'#f3f4f6', 'clusterBorder':'#d1d5db'}}}%%
graph TD
    classDef offChainNode fill:#ffffff,stroke:#4b5563,stroke-width:2px,stroke-dasharray: 5 5;
    classDef onChainNode fill:#eff6ff,stroke:#3b82f6,stroke-width:2px;
    classDef tokenNode fill:#ecfdf5,stroke:#10b981,stroke-width:2px;
    classDef databaseNode fill:#faf5ff,stroke:#8b5cf6,stroke-width:2px;

    subgraph OffChain ["🌐 OFF-CHAIN (Matching & Wallet Signatures)"]
        A["Client & Provider Match"]:::offChainNode --> B["Wallet Signatures <br> (RLUSD Allowance & Transaction Approval)"]:::offChainNode
        B --> C["Relay Transaction to Blockchain"]:::offChainNode
    end

    subgraph OnChain ["⛓️ ON-CHAIN (Ripple EVM Sidechain & Escrow)"]
        C --> D{"Escrow Contract <br> atomicMintAndLock"}:::onChainNode
        
        subgraph Vault ["Escrow Vault"]
            Pool[("RLUSD Vault")]:::databaseNode
        end
        
        D -- "1. Pull X RLUSD (Client)" --> Pool
        D -- "2. Pull Y RLUSD (Provider)" --> Pool
        
        D -- "3. Mint (Slot=2, ID=ClientPubKey, Value=X)" --> OT["Obligation Token <br> Escrow Payment Tracking"]:::tokenNode
        D -- "4. Mint (Slot=1, ID=ProviderPubKey, Value=Y)" --> ST["Service Token <br> Provider Collateral"]:::tokenNode
    end
```

---

## 2. Time-Fragmented Consumption (Milestone Release)

As milestones/vesting periods elapse and off-chain oracle verifications are submitted, the Centralized Algorithm (with Admin signature) triggers the release function. The value in the Obligation Token is reduced and paid to the provider.

```mermaid
%%{init: {'theme': 'neutral', 'themeVariables': { 'primaryColor': '#f9fafb', 'edgeLabelBackground':'#ffffff', 'clusterBkg':'#f3f4f6', 'clusterBorder':'#d1d5db'}}}%%
graph TD
    classDef offChainNode fill:#ffffff,stroke:#4b5563,stroke-width:2px,stroke-dasharray: 5 5;
    classDef onChainNode fill:#eff6ff,stroke:#3b82f6,stroke-width:2px;
    classDef tokenNode fill:#ecfdf5,stroke:#10b981,stroke-width:2px;
    classDef databaseNode fill:#faf5ff,stroke:#8b5cf6,stroke-width:2px;

    subgraph OffChain ["🌐 OFF-CHAIN (Verification & Trigger)"]
        A["Vesting Milestone Reached"]:::offChainNode --> B["Oracle / Work Delivery Verification"]:::offChainNode
        B --> C["Centralized Algorithm <br> (Trigger with Admin Signature)"]:::offChainNode
    end

    subgraph OnChain ["⛓️ ON-CHAIN (Ripple EVM Sidechain)"]
        C --> D{"Escrow Contract <br> releaseMilestone"}:::onChainNode
        
        D --> OT["Obligation Token <br> Slot 2"]:::tokenNode
        OT -- "1. Deduct Value <br> (burnValue)" --> OT
        
        D --> Pool[("RLUSD Vault")]:::databaseNode
        Pool -- "2. Transfer Released RLUSD" --> Prov["Provider Wallet <br> (ProviderPubKey)"]:::onChainNode
    end
```

---

## 3. Market/Time-Fragmented Duty Delegation (Transfer / Replacement)

When a service provider delegates/transfers their duties to a new provider, the **Client's signature is required**. The Centralized Algorithm updates the escrow state: the old provider's Service Token is burned and collateral returned, a new Service Token is minted for the new provider, and the Obligation Token is transferred to the new provider.

```mermaid
%%{init: {'theme': 'neutral', 'themeVariables': { 'primaryColor': '#f9fafb', 'edgeLabelBackground':'#ffffff', 'clusterBkg':'#f3f4f6', 'clusterBorder':'#d1d5db'}}}%%
graph TD
    classDef offChainNode fill:#ffffff,stroke:#4b5563,stroke-width:2px,stroke-dasharray: 5 5;
    classDef onChainNode fill:#eff6ff,stroke:#3b82f6,stroke-width:2px;
    classDef tokenNode fill:#ecfdf5,stroke:#10b981,stroke-width:2px;
    classDef databaseNode fill:#faf5ff,stroke:#8b5cf6,stroke-width:2px;

    subgraph OffChain ["🌐 OFF-CHAIN (Delegation Request & Approval)"]
        Req["Provider Delegation Request"]:::offChainNode --> CustSign["Client Consent Signature"]:::offChainNode
        CustSign --> Alg["Centralized Algorithm <br> (Trigger with Admin Signature)"]:::offChainNode
    end

    subgraph OnChain ["⛓️ ON-CHAIN (Ripple EVM Sidechain)"]
        Alg --> TransferCall{"Escrow Contract <br> transferProviderDuties"}:::onChainNode
        
        subgraph Branch1 ["Obligation Token Update"]
            TransferCall -- "1. Transfer Ownership" --> OT["Obligation Token <br> Slot 2"]:::tokenNode
            OT --> NewProv["New Provider Wallet <br> (NewProviderPubKey)"]:::onChainNode
        end

        subgraph Branch2 ["Service Collateral Swap"]
            TransferCall -- "2. Refund Remaining Collateral" --> OldST["Old Service Token"]:::tokenNode
            OldST --> OldProv["Old Provider Wallet <br> (OldProviderPubKey)"]:::onChainNode
            OldST -- "3. Burn Token" --> BurnX[X]:::offChainNode
            
            NewProv -- "4. Lock Collateral (RLUSD)" --> Pool[("RLUSD Vault")]:::databaseNode
            Pool -- "5. Mint New Service Token" --> NewST["New Service Token <br> Slot 1"]:::tokenNode
        end
    end
```

---

## 4. Slashing (Collateral Forfeiture)

If the provider violates their service commitments, the off-chain system detects the breach. The Centralized Algorithm (with Admin signature) triggers the slash function: the Provider's Service Token is burned and their locked collateral is sent to the Client. The Client's remaining unspent balance in the Obligation Token is refunded.

```mermaid
%%{init: {'theme': 'neutral', 'themeVariables': { 'primaryColor': '#f9fafb', 'edgeLabelBackground':'#ffffff', 'clusterBkg':'#f3f4f6', 'clusterBorder':'#d1d5db'}}}%%
graph TD
    classDef offChainNode fill:#ffffff,stroke:#4b5563,stroke-width:2px,stroke-dasharray: 5 5;
    classDef onChainNode fill:#eff6ff,stroke:#3b82f6,stroke-width:2px;
    classDef tokenNode fill:#ecfdf5,stroke:#10b981,stroke-width:2px;
    classDef databaseNode fill:#faf5ff,stroke:#8b5cf6,stroke-width:2px;

    subgraph OffChain ["🌐 OFF-CHAIN (Breach Detection)"]
        Breach["Provider Commitment Breach"]:::offChainNode --> Decision["Backend Slashing Decision"]:::offChainNode
        Decision --> AlgTrig["Centralized Algorithm <br> (Trigger with Admin Signature)"]:::offChainNode
    end

    subgraph OnChain ["⛓️ ON-CHAIN (Ripple EVM Sidechain)"]
        AlgTrig --> SlashCall{"Escrow Contract <br> slashProvider"}:::onChainNode
        
        subgraph VaultState ["Escrow Vault"]
            Pool[("RLUSD Vault")]:::databaseNode
        end

        subgraph TokenState ["Token States"]
            ST["Service Token <br> Slot 1"]:::tokenNode
            OT["Obligation Token <br> Slot 2"]:::tokenNode
        end

        SlashCall --> ST
        SlashCall --> OT

        %% Provider Collateral Compensation
        ST -- "1. Burn Token" --> BurnST[X]:::offChainNode
        Pool -- "2. Send Provider Collateral (Y) to Client" --> Cust["Client Wallet <br> (ClientPubKey)"]:::onChainNode

        %% Client Escrow Refund
        OT -- "3. Burn Token" --> BurnOT[X]:::offChainNode
        Pool -- "4. Refund Unspent Balance (X_remaining) to Client" --> Cust
    end
```