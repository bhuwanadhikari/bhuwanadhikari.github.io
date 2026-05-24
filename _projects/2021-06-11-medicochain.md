---
layout: project
title: "How I Built Medicochain: A Blockchain-Powered Drug Anti-Counterfeiting Platform Using Hyperledger Fabric"
description: Medicochain tracks drug provenance across the entire supply chain using a permissioned blockchain, so every stakeholder from manufacturer to patient can verify drug authenticity and fight counterfeiting.
date: 2021-06-11
image: "/images/projects/medicochain_thumbnail_simple.png"
tags: [blockchain, hyperledger-fabric, nodejs, flutter, docker, supply-chain]
github: https://github.com/bhuwanadhikari/medicochain
favorite: false
published: true
---

Someone's grandmother in Kathmandu buys a blood pressure medication. She takes it every day for two weeks. Her blood pressure doesn't improve. She goes back to the clinic. The doctor is puzzled — the dosage should be working. But the pills she bought contained almost no active ingredient. They were fake. The WHO estimates that **10% of medicines in low- and middle-income countries are substandard or falsified**. Every year, counterfeit drugs kill hundreds of thousands of people, not dramatically but quietly, invisibly, in clinics and homes where nobody connects the bad outcome back to the bad pill.

I built Medicochain because I wanted to understand whether blockchain could actually solve this in working code, not just in a whitepaper.

## The Problem with the Status Quo

Drug supply chains are long and opaque. A single tablet might pass through a manufacturer, a regional distributor, a national importer, a local wholesaler, and finally a pharmacy, with each link a potential entry point for counterfeits. The obvious response is centralized tracking: a database that records every handoff. But centralized databases fail this problem in two fundamental ways.

**Trust.** Who runs the database? If it's the manufacturer, distributors have no reason to trust it hasn't been altered to shift blame. If it's a government body, it becomes a regulatory bottleneck. Every participant in the supply chain has an incentive to manipulate records: to deny a shipment was tampered with, to cover up a cold-chain breach, to hide expired stock that was relabeled. A single authoritative database means a single authoritative owner, and no one in a competitive supply chain can fully trust that owner.

**Tamperability.** Traditional databases can be edited. Rows can be deleted, timestamps can be changed, records can be silently updated. A centralized drug tracking system that can be edited by its administrator is not meaningfully different from no tracking system at all. What you need is a ledger where no single party can rewrite history.

This is precisely the problem blockchain was designed for.

## Why Blockchain? Why Hyperledger Fabric Specifically?

Let me separate two things that often get conflated: **public blockchains** (like Ethereum or Bitcoin) and **permissioned blockchains** (like Hyperledger Fabric).

Public blockchains are open. Anyone can join, anyone can read or write, transactions are validated by anonymous nodes scattered across the world, and everything is transparent to everyone forever. That's appropriate for a cryptocurrency. It's not appropriate for a pharmaceutical supply chain where manufacturers don't want competitors to read their production volumes and distributors don't want rivals knowing their customers.

Think of it this way: a public blockchain is like a bulletin board in a town square. Everything posted is visible to everyone, forever. A permissioned blockchain is like a shared ledger kept inside a sealed room where only invited participants can enter, every entry is still permanent and tamper-proof, but outsiders can't see what's inside.

**Hyperledger Fabric** is the most mature permissioned blockchain framework available. It has several properties that make it the right choice for pharma supply chains:

- **No cryptocurrency.** There's no ETH or BTC involved. Just data, transactions, and consensus. No wallets, no gas fees, no mining.
- **Known participants.** Every node is authenticated through an MSP (Membership Service Provider). You know exactly who signed every transaction.
- **Private channels.** Different participants can share data on separate channels. A manufacturer and their exclusive distributor can have a private ledger that pharmacies can't read.
- **High throughput.** Public blockchains process ~15-30 transactions per second. Fabric can handle thousands. A national drug supply chain generates millions of events; you need speed.
- **Pluggable consensus.** Fabric lets you choose the ordering mechanism that fits your trust assumptions. No proof-of-work energy waste.

