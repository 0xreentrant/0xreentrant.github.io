---
layout: distill
title: Don't let mocks in your protocol tests fool you
date: 2023-10-19
description: The biggest threat to developing a protocol is the mental model 
tags: 
categories: 
toc:
  - name: Our case
  - name: Mocks have tradeoffs  
  - name: But what are the consequences?
  - name: How can I ensure our mocks are correct?
  - name: What do these possibilities look like?
  - name: Smoke tests
  - name: Manual testing on a testnet
  - name: Forking tests
  - name: Live Integration Tests
  - name: Conclusion
---

# Don't let mocks in your protocol tests fool you

> Note: This article was written for web3 protocol developers.  Security researchers will also get a little glimpse into techniques for testing protocols.

*Mocking expensive, slow, or complex-to-set-up components is a great way to make testing faster, cheaper, and simpler to leverage.  What can happen though, is that the expectations of the mock can diverge from the expectations of the real-world object. When that happens, you can be oblivious to the fact your codebase does not, in fact, work, green checkmarks be damned.*

*The biggest threat to developing a protocol is the mental model of the protocol within the mind of the developer. A flawed mental model happens from numerous sources, but today we'll focus on a single one: the feeling of security that comes with passing tests*

> Note: I'm using the definition of "mock" to be along the lines of how Martin Fowler uses it here: https://martinfowler.com/articles/mocksArentStubs.html

