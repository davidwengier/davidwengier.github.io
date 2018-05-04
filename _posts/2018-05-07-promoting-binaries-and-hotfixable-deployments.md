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

Tagging the commit hash on the end is done by adding a hyphen after the build counter, and then inserting the commit hash. In TeamCity this is the `%build.vcs.number%` variable, so our full build number format is as follows.

```
1.0.%build.counter%-%build.vcs.number%
```

This will give a build number something like `1.0.134-770ac6d169006ce42b5bbc022a6a166135bbe8a7`. Success in that we have the commit hash in the build number, but its a bit ugly and unnecessarily long. You only need around 7 or 8 characters to be unique for most repos (the Linux Kernel is starting to need 12 but they have hundreds of thousands of commits) so I like to shorten the hash down a bit. Doing this in TeamCity is a little un-intuitive as there are no operations that can be performed in the simply macro language you use to specify the build format. To change the build number you need to use a build step and use the TeamCity feature called [service messages](https://confluence.jetbrains.com/display/TCD10/Build+Script+Interaction+with+TeamCity#BuildScriptInteractionwithTeamCity-servMsgsServiceMessages), which is a standard pre-defined structure of output, written to standard output, that TeamCity will pick up and process. I've done this with a short PowerShell script as the first step in each build I define.

```powershell
Write-Host "Old build number was: %build.number%"
$buildNumber = "1.0.%build.counter%"
$shortHash = "%build.vcs.number%"
$shortHash = $shortHash.substring(0, 10)
$buildNumber = "$buildNumber-$shortHash"
Write-Host "New build number is: $buildNumber"
Write-Host "##teamcity[buildNumber '$buildNumber']"
```

My script is overly long because of the debugging output but I find build logs verbose enough already so keeping a couple of lines out of it isn't worth worrying about. Strictly speaking the whole script could be a one-liner.

```powershell
Write-Host "##teamcity[buildNumber '1.0.%build.counter%-$("%build.vcs.number%".substring(0, 10))']"
```

I'm keeping the short has at 10 characters for no good reason, you could easily change that to whatever you desire. Its worth noting that with this as the first step of the build plan the "Build number format" setting has been rendered effectively useless for all but the first few seconds of the build, until it runs this script. With the script in blame the build number will now be `1.0.134-770ac6d169`.

## Pass hashes through to Octopus

Now that we have our short build number its important to use that in the version number for any package pushed to Octopus, and the release made from those packages. This gives full traceability from git commit, to build, through to deployment. If you also use something like [NerdBank.GitVersioning](https://github.com/AArnott/Nerdbank.GitVersioning) you can tag your DLLs with the same commit hash, which means you can also include it in your application logs or audit tracking.

With the version number in the package being deployed in Octopus it means we can now create a powershell script, and put it in the process for a production deployment. That script  fast forward the master branch to the specific commit that has been deployed, guaranteeing that the master branch will point at exactly where the develop branch was at when that package was built.

```powershell
$hash = "$($OctopusParameters["Octopus.Release.Number"])".Split("-")[1]

Write-Host "Fetching latest origin just to be sure"
git fetch origin
Write-Host "Resetting to current master"
git reset origin/master --hard
Write-Host "Fast forwarding to $hash"
git merge $hash --ff-only
Write-Host "Pushing back to origin"
git push origin
```

The script needs to be run in a git working copy and assumes master is checked out, though that could be added easily enough. I could have reset to the specific commit and just pushed that, but I like the extra protection that `--ff-only` provides. It ensures that if anything goes wrong with the working copy, or the script gets run at an incorrect time, there at least won't be any commits lost that will require navigating the reflog for. There might be a better way to achieve this, or perhaps that worry is for nothing, but I don't profess to be a git expert.

## Hotfixes are now just another build

Now that master is at the point of the deployed production build, hotfix branches can be created from, and merged back into, the master branch, which can then be built and deployed with the normal build and deployment process knowing that any changes that have been made to the develop branch will not be included. In an ideal world develop remains deployable and this process isn't needed, but an insurance policy is a good idea and in this case, cheap to have.

In my case I've set up a separate build on TeamCity for the master branch that is not automatically triggered, and considering each production deploy will change the master branch thats advisable. Additionally this build releases on a hotfix channel in Octopus so that it can deploy direct to staging, avoiding test. This way test still maps to the develop branch so that process isn't interrupted. 