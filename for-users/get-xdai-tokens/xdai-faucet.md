---
description: Different ways to obtain small amounts of xDai
---

# xDai Faucets

## OmniBridge Faucet

The OmniBridge faucet now automatically deposits a small amount of xDai ($0.01) to users bridging tokens from Ethereum or BSC. This is enough to start making transactions. If more is needed, bridged tokens can be [swapped on a DEX](../../about-gc/project-spotlights/#defi) for additional xDai. Only empty wallets (non-contract wallets with a 0 xDai balance) receive xDai when bridging.&#x20;

Example transaction: [https://blockscout.com/poa/xdai/tx/0x4f071fda836ca2fb50854c80851da7040776700247e22f9c9f99d980e74eb527](https://blockscout.com/poa/xdai/tx/0x4f071fda836ca2fb50854c80851da7040776700247e22f9c9f99d980e74eb527)

Change raw input field from HEX to UTF-8 to view the corresponding note.&#x20;

![](../../.gitbook/assets/omni-faucet.png)

## BlockScout Faucet

Faucet is now live with SMS phone verification through Twilio\* and multiple hCaptcha prompts to prevent abuse. Virtual numbers are not accepted for faucet transactions.

{% hint style="info" %}
If the faucet is empty there is no time table for a refill. You can use the [OmniBridge faucet ](xdai-faucet.md#blockscout-faucet)when bridging, try the [3rd party faucets](xdai-faucet.md#3rd-party-faucets) below, or [obtain xDai in other ways](../getting-started-with-xdai/#2-get-a-little-xdai).
{% endhint %}

### Instructions

1\) Go to [https://blockscout.com/xdai/mainnet/faucet](https://blockscout.com/xdai/mainnet/faucet)

1. Enter an `0x...` address where you will receive xDai
2. Enter a valid phone number where you can receive a SMS text message. Be sure to select the correct country with the flag icon.
3. Complete the hCaptcha process.
4. Click **Send SMS.**

![](../../.gitbook/assets/f1.png)

2\) You will be forwarded to the next screen and should receive a text message with a 6 digit code.

1. Enter the 6-digit verification code.
2. Complete the hCaptcha.
3. Click **Request 0.01 xDai**

![](../../.gitbook/assets/f2.png)

You will see a success message. Check your balance on the xDai chain to see that that you received funds from the faucet (See [getting started with xDai](../getting-started-with-xdai/) if you have questions around connecting to the xDai chain).

3\) It is also possible to donate to the faucet and help your community! Click the **Donate button** and complete the transaction in MetaMask to add xDai to the faucet. Donations are automatically set to 10 xDai.

![](../../.gitbook/assets/f3.png)

4\) You are ready to start transacting! You can acquire more xDai directly using a DEX like [HoneySwap](https://honeyswap.org) without paying high gas fees to bridge Dai from Ethereum.

{% hint style="info" %}
The xDai Faucet can be used by an address once every 24 hours to top-off your xDai balance.
{% endhint %}

## 3rd Party Faucets

A number of additional xDai faucets have been created by projects to assist with xDai onboarding.

* [BAO Community xDai Faucet](https://xdai-app.herokuapp.com/faucet)
* [Minerva Wallet Faucet ](https://minerva.digital)(scroll down page to see, must have MIVA to use)

If you've gotten this far and still haven't received any xDai, try the [xDai Discord](https://discord.gg/mPJ9zkq) #faucet-requests channel!
