---
title: "Using cache and incremental builds in real world scenarios"
created_at: Fri Jul 16 2021 15:00:00 EDT
author: Montana Mendy
layout: post
permalink: 2021-07-16-incbuilds
category: news
excerpt_separator: <!-- more --> 
tags:
  - news
  - feature
  - infrastructure
  - community
---

![TCI-Graphics for AdsBlogs (1)](https://user-images.githubusercontent.com/20936398/125991579-9ef67346-70cc-461b-9f31-624c2d3c3276.png)

We all want shorter builds, what are some ways we can accomplish that? In this post I'll give you a variety of ways to help reduce build times, and overall optimization of your `.travis.yml` file. We'll talk about caching and the incremental build method in Travis CI. In general and the majority, short build durations are associated with Travis builds that are configured `cache` content that does not change often (e.g. bash scripts, some dependency), to finish as soon as all the
Travis `workers` finish the required jobs.

<!-- more --> 

## Why use incremental builds? 

In short, incremental builds an approach to split builds into a set of incremental set of smaller builds in order to speed up the build generation process, in turn giving you a faster build time. So when deciding to make a Travis build have incremental hallmarks, speed of incremental builds is crucial to get a short edit-compile-test loop, thus sparing you time. 

## Examples of parrallel/incremental builds 

In this example, I'll be using `sbt` some of you may have heard of it, if not I recommend taking a peek at it. The short description of `sbt` is that `sbt` is specifically built for Scala and Java projects. It's mainly used by developers via the ability to cross build your project against multiple Scala versions, and this is one of the reasons I personally use it. Let's start with a `.travis.yml`: 

```yaml
language: scala

jdk: openjdk8

scala:
   - 2.10.4
   - 2.12.14
```

Now by default Travis will run `sbt ++$TRAVIS_SCALA_VERSION test`, but let's set a conditional in the `.travis.yml` file that enforces this. To enforce this, we will be adding a `script` hook in your `.travis.yml` file: 

```yaml
script:
   - sbt ++$TRAVIS_SCALA_VERSION test
```

Since this blog post is about saving time and in particular the scope of this post is Java/Scala, we will be using `scripted`. As I was talking about earlier so you don't have to test and build cross platform, so let's try another `.travis.yml` scenario where we utilize `scripted`: 

```yaml
language: scala

jdk: openjdk8

script:
   - sbt scripted
```

I recommend making these type of "time-saving" builds as verbose as possible, in the build log you'll see the following lines:

```bash
export JVM_OPTS=@/etc/sbt/jvmopts
export SBT_OPTS=@/etc/sbt/sbtopts
```
You'll then learn what `env vars` are being passed where. In turn giving you a garden variety ways of saving time. 


## Caching 

Caching in Travis CI is always a crucial way to speed up build times. In this example we're even going to cut unnecessary cache updates if you wish to even speed your build times up even further. So what we can do is use the `before_cache:` hook, and use the `rm` command which means `remove. Let's give you an exmaple:

```yaml
before_cache:
- rm -fv $HOME/.ivy2/.sbt.ivy.lock
- find $HOME/.ivy2/cache -name "ivydata-*.properties" -print -delete
- find $HOME/.sbt -name "*.lock" -print -delete
- echo "We saved time"
```
Now let's actually use the `cache` hook. When using `sbt` depending on what `os` you're building on, the directories could change, so please be cognizant of this:

```yaml
cache:
directories:
- $HOME/.cache/coursier
- $HOME/.ivy2/cache
- $HOME/.sbt
```
This usually will take on average ~3-4 minutes off your build time using `cache` whilst using this in conjunction with the `before_cache` script hook.

## Using Bash 

We talked about `sbt` above and `Scala` but let's look at broader scope of improving build times. One of the tools I use is `bash`, to some degree it helps us automate things, which in itself is a time saver. So let's create variables for a frequently-referenced file in paths and branches:

```bash
export BRANCH=$TRAVIS_BRANCH
export branch=$TRAVIS_BRANCH
```
One thing I've noticed is ensuring that `inexact rename detection` error isn't skipped due to too many files is a common cause (from my own experience) for slower builds, so lets set the `renameLimit` in git: 

```bash
git config --global diff.renameLimit 9999999
```
Another way we can save time is by running a `git diff` for the incremental build depending on the checked-out branch at the time: 

```bash
if [ "$BRANCH" == "dev" ]; then
  echo 'Preparing for an incremental deployment'
  for branch in $(git branch -r|grep -v HEAD); do
    #create tracking branch:
    git checkout -qf ${branch#origin/}
  done;
  git checkout (branch name)
 
 # Let's run the git diff I mentioned above
 
  git diff --diff-filter=UMA --name-only master force-app/ |
  while read -r file; do
  
    # Copy the files from git diff into the deploy directory:
    
    sudo cp --parents "$file" $DEPLOYDIR 2>/dev/null
    
    # For any changed class, trigger, page file, it's associated meta-data from the parent file is copied to the deploy directory (and vice versa):
    
    if [[ $file == *.cls ]]; then
      find $classPath -samefile "$file-meta.xml" -exec sudo cp --parents -t $DEPLOYDIR {} \;
    elif [[ $file == *.cls-meta.xml ]]; then
      parsedfile=${file%.cls-meta.xml}
      find $classPath -samefile "$parsedfile.cls" -exec sudo cp --parents -t $DEPLOYDIR {} \;
    elif [[ $file == *Test.cls ]]; then
      find $classPath -samefile "$file-meta.xml" -exec sudo cp --parents -t $DEPLOYDIR {} \;
    elif [[ $file == *Test.cls-meta.xml ]]; then
      parsedfile=${file%.cls-meta.xml}
      find $classPath -samefile "$parsedfile.cls" -exec sudo cp --parents -t $DEPLOYDIR {} \;
    elif [[ $file == *.trigger ]]; then
      find $triggerPath -samefile "$file-meta.xml" -exec sudo cp --parents -t $DEPLOYDIR {} \;
    elif [[ $file == *.trigger-meta.xml ]]; then
      parsedfile=${file%.trigger-meta.xml}
      find $triggerPath -samefile "$parsedfile.trigger" -exec sudo cp --parents -t $DEPLOYDIR {} \;
    elif [[ $file == *.page ]]; then
      find $pagesPath -samefile "$file-meta.xml" -exec sudo cp --parents -t $DEPLOYDIR {} \;
    elif [[ $file == *.page-meta.xml ]]; then
      parsedfile=${file%.page-meta.xml}
      find $pagesPath -samefile "$parsedfile.page" -exec sudo cp --parents -t $DEPLOYDIR {} \;
    fi
  done 
```
<img width="976" alt="gentoo" src="https://user-images.githubusercontent.com/20936398/125999356-ae994db5-6f6b-499b-adfc-e81872fcecab.png">

We can of course name that bash file anything we want, e.g. `timesaver.sh`, of course make it executable via: `chmod u+x timesaver.sh` (or whatever you decide to call your bash script.)

So let's say you have a `C++` project, you have a project where a directory is cached, we know that `$CACHED_DIR` is the directory with the build files. So then before you start building, add a command that uses this information to fix the `mtimes`  of the files that were changed since the previous commit:

```bash
cd "${TRAVIS_BUILD_DIR}"
read PREVIOUS_GIT_COMMIT < $CACHED_DIR/previous_git_commit.txt
changed_files=`git diff --name-only $PREVIOUS_GIT_COMMIT HEAD` 
touch `echo $changed_files`
```
Last thing I'll add for Bash is `mtime`. Let's choose a directory name the `json cache` (for example .mtime_cache) and add it to your `.gitignore.` Then add it to your Travis CI cache. Then your cache section in `.travis.yml` might look like this:

```yaml
cache:
  directories:
  - my/build/dir
  - .mtime_cache
```

## Conclusion 

I demonstrated ways you can save time in your builds in Scala using `sbt`, C++ utilizing `mtime`, various Travis technologies and existing technologies like `Bash`. This is the tip of the iceberg of ways you can save time on builds. In the time coming I will be creating a comprehensive guide on saving time on your Travis CI builds. Be on the lookout for that, it'll be coming soon.

In the time benig, if you have any questions please email me at [montana@travis-ci.com](montana@travis-ci.com). Happy building! 