**TL;DR** If you want to skip to what kind of things you can do to prevent mocks from fooling you, jump to [How can I ensure our mocks are correct?](#how-can-I-ensure-our-mocks-are-correct)

## Our case

Imagine getting green checkmarks on all your tests.  You've gone ahead and built a suite to exercise the entire happy path of your codebase, and even some edge cases to boot.  And even better: every test passed.  It's a good time to grab a beer or wine and take a little break, because next you're going to discover something unsavory when you try to run through the deployed example.  

{% include figure.html path="assets/img/all-tests-green.png" class="img-fluid" %}

> Note: Before you hire a firm or to audit your code, do make sure to fully deploy your code to a testnet with all 3rd-party contracts and features available. For example: testing cross-chain protocols

Cutting to the chase, you're going to "Oh no, what's wrong, I coded this *perfectly* - all the tests pass!" And you're going to feel so right, because your mental model was impeccable and your tests reflected that... *but your mocks fooled you*.  Actually, it's just a case of a "[false positive](https://en.wikipedia.org/wiki/False_positives_and_false_negatives)" (lest we lose our last shred of sanity and [anthropomorphize](https://psychcentral.com/health/why-do-we-anthropomorphize) our code).  Green checks that don't give confidence are the antithesis of any testing methodology, so let's break down what to keep in mind.

## Mocks have tradeoffs

When we use mocks, we're taking a stance. The two main ones I hear and have held are:

1. Our dependencies are too expensive, slow, or complex to spin up each time we test (*fair!*)
2. We want our tests to run quickly, and we can't wait for each test iteration to fully pass or well lose out on momentum (*fair!*)

### But what are the consequences?

These tradeoffs, again, are fair to make when the stakes are low, like:

- When you're building the protocol, pre-launch
- When you're adding to an existing protocol and need the flexibility to refactor highly-coupled parts of a system
- When you're exploring how to implement a new system 

Etc.

But in the case of running a web3, immutable, smart-contract-based, financial protocol the stakes are high. As protocol developers in a hyper-competitive environment with close to zero consequences for exploitation by psuedo-anonymous actors that include globally-sanctioned blackhats, more care must be taken before accepting real people's assets. I'll let our industry's favorite rug-pull [journalist](https://rekt.news/stake-rekt/) say it best:

They say the house always wins. Not in crypto.

## How can I ensure our mocks are correct?

"But reentrant, we read the documentation front-to-back and everything is working just like they said!" 

I 100% get it: you've put in the work.  

But, what if:

- The documentation is wrong
- The implementation has changed
- You used the wrong flag for a key function while trying to get that last PR in at 3am Saturday morning

Then things break. 

What do you do? Add *at least* one more step, especially before deploying your new protocol.

1. Create "smoke-tests" - as in short, typically disposable scripts that call major 3rd-party components the same as the mocks would be called.
3. Deploy on a fully-functional testnet and run manual tests for all major features.
4. Build "forking tests" that fork a live network locally and test against it. Some dependencies may not work if they are off-chain.
5. Build integration tests for all major features, deploy on a fully-functional testnet, and test against it.

< decision tree for determinnig when mocks fail >
## What do these possibilities look like?

There's nothing like a real environment to expose the brittleness of a test suite's assumptions. While it might seem unnecessary "because it's there in the code," the unique case of web3 warrants the extra paranoia.
### Smoke tests

This one makes sense when your contracts interface with 3rd-party code. Each feature that touches a 3rd-party contract is either triggered in a script or run manually. If the feature breaks, but your mocking tests were all passing, there is an issue with your mock.

{% include figure.html path="assets/img/smoke_tests.png" class="img-fluid" %}

For example: imagine you are a protocol developer lead that wants to test that each feature that is mocked works.  You would:

1. Identify all the features that touch 3rd-party code.
2. Write a script to exercise the 3rd-party contract code for each relevant feature, according to how you expect the API of that code to work.
3. Run your script.
4. Make a note if something broke, and how it broke

You will find that increased logging through events or console logs will be useful when creating your scripts.

We won't go much deeper on the subject of smoke tests in web3, but here's a great article for learning more about smoke tests: https://www.edureka.co/blog/what-is-smoke-testing/
### Manual testing on a testnet

This one is pretty obvious in operation. The goal here is to make sure every feature that interfaces with 3rd-party components has been run, along with some edge-cases that have been identified.  That way you are exercising whatever assumptions your code is making about the interface of code you don't control.  

The advantage to running these on an existing testnet is that any off-chain systems your preferred 3rd-party system uses will be more or less accessible.  A downside would be the necessity of amassing gas tokens for the respective testnet.  

{% include figure.html path="assets/img/manual-tests.png" class="img-fluid" %}

Similarly to running smoke tests, the point is to get an actual look at whether any assumptions encoded in the codebase, and your mocks, match the real-world requirements of any 3rd-party component.  

Doing this yourself:
1. Identify all the features that touch 3rd-party code.
2. Perform all the operations in your app to exercise the 3rd-party contract code for each relevant feature, according to how you expect the API of that code to work.
3. Make a note if something broke, and how it broke

If any of your mocking tests passed, but manual testing fails where your tests had mocks, then that's an unmistakable sign something is broken in your codebase *and* tests.
### Forking tests

When there is no need to monitor the results of any off-chain systems, you can fork any live chain and run a version of it locally for more control and execution speed. This also is advantageous for the off-chance that your 3rd-party of choice decides not to run a version on a testnet.  

{% include figure.html path="assets/img/forking-tests.png" class="img-fluid" %}

Unlike the smoke tests and the manual tests, we are back to writing automated tests.  We can use our existing testing suites, with the modification that now any mocked calls will instead call actual contracts hosted on the forked testnet.  That way any assumptions that mocks encode will be instead tested against actual code.  

A disadvantage of this approach is that the tests themselves will likely take longer to run each cycle compared to the mocking tests.  However, compared to smoke tests and manual testing, you'll get the increased coverage and repeatability of automated testing along with the better assurance of correctness.
### Live Integration Tests

The most intensive version of testing that would shake out any assumptions about 3rd-party interfaces in your codebase would be integration tests on a live testnet.  All components are deployed to the testnet, mocks in tests are replaced with calls to actual on-chain resources, all tests run against the live testnet, and off-chain resources are available to monitor.  You will need any testnet gas tokens for every run of the test suite.  

{% include figure.html path="assets/img/integration-tests.png" class="img-fluid" %}

Every test will exercise live code as close to the final, real-world software environment possible.  Any tests that fail here will give a lot of information about assumptions encoded in your codebase.  Because there will be subtle differences between a local fork and an actual live chain, and the cost of running the test suite (in time and testnet gas) it will be helpful to monitor the results of the tests with other tools such as chain explorers like [Tenderly](https://tenderly.co/) for delving into any edge cases that the other techniques did not uncover.

## Conclusion

We can see that mocks are helpful, but they have tradeoffs.  To counter the possibility that mocks will give us false-positives, we can increase the assurances by running tests against actual 3rd-party code.  

In web3, it is imperative that all precautions are made to ensure real money isn't lost to thefts or mishandling of digital currency.  Test more, test often, and test comprehensively.
