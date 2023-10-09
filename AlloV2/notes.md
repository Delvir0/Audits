# Sherlock's Allo-V2 contest not-submitted issues
Don't ask me why mkay?
## High: It's possible to front run allocation or distribution and change the approved allocation to a higher value

### Contract: `RFPSimpleStrategy.sol`
### Summary:
Flow of the strategy is: `_registerRecipient -> _allocate -> _distribute` where the last function distributes the funds.
Both _allocate and _distribute can be frontrun to alter the saved values in _registerRecipient.
Both have their own failures in functions's requirements which leads to the frontrun.

### Detail
There are two scenario's in which we frontrun before:
1. _allocate is called,
2. _distribute is called.

#### 1. The _allocate function (or the _registerRecipient's status definition/ storage) fails to distinguish multiple _registerRecipient calls.
The attack: 
- _registerRecipient with a proposalBid of x amount (assuming that one is getting accepted)
- _allocate txn is called
- call _registerRecipient again with a proposalBid of x + x amount and a higher gas value

This would only be revented if the owner double checks before distributing.

#### 2. The _distribute's deactived pool requirement can be altered by anyone due missing AC in setPoolActive
Note: while missing AC on setPoolActive is an issue, it does not create any damage on it's own. 
While the function can be used to create a DOS scenario on all functions that require either an active or inactive pool, this can be prevented by calling the needed function in one txn.

The issue arises due to the combination of the following:
1. _distribute's onlyInactivePool can be altered by anyone due to missing AC in setPoolActive
2. __distribute does not check the status of the recipient_

The attack:
- _registerRecipient with x amount -> _allocate: this changes the status to Status.Accepted and deactivates the pool
- Call setPoolActive(true) -> _registerRecipient with x + x amount -> setPoolActive(false)
- recipient.Status is changed from Accepted to Pending
- Owner calls _distribute (the only requirement is that the pool is deactived)

## Medium: It's possible to alter anyone's recipient registration if useRegistryAnchor is set to false
### Contract: `RFPSimpleStrategy.sol`
### Summary:
Due to insufficient input validation and/ or AC, in the case where useRegistryAnchor = false, registration of any recipientId can be changed

### Detail
Below is _registerRecipient's else statement which triggers when useRegistryAnchor == false:

```solidity
else {
            //  @custom:data when 'false' -> (address recipientAddress, address registryAnchor, uint256 proposalBid, Metadata metadata)
            (recipientAddress, registryAnchor, proposalBid, metadata) =
                abi.decode(_data, (address, address, uint256, Metadata));

            // Check if the registry anchor is valid so we know whether to use it or not
            isUsingRegistryAnchor = registryAnchor != address(0);

            // Ternerary to set the recipient id based on whether or not we are using the 'registryAnchor' or '_sender'
            recipientId = isUsingRegistryAnchor ? registryAnchor : _sender; 

            // Checks if the '_sender' is a member of the profile 'anchor' being used and reverts if not
            if (isUsingRegistryAnchor && !_isProfileMember(recipientId, _sender)) revert UNAUTHORIZED(); 
        }
```
(`_sender` is an user input)

In the else statement, if `isUsingRegistryAnchor` is set to address(0), the _sender input is used to set recipientId which could be anything.
In the following code example, the recipientId is used to fetch the existing position (assume it exists) to alter the inputs: 

```solidity
Recipient storage recipient = _recipients[recipientId];
....
recipient.recipientAddress = recipientAddress;
recipient.useRegistryAnchor = isUsingRegistryAnchor ? true : recipient.useRegistryAnchor;
recipient.proposalBid = proposalBid; 
recipient.recipientStatus = Status.Pending; 
```

## Low/ informational
### 1. Incorrect use of time comparison in _isPoolActive
```solidity
 if (registrationStartTime <= block.timestamp && block.timestamp <= registrationEndTime) { //@audit-issue should be block.timestamp < registrationEndTime
            return true;
        }
        return false;
```
Code includes being at the timestamp as in time, meaning the requirement hits when the timestamp matches. 
e.g. 
action = start, registrationStartTime = 100, if timestamp is at 100: perform action.
action = stop, registrationEndTime = 100, if timestamp is at 100: perform action.

### 2. Populating recipient.useRegistryAnchor in the register process is inconsistent.
Each contract performs the following:
- RFPSimpleStrategy: `recipient.useRegistryAnchor = isUsingRegistryAnchor ? true : recipient.useRegistryAnchor;`
- QVBaseStrategy: `recipient.useRegistryAnchor = registryGating ? true : isUsingRegistryAnchor;`
- DonationVotingMerkleDistributionBaseStrategy: `recipient.useRegistryAnchor = useRegistryAnchor ? true : isUsingRegistryAnchor;`

Possible scenario in RFPSimpleStrategy: if recipient registered with isUsingRegistryAnchor == true and decides to re-register where isUsingRegistryAnchor == false, recipient.isUsingRegistryAnchor is kept true since that's the previous value
