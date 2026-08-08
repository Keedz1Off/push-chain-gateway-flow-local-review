# About Me

I am a junior smart contract security researcher focused on bridge security, L2/L3 architecture, and cross-chain flow analysis.

My main portfolio work covers:

```text
Push Chain Gateway contest-oriented review
Arbitrum bridge deposit and withdrawal flows
Optimism bridge and messenger flows
LayerZero OFT send, receive, and trusted-peer flows
```

My current Push Chain review follows the EVM Gateway contest scope. I traced the source flow, event-based message boundary, TSS/Vault finalization, CEA execution, and revert/refund flow. I then verified invariants directly against the relevant functions and recorded protected and suspicious cases separately.

My review method is:

```text
Understand the architecture
-> trace funds and data
-> define invariants
-> verify each invariant in code
-> isolate suspicious behavior
-> prove or reject it with a PoC
```

Additional portfolio studies include Arbitrum L3 architecture and the Sky/DAI Optimism-based bridge fork. These are supporting architecture reviews, while Push Chain, Arbitrum, Optimism, and LayerZero form the primary security scope of my portfolio.

Portfolio: https://github.com/Keedz1Off
