---
layout: post
title:  "My open-source workflow"
date:   2022-01-19 11:11:11 -0500
categories: open-source
---

I've been doing open source for quite a while, but the moment it really kicked in was when I started programming Rust. The community is very friendly, and people tend to quickly organize the work around interesting ideas. However, it turns out that there are different ways of organizing the workflow, and not everybody is doing it the same way. Hence, in this post I decided to describe what I do in all my [open source projects](https://github.com/kvark). I don't pretend this to be ideal, in fact it's all fairly obvious, but it worked fairly well so far. I'm sure there is a few things that could be simplified here. At the very least, if you do something different, you'd be able to say "at least it's better than @kvark's one" :)

## Repository

Let's start with an openly available git repository. We use Github for most things, even if I'd like to get away from it, the network effects and high usability are too strong with this one. Basic things every repository needs:
  - well-thought name and good description
  - proper README describing what the heck
  - ways to communicate, like Matrix room badges in README, or enabled Discussions
  - continuous integration setup
  - CHANGELOG file
  - (in Rust) include `Cargo.lock` if the project has important binary targets

All the development happens in the default branch, let's call it trunk. We configure the repository to make it "protected" by requiring at least CI checks to pass on it. We also enable auto-merge and linear history. Auto-merge ensures that the merged code is CI-checked. Linear history allows us to bisect any regression later on. For this reason, it's also important to have each commit working.

Smaller PRs are generally squashed. Larger can sometimes be rebased, if there is a trust in the author to do the proper checks on each commit. I wish Github had an option to do this for us, i.e. run CI on every commit when rebasing.

## Contribution

First thing is forking the repository. One-time initialization goes as follows:
```bash
git clone https://github.com/<me>/<project-name>
cd <project-name>
git remote add upstream https://github.com/<owner>/<project-name>
```

Before writing a patch, set the stage ready on a temporary branch:
```bash
cd <project_name>
git checkout trunk
git pull -r upstream trunk # this updates my local trunk to the upstream
git checkout -b my-great-fix # branch out locally
```

If you accidentally worked in trunk and committed something, you can fix this by renaming it and forking the trunk again:
```bash
git branch -m my-great-fix
git checkout HEAD^ -b trunk # in case of 1 extra commit, use HEAD^^ for 2, etc
git checkout my-great-fix # continue working on it
````

It's important to have a quick way to check that everything is fine with the code. Most often it's just `cargo test`. Sometimes it's `make`, if there are extra steps involved. For example, [naga](https://github.com/gfx-rs/naga)'s `make` formats the code, runs tests and updates the reftests, and scans the code with clippy.

When you are done, push the changes to your fork:
```bash
git push origin HEAD
```
This will prompt you with a link to create a Github PR.
If you need to tweak something:
```bash
# quick and dirty way
git commit --amend --no-edit
git push -f origin HEAD
# more powerful way
git rebase -i HEAD^ # for one commit
```

We prefer the PRs to be as close to the tip as possible, so you may need to rebase if PR took a while to review:
```bash
git pull -r upstream trunk
# resolve conflicts
cargo test # preferably, for each commit
git push -f origin HEAD
```

Oh, and generally reviews should provide the first wave of feedback within a day or two. Anything longer is a big concern for the health of the project.

After the PR got merged we can either delete the branch:
```bash
git checkout trunk
git branch -D my-great-fix
```
Or continue with a different name:
```bash
git branch -m another-thing
git pull -r upstream trunk
````

## Releases

General release procedure consists for the following steps:
  1. Triage all bugs that could be API-breaking and see if anything should be done before the release, to be included in the breaking change, if any.
  2. Check all the dependencies to see if they need to be released first (i.e. switch from Github to Crates dependencies).
  3. Update the changelog by scanning the list of merged PRs (or commits) between the tip and the previous release.
  4. Bump the package versions.
  5. Test it thoroughly. Make a PR and see how CI likes it.
  6. Publish on Crates and merge the PR.

For breaking changes, we create a new branch from trunk, e.g. `v0.10`, after the merge. For patches, we work off the branch being patched (and the PR goes against that branch), and [cherry-pick](https://github.com/gfx-rs/naga/pull/1663) the necessary changes from trunk. Some people insist on tagging each and every release, but I don't find it necessary. As long as you can see when crates versions were updated, and you followed this routine, it's clear what code was changed for each version (in addition to the changelog entries).

Note: since the work is done in the branch, trunk doesn't get its patch version updated. The result is often that trunk package version is *behind* the release branch version. That may be confusing to some users, but I think it's totally fine. The version only matters for something that is either on crates, or used as a override for something on crates. You are never supposed to override a released version by trunk, because generally speaking trunk has some breaking changes in it.

Note: Rust's SemVer culture is different from the general SemVer in that the patch version in "0.x.patch" is required to be backwards compatible with other "0.x.y". So you can see a lot of "pre-1.0" crates, and [that's OK](https://github.com/kvark/mint/issues/31).

For an actively developed project releasing once in a few months (say, 3-4 releases per year) is a good rate. It's a balance between making other people do the work for updating, and keeping up with all the latest and greatest things.

## Growing the community

This stuff I haven't figured out totally yet. Making something good and usable may end up with people happily using it without contributing back. Making something too rough will repel potential users. The [itch-to-scratch](https://en.wikipedia.org/wiki/The_Cathedral_and_the_Bazaar) model seems to work somewhat:
> Every good work of software starts by scratching a developer's personal itch.

We generally use Matrix rooms for communication. I've seen projects with very active community engagement, and others with strong BDFL but a weak community. One thing that I value most is doing quick reviews and providing detailed feedback.