For a domain where privacy, speed, regulatory compliance, and participant accountability all matter, Fabric is the right tool.

## How Medicochain Works: The Architecture

The core idea is simple: every drug batch gets a unique identity on the blockchain, and every time that batch changes hands, a transaction is recorded that cannot be deleted or modified. Here's the full lifecycle:

**1. Manufacturer registers a batch.**
A pharmaceutical company produces 10,000 units of a drug. They call Medicochain's API with the batch ID, drug name, manufacturing date, expiry date, and composition. The chaincode creates an asset on the ledger: this batch exists, it was created by this manufacturer, at this time. The record is signed by the manufacturer's cryptographic identity and committed to the blockchain.

**2. Distributor receives and signs.**
The batch is shipped. When the distributor receives it, they scan the batch ID and submit a transfer transaction. The chaincode verifies that the batch exists, that the previous owner was the manufacturer, and records the new owner as the distributor. Both parties have signed off on this handoff, and the record shows the chain of custody.

**3. Pharmacy accepts delivery.**
Same process. The pharmacy receives the batch from the distributor, submits a transaction, the chaincode updates the owner field on the ledger asset. At this point, the full provenance history is on-chain: manufacturer to distributor to pharmacy, each step timestamped and signed.

**4. Patient or pharmacist verifies.**
When a patient comes in, the pharmacist opens the Flutter mobile app, scans the batch ID (via QR code), and the app queries the blockchain through the Express.js API. The response shows the complete chain of custody. If the drug arrived without passing through a registered distributor, or if the batch ID doesn't exist at all, the system flags it immediately.

**Chaincode** is Fabric's term for smart contracts, code that runs on the blockchain itself and defines what transactions are valid. Think of it as the enforcer: it can't be bribed, it can't be edited mid-execution, and it applies the same rules to every participant. My chaincode was written in Node.js and handles drug registration, ownership transfer, and provenance queries.

**Orderer nodes** are responsible for sequencing transactions. They take incoming transactions from all peers and determine the canonical order in which they get committed to the ledger. Think of the orderer as the referee who decides which move happened first.

**Peer nodes** are where the actual ledger lives and where chaincode executes. Each organization in the network (manufacturer, distributor, pharmacy) runs their own peer node. All of this runs inside Docker containers, so the entire network topology is defined as Docker services, making the environment fully reproducible.

## The Tech Deep-Dive

### Chaincode in Node.js

Fabric chaincode can be written in Go, Java, or Node.js. I chose Node.js because it's fast to iterate on and the Fabric SDK for JavaScript is well-documented.

The core of the drug transfer chaincode looks like this:

```javascript
// [INSERT: chaincode snippet for drug transfer transaction]
async transferDrug(ctx, batchId, newOwner) {
  const drugJSON = await ctx.stub.getState(batchId);
  if (!drugJSON || drugJSON.length === 0) {
    throw new Error(`Drug batch ${batchId} does not exist`);
  }
  const drug = JSON.parse(drugJSON.toString());
  drug.currentOwner = newOwner;
  drug.transferHistory.push({
    from: drug.currentOwner,
    to: newOwner,
    timestamp: ctx.stub.getTxTimestamp()
  });
  await ctx.stub.putState(batchId, Buffer.from(JSON.stringify(drug)));
}
```

The `ctx.stub` is Fabric's interface to the ledger. `getState` reads an asset, `putState` writes it. Once `putState` is called and the transaction is committed, that record is permanent. There's no `UPDATE` or `DELETE` equivalent.

The provenance query uses `getHistoryForKey`, which returns every version of an asset that has ever existed on the ledger, giving you a full audit trail that cannot be pruned.

### Express.js as the API Gateway

The Flutter mobile app doesn't talk to the blockchain directly. It talks to an Express.js server, which uses the Hyperledger Fabric Node.js SDK to submit transactions and query the ledger. This separation matters: it keeps the mobile client simple and allows the API layer to handle authentication, input validation, and wallet management (Fabric identities are X.509 certificates, not passwords).

