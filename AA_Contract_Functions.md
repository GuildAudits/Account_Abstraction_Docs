
# Account Abstraction Analysis

## Stake Manager contract

| Function Name | Function Argument | Visibility | Return Type | One Line Description |
|---------------|-------------------|------------|-------------|----------------------|
| getDepositInfo | address account | external | DepositInfo | Returns the full deposit information of the given account |
| balanceOf | address account | external | uint256 | Returns the deposit (for gas payment) of the account |
| depositTo | address account | external | payable | Adds to the deposit of the given account |
| addStake | uint32 _unstakeDelaySec | external | payable | Adds to the account's stake - amount and delay |
| unlockStake | N/A | external | N/A | Attempts to unlock the stake |
| withdrawStake | address payable withdrawAddress | external | N/A | Withdraws from the (unlocked) stake |
| withdrawTo | address payable withdrawAddress, uint256 withdrawAmount | external | N/A | Withdraws from the deposit |

----------------------------------
## Sender Creator contract

| Function Name | Function Argument | Visibility | Return Type | One-line Description |
|---------------|-------------------|------------|-------------|----------------------|
| createSender  | bytes calldata initCode | external | address | Call the "initCode" factory to create and return the sender account address. |

-------------------

## Nonce Manager contract

Return the next nonce for this sender.

* Within a given key, the nonce values are sequenced (starting with zero, and incremented by one on each userop)
* But UserOp with different keys can come with arbitrary order.

| Function Name         | Function Argument                | Visibility  | Return Type | One-line Working                                                                                                          |
|-----------------------|----------------------------------|-------------|-------------|---------------------------------------------------------------------------------------------------------------------------|
| getNonce              | address sender, uint192 key      | public view | uint256     | Returns the next valid sequence number for a given nonce key by bitwise ORing the value of nonceSequenceNumber[sender][key] with the left-shifted value of key by 64. |
| incrementNonce        | uint192 key                      | public      | N/A         | Allows an account to manually increment its own nonce by incrementing the value of nonceSequenceNumber[msg.sender][key] by 1. |
| _validateAndUpdateNonce | address sender, uint256 nonce   | internal    | bool        | Validates and updates nonce uniqueness for the given account by checking if nonceSequenceNumber[sender][key] incremented by 1 is equal to the value of seq, which is extracted from nonce by bitwise ANDing it with 64 bits and converting it to uint64. Returns true if validation is successful. |


---------------------------
## Base Account contract

| Function Name | Function Argument | Visibility | Return Type | Working |
|---------------|-------------------|------------|-------------|---------|
| nonce()       | None              | public     | uint256     | Returns the account nonce value. |
| entryPoint()  | None              | public     | IEntryPoint | Returns the current entryPoint used by this account. |
| validateUserOp() | UserOperation calldata userOp, bytes32 userOpHash, uint256 missingAccountFunds | external | uint256 | Validates user's signature and nonce, and updates the account state to prevent replay attacks. Also sends missing funds to the entrypoint. |
| _requireFromEntryPoint() | None | internal | None | Ensures that the request comes from the known entrypoint. |
| _validateSignature() | UserOperation calldata userOp, bytes32 userOpHash | internal | uint256 | Validates the signature of the userOp against the provided userOpHash and returns validation data. |
| _validateAndUpdateNonce() | UserOperation calldata userOp | internal | None | Validates that the current nonce matches the UserOperation nonce, and updates the account's state to prevent replay attacks. |
| _payPrefund() | uint256 missingAccountFunds | internal | None | Sends missing funds to the entrypoint (msg.sender) for the current transaction. |


--------------------------------------------------------
## paymaster Contract

