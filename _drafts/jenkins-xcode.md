---
layout: post
title: Jenkins, Pipelines, Xcode, and Us.
---
# Setting up a cross-platform Jenkins installation at Push Interactions

## Frustration In The Past
I have been trying to get our apps onto a Jenkins machine for years. Most of the time, I have run into roadblocks that centred around the iOS Simulator, which has some fairly stringent requirements:
- The iOS Simulator must be run in a GUI session
- You can't run multiple simulators at once

## All Together With Android
The most recent attempt to get this going at work was in conjunction with an Android colleague. He was really interested in getting our Android projects up and running, so he drove a lot of the advancements.

## Enter Xcode 9
With Xcode 9, the good folks on the Simulator team at Apple have changed things considerably. 
- Multiple Simulators can be run concurrently
- Simulators no longer require a GUI Session ( <- This is very exciting.)

## This time, for sure!
### Jenkins Installed
There are a number of paths to installing Jenkins. Unfortunately, the method of installation matters for iOS projects. I mentioned the simulator needing a GUI session earlier, and you get bitten quite a bit on this point if you don't install Jenkins in the correct manner.
> Specifically, you have to install Jenkins as the logged-in user.

If you don't, then when it comes time to run your tests, nothing will work. This is because you'll run `xcodebuild test`, and nothing will happen. 

I installed Jenkins using homebrew. I loosely followed this post: 
[Installing Jenkins on OSX Yosemite](https://nickcharlton.net/posts/installing-jenkins-osx-yosemite.html)

While Yosemite is getting a little long-in-the-tooth, the steps I used on El Capitan were largely the same.

I ignored the bits about creating a new jenkins user, as we have a CI user on the machine. I installed Jenkins using homebrew, and created a new plist in the ~/Library/LaunchAgents folder.


> #### A segue on launchd
> It's worth noting that I created the plist in LaunchAgents, and not LaunchDaemons. This is because macOS services use different strategies for plists in each folder.
> - LaunchDaemons are run 'headless', ie without a GUI session
> - LaunchAgents are run as the logged-in user. Essentially, they are run **on your behalf** when you log in.

## Pipelines are different

These days, Jenkins is pushing us towards a new UI called 'Blue Ocean'. It looks great, and brings with it a radical shift in how Jenkins handles setting up a project. 

In the past, we'd use the Jenkins plugin architecture and configure our project using the provided configuration web interface.

Pipelines and Blue Ocean eschew the web-interface in favour of the Jenkinsfile: A file containing a Groovy-based DSL that allows us to define our projects completely. A Jenkinsfile also allows us to split up our build into parallel steps.

### The Basic Pipeline

Here's my basic Jenkinsfile:
{% highlight groovy %}
node('GUISession') {
    withEnv(["DEVELOPER_DIR=/Applications/Xcode-beta.app/Contents/Developer"]) {
        stage('Checkout/Build/Test') {
            checkout scm
            sh "" +
            "source /Users/canary/.bash_profile\n" +
            "bundle install\n" +
            "bundle update\n" +
            "launchctl remove com.apple.CoreSimulator.CoreSimulatorService || true\n" +
            "open -b com.apple.iphonesimulator\n" +
            "set -o pipefail && /usr/bin/xcodebuild -scheme 'ViewPoint' -workspace ViewPoint.xcworkspace -destination 'name=iPad Air,OS=latest' -enableCodeCoverage YES -configuration Debug clean build analyze test | XCPRETTY_JSON_FILE_OUTPUT=xcodebuild.json xcpretty -r junit -f `xcpretty-json-formatter`\n"
            
            step([$class: 'JUnitResultArchiver', allowEmptyResults: true, testResults: 'build/reports/junit.xml'])
        }
    }
}
{% endhighlight %}

Let's step through some of the highlights:

`node('GUISession')`

`GUISession` is a label I've applied to the "master" Node on my Jenkins instance. This is the only one that will consistently be able to fire up a Simulator and run the tests.

`withEnv([DEVELOPER_DIR=...])`

Did you know you can specify **which** version of Xcode that Jenkins runs with `xcodebuild`? You just have to specify an environment variable. Normally on the command line you'd do this with a call like: `export DEVELOPER_DIR=[your Xcode Developer folder]`. In your Jenkins file, you do it by wrapping your code in a `withEnv` block.

`sh "" + ....`

This allows me to chain a whole bunch of shell scripts together. This is needed, because each call to `sh` within the Jenkinsfile has its own context. So when I run `bundle install + upgrade` earlier on, I need those ruby gems to exist when I call my `xcodebuild` later. Otherwise (for example), I wouldn't be able to pipe the output to `xcpretty` and get a nicely formatted log.

`XCPRETTY_JSON_FILE_OUTPUT=xcodebuild.json xcpretty -r junit -f 'xcpretty-json-formatter'`

I'm piping the output of xcodebuild to xcpretty, which outputs everything into a json file with xcpretty-json-formatter. This means that all test/analyze issues found by Xcode will be in a nice, easy-to-consume list.

### Post results to Bitbucket PR

Our Android colleagues have the ability to post to our Pull Requests with issues found during the build process, but I want to go a little further. I know that Bitbucket Server/Cloud allows you to use a REST API call to post comments to individual files/lines. That's what I want for our app.

I want for warnings and linter issues to show up **inline** in the PR. In order to get that, I have to use the Bitbucket Server API.

The idea is that for each entry in the `xcodebuild.json` file, I'll create a new comment on the Pull Request. 

So I've added the following to the stage:

{% highlight groovy %}
def fileContents = readFile "xcodebuild.json"
def buildLog = new JsonSlurperClassic().parseText(fileContents)
Integer branchId = getBranchId()
buildLog.compile_warnings.each {
    postComment(projectKey, repo, branchId, it.line+'\n'+it.cursor)
}
{% endhighlight %}

We have to use `JsonSlurperClassic` because the newer `JsonSlurper` isn't thread-safe. [Or something like that.](https://stackoverflow.com/questions/37864542/jenkins-pipeline-notserializableexception-groovy-json-internal-lazymap/38439681#38439681)