```javascript
// [INSERT: Express route for drug provenance query]
app.get('/drug/:batchId/provenance', async (req, res) => {
  const gateway = new Gateway();
  await gateway.connect(ccp, { wallet, identity: 'pharmacist1' });
  const network = await gateway.getNetwork('drug-channel');
  const contract = network.getContract('drug-transfer');
  const result = await contract.evaluateTransaction('getDrugHistory', req.params.batchId);
  res.json(JSON.parse(result.toString()));
});
```

### The `cheatseat.sh` Problem

Setting up a local Hyperledger Fabric network manually is one of the most friction-heavy experiences in enterprise blockchain. You need to generate cryptographic material for each organization, create the channel genesis block, start the orderer container, start peer containers for each org, join peers to the channel, install chaincode on each peer, approve chaincode for each org, commit the chaincode definition, and only then can you submit a transaction.

That's roughly 20 shell commands, each depending on the previous one, each with its own set of environment variables. One wrong path breaks everything silently.

I wrote `cheatseat.sh` to orchestrate the entire bootstrap sequence. It's not glamorous, but it's the script that made local development actually possible. One command to stand up the full network, deploy chaincode, and have a testable environment. This kind of automation is often the unsexy but critical work in blockchain projects that blog posts don't show.

## Impact and Real-World Relevance

The WHO's IMPACT initiative, the EU Falsified Medicines Directive, and the US Drug Supply Chain Security Act all point to the same conclusion: the pharmaceutical industry needs end-to-end traceability. Medicochain is a working proof of concept for exactly that.

The immediate applicability is in places like Nepal, where regulatory infrastructure is thinner and counterfeit drugs flow more easily through informal supply chains. But the architecture is geography-agnostic. The same network configuration, with different organizations enrolled, could serve a national drug authority in any country.

Beyond counterfeiting, the provenance data has secondary value: cold-chain monitoring (was this batch always stored at the right temperature?), recall management (which pharmacies received batches from this compromised shipment?), and regulatory audit trails. Once the data is on an immutable ledger, every downstream use case becomes easier.

## What I Learned

Hyperledger Fabric is powerful and genuinely well-suited to enterprise supply chain problems, but it has a steep learning curve. The documentation has improved significantly, but when I built this, configuration errors failed silently and debugging required reading Docker container logs across four simultaneous services while mentally modeling what the consensus protocol was doing.

If I were starting over, I'd make three changes. First, I'd invest more in the Flutter UI. The verification flow works, but the UX doesn't feel polished enough for a pharmacist to adopt in 30 seconds. QR scanning needs to be faster and more fault-tolerant. Second, I'd add proper key management from day one. Managing Fabric identities via the file system is fine for a prototype, but a production deployment needs a proper wallet service. Third, I'd explore **Hyperledger Besu** as an alternative, which runs the Ethereum Virtual Machine (EVM), meaning you can write Solidity smart contracts while keeping the permissioned network model. That would make the chaincode more portable and easier to audit for developers already familiar with Ethereum tooling.

The project was originally scoped as a university assignment. It grew into something I actually believe in. The counterfeiting problem is real, the technology to address it exists, and the missing piece is implementation, someone building it, running it, and getting stakeholders to trust it.

## Explore the Code

If you're building in the blockchain space, working on supply chain transparency, or just curious how Hyperledger Fabric works end-to-end, the full source is on GitHub:

**[github.com/bhuwanadhikari/medicochain](https://github.com/bhuwanadhikari/medicochain)**

The repo includes the full network configuration, Node.js chaincode, the Express.js API, and the `cheatseat.sh` bootstrap script. Clone it, run the script, and you'll have a local Fabric network with deployed chaincode in under five minutes.

If you find a bug, have a question about the architecture, or want to extend it, PRs and issues are open. This problem is worth solving. Every fake pill that gets flagged before it reaches a patient is a life the system protected. That's worth building for.
