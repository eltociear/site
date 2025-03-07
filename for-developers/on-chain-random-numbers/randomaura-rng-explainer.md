# RandomAura RNG Explainer

{% hint style="success" %}
The following example uses methods from the [RandomAura](https://github.com/poanetwork/posdao-contracts/blob/master/contracts/RandomAuRa.sol) contract. A [RANDAO ](https://github.com/randao/randao)methodology implemented by the validator nodes generates pseudorandom numbers.

The following is designed to work with[ OpenEthereum's AuRa ](https://openethereum.github.io/Aura)consensus protocol v3.2.5. In this protocol, validators take turns sealing blocks, one after the other, in a prescribed order.
{% endhint %}

### Collection Rounds

A series of **collection rounds** are used for random number generation. The  collection round length is configurable - in this example we use 38 blocks for a round, **** split into two equal 19 block phases:

* `commit phase`&#x20;
* `reveal phase`

The length of each phase is  `19` blocks ( `38 / 2 = 19`). When one collection round finishes, the next collection round starts, and so on. For example:

```
Block number   |   Phase
...
101                Commit phase     <-- collection round #1 started here
102                Commit phase
103                Commit phase
...
117                Commit phase
118                Commit phase
119                Commit phase
120                Reveal phase
121                Reveal phase
122                Reveal phase
...
136                Reveal phase
137                Reveal phase
138                Reveal phase     <-- collection round #1 finished here
139                Commit phase     <-- collection round #2 started here
140                Commit phase
141                Commit phase
...
```

During each collection round, the RandomAura contract (see below) collects the random numbers generated during that round.

### Commit Phase

1\) Each validator in the set generates a random `number` locally with their node, hashes the secret, and calls the `commitHash` function when it is their turn to create a block. &#x20;

```javascript
/// @dev Called by the validator's node to store a hash and a cipher of the validator's number on each collection
/// round. The validator's node must use its mining address (engine_signer) to call this function.
/// This function can only be called once per collection round (during the `commit phase`).
/// @param _numberHash The Keccak-256 hash of the validator's number.
/// @param _cipher The cipher of the validator's number. Can be used by the node to restore the lost number after
/// the node is restarted (see the `getCipher` getter).
function commitHash(bytes32 _numberHash, bytes calldata _cipher) external {
    address miningAddress = msg.sender;

    require(block.coinbase == miningAddress);
    require(isCommitPhase()); // must only be called in `commit phase`
    require(_numberHash != bytes32(0));
    require(!isCommitted(currentCollectRound(), miningAddress)); // cannot commit more than once

    uint256 collectRound = currentCollectRound();

    _commits[collectRound][miningAddress] = _numberHash;
    _ciphers[collectRound][miningAddress] = _cipher;
}
```

2\) This function accepts the hash of the secret `number` and its `cipher`. The `cipher` is the `number` encrypted with a validator's key, and  is needed for the `reveal phase` (see below).

For example, if there are three validators, they will call `commitHash` in the following order (for the sample case above):

```
Block number   |   Phase   |       What happens
...
101             Commit phase    Validator1 calls `commitHash`
102             Commit phase    Validator2 calls `commitHash`
103             Commit phase    Validator3 calls `commitHash`
104             Commit phase    Nothing happens
105             Commit phase    Nothing happens
106             Commit phase    Nothing happens
...
117             Commit phase    Nothing happens
118             Commit phase    Nothing happens
119             Commit phase    Nothing happens
120             Reveal phase    Validator1 calls `revealSecret`
121             Reveal phase    Validator2 calls `revealSecret`
122             Reveal phase    Validator3 calls `revealSecret`
123             Reveal phase    Nothing happens
125             Reveal phase    Nothing happens
126             Reveal phase    Nothing happens
...
136             Reveal phase    Nothing happens
137             Reveal phase    Nothing happens
138             Reveal phase    Nothing happens
139             Commit phase    Validator1 calls `commitHash`
140             Commit phase    Validator2 calls `commitHash`
141             Commit phase    Validator3 calls `commitHash`
...
```

When the `commit phase` finishes, the `reveal phase` starts.

### Reveal Phase

When a validator's turn arrives to create a block:

1\) The validator gets the cipher of the number using the `getCommitAndCipher` public getter.

