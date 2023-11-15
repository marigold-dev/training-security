# training-security

## Prerequisites

To run the code, a Ligo compiler is required and can be installed at this [location](https://ligolang.org/docs/intro/installation/?lang=jsligo)

Compile the contract with the Ligo compiler

```bash
ligo compile contract ./contracts/main.jsligo
```

or though Taqueria

```bash
npm i
TAQ_LIGO_IMAGE=ligolang/ligo:1.1.0 taq compile main.jsligo
```

## Context

training-security => TODO (https://github.com/InferenceAG/TezosSecurityBaselineChecking/blob/master/readme.md)
https://opentezos.com/smart-contracts/avoiding-flaws/#8-using-computations-that-cause-tez-overflows

///

### Off chain attacks

- bug
  - frontend pointing to unexisting KT1 address, wrong call of entrypoint entries etc ... => problem of maintenance
- impersonalization :
  - fake apps : fake frontends, emails scams, etc ...
  - contracts copies : dup contract deployed on the network
- replay attack : is it possible to copy a L1 tx to a L2 tx ? Ex : gas station use case
  - solution : introduce NONCE + CHAIN_ID. It can be also a timestamp as nonce (easy to spot)
- trust and no verify
  - fake data : FTX & co. You cannot just trust offchain APIs as granted
  - sign blindly any Tx payload from a wallet

### Onchain attacks

- programming errors
- leaks
- user management

We will start with a buggy code and try to fix issues one by one

## Programming errors

- bugs : // deploy a broken Michelson code, not using Ligo compiler for example ...

  - //const d : option<tez> = 1mutez - 2mutez; (https://github.com/InferenceAG/TezosSecurityBaselineChecking/blob/master/testcases/TC-004/mutezUnderflow.tz)
    => use Ligo
  - rounding issues : https://opentezos.com/smart-contracts/avoiding-flaws/#9-contract-failures-due-to-rounding-issues
  - still some bitwise operations are unsecure , like for Bitwise.shift_right or Bitwise.shift_left , it will raise Michelson error if oveflow is reached
    => solution : do lot of checks on the code or leave default behavior if the consequence is not important

  - Mistake of sender vs source
    => think twice about potential use case of your endpoint

- lib code change : //TODO example of devops issue
  => do more unit tests and CI reports

- private
  - any secret value can be read
    => don't store secret or encrypt it and reveal it later
- predictable information used for random :

  - use block timestamp. Can be predictable as it is in seconds and we know the block time more or less
  - use contract origination address : it is composed of hash of operation + origination index
    => solution :
    - ask independant participat to submit a random number. A bit painful as it require to do commit/reveal and a way to unlock a locked situation
    - good randomness Oracle. It is, in theory, possible to create a good off-chain random Oracle. Chainlink offers a randomness Oracle based on a verifiable random function (VRF), and may be one of the few, if not the only reasonably good available randomness Oracle but not available on Tezos

- leave the contract blocked on a state waiting for a user action. Ex : Shifumi game based on 10min timout to cliam a victory in case of opponent unfair behavior to not play
  => solution : Always have a way to unlock a situation, setting an admin control or timestamp based resolution

## Leaks

- replay attack : return same op 2 times on the result list !!! //TODO code

- memory (overflow (ex on FA1.2 : https://inference.ag/blog/2023-10-09-FA12_spenders/))

  - integers and nats, as they can be increased to an arbitrary large value
  - strings, as there is no limit on their lengths
  - lists, sets, maps, that can contain an arbitrary number of items

  Ex : Anyone could attack this contract by calling the lockAmount entry point with 0 tez many times, to add so many entries into the lockedAmount list, that simply looping through the entries would consume too much gas.
  From then on, all the funds would be forever locked into the contract.
  Even simply loading the list into memory and deserializing the data at the beginning of the call, could use so much gas that any call to the contract would fail.

  => solution :

  - ask the user to pay a minimum tez for each call
  - set a threshold limit
  - store data on a big_map
  - avoid unnecessary onchain coputation that can be do off chain. Ex : do not loop onchain and just update the part to purge a map in cache on an indexer ?

- reentrancy (hard but still possible on Tezos ..., the ETH loop of callback does not work but ... . Ex : no idea if possible to send money and loop on withdraw because balance is reset to 0 ???). Implement the test and see that it is not possible on a Test ! //TODO demonstarte it + SHOW this example (https://opentezos.com/smart-contracts/avoiding-flaws/#10-re-entrancy-flaws)

- overflow ??? There is no SafeMath in ligo . Do not confuse with https://packages.ligolang.org/package/@ligo/math-lib that is to manipulate float instead or multiply/deivide by 10^6. For the nat, int, and timestamp types, the Michelson interpreter uses arbitrary-precision arithmetic provided by the OCaml Zarith library. It means that their size is only limited by gas or storage limits. You can store huge numbers in a contract without reaching the limit

=> solution : do operation on int or nat instead of tez as it has larger values

- frontRunning / MEV : It can be done by the baker itself as the list is known in advance at each period ... or any bots litening to the gossip network ...
  - buy before big BUY TX , sell token after === sandwich attack
  - BOT : whataver increase user balance, you just copy with higher fees to pass first
  - BAKER : just change the order

## User trust & management

- administrators : who can do what on the contract ?
- lambda / mutable code : which part of the code can change, who can do it ? Is it audited ?
- check roles : Does the user can execute this part of the code? Ex : update_operator, check if sender == owne othrwise, anyone can give permissions.
- trustable oracles : Do I trust data from this Oracle ? Is he centralized ? Who is behind it ? What is the refresh time policy ?
