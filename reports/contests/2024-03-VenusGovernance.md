## Findings Summary

| ID | Description | Severity |
| - | - | - |
| [M-01](#m-01-failed-proposals-can-be-executed-by-everyone-when-contracts-are-paused-due-to-missing-whennotpaused) | Failed proposals can be executed by everyone when contracts are `paused` due to missing `whenNotPaused` | Medium |
| [L-01](#l-01-cross-chain-messaging-layer-can-be-dos-using-low-gas-in-_adapterparams) | Cross-chain messaging layer can be DoS using low gas in `_adapterParams` | Low |
| [L-02](#l-02-user-can-block-the-destination-_nonblockinglzreceive-by-retrying-multiple-failed-proposals) | User can block the destination `_nonBlockingLzReceive` by retrying multiple failed proposals | Low |

# [M-01] Failed proposals can be executed by everyone when contracts are `paused` due to missing `whenNotPaused`

**Description:**

Venus Governance system doesn’t handle the situation when the VIP fails in the destination chain’s `nonBlockingLzReceive` function and then at a later stage, `OmnichainGovernanceExecutor` is paused, for example, to be replaced or an exploit occurs in the system. 

The problem is that the failed messages are being hashed and saved in a mapping: 

`failedMessages[srcChainId_][srcAddress_][nonce_] = hashedPayload;`

Then everyone incentivized can call `NonBlockingLzApp::retryMessage` and add the failed proposal to the queue, then he only needs to wait for the delay period and execute the proposal **even when the contracts are in a paused state**.

An especially dangerous scenario is when there is a `CRITICAL` proposal that failed for some reason, since the delay is the shortest. 
Consider the example:

1. An important grant is approved to be made on the Arbitrum chain but the **funds** **receiver** passed to the proposal is for the Ethereum chain.
2. Daily Limit is hit before this proposal and `nonBlockingLzReceive` reverts, storing the message in `failedMessages`
3. There is nothing to do from the protocol’s side on the Arbitrum chain, regarding the failed message.
4. The right proposal is executed and funds are granted to the desired Ethereum recipient.
5. A user sees the actions and manages to deploy a contract on the address **passed as a recipient in step 1 with CREATE2 opcode on the Arbitrum chain**.
Similar to this: https://rekt.news/wintermute-rekt/
6. Then at a later stage when the `OmnichainGovernanceExecutor` is **paused** and `GUARDIAN` doesn’t monitor it frequently, the user will retry the message, put it in a queue, and execute it right after the delay passes, receiving the funds specified.

All the functions needed for this attack are lacking `whenNotPaused` modifier (compared to `OmnichainProposalSender`):

`retryMessage` → `_nonblockingLzReceive` → `_queue` → `execute`.

```solidity
OmnichainGovernanceExecutor.sol

// @audit missing whenNotPaused
function _nonblockingLzReceive(uint16, bytes memory, uint64, bytes memory payload_) internal virtual override {

function _queue(uint256 proposalId_) internal {

function execute(uint256 proposalId_) external nonReentrant {
```

**Recommendation:**

In order to fully mitigate the issues originating from the `retryMessage` function the following changes should be implemented:

- All the functions involved in proposal execution should have `whenNotPaused` modifier.
- Apply `onlyOwner` modifier to the retry function, allowing only the `OmnichainGovernanceExecutor`’s owner to retry transactions.

# [L-01] Cross-chain messaging layer can be DoS using low gas in `_adapterParams`

**Description:** Anyone can disrupt cross-chain communication by setting a low gas fee in the `_adapterParams`.
When using LayerZero to send messages, the sender decides how much gas to give the Relayer for delivering the payload to the destination chain. 

Can read here: https://docs.layerzero.network/v1/developers/evm-guides/advanced/relayer-adapter-parameters

In `OmnichainProposalSender`, it's assumed that specifying a small amount of gas isn't possible. However, in reality, as a proposer, you can pass any amount of gas you want.

The attacker propose and passing low gas in `_adapterParams`.

We'll use the test payload, which is `Timelock.setDelay()`, but the user can pass whatever they want. This execution cost roughly 3к gas in the tests, but he will specify it to be 2k (or lower). The transaction will be executed on the source chain and `lzReceive` on the destination chain will be called, but first it will be validated [here](https://github.com/LayerZero-Labs/LayerZero/blob/main/contracts/Endpoint.sol#L118). Due to the low gas, this call will fail inside (it won't even reach to `OmnichainGovernanceExecutor._blockingLzReceive` [here](https://github.com/LayerZero-Labs/solidity-examples/blob/627caff1b7955e0473540b15effd1aa72f685f5c/contracts/lzApp/LzApp.sol#L51)) `LzApp.lzReceive`, and then it will be stored in [storedPayload](https://github.com/LayerZero-Labs/LayerZero/blob/48c21c3921931798184367fc02d3a8132b041942/contracts/Endpoint.sol#L122), and since it is in `try/catch` the upper calling function `OmnichainProposalSender.execute()` will continue and emit `ExecuteRemoteProposal` assuming that `lzReceive` was successful, causing a blocking of the cross-chain commutation and lying with wrong emit. 
Then all further proposes with the same payload but passing real gas, will fail in [Layer zero Endpoint.receivePayload](https://github.com/LayerZero-Labs/LayerZero/blob/48c21c3921931798184367fc02d3a8132b041942/contracts/Endpoint.sol#L116), because they was already store it in `storedPayload`, and because of that it will trigger the `catch` block of the `OmnichainProposalSender.execute()`, which will store it in `storedExecutionHashes`. 
By protocol design, if anything is stored in `storedExecutionHashes`, `OmnichainProposalSender.retryExecute` will be called, but any call to this will fail again in [Layer zero Endpoint.receivePayload](https://github.com/LayerZero-Labs/LayerZero/blob/48c21c3921931798184367fc02d3a8132b041942/contracts/Endpoint.sol#L116).

To run the PoC, paste the test in `Omnichain.ts` and run it with:

```solidity
npx hardhat test --grep "Test issues"
```

But since the tests use [LZEndpointMock](https://github.com/LayerZero-Labs/solidity-examples/blob/627caff1b7955e0473540b15effd1aa72f685f5c/contracts/lzApp/mocks/LZEndpointMock.sol#L174) and this file do not revert after put something is storedPayload as the original Endpoint. You need to change the `LZEndpointMock` in `node_modules` to stimulate the real [LayerZero Endpoint](https://github.com/LayerZero-Labs/LayerZero/blob/main/contracts/Endpoint.sol).

Change the `node_modules/@layerzerolabs/solidity-examples/contracts/lzApp/mocks/LZEndpointMock.sol::receivePayload` as this: https://gist.github.com/Slavchew/072992cdc1a3a071bf649fa3c23fd657

Add the following test to `tests/Cross-chain/Omnichain.ts` and run it with:

```
npx hardhat test --grep "Test C1"
```

```solidity
it("Test C1", async function () {
    const payload = await getPayload(NormalTimelock.address);
    
    // Change the gas to 2000
    adapterParams = ethers.utils.solidityPack(["uint16", "uint256"], [1, 2000]);
    // Calculate the nativeFee for updated adapterParams
    nativeFee = (await sender.estimateFees(remoteChainId, payloadWithId(payload), adapterParams))[0];
    
    // This will fail on destination lzReceive and store the execution in storedPayload, but will assume successful execution and emit ExecuteRemoteProposal
    await expect(await sender.connect(signer1).execute(remoteChainId, payload, adapterParams, {
      value: nativeFee,
    })).to.emit(sender, "ExecuteRemoteProposal");

    // Update the gas and try again the same proposal
    adapterParams = ethers.utils.solidityPack(["uint16", "uint256"], [1, 10000]);
    nativeFee = (await sender.estimateFees(remoteChainId, payloadWithId(payload), adapterParams))[0];

    // This will fail in execute try/catch because it was already stored in storedPayload, and will store it in storedExecutionHashes
    await expect(await sender.connect(signer1).execute(remoteChainId, payload, adapterParams, {
      value: nativeFee,
    })).to.emit(sender, "StorePayload");

    const pId = await getLastSourceProposalId();

    // This will fail with 'LayerZero: in message blocking', because the cross-chain communication is blocked
    await sender
         .connect(signer1)
         .retryExecute(pId, remoteChainId, payloadWithId(payload), adapterParams, nativeFee, {
           value: nativeFee,
         });
  });
```

![Screenshot 2024-04-01 at 0.53.33.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/266b66e0-da3c-4ce5-9dfd-48d3055d679d/29da3ff1-99f7-4f69-92f7-220d2d7c500b/Screenshot_2024-04-01_at_0.53.33.png)

Issue with the same root case was founded in Maia contest, [M-09](https://code4rena.com/reports/2023-09-maia#m-09-message-channels-can-be-blocked-resulting-in-dos)

**Recommendation:**

A minimum gas need to be required

> decode `adapterParams` and check the `gasLimit` which will mainly be the second value.
See the layout here: https://github.com/LayerZero-Labs/LayerZero/blob/48c21c3921931798184367fc02d3a8132b041942/contracts/RelayerV2.sol#L94-L101)
> 

```diff
function execute(
    uint16 remoteChainId_,
    bytes calldata payload_,
    bytes calldata adapterParams_
) external payable whenNotPaused {
    _ensureAllowed("execute(uint16,bytes,bytes)");

    // A zero value will result in a failed message; therefore, a positive value is required to send a message across the chain.
    require(msg.value > 0, "OmnichainProposalSender: value cannot be zero");
    require(payload_.length != 0, "OmnichainProposalSender: Empty payload");

    bytes memory trustedRemote = trustedRemoteLookup[remoteChainId_];
    require(trustedRemote.length != 0, "OmnichainProposalSender: destination chain is not a trusted source");
    _validateProposal(remoteChainId_, payload_);
    uint256 _pId = ++proposalCount;
    bytes memory payload = abi.encode(payload_, _pId);

+   require(gasLimit >= gasLimitThreshold, "gasLimit too low");

    try
        LZ_ENDPOINT.send{ value: msg.value }(
            remoteChainId_,
            trustedRemote,
            payload,
            payable(msg.sender),
            address(0),
            adapterParams_
        )
    {
        emit ExecuteRemoteProposal(remoteChainId_, _pId, payload);
    } catch (bytes memory reason) {

        console.log("IN CATCH StorePayload");
        //console.log(_pId);
        storedExecutionHashes[_pId] = keccak256(abi.encode(remoteChainId_, payload, adapterParams_, msg.value));
        emit StorePayload(_pId, remoteChainId_, payload, adapterParams_, msg.value, reason);
    }
}
```

and the amount must be something over 50k at least because when `LzApp::lzReceive` is called on destination and then `OmnichainGovernanceExecutor._blockingLzReceive` will try to cache 30k gas for the emit and storing if `nonblockingLzReceive` failed, and if pass 10k and the `lzReceive` works till here it will fail due to underflow in `_blockingLzReceive`, because it doesn't guarantee that `gasleft()` is higher than 30k.

```solidity
function _blockingLzReceive(
    uint16 srcChainId_,
    bytes memory srcAddress_,
    uint64 nonce_,
    bytes memory payload_
) internal virtual override whenNotPaused {
    uint256 gasToStoreAndEmit = 30000; // enough gas to ensure we can store the payload and emit the event

    require(srcChainId_ == srcChainId, "OmnichainGovernanceExecutor::_blockingLzReceive: invalid source chain id");

    (bool success, bytes memory reason) = address(this).excessivelySafeCall(
        gasleft() - gasToStoreAndEmit, // @audit underflow
        150,
        abi.encodeCall(this.nonblockingLzReceive, (srcChainId_, srcAddress_, nonce_, payload_))
    );
    // try-catch all errors/exceptions
    if (!success) {
        bytes32 hashedPayload = keccak256(payload_);
        failedMessages[srcChainId_][srcAddress_][nonce_] = hashedPayload;
        emit ReceivePayloadFailed(srcChainId_, srcAddress_, nonce_, reason); // Retrieve payload from the src side tx if needed to clear
    }
}
```

To prevent `retryExecute` from failing, it must remove the stored payload from the endpoint contract, before retry in cases like this.

# [L-02] User can block the destination `_nonBlockingLzReceive` by retrying multiple failed proposals

**Description:**

All cross-chain proposals that failed in the `_nonblockingLzReceive` function are saved in a mapping provided by LayerZero:

```solidity
function _blockingLzReceive(uint16 srcChainId_, bytes memory srcAddress_, uint64 nonce_, bytes memory payload_)
    internal
    virtual
    override
    whenNotPaused
{
    uint256 gasToStoreAndEmit = 30000; // enough gas to ensure we can store the payload and emit the event
    require(srcChainId_ == srcChainId, "OmnichainGovernanceExecutor::_blockingLzReceive: invalid source chain id");

    (bool success, bytes memory reason) = address(this).excessivelySafeCall(
        gasleft() - gasToStoreAndEmit,
        150,
        abi.encodeCall(this.nonblockingLzReceive, (srcChainId_, srcAddress_, nonce_, payload_))
    );
    // try-catch all errors/exceptions
    if (!success) {
        bytes32 hashedPayload = keccak256(payload_);
        failedMessages[srcChainId_][srcAddress_][nonce_] = hashedPayload; //@audit failed
        emit ReceivePayloadFailed(srcChainId_, srcAddress_, nonce_, reason); // Retrieve payload from the src side tx if needed to clear
    }
}
```

The most frequent reason for failure will be the daily execution limit checked in the `_isEligibleToReceive`:

```solidity
function _isEligibleToReceive(uint256 noOfCommands_) internal {
    uint256 currentBlockTimestamp = block.timestamp;

    // Load values for the 24-hour window checks for receiving
    uint256 receivedInWindow = last24HourCommandsReceived;

    // Check if the time window has changed (more than 24 hours have passed)
    if (currentBlockTimestamp - last24HourReceiveWindowStart > 1 days) {
        receivedInWindow = noOfCommands_;
        last24HourReceiveWindowStart = currentBlockTimestamp;
    } else {
        receivedInWindow += noOfCommands_;
    }

    // Revert if the received amount exceeds the daily limit
    require(receivedInWindow <= maxDailyReceiveLimit, "Daily Transaction Limit Exceeded");//@audit can be filled with failed proposals

    // Update the received amount for the 24-hour window
    last24HourCommandsReceived = receivedInWindow;
}

```

This check is implemented as a relay which restricts the amount of proposals in a day. It is possible this amount to be changed through `setMaxDailyReceiveLimit` but only with another proposal.

This daily limit logic can be leveraged from an malicious user to block the entire proposal execution for a day in any given moment by executing multiple previously failed proposals from the `NonblockingLzApp::retryMessage` and hitting the max limit of queued targets. Even `GUARDIAN` can’t unblock the execution because the max daily limit isn’t being replenished when cancelling transactions from the `cancel` function.

Consider the following VIP: https://community.venus.io/t/move-bad-debt-from-the-current-debtors-to-the-bnb-bridge-exploiter-account-to-eliminate-remaining-shortfall/4071/5 and a `maxDailyLimit` = 100 (specified in deploy configs)
This VIP heavily relies on the prices of the assets involved and everyone interested can freely block the path by querying 10 proposals with targets counts 10. This will result in `_isEligibleToReceive` to return false effectively blocking all the subsequent proposals and potential liquidation avoidance.

**Recommendation:**

Apply onlyOwner modifier to the `retryMessage` function, allowing only the owner to execute failed functions or lower the limit when a transaction fail, so that way to miss it from including twice in the limit.