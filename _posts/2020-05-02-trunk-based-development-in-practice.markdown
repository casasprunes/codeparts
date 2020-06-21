---
layout: post
title:  "Trunk-Based Development in practice"
date:   2020-05-02 20:00:00 +0100
modified: 2020-06-21 10:00:00 +0100
tags: DevOps
excerpt: "By following this model, we take a more continuous delivery approach to software development, we can release new features quickly while avoiding merge hell."
author: "casasprunes"
---
## 1. Introduction

There are two ways of doing Trunk-Based Development; the first is by committing straight into master, the second by using short-lived feature branches, this article describes the second one.

## 2. What is Trunk-Based Development?

Let's look at the definition of Trunk-Based Development from [trunkbaseddevelopment.com][tbd], by [Paul Hammant][paul]:

> A source-control branching model, where developers collaborate on code in a single branch called trunk *, resist any pressure to create other long-lived development branches by employing documented techniques. They therefore avoid merge hell, do not break the build, and live happily ever after.
>
> \* master, in Git nomenclature

And we can do it by committing straight into master or, in our case, by using short-lived feature branches:

> one person/pair over a couple of days (max) and flowing through Pull-Request style code-review & build automation before "integrating" (merging) into the trunk (or master)

## 3. Trunk-Based Development in practice

We are going to see now an example on how to use this model, assuming that we have in place a continuous integration workflow similar to the one described in [this article][workflow] that will run some checks whenever we create a Pull Request.

### 3.1 Start work on a new feature

There is a feature ready to be picked up in the Backlog; we take it.

> Features in the Backlog should take a max of two days to complete; otherwise, we should divide them into smaller ones.

We make sure we have the latest changes from master:

{% highlight shell %}
➜  piggybox git:(upgrade-ratpack) git checkout master
Switched to branch 'master'
Your branch is up to date with 'origin/master'.
➜  piggybox git:(master) git pull
Already up to date.
➜  piggybox git:(master) 
{% endhighlight %}

We start by implementing the test first and the code later, following a TDD approach. 

### 3.2 Create a Pull Request

We have finished the implementation of the new feature in less than a day ideally, two days max.

We create a short-lived feature branch and commit our changes:

{% highlight shell %}
➜  piggybox git:(master) git checkout -b feature-name
Switched to a new branch 'feature-name'
➜  piggybox git:(feature-name) git commit -a -m "Feature message"
[feature-name 04c92af] Feature message
 1 file changed, 1 insertion(+), 1 deletion(-)
➜  piggybox git:(feature-name) 
{% endhighlight %}

To create the Pull Request, we push the changes with the following command:

{% highlight shell %}
➜  piggybox git:(feature-name) git push --set-upstream origin $(git_current_branch)
remote: Create a pull request for 'feature-name' on GitHub by visiting:
remote:      https://github.com/casasprunes/piggybox/pull/new/feature-name
remote: 
To https://github.com/casasprunes/piggybox.git
 * [new branch]      feature-name -> feature-name
Branch 'feature-name' set up to track remote branch 'feature-name' from 'origin'.
➜  piggybox git:(feature-name) 
{% endhighlight %}

As a result, git shows us a link in the terminal that we can use to create the Pull Request on GitHub. 

After creating the Pull Request, automated checks run on it. 

### 3.3 Make requested changes

Another programmer/pair has reviewed our Pull Request and pointed out some improvements that we need to do and some flaws we need to correct.
We proceed to make the changes, and when we finish, we commit them:

{% highlight shell %}
➜  piggybox git:(feature-name) git commit -a -m "Changes message"
[feature-name 4d7d7f6] Changes message
 1 file changed, 1 insertion(+), 1 deletion(-)
➜  piggybox git:(feature-name) 
{% endhighlight %}

### 3.4 Rebase

Another programmer/pair has merged changes into master before us. In this case, we need to rebase before we can merge our Pull Request into master:

{% highlight shell %}
➜  piggybox git:(feature-name) git pull --rebase origin master
From https://github.com/casasprunes/piggybox
 * branch            master     -> FETCH_HEAD
   afc7492..e80f7dd  master     -> origin/master
First, rewinding head to replay your work on top of it...
Applying: Feature message
Applying: Changes message
➜  piggybox git:(feature-name) 
{% endhighlight %}

We resolve conflicts if any, and then push:

{% highlight shell %}
➜  piggybox git:(feature-name) git push --force-with-lease
To https://github.com/casasprunes/piggybox.git
 + 04c92af...74757c6 feature-name -> feature-name (forced update)
➜  piggybox git:(feature-name) 
{% endhighlight %}

### 3.5 Squash and Merge

All checks pass on the Pull Request, and we got the approval from our peers. Now we  click the Squash and Merge button in GitHub to merge the changes into master.

## 4. Why use Trunk-Based Development with short-lived feature branches?

By following this model, we take a more continuous delivery approach to software development, with short-lived branches, we can release new features quickly while avoiding merge hell. 

For unfinished work, we can use feature flags to keep incomplete features hidden until all the necessary code is shipped.

We keep master green and deployable at all time by making sure that before merging a Pull Request, we always run automatic checks: build, validate code formatting, run tests, measure code coverage, etc. These are just examples of checks you may run, depending on your needs. 

Also, having a code review from another programmer/pair before merging any changes into master might help us to find possible problems that the automatic checks can't find.

## 5. Conclusion

In this article, we have looked at how to use Trunk-Based Development with short-lived feature branches. I’m not saying this is the best branching model. There are other models like GitFlow, GitHub Flow, Trunk-Based Development committing straight to master, which might work better for you depending on the project you are working. 

It's always good to try different models until you find the one that works better for your team.

### References:

* [Introduction to Trunk Based Development][introduction]
* [Short-Lived Feature Branches][feature-branches]

[tbd]: https://trunkbaseddevelopment.com/
[paul]: https://twitter.com/paul_hammant
[workflow]: https://code.parts/2020/03/15/github-actions-for-a-kotlin-project-with-kafka-and-gradle/
[introduction]: https://trunkbaseddevelopment.com/
[feature-branches]: https://trunkbaseddevelopment.com/short-lived-feature-branches/
