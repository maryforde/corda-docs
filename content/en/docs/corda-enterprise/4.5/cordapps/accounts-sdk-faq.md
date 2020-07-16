---
date: '2020-05-10T12:00:00Z'
menu:
  corda-enterprise-4-5:
    parent: corda-enterprise-4-5-accounts-sdk
weight: 200
tags:
- Accounts SDK
- Accounts library
- Accounts CorDapp
- accounts FAQs
title: Accounts SDK FAQs
---

# Accounts SDK Frequently Asked Questions (FAQs)

## What's the difference between an account and a node?

A **node** is a process that runs `corda.jar`. Nodes have a network identity issued to them by the trust root of the network that the node participates in. The node has a vault to store state objects, a transaction store to store transactions, and a flow state machine manager to run flows. The node is essentially a container for running business logic written in CorDapps.

Corda nodes have:  

* a "default" `PublicKey` which is associated with the `CordaX500Name` for the node. Both the `PublicKey` and the
  `CordaX500Name` are paired with each other inside the `Party` object for the node.
* the capability to create new key pairs and allocate them to an "external ID" using the `IdentityService`. These keys   are often called "confidential identities" (`AnonymousParty`s) because they have no `CordaX500Name` associated with them by default.
* the capability, using the `IdentityService`, to allocate `PublicKey`s created by another node to the `CordaX500Name` of that node and optionally an "external ID" associated with that node.
* the capability to look up the `CordaX500Name` (and therefore the `Party` object) for an `AnonymousParty` (confidential identity) using the `IdentityService`.

An **account** is not a node. Instead, an account is "hosted" on a Corda node and an account represents a logical subset of the host node's vault.

Accounts differ from nodes in the following additional ways:
* Unlike a Corda node, an account does not have a default `PublicKey` associated with it. To transact with an account, you must request that the node hosting the account generate a new key pair for the account.
* An account does not have a unique identity associated with it. At the network level, the account's "identity" is the `CordaX500Name` of the host node. However, accounts are _identifiable_, as an identity can be assigned to an account, but this must be done at the application level.


## Do I need to add account IDs to my state objects?

No! In the Accounts SDK, `PublicKey`s are mapped to account IDs, specifically so that accounts are not tightly coupled to state objects. The benefit of this design is that you can use the same state objects for both workflows where accounts have been enabled and workflows where accounts have not been enabled. To find out which account the state belongs to, you can look up the participation/owning keys to the account ID they are allocated to.

## How do I assign identities to accounts?

This is an application-level concern and it's up to you how you solve it.

## How do nodes become aware of accounts?

Again, this is an application-level concern. Version n.n of the Accounts SDK does not ship with any sort of automatic discovery mechanism. However, there are primitive flows called `RequestAccountInfo`and `ShareAccountInfo` for sharing account information, as well as `ShareStateAndSyncAccounts` for sharing a state with another node, along with all the `AccountInfo`s and `PublicKey` mapping for the accounts involved in that state.

If you are writing an "accounts-enabled" CorDapp you'll probably want to write a flow that shares new accounts with all the other nodes in your business network that need to be aware of the new accounts.

## How do I mix workflows with accounts and non-accounts?

For now, in the interests of simplicity, we would suggest that if you need to use accounts, then use accounts on all nodes on your business network. For a node that does not _need_ to use accounts then you can just set up a "default" account for that node.

## How do I write a CorDapp if I want to have some nodes that are not using accounts?

This is a bit tricky for now. You'll need to update your flows with logic to determine if a state participant/owner is an account holder or not. You can do this using the `AccountService` APIs. You will also need to take some additional steps to share the required `AccountInfo`s and `PublicKey` mappings with the other nodes that need to see them.

Working with accounts is much like working with confidential identities but with the addition of the `AccountInfo` state. For example, when using confidential identities, a new `PublicKey` would be requested from a counterparty node before creating a transaction with that counterparty. The same workflow is required with accounts - a new `PublicKey` should be requested before creating a state and transaction. The biggest difference is that, in your flow, you must also specify which account the new `PublicKey` should be allocated to. However, of course, this should be known up-front by the node invoking the flow.

## How does state selection work if I am using accounts and unaccounted states on the same node?

Currently, if accounts and non-accounts workflows are mixed on the same node, then you need to be careful with state selection. When selecting states for non-accounts workflows, the state selection code will currently pick states which could be assigned to accounts. However, when selecting states for accounts workflows, you will supply the account ID for the account you want to select states from, discriminating the states query to only states held by the specified account. The best solution for now is to mandate that a node uses accounts for _all_ states or not at all.

## Are states for different accounts segregated?

No. They are all stored in the same `VaultService` operated by the host node. States held by different accounts on the same node can be partitioned by their account ID when querying the vault.

## How do I handle permissions on a per account basis?

This must be done at the application level for now and can be handled in the RPC client part of your CorDapp.
