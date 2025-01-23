---
type: docs
title: Clever KMS
description: A serverless Hashicorp Vault API compatible FoundationDB layer
tags:
- addons
keywords:
- vault
- iam
- biscuit
- foundationdb

type: docs
---

This document explains the overall infrastructure of KMS product, as well as different roles and responsabilities for managed KMS instances.

## Clever Cloud Actions VS User Actions

## OFFLINE - Clever Cloud Action

Clever Cloud uses a tool that allows offline creation of the platform's ADMIN context. The administration creation process is simple and enables:

- Creation of a master key in XChaCha20Poly1305 format
- Splitting this key into multiple parts using the Shamir Secret Sharing Scheme algorithm (with customizable k and n)
- Encryption of each obtained part with the PGP keys of the expected recipients
- Optional archiving of the master key on an HSM or a subset of PGP keys (e.g., for vault storage)

## ONLINE - Clever Cloud Action

Starting a Clever Cloud KMS instance requires a ceremony to provide the minimum **k** keys out of **n**. This procedure initializes the KMS service, materializing the master key in memory. Once this step is completed, the service becomes available.

## ONLINE - User Action

A KMS service user will request access tokens from the IAM, which are of two types:

- Administrator
- User

The Administrator token allows a user to create a personal digital vault (Vault / KMS) through the Clever Cloud KMS service, which implements the Vault API. Thus, the operation can be done directly from a Vault CLI (https://developer.hashicorp.com/vault/docs/commands). After logging in, the administrator can create the vault and obtain n keys to reconstruct a unique master key (also using the Shamir Secret Sharing Scheme algorithm with k and n chosen by the administrator). **The distribution and securing of these keys is a process to be implemented by the user themselves**, as well as the ceremony to lock/unlock the vault. (Refer to the Vault documentation regarding Seal/Unseal). If the unlocking operation is valid, the user's vault master key is reconstructed and stored encrypted by the platform's versioned master key. The version allows us to perform a key change operation (rekey) in case of compromise or suspected compromise of a key or part of a key.

The user token allows interaction with the unlocked vault through the available Vault APIs (KV and Transit to date). When creating a transit key, the user's vault master key is used to store this key encrypted on the storage, which is itself encrypted. In case of a vault locking operation, the user master key is forgotten, and the vault data is inaccessible under any condition. Proper management of key parts is therefore crucial.

All interactions with storage are transactional, ensuring the integrity of writes during API calls.

## FAQ

- **Do you rely on the HashiCorp Vault solution, or a fork?** No. Clever Cloud provides an API created by us, which is compatible with HashiCorp's Vault, and locked by the token.
- **A is the Vault a dedicated instance on a VM that Clever Cloud manages?** Yes. Clever Cloud manages this instance, and is responsible for the proper functioning of the servers. You, the user, is responsible for the proper functioning of the service Clever Cloud provides you. The KMS instance is serverless, so as soon as the layer receives an incoming connection, it automatically opens a context which puts in memory the required information to perform operations over user data. Once the connection is cut by the user, the context disappears. DNS is responsible for choosing a physical server that receives the request, and to get the user context out of FoundationDB.
- **Is KMS a cluster, or a single instance?** It's a cluster of 3 machines ensuring the high availability, and the database is a cluster with replication counting several database instances.
- **Is there an associated hardware (like HSM), or is the key is kept in the Vault process memory?** There is no HSM. When Clever Cloud spin up a node, this node is sealed. To unseal it, a quorum of 3 people is necessary to reconstruct the cluster's master key, which allows to handle the KMS instance traffic. This fractionned secret (Shamir) belongs to Clever Cloud to manage the instance, and has nothing to do with your administrative rights.
- **Where's the key stored?**: FoundationDB stores the encrypted Tenant Root Key (that ciphers secrets), while your Master Key remains in memory in the unsealed node. This last key can't decrypt tenant secrets.
- **Who is responsible for organizing the quorum of secret bearers (Shamir) and their availability in case we need to restore the key?**:
In case of Vault physical restart, Clever Cloud must gather a quorum to re-inject the secret into the Vault. A physical restart of the KMS instance would be invisible to you, the user. To summarize, the physical server manages Clever Cloud's master key (this operation is invisible to you on the user side). This key that allows you to launch a KMS instance. **Your master key is only created when your quorum meets and assembles its shares of the secret. Your master key is then stored encrypted in FoundationDB.
- **Will I notice when the servcie is down?** The only time you, on the user side, might see a service down is if all of Vault nodes crash at the same time, or if all multiple FoundationDB nodes die. During a serious incident, like a fiber cut, for example.
- **Is the initial recovery of Shamir secrets, as well as the injection during a restart, done via the internet?**: Yes, this is done through [the official HashiCorp Vault CLI](https://developer.hashicorp.com/vault/docs/commands/operator/unseal).

