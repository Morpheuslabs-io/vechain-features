# VeChainThor Docs

Information is provided by VeChain to Morpheus Labs.

## Bi-Token Design

Financial characteristics are inherent in every blockchain. A proper economic model is one of the fundamental elements in a blockchain ecosystem, and a key factor for its success. We have learned from our business partners, especially corporations and enterprise business owners, that one of major obstacles to adopting blockchain technologies is the unpredictability of the cost of using blockchain, thanks to the volatility of cryptocurrencies. 

To tackle the problem, we design a bi-token system that includes the VeChain Token (VET) and VeThor Token (VTHO). The function of VET is to serve as value-transfer medium, or in other words, smart money, to enable rapid value circulation within the VeChainThor ecosystem. On the other hand, VTHO represents the underlying cost of using VeChainThor and will be consumed (or, in other words, destroyed) after on-chain operations are performed. According to our design, VTHO is generated from holding VET with a constant speed. In this way, we are able to detach the direct cost of using VeChainThor from the VET price.

Let $V$ be the amount of VET, $t$ the amount of time (in terms of the number of blocks) and $v$ the VTHO generation speed. Mathematically, we can write

$E_{\textrm{gen}} = v \cdot V \cdot t$

where $E_{\textrm{gen}}$ denotes the amount of VTHO generated from holding $V$ VET. On the other hand, for each transaction, given $G$ be the amount of Gas required to process the transaction by the system and $p$ the gas price in VTHO given by the transaction sender, we can calculate the amount of VHTO consumed for the transaction as:

$E_{\textrm{con}} = p\cdot G$

Velocity $v$ is a constant equal to $5\times10^{-8}$ VTHO per VET per block. In other words, if you had 10K VET, you would be given at most 4.32 VTHO every 24 hours. The gas price $p$ can vary in the range $\big[p^{\textrm{base}},2\cdot p^{\textrm{base}}\big]$ where $p^{\textrm{base}}$ is a parameter that can be adjusted according to the market supply and demand of VTHO. Currently, we set $p^{\textrm{base}} = 1 \,\textrm{VTHO}\,/\,\textrm{Kgas}$. 



## Block