| Function Name | Function Argument | Visibility | Return Type | Working |
|---------------|-------------------|------------|-------------|---------|
| validatePaymasterUserOp | UserOperation calldata userOp, bytes32 userOpHash, uint256 maxCost | external | bytes memory | This function is used for payment validation. It takes in user operation details, including the user operation data, its hash, and the maximum cost of the transaction (based on gas and gas price from userOp). It returns a context value to be sent to the postOp function, and a validationData which includes the signature and time-range of the operation. |
| postOp | PostOpMode mode, bytes calldata context, uint256 actualGasCost | external | N/A | This function is the post-operation handler. It takes in the mode which can be opSucceeded, opReverted, or postOpReverted, the context value returned by validatePaymasterUserOp, and the actual gas cost used so far. It handles the post-operation logic based on the mode received. |


--------------------------------------------------------
## UserOperation Contract

* struct UserOperation


| Struct Name         | Params                        | Description                                                                                                              |
|--------------------|--------------------------------|----------------------------------------------------------------------------------------------------------|
| UserOperation       | `sender`                       | The sender account of this request.                                                                                |
|                    | `nonce`                        | Unique value the sender uses to verify it is not a replay.                                                   |
|                    | `initCode`                     | If set, the account contract will be created by this constructor.                                              |
|                    | `callData`                     | The method call to execute on this account.                                                                      |
|                    | `callGasLimit`                 | The gas limit passed to the callData method call.                                                               |
|                    | `verificationGasLimit`         | Gas used for validateUserOp and validatePaymasterUserOp.                                                       |
|                    | `preVerificationGas`           | Gas not calculated by the handleOps method, but added to the gas paid. Covers batch overhead.              |
|                    | `maxFeePerGas`                 | Same as EIP-1559 gas parameter.                                                                                     |
|                    | `maxPriorityFeePerGas`        | Same as EIP-1559 gas parameter.                                                                                   |
|                    | `paymasterAndData`            | If set, this field holds the paymaster address and paymaster-specific data. The paymaster will pay for the transaction instead of the sender. |
|                    | `signature`                     | Sender-verified signature over the entire request, the EntryPoint address, and the chain ID.         |



* Functions

| Function Name | Function Argument | Visibility | Return Type | Working |
|---------------|-------------------|------------|-------------|---------|
| getSender | UserOperation calldata userOp | internal pure | address | Returns the sender address from the UserOperation struct by reading the first member of the calldata using assembly. |
| gasPrice | UserOperation calldata userOp | internal view | uint256 | Calculates the gas price for the UserOperation struct by considering maxFeePerGas and maxPriorityFeePerGas, and taking into account the basefee of the current block. |
| pack | UserOperation calldata userOp | internal pure | bytes | Packs the fields of the UserOperation struct into a bytes array using abi.encode. |
| hash | UserOperation calldata userOp | internal pure | bytes32 | Calculates the hash of the packed UserOperation struct using keccak256. |
| min | uint256 a, uint256 b | internal pure | uint256 | Returns the minimum value between two uint256 values. |


-------------------------------------------
## Entry Point contract

| Function Name | Arguments | Visibility | Description |
|---------------|-----------|------------|-------------|
| UserOperationEvent | userOpHash: bytes32<br>sender: address<br>paymaster: address<br>nonce: uint256<br>success: bool<br>actualGasCost: uint256<br>actualGasUsed: uint256 | event | Event emitted after each successful request |
| AccountDeployed | userOpHash: bytes32<br>sender: address<br>factory: address<br>paymaster: address | event | Event emitted when an account is deployed |
| UserOperationRevertReason | userOpHash: bytes32<br>sender: address<br>nonce: uint256<br>revertReason: bytes | event | Event emitted when UserOperation callData reverts with non-zero length |
| BeforeExecution | - | event | Event emitted before starting the execution loop |
| SignatureAggregatorChanged | aggregator: address | event | Event emitted when the signature aggregator is changed |
| FailedOp | opIndex: uint256<br>reason: string | error | Custom revert error for handleOps function |
| SignatureValidationFailed | aggregator: address | error | Error case when signature aggregator fails to verify the aggregated signature |
| ValidationResult | returnInfo: ReturnInfo<br>senderInfo: StakeInfo<br>factoryInfo: StakeInfo<br>paymasterInfo: StakeInfo | error | Successful result from simulateValidation |
| ValidationResultWithAggregation | returnInfo: ReturnInfo<br>senderInfo: StakeInfo<br>factoryInfo: StakeInfo<br>paymasterInfo: StakeInfo<br>aggregatorInfo: AggregatorStakeInfo | error | Successful result from simulateValidation with aggregator info |
| SenderAddressResult | sender: address | error | Return value of getSenderAddress |
| ExecutionResult | preOpGas: uint256<br>paid: uint256<br>validAfter: uint48<br>validUntil: uint48<br>targetSuccess: bool<br>targetResult: bytes | error | Return value of simulateHandleOp |