```javascript
/// @dev Returns the Keccak-256 hash and cipher of the validator's number for the specified collection round
/// and the specified validator stored by the validator through the `commitHash` function.
/// @param _collectRound The serial number of the collection round for which hash and cipher should be retrieved.
/// @param _miningAddress The mining address of validator (engine_signer).
function getCommitAndCipher(
    uint256 _collectRound,
    address _miningAddress
) public view returns(bytes32, bytes memory) {
    return (_commits[_collectRound][_miningAddress], _ciphers[_collectRound][_miningAddress]);
}
```

2\) The validator decrypts the cipher with their key and retrieves the `number`.

3\) The validator calls the `revealNumber`function to reveal their committed number (the function XORs the number with the previous seed to create a new random seed stored in the `currentSeed` state variable).

```javascript
/// @dev Called by the validator's node to XOR its number with the current random seed.
/// The validator's node must use its mining address (engine_signer) to call this function.
/// This function can only be called once per collection round (during the `reveal phase`).
/// @param _number The validator's number.
function revealNumber(uint256 _number) external {
    address miningAddress = msg.sender;

    bytes32 numberHash = keccak256(abi.encodePacked(_number));
    uint256 collectRound = currentCollectRound();

    require(block.coinbase == miningAddress);
    require(!isCommitPhase()); // must only be called in `reveal phase`
    require(numberHash != bytes32(0));
    require(numberHash == _commits[collectRound][miningAddress]); // the hash must be commited
    require(!_sentReveal[collectRound][miningAddress]); // cannot reveal more than once during the same collect round

    currentSeed = currentSeed ^ _number;

    _sentReveal[collectRound][miningAddress] = true;
    delete _commits[collectRound][miningAddress];
    delete _ciphers[collectRound][miningAddress];
}
```

{% hint style="info" %}
_Note: Randomness created in a deterministic manner, through computerized means, it is called pseudorandomness. Pseudorandom numbers exhibit the same properties as random numbers. The method described above is technically a pseudorandom number generator (PRNG)_
{% endhint %}

## RandomAura Contract Code

The RandomAura Contract interfaces with the Authority Round consensus process to store and iterate the `currentSeed` , control when the seed is revealed, and report on skipped reveals by Validators.