VeChainThor defines a [block](https://github.com/vechain/thor/blob/master/block/block.go) in Golang as:

```go
type Block struct {
	header *Header
	txs    tx.Transactions
}

type Header struct {
	body headerBody
}

type headerBody struct {
	ParentID    thor.Bytes32
	Timestamp   uint64
	GasLimit    uint64
	Beneficiary thor.Address

	GasUsed    uint64
	TotalScore uint64

	TxsRoot      thor.Bytes32
	StateRoot    thor.Bytes32
	ReceiptsRoot thor.Bytes32

	Signature []byte
}

type Transactions []*Transaction
```

where `ParentID` is the ID of the parent block, `Beneficiary` the address assigned by the block generator to receive reward (in VTHO) and `TotalScore` the accumulated witness number of the chain branch headed by the block. We will describe what the score means when describing the [Proof of Authority consensus algorithm](#poa).

Let $\Gamma$ denote `headerBody`. The block ID (`thor.Bytes32`) can be computed as:

$BlkID = h \circ \big(hash\,(\Gamma-\{sig\})\big)[4:]$

where $h$ is the block number stored as a `uint32` and $[4:]$ the operation that discards the first four bytes. 

## Transaction Model

VeChainThor adopts a new transaction model to tackle some of the fundamental problems that hinder a broader use of blockchain at the moment. The model is defined in Golang as:

```go
// transaction.go

type Transaction struct {
	body body
}

type body struct {
	ChainTag     byte			
	BlockRef     uint64
	Expiration   uint32
	Clauses      []*Clause
	GasPriceCoef uint8
	Gas          uint64
	DependsOn    *thor.Bytes32 `rlp:"nil"`
	Nonce        uint64
	Reserved     []interface{}
	Signature    []byte
}
```
 
Fields within the transaction `body`, $\Omega$, are defined as:

* `ChainTag` – last byte of the genesis block ID which is used to identify a blockchain to prevent the cross-chain replay attack;
* `BlockRef` - reference to a specific block;
* `Expiration` – how long, in terms of the number of blocks, the transaction will be allowed to be mined in VeChainThor;
* `Clauses` – an array of *Clause* objects each of which contains fields `To`, `Value` and `Data` to enable a single transaction to carry multiple tasks issued by the transaction sender;
* `GasPriceCoef` – coefficient used to calculate the gas price for the transaction.
* `Gas` – maximum amount of gas allowed to pay for the transaction;
* `DependsOn` – ID of the transaction on which the current transaction depends;
* `Nonce` – number set by user;
* `Reserved` - Reserved field for backward compatibility. It MUST be set empty for now otherwise the transaction will be considered invalid.
* `Signature` - transaction signature, $sig=sign\Big(hash\big(rlp(\Omega-\{sig\})\big),\,sk\Big)$ where $sk$ is the transaction sender's private key.

### Transaction ID

Every blockchain system must find a way to uniquely identify each transaction. Otherwise the system would be vulnerable to the transaction replay attack. In VeChainThor, we give every transaction a unique ID to identify itself. In particular, the transaction ID, $TxID$, can be calculated as:

$TxID=hash\big(hash(\Omega-\{sig\}),\textrm{signer_address}\big)$

When validating a given transaction, VeChainThor computes its $TxID$ and checks whether it has been used before. 

Suppose Alice has signed a transaction that sends 10 VET to Bob and Bob wants to re-use the transaction to get 10 VET from Alice. Obviously, this is not going to work for Bob. Since the two transactions have exactly the same ID, the one broadcast by Bob would be rejected due to the existence of the transaction ID. 

For any two transactions, as long as they had a field in $\Omega-\{sig\}$ with different values, their transaction IDs would be different. Moreover, we can always adjust the *Nonce* field to result in a new ID. In contrary to Ethereum, VeChainThor users can easily assemble multiple transactions sent from the same account with different IDs, which means that they could be sent off at the same time and would be processed by VeChainThor independently.

### Multi-Task Transaction

VeChainThor allows a single transaction to carry out multiple tasks. To do that, we introduce the `Clause` structure to represent a single task and allow multiple tasks defined in one transaction. 

The structure is defined in Golang as follows:

```go
type Clause struct {
	body clauseBody
}

type clauseBody struct {
	To    *thor.Address `rlp:"nil"`
	Value *big.Int
	Data  []byte
}
```

and contains three fields:

* `To` – recipient’s address;
* `Value` – amount transferred to the recipient;
* `Data` – input data.

We then define `Clauses` as a `Clause` array in the transaction model to make it possible for a transaction to conduct multiple tasks. 

The multi-task mechanism has two interesting characteristics:

* Since tasks (clauses) are included in a single transaction, their executions can be considered as atomic, meaning that, they either all succeed, or all fail.

* Tasks (clauses) are processed one by one in the exact order defined in `Clauses`.

The multi-task mechanism provides us a secure and efficient way to handle, for instance, tasks such as fund distribution, token airdrop, mass product registration, etc. Moreover, due to the fact that the tasks are processed sequentially, it can be used to conduct a multi-step process. 

#### Transaction Gas Calculation

The total gas, $g_{\textrm{total}}$, required for a transaction can be computed as:

$g_{\textrm{total}}=g_0+\sum_i\big(g_{\textrm{type}}^i+g_{\textrm{data}}^i+g_{\textrm{vm}}^i\big)$

where 

$g_0=5,000$, 

$g_{\textrm{type}}^i=48,000$ if the $i^{\textrm{th}}$ clause is to create a contract or $g_{\textrm{type}}^i=16,000$ otherwise,

$g_{\textrm{data}}^i = 4 * n_{z}^i + 68 * n_{nz}^i$ where $n_{z}^i$ is the number of bytes equal to zero within the data in the $i^{\,\textrm{th}}$ clause and $n_{nz}^i$ the number of bytes not equal to zero,

and $g_{\textrm{data}}^i$ is the gas cost returned by the virtual machine for executing the $i^{\,\textrm{th}}$ clause.

### Other New Features

Besides *Clauses*, VeChainThor's transaction model includes fields `DependsOn`, `BlockRef` and `Expiration` to allow us to further empower a transaction. Let us first revisit these fields as follows:

* `DependsOn` stores the ID of the transaction on which the current transaction depends. In other words, the current transaction cannot be processed without the success of the transaction referred by `DependsOn`. Here by “success”, we mean that the referred transaction has been executed without state reversion.

* `BlockRef` stores the reference to a particular block whose next block is the earliest block the current transaction can be included. In particular, the first four bytes of `BlockRef` contains the block height, $h$, while the second four bytes can be used to prove that the referred block is known before the transaction is assembled. If that is the case, the value of `BlockRef` should match the first eight bytes of the ID of the block with height $h$. 

* `Expiration` stores a number that can be used, together with `BlockRef`, to specify when the transaction expires. Specifically, the sum of `Expiration` and the first four bytes of `BlockRef` defines the height of the last block that the transaction can be included.

`DependsOn` allows us to systematically define an order for a sequence of transactions and such an order is guaranteed by the rules hard-coded as part of the consensus of VeChainTor. Moreover, the system requires the prior transaction depended on by the current transaction to be truly executed, adding another useful layer of security on the dependency.

`BlockRef` and `Expiration` allows us to set the life cycle of a transaction that has not been included in a block. The former defines the starting point and the latter its active period. With such a handful feature, we would no longer be troubled by the situation that a transaction was stuck for hours or even days waiting to be processed and we could not do anything about it. The inclusion of two fields would make transactions safer since it prevents them from being hijacked and later re-used to cause problems.

### Proof of Work

VeChainThor allows the transaction-level proof of work and converts the proved work into extra gas price that will be used by the system to generate more reward to the block generator that includes the transaction. In other words, users can utilize their local computational power to make their transactions more likely to be included in a new block. 

In particular, the computational work can be proved through fields `Nonce` and `BlockRef` in the transaction model. Let $n$ be the value of `Nonce`, $h_0$ the height of the block referred by `BlockRef`, $h$ the height of the block that includes the transaction and $g$ the value of field `Gas`. The extra gas price $\Delta p$ can be computed as:

$w = \frac{2^{256}-1}{hash\,\big(rlp\,(\Omega\,-\{n,\,sig\}\,+\,signer)\,\circ\,n\big)}$

$\Delta p=p^{\textrm{base}}\Big(\frac{1}{g}\big(\frac{w}{10^3}\big)\big(\frac{1}{1.04}\big)^{\frac{h}{2.592\times 10^5}}\Big)$.

where $\circ$ denotes the operator that concatenates two byte array, $rlp$ the function that performs the RLP encoding, $sig$ the transaction signature and $signer$ the address of the account that signs the transaction. 

Using the above formula, users can keep trying various *Nonce* value to maximize $\Delta p$. However, a transaction sender is not given an infinite amount time to search for the desirable $n$.  In particular, a transaction has to satisfy $\vert h-h_0\vert\leq 30$ or otherwise $\Delta p=0$ no matter how large $\Delta p$ can be computed. Moreover, to prove that the local computation starts no later than the block with height $h_0$, the transaction sender has to fill `BlockRef` with the value equal to the first eight bytes of the ID of the referred block. 

### Total Gas Price

The total gas price for the transaction sender is computed as:

$p^{\textrm{total}}=p^{\textrm{base}}+p^{\textrm{base}}\frac{\phi}{255}$

and the total price for block generators as 

$p^{\textrm{total}}=p^{\textrm{base}}+p^{\textrm{base}}\frac{\phi}{255}+\Delta p$

where $\phi$ is the value of field `GasPriceCoef` and $\Delta p$ the extra gas price converted from the proven local computational work. 

It can be seen that the gas price used to calculate the transaction cost depends solely on the input gas-price coefficient while the reward for packing the transaction into a block varies due to the transaction-level proof-of-work mechanism.

## Built-in Smart Contracts

There are six smart contracts deployed in VeChainThor's genesis block. Their source code can be found [here](https://github.com/vechain/thor/tree/master/builtin/gen). We describe briefly what they do as follows:

* `authority.sol` 

 facilitates the Proof of Authority consensus algorithm. The contract manages a list of candidate proposers for generating new blocks. They are authorized by the steering committee of the VeChain Foundation. Only the first 101 of them in the list, the so-called authority masternodes, can generate new blocks at the moment. Information of a candidate proposer stored in the system includes the signer address, endorsor address and identity. They need to use the private key corresponding to the signer address to sign any block they assemble and put sufficient VET as a deposit at the endorsor address. The identity is the hash of the proposer's real identity.

* `energy.sol` 
 
 is the smart contract that creates the VeThor token (VTHO) on VeChainThor. It conforms the VIP180 (or equivalently, the ERC20) standard. The smallest unit of VTHO is wei where 1 VTHO = 1e18 wei.
 
* `executor.sol`

 contains the core code for the on-chain governance. The contract enables a proposal to be executed automatically on VeChainThor if it is approved by at least two-third of the steering committee members. A proposal can be registered either by an approver (a steering committee member) or by an authorized voting contract. 
 
* `extension.sol`

 defines functions to get information of a certain block or transaction. 
 
* `params.sol` 

 defines functions to get and set the governance parameters of VeChainThor. Note that only the built-in "Executor" contract is allowed to set new values for the parameters. You can find the list of the governance parameters in [`params.go`](https://github.com/vechain/thor/blob/master/thor/params.go).
 
* `prototype.sol`

 is the smart contract that implements the [multi-party payment protocol](#mpp) which will be discussed in detail shortly.  

Examples of how to use the built-in contracts can be found [here](https://github.com/vechain/thor-builtins). 
  
## Multi-Party Payment Protocol {#mpp}

One of the major obstacles for ordinary people, or even enterprises, to adopt a public blockchain is the uncertainty and complexity in dealing with crypto assets. On one hand, users have to face the high price volatility when acquiring crypto from the market; on the other hand, they need to understand related concepts and get familiar with various tools to be able to use and manage their crypto assets. 

Can we find a way around the above-mentioned difficulties? For the existing blockchain networks such as Bitcoin and Ethereum, the answer is most likely negative. This is due to the fact that for those systems transaction fee has to be paid by whom sends the transaction and sending transactions is the way we interact with a public blockchain.

In VeChainThor, we come up with the multi-party payment protocol (MPP) to tackle this problem. Basically, MPP says that transaction fee can be paid by someone other than who sends the transaction if certain conditions meet. In this way, users can interact with VeChainThor even with a zero balance. 

Let us first define the terminology to be used to describe MPP as follows:

* *Sender* - account that signs the transaction;
* *Recipient* - account to which the transaction is sent;
* *Sponsor* - account that sponsors the recipient to pay for the transaction fee;
* *User* - VeChainTor allows any account to register other accounts as its users and conditionally pay for the cost of the transactions sent them;
* *Credit* - available VHTO for paying for transaction cost for a particular user of a particular account. 

![](MPP_text.png)

The above figure shows the decision-making flow within MPP. When it comes to the question of who should pay for a transaction, VeChainThor first checks the usership and sponsorship associated with the *Sender* and *Recipient*. It then tries to deduct the transaction fee from the corresponding account. For instance, if both the usership and sponsorship are in place, the system will first try to deduct the transaction fee from the *Sponsor*’s balance; if it fails, then from the *Recipient*'s balance; and if it fails again, from the *Sender*’s balance. 

In practice, a dapp is most likely built upon multiple smart contracts deployed on VeChainThor. Its users interact with our public blockchain through sending transactions to the smart contracts to call a certain function. With MPP, the dapp owner can register its users' accounts as the *User* of the smart contracts such that all the legit transactions from the dapp users can be paid by the owner. In this way, people can use the dapp almost in the same way they use other apps without dealing with crypto. Moreover, the owner can set up a single account to sponsor all the smart contract, which makes the maintenance a lot easier. 

### Credit Plan

To prevent MPP from being abused by malicious users, the owner of a smart contract can set a credit plan for the contract to set up rules on how to pay for the transactions from users. In particular, a credit plan can be defined as: 

```go
type creditPlan struct {
	Credit       *big.Int
	RecoveryRate *big.Int
}
```

where `RecoveryRate` is the amount of VTHO (in wei) accumulated per block to pay for transactions for each user and `Credit` the maximum amount of VTHO (in wei) that can be accumulated.

When the system checks whether an account's user has a sufficient amount of credit to pay for the transaction, it calculates the available credit $c$ as:

$c = \min{\big(C,C-c_{\textrm{used}}+r\cdot\max{(0, h-h_0)}\big)}$

where $C$ denotes `Credit`, $r$ `RecoverRate`, $h$ the current block height, $h_0$ the block height when the user uses credit last time and $u_{\textrm{used}}$ the amount of credit consumed after the user's last transaction is paid by the account. Note that $C-c_{\textrm{used}}$ is the remaining credit  after the last transaction is paid.

### Master Account

In VeChainThor, we introduce the concept of the master account to make it easier for dapp owners to user MPP and manage their dapps. Specifically, every account, including a smart contract, can have a *Master* account which is allowed by the system to register/remove *Users*, set a credit plan and select the active *Sponsor* for the account. Note that the account that deploys a smart contract becomes the *Master* of the contract by default. A normal account can also set its *Master* through calling function `setMaster` implemented in the built-in contract `Prototype`. We will describe the implementation of MPP shortly.

In practice, we may most likely build a dapp based on multiple smart contracts. Each contract may have its own *Users* and be sponsored by multiple *Sponsors*. How to manage these accounts suddenly becomes a challenging task the dapp owner has to think about. With the *Master* mechanism and built-in contract `Prototype`, the owner does not have to implement anything on the contract code level to user MPP. He/she can now use even a single *Master* to manage all the contracts through calling functions of contract `Prototype`. 

### MPP Implementation

The multi-party payment protocol is implemented by the built-in smart contract `Prototype` deployed at `0x000000000000000000000050726f746f74797065` in the genesis block of VeChainThor. 

#### Functions related to *User*

`isUser`

Check whether an account is a registered *User* of another account.

Input:

* `address _self`: account address
* `address _user`: *User* address

Return:

* `true` if `_user` is a *User* of `_self` or `false` otherwise

---

`addUser` / `removeUser`

Add / remove a *User* for an account. The transaction sender has to be the account itself or its current *Master*.

Input:

* `address _self`: account address
* `address _user`: *User* address

#### Functions related to the *credit plan*

`creditPlan`

Get the credit plan associated with an account.

Input:

* `address _self`: account address

Return:

* `uint256 credit`: maximum amount of credit (VTHO in wei) allowed for each  *User* of the account
* `uint256 recoveryRate`: amount of credit (VTHO in wei) generated per block for  each *User* of the account

---

`setCreditPlan`

Set a credit plan for an account. The transaction sender has to be either the account itself and its current *Master*.

Input: 

* `uint256 credit`: maximum amount of credit (VTHO in wei) allowed for each  *User* of the account
* `uint256 recoveryRate`: amount of credit (VTHO in wei) generated per block for  each *User* of the account

---

`userCredit`

Get the available credit for a particular *User* of an account.

Input:

* `address _self`: account address
* `address _user`: *User* address

Return:

* `uint256`: available credit (VTHO in wei) for the *User*

#### Functions related *Master*

`master`

Get the *Master* address of the given account address.

Input:

* `address _self`: account address

Return:

* `address`: address of the *Master* of `_self`.

---

`setMaster`

Set the *Master* for a particular account. The transaction sender has to be either the account itself or its current *Master*.

Input:

* `address _self`: account address
* `address _newMaster`: address of the new *Master* of `_self`

#### Functions related to *Sponsor* 

`sponsor` / `unsponsor`

Sponsor / unsponsor an account. The transaction sender has to be the *Sponsor* account.

Input:

* `address _self`: address of the account to be sponsored / unsponsored

---

`isSponsor`

Check whether an input account is a *Sponsor* of another account.

Input:

* `address _self`: account address
* `address _sponsor`: *Sponsor* address

Return:

* `true` if `_sponsor` is a *Sponsor* of `_self`

---

`selectSponsor`

Select a *Sponsor*. The transaction sender has to be either the sponsored account or its *Master*.

Input:

* `address _self`: account address
* `address _sponsor`: *Sponsor* address

---

`currentSponsor`

Get the current active *Sponsor*.

Input:

* `address _self`: account address

Return:

* `address`: address of the current active *Sponsor* of `_self`. 

 
## Proof of Authority {#poa}

One of the biggest decisions when designing a public blockchain system is about designing the consensus algorithm. The protocol not only dictates how blockchain participants agree on the blockchain grows but embodies the governance model imposed upon the system. 

Recall that the underlying design philosophy of our governance model is that 

*neither a total centralization nor a total decentralization would be the correct answer, but a compromise from and balance of both would*. 

VeChainThor implements the Proof of Authority (PoA) consensus algorithm which suits our governance model which states that there would not be anonymous block producers, but a fixed number of known validators (Authority Masternodes) authorized by the steering committee of the VeChain Foundation. 

>“It takes twenty years to build a reputation and five minutes to ruin it. If you think about that, you’ll do things differently.” – Warren Buffet

To be an Authority Masternode (AM), the individual or entity voluntarily discloses who they are (identity and reputation by extension) to the VeChain Foundation in exchange for the right to validate and produce blocks. It is their identities and reputations placed at stake that give all the AMs additional incentives to behave and keep the network secure. In VeChainThor, each AM has to go through a strict know-your-customer (KYC) procedure and satisfy the minimum requirements set by the Foundation.

When discussing a consensus algorithm, we must answer the following questions: 

* When is a new block produced? 
* Who generates the block? 
* How to choose the "trunk" from two legitimate blockchain branches?

### When

VeChainThor schedules a new block to be generated once every $\Delta$ seconds. We set $\Delta=10$, which is based on our estimation of the usage of VeChainThor.  Let $t_0$ be the timestamp of the genesis block. The timestamp of the block with height $h>0$, $t_h$, must satisfy $t_h=t_0+m\Delta$ where $m\in \mathbb{N}^+$ and $m\geq h$. 

### Who

PoA allows every available AM to have an equal opportunity to be selected to produce blocks. To do that, we introduce a deterministic pseudo-random process (DPRP) and the “active/inactive” AM status to decide whether a particular AM $a$ is legitimate for producing a block $B(h,t)$ with height $h$ (`uint32`) and timestamp $t$ (`uint64`). Here $t$ must satisfy $(t-t_0)\,\textrm{mod}\,\Delta=0$. We first define the DPRP to generate a pseudo-random number $\gamma(h,t)$ as:

$\gamma(h,t)=DPRP\,(h,t)= hash\,(h\circ t)$

where $\circ$ denotes the operation that concatenates two byte arrays. 

Let $A_B$ denote the sorted set of AMs with the “active” status in the state associated with block $B$. Note that in VeChainThor each AM is given a fixed index number and the numbers are used to sort elements in $A_B$. To verify whether $a$ is the legitimate AM for producing $B(h,t)$, we first define 

$A_{B(h,t)}^a=sort\big(A_{PA(B(h,t))} \cup a\big)$ 

where $PA(\cdot)$ returns the parent block. We then compute index $i^a(h,t)$ as:

$i^a (h,t)=\gamma(h,t)\,\textrm{mod}\,\|A_{B(h,t)}^a\|$

AM $a$ is the legitimate producer of $B(h,t)$ if and only if $A_{B(h,t)}^a\big[i^a (h,t)\big]=a$. Note that we put double quotes around the word “active” to emphasize that the status does not directly reflect the physical condition of a certain AM, but merely a status derived from the incoming information from the network. 

#### AM Status Updating

Given the latest block $B(h,t_1)$ and its parent $B(h-1,t_0)$, for any $t_0<t<t_1$ and $(t-t_0)\,\textrm{mod}\,\Delta=0$, the system computes AM $a_t$ such that 

$A_{B(h,t_1)}^{a_t}\big[i^{a_t}(h,t)\big]=a_t$

and mark $a_t$ as "inactive" in the state associated with $B(h,t_1)$. In addition, the system always sets the status of the AM that generates $B(h,t_1)$ as "active". Note that we set all the AMs as "active" from the beginning.

### Trunk

The final question we need to answer is how to choose the “trunk” from two legitimate blockchain branches. Since there is no computational competition in PoA, the “longest chain” rule does not apply. Instead, we consider the better branch as the one witnessed by more AMs. 

To do that, we compute the accumulated witness number (AWN), $\pi$, for block $B(h,t)$ as:

$\pi_{B(h,t)}=\pi_{PA(B(h,t))}+\|A_{B(h,t)}\|$

with $\pi_{B_{\textrm{genesis}}}=0$. Since $\|A_{B(h,t)}\|$ computes the number of AMs with “active” status associated with $B(h,t)$, it can be viewed as the number of AMs that witness the generation of $B(h,t)$. Therefore, we select the branch with the larger AWN as the trunk. If the AWNs are the same, we choose the branch with less length. Note that the AWN is stored in the block header as `TotalScore`.

Formally, given two branches $\mathcal{B}_1$ and $\mathcal{B}_2$ with their latest blocks $B_1(h_1,t_1)$ and $B_2(h_2,t_2)$, respectively, we first calculate their AWNs $\pi_{B_1}$ and $\pi_{B_2}$. The system then makes the following decision: choose $\mathcal{B}_1$ as the trunk if $\pi_{B_1}>\pi_{B_2}$, or $\mathcal{B}_2$ if $\pi_{B_1}<\pi_{B_2}$. In case $\pi_{B_1}=\pi_{B_2}$, choose $\mathcal{B}_1$ if $h_1<h_2$ or $\mathcal{B}_2$ if $h_1>h_2$. If $h_1=h_2$, keep the current trunk.