| Function Name | Arguments | Visibility | Description |
|---------------|-----------|------------|-------------|
| handleOps | ops: bytes[],beneficiary | external | Function to handle multiple operations in a batch |
| getUserOpHash | UserOperation | view | Function to handle a single operation |
| getSenderAddress | bytes initcode| external |generate a request Id - unique identifier for this request.the request ID is a hash over the content of the userOp (except the signature), the entrypoint and the chainid.|
| simulateValidation | userOperation | external|Simulate a call to account.validateUserOp and paymaster.validatePaymasterUserOp.|
| simulateHandleOp | op: UserOperation<br> address target<br> bytes targetcallData | external |simulate full execution of a UserOperation (including both validation and target execution)|
| handleAggregatedOps | UserOpsPerAggregator[],beneficiary | external | Execute a batch of UserOperation with Aggregators |

----------------------------------------

## Wallet contract

| Function Name         | Arguments              | Visibility | Return Type | One-line Description                                  |
|-----------------------|------------------------|------------|-------------|------------------------------------------------------|
| initialize            | _entryPoint<br> _owner<br><br> _upgradeDelay<br> _guardianDelay<br> _guardian   | public     | N/A         | Initialized function to setup the smart Wallet contract |
| nonce                 | N/A                    | public     | uint256     | Return the contract nonce                            |
| entryPoint            | N/A                    | public     | IEntryPoint | Return the entrypoint address                        |
| onlyOwner             | N/A                    | modifier   | N/A         | Modifier for only owner                               |
| onlyOwnerOrFromEntryPoint | N/A               | modifier   | N/A         | Modifier that can be called by owner or from entrypoint |
| fallback              | N/A                    | fallback   | N/A         | Fallback function to receive Ether                   |
| registerCallback      | _callback              | external   | N/A         | Register a callback function                         |
| upgradeLogic          | _newLogic              | external   | N/A         | Upgrade the logic contract                           |
| upgradeAccount        | _account              | external   | N/A         | Upgrade an account to the new logic contract         |
| isOwner               | _address               | public     | bool        | Check if an address is the owner of the contract      |
| isGuardian            | _address               | public     | bool        | Check if an address is a guardian                    |
| authorizeGuardian     | _guardian             | external   | N/A         | Authorize a guardian                                 |
| removeGuardian        | _guardian             | external   | N/A         | Remove authorization of a guardian                   |
| getGuardian           | N/A                    | public     | address     | Get the current guardian address                     |
| setGuardian           | _guardian             | external   | N/A         | Set a new guardian                                   |
| setGuardianDelay      | _guardianDelay        | external   | N/A         | Set the guardian delay                               |
| getGuardianDelay      | N/A                    | public     | uint32      | Get the current guardian delay                       |
| isUpgrading           | N/A                    | public     | bool        | Check if the contract is currently upgrading         |
| setUpgradeDelay       | _upgradeDelay         | external   | N/A         | Set the upgrade delay                                |
| getUpgradeDelay       | N/A                    | public     | uint32      | Get the current upgrade delay                        |
| getUpgradeStartTime   | N/A                    | public     | uint256     | Get the start time of the current upgrade            |
| getUpgradeEndTime     | N/A                    | public     | uint256     | Get the end time of the current upgrade              |
| getUpgradeProgress    | N/A                    | public     | uint256     | Get the progress of the current upgrade              |






