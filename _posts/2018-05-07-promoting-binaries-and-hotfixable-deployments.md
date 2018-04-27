---
layout: post
title: ! Promoting Binaries and Hotfixable Deployments
published: false
---

There are a two different schools of thought when it comes to deploying to production environments. Well okay, we're developers, so there are probably 100 different schools of thought but bear with me. One option is to promote the same binaries from testing, through staging, and all the way to production, and the other is to maintain a branch in your source repository for the current state of production, and deploy from that. The general thinking is that with the former you get safety in knowing that your production deployments is _exactly_ what has been through your testing cycles, and with the latter you're always in a position to hotfix and correct a production issue regardless of what state your testing branch might be in.

Fortunately its an argument that can be avoided, and instead you can set up an environment where you get the best of both worlds: Predictable results from promoting binaries to production and an insurance policy in case you need to hotfix. I am using [TeamCity](https://www.jetbrains.com/teamcity/) and [Octopus Deploy](https://www.jetbrains.com/teamcity/) to do this but the ideas are the same no matter what technology you use.

## Commit hashes are important

One of the best pieces of advice I have for anyone setting up any kind of CI/CD, automation, devops workflow is to get your commit hashes into your binaries and packages as early and as often as possible. Having a known identifier that can track binaries and directly correlate them to source code is invaluable for all sorts of things, but in this case its especially important so that the build server and deployment packages know what each other is talking about.

To get commit hashes into your build output in TeamCity is as straightforward as configuring a setting on the build in question. The "Build number format" setting dictates how TeamCity should format build numbers in its output, and also the format of the `%build.number%` variable that you can use in, or pass in to, scripts and build steps. The normal approach for a build number would be something like `1.0.%build.counter%`, where the major and minor versions are hardcoded to 1.0, and the build number increments automatically with every build. Personally I'm a fan of using something like GitVersion to allow the number of commits to be used instead of the build number, as it allows resiliency across build server reinstalls, but thats for another discussion.

Tagging the commit hash on the end is done by adding a hyphen after the build counter, and then inserting the commit hash. In TeamCity this is the `%vcs.commit.number%` variable, so our full build number format is as follows.

```
1.0.%build.counter%-%vcs.commit.number%
```

This will give a build number something like `1.0.134-770ac6d169006ce42b5bbc022a6a166135bbe8a7`.