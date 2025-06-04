## High-Level Protocol Overview

The protocol is designed to efficiently manage and provide liquidity for a diverse range of yield-bearing assets. Its core purpose is to streamline the complexities typically associated with these assets by offering automated redemption processes and sophisticated rebalancing mechanisms. This ensures optimal yield generation and risk management for participants. By abstracting away the operational overhead, the protocol aims to make yield-bearing assets more accessible and attractive to a broader audience.

## Main Actors

The protocol involves several key actors, each with distinct roles and responsibilities:

*   **Users:**
    *   **Liquidity Providers (LPs):** These users supply assets to the protocol's liquidity pools. In return for providing liquidity, they typically earn a share of the protocol's revenue, often in the form of yield or fees generated from the pooled assets. LPs are crucial for the protocol's ability to offer its services.
    *   **Traders:** These users interact with the protocol to swap or trade assets. They benefit from the liquidity provided by LPs, allowing them to execute trades efficiently. Traders might also be seeking to gain exposure to specific yield-bearing assets or arbitrage price differences.

*   **Operators:** These are entities responsible for the day-to-day operational management of the protocol. Their tasks may include monitoring the health of the system, executing automated strategies (like rebalancing or redemption), managing oracle feeds, and ensuring the smooth functioning of smart contracts. Operators might be a decentralized network of participants or a designated team.

*   **Owners:** These are the individuals or entities that have governance rights and control over the protocol. They are typically responsible for making decisions on protocol upgrades, parameter changes (e.g., fee structures, risk parameters), and the overall strategic direction of the protocol. Ownership is often represented by governance tokens.

## Top-Level Architecture Diagram

```mermaid
graph TD
    subgraph UserLayer [User Facing]
        User
        subgraph Zappers
            ZapperLidoARM["ZapperLidoARM (Proxy)"]
            ZapperARM["ZapperARM (Proxy)"]
        end
    end

    subgraph CoreProtocol [Core Protocol Contracts]
        subgraph ARMs ["Automated Redemption Modules (Proxied)"]
            direction LR
            LidoARM["LidoARM (Proxy)"]
            OethARM["OethARM (Proxy)"]
            OriginARM["OriginARM (Proxy)"]
        end
        CapManager["CapManager (Proxy)"]
        SonicHarvester["SonicHarvester (Proxy)"]
        LPTokenUser["LP Tokens (User Receives)"]
        LPTokenARM["LP Tokens (ARM Mints/Burns)"]
    end

    subgraph ExternalIntegrations [External Protocols & Strategies]
        Lido[/"Lido Protocol"/]
        OETHVault[/"OETH Vault"/]
        GenericYield[/"Generic Yield Strategies"/]
    end

    %% User Interactions
    User -- "Deposit/Swap" --> ZapperLidoARM
    User -- "Deposit/Swap" --> ZapperARM
    User -- "Direct Deposit/Withdraw" --> LidoARM
    User -- "Direct Deposit/Withdraw" --> OethARM
    User -- "Direct Deposit/Withdraw" --> OriginARM
    ARMs --> LPTokenUser

    %% Zapper Interactions
    ZapperLidoARM -- "Zap In/Out" --> LidoARM
    ZapperARM -- "Zap In/Out" --> LidoARM
    ZapperARM -- "Zap In/Out" --> OethARM
    ZapperARM -- "Zap In/Out" --> OriginARM

    %% ARM Interactions
    LidoARM --> LPTokenARM
    OethARM --> LPTokenARM
    OriginARM --> LPTokenARM
    ARMs -- "Check/Update Caps" --> CapManager
    LidoARM -- "Redeem/Withdraw" --> Lido
    OethARM -- "Redeem/Withdraw" --> OETHVault
    OriginARM -- "Redeem/Withdraw" --> GenericYield

    %% SonicHarvester Interactions
    SonicHarvester -- "Harvest Rewards" --> GenericYield
    SonicHarvester -- "Distribute Rewards" --> ARMs %% Or a central rewards contract

    %% Conceptual Proxy Indication (actual proxy use is per contract)
    %% This is simplified; in reality, each contract like LidoARM, CapManager etc. would have its own proxy.
    %% The (Proxy) in the name indicates this.
end
```
