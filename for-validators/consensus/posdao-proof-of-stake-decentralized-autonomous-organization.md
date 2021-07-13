# POSDAO: Proof of Stake Decentralized Autonomous Organization

POSDAO describes the validator selection method for the xDai Stable Chain.  Validators provide consensus for xDai chain transactions.

Validators are selected based on the amount of STAKE they place into the protocol along with an on-chain RNG. The validator set is capped at 19, and validator candidates need to place minimums of 2K STAKE \(current minimum\) and setup a valid node to be eligible for participation. In addition, public delegators can place STAKE on candidates, increasing their chances of becoming validators in the next set. The validator set can change weekly based on the number of eligible validators and their staking amounts.

{% hint style="success" %}
[POSDAO Contract Implementation Addresses](https://github.com/poanetwork/poa-chain-spec/blob/dai/contracts.json#L9)
{% endhint %}

{% hint style="info" %}
Prior to Public POSDAO, Permissioned POSDAO was used nominated validators to sign blocks. xDai transitioned to public POSDAO in December, 2020.
{% endhint %}

