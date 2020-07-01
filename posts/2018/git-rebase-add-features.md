.. title: Some Nice 'git rebase' and 'git add' features
.. slug: git-rebase-add-features
.. date: 2018-05-27 12:57:04 UTC+05:30
.. tags: Git
.. link:
.. description:
.. type: text
.. status:
.. author: Abhijit Gadgil
.. summary: This blog post discusses a couple of handy tricks - using `git rebase` and `git add` to re-organize code where one can add parts of code in a single file into separate commits.

# Choosing a Subset Of Changes in a File

It quite often happens with me, while working on a feature, looking at the code, you I want to 'fix something' that's not directly related to the exact part of the feature that I am working on, but still important to be done for committing.

For instance, recently, I forgot to implement a method in a Class (Class 1) and realized while I started implementing another Class (Class 2). Now ideally What you want in a history is 'changes related to an implementation of a class say Class 1 above be together and ideally in one commit (at-least when pushed to the `origin`). A normal practice while committing work is `git commmit -a`, which will basically 'stage' all modified files and commit them. What if one wants to add only 'a part of a file' in a commit while leaving the other modifications intact but uncommitted. `git add --patch` serves this purpose. It opens an *interactive session* and asks you several choices about what you want to to with different `hunks` that it has identified, which one can add, ignore and so on. But what if there is a 'hunk' that git has identified but you want to only take 'part' of that hunk and leave other 'part' untouched? The above interactive session supports an option called `split` ('s') that can be used to split a given hunk and then the interactive session continues as before. This becomes very handy.

To pick and choose which changes in a file to use in a commit - simply do

`git add --patch` and then continue the interactive session



# Merging More than one Commits together

Before we want to publish our changes to origin (or create a PR), it's often a good idea to `rebase` our feature branch (with the origin branch) as well as combine our multiple commits in the feature branch into a single (or a few meaningful) commit(s). A very interesting feature of `git rebase -i` is you can `pick` and `squash` commits, but not just that, it's possible to 're-organize' the order of the commits themselves. This is quite an interesting feature, that allows the history of the repository to be kept meaningful and consistent.

To re-organize your work in a feature branch -

`git rebase -i` and then 'pick' and 'squash' commits if necessary. You can move the lines of 'pick' in the file. (Keep in mind if you 'delete' the commit, during rebase those changes will be lost). A lot of this is 'documented' (in fact the commit message documents this very well, often we tend to ignore.)