Below is the full RandomAura contract code. The POSDAO implementation is located at [https://github.com/poanetwork/posdao-contracts/blob/master/contracts/RandomAuRa.sol](https://github.com/poanetwork/posdao-contracts/blob/master/contracts/RandomAuRa.sol)

```javascript
pragma solidity 0.5.10;

import "./interfaces/IRandomAuRa.sol";
import "./interfaces/IStakingAuRa.sol";
import "./interfaces/IValidatorSetAuRa.sol";
import "./upgradeability/UpgradeableOwned.sol";


/// @dev Generates and stores random numbers in a RANDAO manner (and controls when they are revealed by AuRa
/// validators) and accumulates a random seed. The random seed is used to form a new validator set by the
/// `ValidatorSetAuRa.newValidatorSet` function.
contract RandomAuRa is UpgradeableOwned, IRandomAuRa {

    // =============================================== Storage ========================================================

    // WARNING: since this contract is upgradeable, do not remove
    // existing storage variables, do not change their order,
    // and do not change their types!

    mapping(uint256 => mapping(uint256 => bytes)) internal _ciphers;
    mapping(uint256 => mapping(uint256 => bytes32)) internal _commits;
    mapping(uint256 => uint256[]) internal _committedValidators;

    /// @dev The length of the collection round (in blocks).
    uint256 public collectRoundLength;

    /// @dev The current random seed accumulated during RANDAO or another process
    /// (depending on implementation).
    uint256 public currentSeed;

    /// @dev A boolean flag defining whether to punish validators for unrevealing.
    bool public punishForUnreveal;

    /// @dev The number of reveal skips made by the specified validator during the specified staking epoch.
    mapping(uint256 => mapping(uint256 => uint256)) internal _revealSkips;

    /// @dev A boolean flag of whether the specified validator has revealed their number for the
    /// specified collection round.
    mapping(uint256 => mapping(uint256 => bool)) internal _sentReveal;

    /// @dev The address of the `ValidatorSetAuRa` contract.
    IValidatorSetAuRa public validatorSetContract;

    // ============================================== Modifiers =======================================================

    /// @dev Ensures the caller is the BlockRewardAuRa contract address.
    modifier onlyBlockReward() {
        require(msg.sender == validatorSetContract.blockRewardContract());
        _;
    }

    /// @dev Ensures the `initialize` function was called before.
    modifier onlyInitialized {
        require(isInitialized());
        _;
    }

    /// @dev Ensures the caller is the ValidatorSetAuRa contract address.
    modifier onlyValidatorSet() {
        require(msg.sender == address(validatorSetContract));
        _;
    }

    // =============================================== Setters ========================================================

    /// @dev Clears commit and cipher for the given validator's pool if the pool
    /// hasn't yet revealed their number.
    /// Called by the ValidatorSetAuRa.changeMiningAddress function
    /// when a validator creates a request to change their mining address.
    function clearCommit(uint256 _poolId) external onlyValidatorSet {
        uint256 collectRound = currentCollectRound();
        if (!_sentReveal[collectRound][_poolId]) {
            delete _commits[collectRound][_poolId];
            delete _ciphers[collectRound][_poolId];
        }
    }

    /// @dev Called by the validator's node to store a hash and a cipher of the validator's number on each collection
    /// round. The validator's node must use its mining address to call this function.
    /// This function can only be called once per collection round (during the `commits phase`).
    /// @param _numberHash The Keccak-256 hash of the validator's number.
    /// @param _cipher The cipher of the validator's number. Can be used by the node to restore the lost number after
    /// the node is restarted (see the `getCipher` getter).
    function commitHash(bytes32 _numberHash, bytes calldata _cipher) external onlyInitialized {
        address miningAddress = msg.sender;

        require(commitHashCallable(miningAddress, _numberHash));
        require(_getCoinbase() == miningAddress); // make sure validator node is live

        uint256 collectRound = currentCollectRound();
        uint256 poolId = validatorSetContract.idByMiningAddress(miningAddress);

        _commits[collectRound][poolId] = _numberHash;
        _ciphers[collectRound][poolId] = _cipher;
        _committedValidators[collectRound].push(poolId);
    }

    /// @dev Called by the validator's node to XOR its number with the current random seed.
    /// The validator's node must use its mining address to call this function.
    /// This function can only be called once per collection round (during the `reveals phase`).
    /// @param _number The validator's number.
    function revealNumber(uint256 _number) external onlyInitialized {
        _revealNumber(_number);
    }

    /// @dev The same as the `revealNumber` function (see its description).
    /// The `revealSecret` was renamed to `revealNumber`, so this function
    /// is left for backward compatibility with the previous client
    /// implementation and should be deleted in the future.
    /// @param _number The validator's number.
    function revealSecret(uint256 _number) external onlyInitialized {
        _revealNumber(_number);
    }

    /// @dev Changes the `punishForUnreveal` boolean flag. Can only be called by an owner.
    function setPunishForUnreveal(bool _punishForUnreveal) external onlyOwner {
        punishForUnreveal = _punishForUnreveal;
    }

    /// @dev Initializes the contract at network startup.
    /// Can only be called by the constructor of the `InitializerAuRa` contract or owner.
    /// @param _collectRoundLength The length of a collection round in blocks.
    /// @param _validatorSet The address of the `ValidatorSet` contract.
    /// @param _punishForUnreveal A boolean flag defining whether to punish validators for unrevealing.
    function initialize(
        uint256 _collectRoundLength, // in blocks
        address _validatorSet,
        bool _punishForUnreveal
    ) external {
        require(_getCurrentBlockNumber() == 0 || msg.sender == _admin());
        require(!isInitialized());
        IValidatorSetAuRa validatorSet = IValidatorSetAuRa(_validatorSet);
        require(_collectRoundLength % 2 == 0);
        require(_collectRoundLength % validatorSet.MAX_VALIDATORS() == 0);
        require(IStakingAuRa(validatorSet.stakingContract()).stakingEpochDuration() % _collectRoundLength == 0);
        require(_collectRoundLength > 0);
        require(collectRoundLength == 0);
        require(_validatorSet != address(0));
        collectRoundLength = _collectRoundLength;
        validatorSetContract = IValidatorSetAuRa(_validatorSet);
        punishForUnreveal = _punishForUnreveal;
    }

    /// @dev Checks whether the current validators at the end of each collection round revealed their numbers,
    /// and removes malicious validators if needed.
    /// This function does nothing if the current block is not the last block of the current collection round.
    /// Can only be called by the `BlockRewardAuRa` contract (by its `reward` function).
    function onFinishCollectRound() external onlyBlockReward {
        if (_getCurrentBlockNumber() % collectRoundLength != 0) return;

        // This is the last block of the current collection round

        address[] memory validators;
        address validator;
        uint256 i;

        address stakingContract = validatorSetContract.stakingContract();

        uint256 stakingEpoch = IStakingAuRa(stakingContract).stakingEpoch();
        uint256 startBlock = IStakingAuRa(stakingContract).stakingEpochStartBlock();
        uint256 endBlock = IStakingAuRa(stakingContract).stakingEpochEndBlock();
        uint256 currentRound = currentCollectRound();

        if (_getCurrentBlockNumber() > startBlock + collectRoundLength * 3) {
            // Check whether each validator didn't reveal their number
            // during the current collection round
            validators = validatorSetContract.getValidators();
            for (i = 0; i < validators.length; i++) {
                validator = validators[i];
                if (!sentReveal(currentRound, validator)) {
                    uint256 poolId = validatorSetContract.idByMiningAddress(validator);
                    _revealSkips[stakingEpoch][poolId]++;
                }
            }
        }

        // If this is the last collection round in the current staking epoch
        // and punishing for unreveal is enabled.
        if (
            punishForUnreveal &&
            (_getCurrentBlockNumber() == endBlock || _getCurrentBlockNumber() + collectRoundLength > endBlock)
        ) {
            uint256 maxRevealSkipsAllowed =
                IStakingAuRa(stakingContract).stakeWithdrawDisallowPeriod() / collectRoundLength;

            if (maxRevealSkipsAllowed > 1) {
                maxRevealSkipsAllowed -= 2;
            } else if (maxRevealSkipsAllowed > 0) {
                maxRevealSkipsAllowed--;
            }

            // Check each validator to see if they didn't reveal
            // their number during the last full `reveals phase`
            // or if they missed the required number of reveals per staking epoch.
            validators = validatorSetContract.getValidators();

            address[] memory maliciousValidators = new address[](validators.length);
            uint256 maliciousValidatorsLength = 0;

            for (i = 0; i < validators.length; i++) {
                validator = validators[i];
                if (
                    !sentReveal(currentRound, validator) ||
                    revealSkips(stakingEpoch, validator) > maxRevealSkipsAllowed
                ) {
                    // Mark the validator as malicious
                    maliciousValidators[maliciousValidatorsLength++] = validator;
                }
            }

            if (maliciousValidatorsLength > 0) {
                address[] memory miningAddresses = new address[](maliciousValidatorsLength);
                for (i = 0; i < maliciousValidatorsLength; i++) {
                    miningAddresses[i] = maliciousValidators[i];
                }
                validatorSetContract.removeMaliciousValidators(miningAddresses);
            }
        }

        // Clear unnecessary info about previous collection round.
        _clearOldCiphers(currentRound);
    }

    // =============================================== Getters ========================================================

    /// @dev Returns the length of the commits/reveals phase which is always half of the collection round length.
    function commitPhaseLength() public view returns(uint256) {
        return collectRoundLength / 2;
    }

    /// @dev Returns the serial number of the current collection round.
    function currentCollectRound() public view returns(uint256) {
        return (_getCurrentBlockNumber() - 1) / collectRoundLength;
    }

    /// @dev Returns the number of the first block of the current collection round.
    function currentCollectRoundStartBlock() public view returns(uint256) {
        return currentCollectRound() * collectRoundLength + 1;
    }

    /// @dev Returns the cipher of the validator's number for the specified collection round and the specified validator
    /// stored by the validator through the `commitHash` function.
    /// For the past collection rounds the cipher is empty as it's erased by the internal `_clearOldCiphers` function.
    /// @param _collectRound The serial number of the collection round for which the cipher should be retrieved.
    /// @param _miningAddress The mining address of validator.
    function getCipher(uint256 _collectRound, address _miningAddress) public view returns(bytes memory) {
        uint256 poolId = validatorSetContract.idByMiningAddress(_miningAddress);
        return _ciphers[_collectRound][poolId];
    }

    /// @dev Returns the Keccak-256 hash of the validator's number for the specified collection round and the specified
    /// validator stored by the validator through the `commitHash` function. Note that for the past collection rounds
    /// it can return empty results because there was a migration from mining addresses to staking addresses.
    /// @param _collectRound The serial number of the collection round for which the hash should be retrieved.
    /// @param _miningAddress The mining address of validator.
    function getCommit(uint256 _collectRound, address _miningAddress) public view returns(bytes32) {
        uint256 poolId = validatorSetContract.idByMiningAddress(_miningAddress);
        return _commits[_collectRound][poolId];
    }

    /// @dev Returns the Keccak-256 hash and cipher of the validator's number for the specified collection round
    /// and the specified validator stored by the validator through the `commitHash` function.
    /// For the past collection rounds the cipher is empty. Note that for the past collection rounds
    /// it can return empty results because there was a migration from mining addresses to staking addresses.
    /// @param _collectRound The serial number of the collection round for which hash and cipher should be retrieved.
    /// @param _miningAddress The mining address of validator.
    function getCommitAndCipher(
        uint256 _collectRound,
        address _miningAddress
    ) public view returns(bytes32, bytes memory) {
        uint256 poolId = validatorSetContract.idByMiningAddress(_miningAddress);
        return (_commits[_collectRound][poolId], _ciphers[_collectRound][poolId]);
    }

    /// @dev Returns a boolean flag indicating whether the specified validator has committed their number's hash for the
    /// specified collection round. Note that for the past collection rounds it can return false-negative results
    /// because there was a migration from mining addresses to staking addresses (on xDai chain).
    /// Also, it intentionally returns a false-positive result during the commit phase when the specified
    /// mining address of a validator is about to be changed. This is needed to prevent committing hash/cipher pair
    /// by Ethereum node to guarantee that a new validator's node (with a new mining address) won't try to wrongly
    /// decrypt the cipher stored by the previous node (created by the private key of the previous mining address).
    /// @param _collectRound The serial number of the collection round for which the checkup should be done.
    /// @param _miningAddress The mining address of the validator.
    function isCommitted(uint256 _collectRound, address _miningAddress) public view returns(bool) {
        uint256 poolId = validatorSetContract.idByMiningAddress(_miningAddress);

        if (poolId != 0 && isCommitPhase()) {
            (uint256 requestPoolId,) = validatorSetContract.miningAddressChangeRequest();
            if (poolId == requestPoolId) {
                return true;
            }
        }

        return _commits[_collectRound][poolId] != bytes32(0);
    }

    /// @dev Returns a boolean flag indicating whether the current phase of the current collection round
    /// is a `commits phase`. Used by the validator's node to determine if it should commit the hash of
    /// the number during the current collection round.
    function isCommitPhase() public view returns(bool) {
        return ((_getCurrentBlockNumber() - 1) % collectRoundLength) < commitPhaseLength();
    }

    /// @dev Returns a boolean flag indicating if the `initialize` function has been called.
    function isInitialized() public view returns(bool) {
        return validatorSetContract != IValidatorSetAuRa(0);
    }

    /// @dev Returns a boolean flag indicating whether the current phase of the current collection round
    /// is a `reveals phase`. Used by the validator's node to determine if it should reveal the number during
    /// the current collection round.
    function isRevealPhase() public view returns(bool) {
        return !isCommitPhase();
    }

    /// @dev Returns a boolean flag of whether the `commitHash` function can be called at the current block
    /// by the specified validator. Used by the `commitHash` function and the `TxPermission` contract.
    /// @param _miningAddress The mining address of the validator which tries to call the `commitHash` function.
    /// @param _numberHash The Keccak-256 hash of validator's number passed to the `commitHash` function.
    function commitHashCallable(address _miningAddress, bytes32 _numberHash) public view returns(bool) {
        if (!isCommitPhase()) return false; // must only be called in `commits phase`

        if (_numberHash == bytes32(0)) return false;

        if (!validatorSetContract.isValidator(_miningAddress)) return false;

        if (isCommitted(currentCollectRound(), _miningAddress)) return false; // cannot commit more than once

        return true;
    }

    /// @dev Returns the number of the first block of the next (future) collection round.
    function nextCollectRoundStartBlock() public view returns(uint256) {
        uint256 currentBlock = _getCurrentBlockNumber();
        uint256 remainingBlocksToNextRound = collectRoundLength - (currentBlock - 1) % collectRoundLength;
        return currentBlock + remainingBlocksToNextRound;
    }

    /// @dev Returns the number of the first block of the next (future) commit phase.
    function nextCommitPhaseStartBlock() public view returns(uint256) {
        return nextCollectRoundStartBlock();
    }

    /// @dev Returns the number of the first block of the next (future) reveal phase.
    function nextRevealPhaseStartBlock() public view returns(uint256) {
        if (isCommitPhase()) {
            return currentCollectRoundStartBlock() + commitPhaseLength();
        } else {
            return nextCollectRoundStartBlock() + commitPhaseLength();
        }
    }

    /// @dev Returns a boolean flag of whether the `revealNumber` function can be called at the current block
    /// by the specified validator. Used by the `revealNumber` function and the `TxPermission` contract.
    /// @param _miningAddress The mining address of validator which tries to call the `revealNumber` function.
    /// @param _number The validator's number passed to the `revealNumber` function.
    function revealNumberCallable(address _miningAddress, uint256 _number) public view returns(bool) {
        return _revealNumberCallable(_miningAddress, _number);
    }

    /// @dev The same as the `revealNumberCallable` getter (see its description).
    /// The `revealSecretCallable` was renamed to `revealNumberCallable`, so this function
    /// is left for backward compatibility with the previous client
    /// implementation and should be deleted in the future.
    /// @param _miningAddress The mining address of validator which tries to call the `revealSecret` function.
    /// @param _number The validator's number passed to the `revealSecret` function.
    function revealSecretCallable(address _miningAddress, uint256 _number) public view returns(bool) {
        return _revealNumberCallable(_miningAddress, _number);
    }

    /// @dev Returns the number of reveal skips made by the specified validator during the specified staking epoch.
    /// @param _stakingEpoch The number of staking epoch.
    /// @param _miningAddress The mining address of the validator.
    function revealSkips(uint256 _stakingEpoch, address _miningAddress) public view returns(uint256) {
        uint256 poolId = validatorSetContract.idByMiningAddress(_miningAddress);
        return _revealSkips[_stakingEpoch][poolId];
    }

    /// @dev Returns a boolean flag indicating whether the specified validator has revealed their number for the
    /// specified collection round. Note that for the past collection rounds it can return false-negative results
    /// because there was a migration from mining addresses to staking addresses.
    /// @param _collectRound The serial number of the collection round for which the checkup should be done.
    /// @param _miningAddress The mining address of the validator.
    function sentReveal(uint256 _collectRound, address _miningAddress) public view returns(bool) {
        uint256 poolId = validatorSetContract.idByMiningAddress(_miningAddress);
        return _sentReveal[_collectRound][poolId];
    }

    // ============================================== Internal ========================================================

    /// @dev Removes the ciphers of all committed validators for the collection round
    /// preceding to the specified collection round.
    /// @param _collectRound The serial number of the collection round.
    function _clearOldCiphers(uint256 _collectRound) internal {
        if (_collectRound == 0) {
            return;
        }

        uint256 collectRound = _collectRound - 1;
        uint256[] storage poolIds = _committedValidators[collectRound];
        uint256 poolIdsLength = poolIds.length;

        for (uint256 i = 0; i < poolIdsLength; i++) {
            delete _ciphers[collectRound][poolIds[i]];
        }

        poolIds.length = 0;
    }

    /// @dev Used by the `revealNumber` function.
    /// @param _number The validator's number.
    function _revealNumber(uint256 _number) internal {
        address miningAddress = msg.sender;

        require(revealNumberCallable(miningAddress, _number));
        require(_getCoinbase() == miningAddress); // make sure validator node is live

        uint256 poolId = validatorSetContract.idByMiningAddress(miningAddress);

        currentSeed = currentSeed ^ _number;
        _sentReveal[currentCollectRound()][poolId] = true;
    }

    /// @dev Returns the current `coinbase` address. Needed mostly for unit tests.
    function _getCoinbase() internal view returns(address) {
        return block.coinbase;
    }

    /// @dev Returns the current block number. Needed mostly for unit tests.
    function _getCurrentBlockNumber() internal view returns(uint256) {
        return block.number;
    }

    /// @dev Used by the `revealNumberCallable` public getter.
    /// @param _miningAddress The mining address of validator which tries to call the `revealNumber` function.
    /// @param _number The validator's number passed to the `revealNumber` function.
    function _revealNumberCallable(address _miningAddress, uint256 _number) internal view returns(bool) {
        if (!isRevealPhase()) return false; // must only be called in `reveals phase`

        bytes32 numberHash = keccak256(abi.encodePacked(_number));

        if (numberHash == bytes32(0)) return false;

        if (!validatorSetContract.isValidator(_miningAddress)) return false;

        uint256 collectRound = currentCollectRound();

        if (sentReveal(collectRound, _miningAddress)) {
            return false; // cannot reveal more than once during the same collectRound
        }

        if (numberHash != getCommit(collectRound, _miningAddress)) {
            return false; // the hash must be commited
        }

        return true;
    }
}

```
