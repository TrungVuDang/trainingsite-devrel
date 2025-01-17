---
title: Use the Tokens SDK
description: Get acquainted with the SDK
slug: how-to-use-tokens-sdk
aliases: [
  "/libraries/how-to-use-tokens-sdk/"
]
menu:
  main:
    parent: token-sdk
    weight: 40
weight: 40
---

## Introduction

In the previous chapter, you discovered the core concepts of the Tokens SDK. In this chapter you will learn to use the SDK by way of example. The code samples in this chapter are used in place in a dedicated unit test file, [`UsdTokenCourseExercise`](https://github.com/corda/corda-training-code/blob/master/030-tokens-sdk/workflows/src/test/java/com/template/usd/UsdTokenCourseExercise.java), and [3 flows](https://github.com/corda/corda-training-code/tree/master/030-tokens-sdk/workflows/src/main/java/com/template/usd) to avoid any doubt about the environment in which the samples can be used.

The SDK comes with flows to `Issue`, `Move`, and `Redeem` tokens. Those flows come in fungible, non-fungible, confidential, inlined, and initiating versions. Review the details here and then delve into the exercise in the next chapter.

You are going to advance through 4 overall steps per this list:

1. Include the SDK in your project.
2. `Issue` 100$ to Alice.
3. `Move` 50$ from Alice to Bob.
4. Have Bob `Redeem` 25$.

Let's start:

## Include the SDK in your project

1. Add the release group and version inside the **project** root `build.gradle`, in the `ext` configuration block:

    ```groovy
    tokens_release_version = '1.1'
    tokens_release_group = 'com.r3.corda.lib.tokens'
    confidential_id_release_version = '1.0'
    confidential_id_release_group = 'com.r3.corda.lib.ci'
    ```
    **If** you are working with the course's repository, you may want to update the `constants.properties` file instead:
    
    ```properties
    tokensReleaseVersion=1.1
    ...
    ```
    And update your `build.gradle` in `010-empty-project` in the same fashion as the other parameters are collected:
     
    ```groovy
    tokens_release_version = constants.getProperty("tokensReleaseVersion")
    ...
    ```

2. Add the repositories inside the root `build.gradle`:

    ```groovy
    repositories {
       maven { url 'https://software.r3.com/artifactory/corda-lib' }
       maven { url 'https://software.r3.com/artifactory/corda-lib-dev' }
    }
    ```

3. Add the necessary `dependencies` in the respective modules' `build.gradle` files. Only add `tokens-money` if you plan on using `FiatCurrency` and `DigitalCurrency` token type definitions:

    ```groovy
    // For contracts
    cordapp "$tokens_release_group:tokens-contracts:$tokens_release_version"
    cordapp "$tokens_release_group:tokens-money:$tokens_release_version" // Optional

    // For workflows
    cordapp "$tokens_release_group:tokens-workflows:$tokens_release_version"
    cordapp "$tokens_release_group:tokens-selection:$tokens_release_version"
    cordapp "$confidential_id_release_group:ci-workflows:$confidential_id_release_version"
    ```

For the next steps, the code examples shown below are actually _rootless_. When you go to the repo itself, you will see that they are split into 3 parts:

1. A somewhat generic flow.
2. A convenience function that calls the flow with specific values.
3. A test function that confirms it got it right.

## Let the US mint issue

Now, on the US Mint's node, within a flow's `call` function, [`Issue`](https://github.com/corda/corda-training-code/blob/master/030-tokens-sdk/workflows/src/main/java/com/template/usd/IssueUsdFlow.java) &#36;100 to [Alice](https://github.com/corda/corda-training-code/blob/master/030-tokens-sdk/workflows/src/test/java/com/template/usd/UsdTokenCourseExercise.java#L46-L51):

```java
// Prepare what we are talking about.
final TokenType usdTokenType = new TokenType("USD", 2);
// Identify the issuer.
if (!getOurIdentity().getName().equals(CordaX500Name.parse("O=US Mint, L=Washington D.C., C=US"))) {
    throw new FlowException("We are not the US Mint");
}
final IssuedTokenType usMintUSD = new IssuedTokenType(getOurIdentity(), usdTokenType);

// Who is going to own the output, and how much?
// Create a 100$ token that can be split and merged.
final Amount<IssuedTokenType> oneHundredUSD = AmountUtilitiesKt.amount(100L, usMintUSD);
final Party alice = getServiceHub().getNetworkMapCache().getPeerByLegalName(
        CordaX500Name.parse("O=Alice, L=New York, C=US"));
final FungibleToken usdToken = new FungibleToken(oneHundredUSD, alice, null);

// Issue the token to Alice.
final SignedTransaction issueTx = subFlow(new IssueTokens(
        Collections.singletonList(usdToken), // Output instances
        Collections.emptyList())); // Observers
```
With its [`@Test mintIssues100ToAlice`]https://github.com/corda/corda-training-code/blob/master/030-tokens-sdk/workflows/src/test/java/com/template/usd/UsdTokenCourseExercise.java#L46-L51). By the way, the `tokens-money` module (which comes with the SDK), has 2 predefined token types: `FiatCurrency` and `DigitalCurrency`, so you can do something like this:

```java
// US Dollar token type.
final TokenType usdTokenType = FiatCurrency.Companion.getInstance("USD");
// Ripple token type.
final TokenType rippleTokenType = DigitalCurrency.Companion.getInstance("XRP");
// Other predefined digital currency types are: BTC, ETH, DOGE.
```

## Let Alice move some to Bob

Then, on Alice's node, [`Move`](https://github.com/corda/corda-training-code/blob/master/030-tokens-sdk/workflows/src/main/java/com/template/usd/MoveUsdFlow.java) &#36;50 from [Alice to Bob](https://github.com/corda/corda-training-code/blob/master/030-tokens-sdk/workflows/src/test/java/com/template/usd/UsdTokenCourseExercise.java#L55-L60):

```java
// Move 50$ that were issued by the US Mint and held by Alice to Bob.
// Prepare what we are talking about.
final TokenType usdTokenType = FiatCurrency.Companion.getInstance("USD");
final Party usMint = getServiceHub().getNetworkMapCache().getPeerByLegalName(
        CordaX500Name.parse("O=US Mint, L=Washington D.C., C=US"));
if (usMint == null) throw new FlowException("No US Mint found");

// Who is going to own the output, and how much?
final Party bob = getServiceHub().getNetworkMapCache().getPeerByLegalName(
        CordaX500Name.parse("O=Bob, L=New York, C=US"));
final Amount<TokenType> fiftyUSD = AmountUtilitiesKt.amount(50L, usdTokenType);
final PartyAndAmount<TokenType> fiftyUSDForBob = new PartyAndAmount<>(bob, fiftyUSD);

// Describe how to find those $ held by Alice.
final QueryCriteria issuedByUSMint = QueryUtilitiesKt.tokenAmountWithIssuerCriteria(usdTokenType, usMint);
final QueryCriteria heldByAlice = QueryUtilitiesKt.heldTokenAmountCriteria(usdTokenType, alice);

// Do the move
final SignedTransaction moveTx = subFlow(new MoveFungibleTokens(
        Collections.singletonList(fiftyUSDForBob), // Output instances
        Collections.emptyList(), // Observers
        issuedByUSMint.and(heldByAlice), // Criteria to find the inputs
        alice)); // change holder
```
With its [`@Test bobGot50Dollars`](https://github.com/corda/corda-training-code/blob/master/030-tokens-sdk/workflows/src/test/java/com/template/usd/UsdTokenCourseExercise.java#L84-L101). Note how the last parameter is explicit: `changeHolder = alice`. Why did it not rely on the [default value of `getOurIdentity()`](https://github.com/corda/token-sdk/blob/b9a1ee76434defd0b234df05c972202c7f1a2a5c/workflows/src/main/kotlin/com/r3/corda/lib/tokens/workflows/flows/move/MoveFungibleTokensFlow.kt#L50)?

* It is best to not leave anything to chance.
* When you are introduced to the _Accounts Library_, this default will get messy.

So better keep this as a habit.

For the avoidance of doubt, the _change holder_ is Alice. In this case, _change_ is meant as in "Do you have _change_ on a &#36;20 bill?" It is not meant as the verb to change hands. What Bob receives is expressed in the `Collections.singletonList(fiftyUSDForBob), // Output instances` part.

## Let Bob redeem some

Then, on Bob's node, let [Bob](https://github.com/corda/corda-training-code/blob/master/030-tokens-sdk/workflows/src/main/java/com/template/usd/RedeemUsdFlow.java) [`Redeem` &#36;25](https://github.com/corda/corda-training-code/blob/master/030-tokens-sdk/workflows/src/test/java/com/template/usd/UsdTokenCourseExercise.java#L64-L69):

```java
// Prepare what we are talking about.
final TokenType usdTokenType = FiatCurrency.Companion.getInstance("USD");
final Party usMint = getServiceHub().getNetworkMapCache().getPeerByLegalName(
    CordaX500Name.parse("O=US Mint, L=Washington D.C., C=US"));
if (usMint == null) throw new FlowException("No US Mint found");

// Describe how to find those $ held by Me.
final QueryCriteria heldByBob = QueryUtilitiesKt.heldTokenAmountCriteria(usdTokenType, bob);
final Amount<TokenType> twentyFiveUSD = AmountUtilitiesKt.amount(25L, usdTokenType);

// Do the redeem
final SignedTransaction redeemTx = subFlow(new RedeemFungibleTokens(
        twentyFiveUSD, // How much to redeem
        usMint, // issuer
        Collections.emptyList(), // Observers
        heldByBob, // Criteria to find the inputs
        bob)); // change holder
```
With its [`@Test bobGot25DollarsLeft`](https://github.com/corda/corda-training-code/blob/master/030-tokens-sdk/workflows/src/test/java/com/template/usd/UsdTokenCourseExercise.java#L104-L120). Note that this time it only created a `QueryCriteria heldByBob` and did not need to create a `QueryCriteria issuedByUSMint` as this is done automatically for us in the [detail of the implementation](https://github.com/corda/token-sdk/blob/b9a1ee76434defd0b234df05c972202c7f1a2a5c/workflows/src/main/kotlin/com/r3/corda/lib/tokens/workflows/flows/redeem/RedeemFlowUtilities.kt#L106-L107).

## Conclusion

You have seen how to include the SDK into your project, and how to do the 3 basic operations that your more complex flows would do as part of a CorDapp.