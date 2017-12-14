<!DOCTYPE html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8"/>
</head>

# GitHub Mini HowTo

A very brief explanation of our [Islandora](https://islandora.ca/) module development workflow.

## Introduction

This document explains --- very briefly and with lots of simplifications and inaccuracies --- the workflow for developing [PHP](https://secure.php.net)-code for the [`lib4ridora`](https://github.com/Lib4RI/lib4ridora)-module.

### @CAVEAT

THIS IS JUST A SKETCH THAT WORKS FOR US (MODULO MISTAKES) AND IN ALL LIKELIHOOD WILL NOT WORK FOR YOU! USE THESE INSTRUCTIONS AT YOUR OWN RISK! CHANCES ARE THAT THEY WILL BREAK YOUR SYSTEM --- PERMANENTLY! UNDER NO CIRCUMSTANCES ARE WE TO BE HELD LIABLE FOR ANY DAMAGE WHATSOEVER! YOU HAVE BEEN WARNED...

### Important notice:

Although below everything is done as user `root`, all the `sudo`-commands should actually be called as user `apache` (using `/usr/bin/sudo -u apache ...` instead of `/usr/bin/sudo ...`). As this probably involves `chown`ing the files first, we leave these instructions as they are for the moment...

## Getting started

1. Have Lib4RI give you push rights to the following repositories:
    * [`lib4ridora`](https://github.com/Lib4RI/lib4ridora)
    * [`libfourri_theme`](https://github.com/Lib4RI/libfourri_theme)
    * [`lib4ri_solr_config`](https://github.com/Lib4RI/lib4ri_solr_config)
    * [`lib4ri_workflow`](https://github.com/Lib4RI/lib4ri_workflow)
2. Create SSH-RSA-IDs (on [`dev`](sftp://<dora-dev-server>) and [`prod`](sftp://<dora-prod-server>)):
    On [`dev`](sftp://<dora-dev-server>):
    ```
    ssh-keygen -t rsa -b 4096 -C "<your-github-username>@users.noreply.github.com" -f ~/.ssh/id_rsa_github-dev
    ```
    On [`prod`](sftp://<dora-prod-server>):
    ```
    ssh-keygen -t rsa -b 4096 -C "<your-github-username>@users.noreply.github.com" -f ~/.ssh/id_rsa_github-prod
    ```
    **IMPORTANT: When prompted for a passphrase, make sure to use one that you did not use for anything else, and make it complicated!**
3. Then, log into [GitHub](https://github.com), go to https://github.com/settings/keys, select "New SSH key" and upload the files `~/.ssh/id_rsa_github-dev.pub` and `~/.ssh/id_rsa_github-prod.pub` (you probably have to first retrieve those from the two servers using `scp`/`sftp`). From now on, you can communicate with GitHub via SSH-publickey (rather than SSH-password).

## The current workflow (N.B.: This is **not** state-of-the-art and should be improved!)

Note: We exemplify the workflow for the [`lib4ridora`](https://github.com/Lib4RI/lib4ridora)-module only. Other modules might have different names for the "`master`" branches, in addition to requiring changes in the workflow. In particular, the [`lib4ri_solr_config`](https://github.com/Lib4RI/lib4ri_solr_config)-module needs special attention regarding deployment (@TODO: Explain this!).

### On [`dev`](sftp://<dora-dev-server>)

1. Check if you are up-to-date
    1. Log into `dev`: `ssh <dora-dev-server>`
    2. Go to the module: `cd /var/www/html/sites/all/modules/lib4ridora`
    3. Verify that `git status` tells you that there is `nothing to commit` and that you are on branch `7.x`
    4. Compare the hash of the top-most entry in `git log` with the last commit on [`lib4ridora`](https://github.com/Lib4RI/lib4ridora)'s GitHub page
    5. (<i>@optional</i>) Inspect `git reflog` for any weirdness
2. Modify the code
    1. Make your modifications using `/usr/bin/sudo /bin/nano <filename>` or `/usr/bin/sudo /usr/bin/vi <filename>`
    2. Check the git-status: `git status` (it should give you a list of files that are `changed, but not updated` and/or a list of `untracked files`; you can review the changes of tracked files using `git diff`
3. Make a commit (subproject by subproject)
    1. Identify yourself:
        1. `git config -l --local` (or, if your `git` version is a bit older, `git config -l -f .git/config`) should give you the current values of `user.name` and `user.email` --- obviously, they should be set to you, since you did the work and are going to commit (NOTE: setting this with option `--global` will just set it for your user on the server; as we are committing as `root` (resp. `apache`), those settings will be ignored. *IT IS BAD STYLE* to set this globally for user `root` (resp. `apache`))
        2. If necessary, use
            ```
            /usr/bin/sudo /usr/bin/git config user.name <your-github-username>
            /usr/bin/sudo /usr/bin/git config user.email <your-github-username>@users.noreply.github.com
            ```
            to set the correct values
    2. While you are at it, review `git config -l --local` and verify the remote, branch, etc...; for proper SSH-publickey functionality, your `remote.<remote_name>.url` should read `git@github.com:/<path-to-repo>.git` (here, `remote.origin.url` should read `git@github.com:/Lib4RI/lib4ridora`)
    3. Add the files to be committed: `/usr/bin/sudo /usr/bin/git add <filename(s)>`
    4. Check `git status` to see if you are satisfied (the list will now include the files ready to be committed --- those not in this list will not be used)
    5. Triple-check `git diff --staged` to see if you are happy with what you will commit
    6. Commit either interactively (`/usr/bin/sudo /usr/bin/git commit`) or non-interactively (`/usr/bin/sudo /usr/bin/git commit -m "<commit_message>"`). Take care to make your message short and poignant, describing what you did and/or why. Note that the first line will appear as the title and might be truncated; additional lines (after an empty line) can be added for more detailed descriptions (see below)
    7. `git status` should now tell you that you are `one commit ahead of 7.x` (or more, if you make more than one commit)
4. Push the commit on GitHub
    1. Check if `ssh-agent` is running:
        ```
        if [ -z "$SSH_AUTH_SOCK" ]; then eval $(ssh-agent -s); fi
        ```
    2. Check if your SSH-identity is loaded (<i>@untested</i>):
        ```
        mysshids=$(ssh-add -l 2>/dev/null)
        if [ $? -ne 0 ] || [ -z "$(echo $mysshids | grep 'id_rsa_github-dev')" ]; then ssh-add ~/.ssh/id_rsa_github-dev; fi
        ```
    3. Push the commit:
        ```
        /usr/bin/sudo SSH_AUTH_SOCK=$SSH_AUTH_SOCK /usr/bin/git push origin 7.x
        ```
        Note that specifying the branch `7.x` or even the remote `origin` might not be necessary; that depends (<i>how?</i>) on your `git config -l --local` (<i>@unchecked</i>)
    4. Verify that your commit(s) made it into the GitHub repository [`lib4ridora`](https://github.com/Lib4RI/lib4ridora) (compare hashes between `git log` and the [Commits-page](https://github.com/Lib4RI/lib4ridora/commits/7.x))

### On [`prod`](sftp://<dora-prod-server>)

1. Check the status
    1. Log into `prod`: `ssh <dora-prod-server>`
    2. Go to the module: `cd /var/www/html/sites/all/modules/lib4ridora`
    3. Verify that `git status` tells you that there is `nothing to commit` and that you are on branch `7.x`
    4. Compare the hash of the top-most entry in `git log` with the last old commit (before your new commit(s)) on [lib4ridora](https://github.com/Lib4RI/lib4ridora)'s GitHub page
    5. (<i>@optional</i>) Inspect `git reflog` for any weirdness
    6. Review `git config -l --local` and verify the remote, branch, etc...; for proper SSH-publickey functionality, your `remote.<remote_name>.url` should read `git@github.com:/<path-to-repo>.git` (here, `remote.origin.url` should read `git@github.com:/Lib4RI/lib4ridora`)
2. Pull the new commit(s)
    1. Check if `ssh-agent` is running:
        ```
        if [ -z "$SSH_AUTH_SOCK" ]; then eval $(ssh-agent -s); fi
        ```
    2. Check if your SSH-identity is loaded (<i>@untested</i>):
        ```
        mysshids=$(ssh-add -l 2>/dev/null)
        if [ $? -ne 0 ] || [ -z "$(echo $mysshids | grep 'id_rsa_github-prod')" ]; then ssh-add ~/.ssh/id_rsa_github-prod; fi
        ```
    3. Pull the commit(s):
        ```
        /usr/bin/sudo SSH_AUTH_SOCK=$SSH_AUTH_SOCK /usr/bin/git pull
        ```
        Note that it might be necessary to specify the remote and/or branch, depending on settings (review `git config -l --local` --- @TODO: Explain this).

## Specialities

Below are a few "tricks" you can use in specific cases...

### Create long commit messages

Note: Conventionally, one puts a very short commit message on the first line (which will serve as a title) and gives more details, when necessary, after an empty line. Using `git commit`, one can edit the whole message in an editor (usually `vi`, `vim`, `emacs`, `nano`, or the like). When skipping the editing part, i.e. when using `git commit -m "..."` to set the message directly, `git` does **not** interpret `\n` as a newline character. There are a few workarounds:
* Use `git commit -m "<title>" -m "<details>"`, each `-m "..."` field creating a new paragraph (ie. with empty lines in between)
* If you need more structure, use `git commit -m "$(echo -e "<title>\n\n<details>")"`, or `echo -e "<title>\n\n<details>" | git commit -F-`
* Type newlines as you go while the quote of the message is "_open_" (`bash` behaves quite well in this regard, but be careful if you use quotes in your message!) --- you should see something like this:
    ```
    git commit -m "<title>
    > 
    > <details>"
    ```
* Use a "here"-document:
    ```
    git commit -F- << EOF
    <title>
    
    <details>
    EOF
    ```

The initial author of the present document personally favours the second item's second option (using a pipe). It is the clearest while allowing to write a one line command (easily editable in the command history). The last alternative is also nice, though.

### Commit as different author (but as same committer)

You might want to do this if you have to commit (and, possibly, push) changes that were made by someone else.

There are, essentially, two ways:
* Use `git commit --author "... <...>"`, where the author string is in the format `"Name <email>"` (e.g. `"John Doe <jj-dd@users.noreply.github.com>"`)
* Use `GIT_AUTHOR_NAME="..." GIT_AUTHOR_EMAIL="..." git commit`

### Commit as different author _and_ committer

You might want to do this to have a better repository history. Similarly to committing in your name but as a different author (which you would use when you have commit rights, but the other does not), you can also override the committer's identity (which you might want to use when your colleague has equal rights in your repository as you). Moreover, this can be _very_ useful if you have to amend a commit (see below), for example when you forgot to check and correct the current identity with `git config -l --local`, `git config user.name "..."`, `git config user.email "..."`.

There are, again, two ways:
* Set the name and email using `git config user.name "..."` and `git config user.email "..."` as described above, then execute `git commit`. Do not forget to change name and email back to your values before continuing
* Use `GIT_AUTHOR_NAME="..." GIT_AUTHOR_EMAIL="..." GIT_COMMITTER_NAME="$GIT_AUTHOR_NAME" GIT_COMMITTER_EMAIL="$GIT_AUTHOR_EMAIL" git commit`

**Remark:** In order to check the author, as well as the committer of a commit, the `--pretty=fuller` switch is often useful (as in `git log --pretty=fuller` or `git show --pretty=fuller HEAD`).

### Commit with different timestamp

This is useful if you have written some code a long time ago and want your history to reflect more realistically your progress.

* To change only the author date, use `git commit --date "..."` where the datestring is in the format `"<yyyy>-<mm>-<dd>T<hh>:<mm>:<ss>+<xxxx>"` (e.g., `"2017-10-16T11:47:55+0200"`)
* To change author and committer date use `GIT_AUTHOR_DATE="..." GIT_COMMITTER_DATE="$GIT_AUTHOR_DATE" git commit`

The following script (<i>@untested</i>) then takes a list of files and sets the commit date to the "youngest" file:
```
#!/bin/bash

CTIME="$(ls -tlA $@ --full-time | sed 's/^.*\([0-9-]\{10\}\) \([0-9:]\{8\}\).*\([-+][0-9]\{4\}\).*$/\1T\2\3/' | sort -r | sed -n '1 p')" || exit $?

echo "Setting author and commit time to \"$CTIME\"" >&2

git add $@ || exit $?

GIT_AUTHOR_DATE="$CTIME" GIT_COMMITTER_DATE="$GIT_AUTHOR_DATE" git commit || exit $?

exit 0
```

**Remark:** Again, the `--pretty=fuller` switch can often be useful to view the author _and_ the committer date (as in `git log --pretty=fuller` or `git show --pretty=fuller HEAD`).

### Track 'empty' directories (a feature `git` does not have)

There is the following workaround to have your repository include a certain empty directory structure (useful if your code does not create working directories when they are missing). This is necessary as `git` can only track files (including their permissions, though), but not directories.

The key is to use `.gitignore` files in the structure you want to save. E.g., assume you would like the following "empty" tree to be contained in (the root of) your repository:
```
parent
  child1
  child2
    grandchild1
    grandchild2
    grandchild3
  child3
```
Then create the following `.gitignore` files in `child1`, `child3`, `grandchild1`, `grandchild2` and `grandchild3`:
```
*
!.gitignore
```
Inside `child2`, create this `.gitignore` file:
```
*
!grandchild1
!grandchild2
!grandchild3
!.gitignore
```
Finally, inside `parent`, create this `.gitignore` file:
```
*
!child1
!child2
!child3
!.gitignore
```
Then, a simple `git add parent` should make `git` include the tree (before this, `parent/` appears as "untracked file").

Note: Actually, what this does, is make `git` track the various `.gitignore` files just created (and only those!). But, since `git` keeps track of the path of each file, this amounts to including the tree. Notice also that this will exclude any other file present, making this a good way to include a working tree.

### Untrack files

As far as the initial author of this document knows, files that are tracked once will continue to be tracked even when the `.gitignore` file in the containing directory would henceforth exclude them. You might want to delete the file to remove it, but there are cases when you would like to keep the file but stop tracking it. Here is how you can do it: `git rm --cached ...` for a list of files, or `git rm -r --cached ...` for directories. You can check the list of tracked files using `git ls-tree HEAD`. Note that you will have to `git commit` the file deletion (although the file itself will remain untouched and untracked henceforth).

## Re-writing history

Sometimes everyone makes mistakes. If you want to keep your history as clean as possible (or hide previously committed sensitive data), it may become necessary to re-write it. Before you do so, consider the following points, though.

* Once you pushed your commits to a remote (such as your public [GitHub](https://github.com) repository), someone else might have checked out the current status. If you re-write history and push again, those other users will have something broken in their history. It is, therefore, discouraged and bad style to rewrite the history in a shared repository. So, _please_, refrain from it (or be sure of what you are doing)
* As far as the initial author of this document can tell, those repositories in which you re-write history will continue to have the obsoleted hashes available. This is useful, on one hand, because the overwritten state is still somehow available. On the other hand, if you re-write history to unpublish sensitive data, then this means that this data will not be removed completely. This is another reason for looking twice in `git diff --staged` before committing, and not rush while doing so --- especially if you intend to push to a publicly available remote. Also, **do not forget** to change passwords, etc..., if you ever pushed one!
* After you re-write history, your `git log` will look clean and in the way you intended to. Your "local hash history" `git reflog`, however, will still reflect your actual history. This is, perhaps, not so bad, since you can re-construct for yourself what happened. Make sure, then, to do the re-writing in a local repository, if you want to hide the steps of the re-write from others
* As far as the initial author of the present document knows, you can only re-write the "top" of your history --- you cannot change something that is buried deep in your commit history. It might then become very tricky to obtain the history you want. Perhaps you should accept the tainted history in this case... (Actually, you can re-write history starting from some buried commit, but be prepared to do a lot of manual labour, and keep in mind that all the hashes on top of the buried commit you change will differ from those inside the old history; maybe also get acquainted with `git rebase` as an additional tool...)

### Amending commits

This comes in very handy if you just committed and you realise that something should be changed (e.g. the commit message, the author, the committer, some code, etc...). Just prepare everything like you would when committing, but then issue `git commit --amend`. This will replace the last commit in `git log` with the one you just made (including the changes relative to `HEAD^`; cf. below). Note that the hash of the amended commit will be different from the one of the unamended commit.

Note that `git reflog` will always exhibit the full hash history, but `git log` will not.

#### A quick remark on hash history
* `HEAD` refers to your current hash (the commit the next one will be based on)
* `HEAD^` or `HEAD~1` refers to the commit before that (according to `git log`), i.e. the parent in the "official" history. Similarly for `HEAD~2`, `HEAD~3`, etc...
* `HEAD@{1}`, `HEAD@{2}`, `HEAD@{3}`, etc... refer to the (local) hash history according to `git reflog`. This might differ significantly from your "official" `git log` history (in particular, it reflects all `git reset`s and the like, showing you the actual history of where `HEAD` pointed at).

### Rolling back

This effectively goes back to a previous state and starts over from there. There are two ways: the soft way and the hard way.

* Assume you just made a commit, but you realise that you misspelled something in the commit message, or you forgot a change relevant to this commit in one particular file. Then you can use the following (or use `git commit --amend` as above):
    1. note the hash for `HEAD` (`git reflog`, first line, or --- even better --- `git rev-parse HEAD`; let us call it `<old_head>`)
    2. `git reset --soft HEAD^` (goes back one commit, staging the files relevant to the commit we wish to amend)
    3. edit your file(s)
    4. `git add ...` (add the modified files, again --- in `git status` they appear as staged _and_ modified before this step)
    5. `git commit -C <old_head>` if you want to keep the commit message, `git commit -c <old_head>` otherwise. This ensures that the author, committer and date are preserved.

* Assume that you pulled two new commits on [`prod`](sftp://<dora-prod-server>) which break your system (say, because they rely on a newer version of PHP, and you only get "Internal server error"s). Then you can do `git reset --hard HEAD~2` to go back and discard all the changes since.

Note that, in all cases, your local instance of the repository will keep track of the hash history (as shown by `git reflog`).

## Branching, merging and sending pull requests

Branches are, basically, deviations from the mainline code (aka. the _master branch_) that can be used for developing new features or test patches. They are based on a particular commit of the master branch and can be incorporated back into the main code at a later stage (i.e. even after several new commits in the master) through so-called _mergers_.

If several people collaborate on a coding project, it can be very useful to have each programmer work on their own branch (and in an own local instance of the code). When a collaborator finishes the implementation of some feature, they can merge their code into the (perhaps evolved) master branch, or --- even better --- submit a so-called _pull request_ to the maintainer of the main repository, who can then review the submitted code and merge it into the master branch.

### Branching

The easiest way to create a new branch is to checkout some commit into a new branch:
```
git checkout -b <new_branch_name> [<starting_point>]
```
where `<starting_point>` is the name of a commit you wish to base your branch on (defaults to `HEAD` if not specified). To check which branch you are on (and what branches are available in your local repository instance), use `git branch`. You can easily switch between branches using
```
git checkout <branch_name>
```
(but avoid switching while having uncommitted work).

In addition, you can also set different remotes for each branch (e.g., in case you wish to follow the master branch of an outside repository in order to merge their updates into your fork). You can either do this by specifying (forget the last two lines if the remote is already configured)
```
git config branch.<branch_name>.remote <remote_name>
git config branch.<branch_name>.merge refs/heads/<remote_branch_name>
git config remote.<remote_name>.url git@github.com:<path_to_repo>
git config remote.<remote_name>.fetch +refs/heads/*:refs/remotes/<remote_name>/*
```
or, much easier and less error prone, by executing when starting out
```
git remote add <remote_name> git@github.com:<path_to_repo>
git fetch --all
git checkout -b <new_branch_name> <remote_name>/<remote_branch_name>
```
This will create the branch `<new_branch_name>` tracking the branch `<remote_branch_name>` in the `git@github.com:<path_to_repo>` repository.

### Merging

So assume that you are ready to incorporate the (already committed!) changes you made in the branch `<branch_name>` into the `master` branch. Then one way of doing this would be as follows.
1. Check out the master branch: `git checkout master`
2. Merge the new commits into `master`: `git merge <branch_name>`
3. In case `git` can figure out a way to incorporate your code into `master`, you are done. Otherwise, `git` will complain about one or several _merge conflict_s. In this case, you will need to manually select the code you want to keep (if not re-code some parts) in each file where a conflict occurred. Such files are categorised as "both added" or "both modified" in the output of `git status`, and they contain so-called _conflict markers_. Such a file will look --- possibly in _several_ places --- as follows:
    ```
    [...]
    [...merged code...]
    [...]
    <<<<<<< HEAD
    [...]
    [...conflicting code in the master branch...]
    [...]
    =======
    [...]
    [...conflicting code in the branch <branch_name>...]
    [...]
    >>>>>>> <branch_name>
    [...]
    [...merged_code...]
    [...]
    ```
    `git` tells you to resolve the conflict and then `git add` the file (all files that were automatically merged are already staged at this point).
    **Remark:** It is not always wise to just select either the part between `<<<<<<< HEAD` and `=======` or the part between `=======` and `>>>>>>> <branch_name>` (deleting the other part as well as the three conflict markers), since `git` cannot know if the code would still run. If things get complicated, the following three commands can prove extremely helpful:
    * `git show :1:<filename>` to see the common parent of the file `<filename>`
    * `git show :2:<filename>` to see the content of `<filename>` according to branch `master` (i.e. like it was before you called `git merge <branch_name>`)
    * `git show :3:<filename>` to see `<filename>` according to the branch `<branch_name>` (i.e. like it would have looked had you just typed `git checkout <branch_name>`)

    Once you are done solving all the conflicts and `git add`ing all the problematic files, issue a `git commit` in order to finish the merge (`git` will suggest the commit message `Merge branch '<branch_name>' into master`).

    Note: `git` keeps track that you are currently solving a merge conflict by placing the file `MERGE_HEAD` into the `.git` subdirectory. Therefore, look out for this file if you start working in a repository for which you are unsure about the status (and perhaps solve any merge conflict before you start coding new features).
4. (<i>@optional</i>) `git push` the merged code to your repository

**Remark:** An alternative way of merging a branch into `master` is to `git rebase` the code. This essentially places the commits in the branch `<branch_name>` _on top_ of the commits in the `master` branch. It is a feature-rich tool which has an interactive mode allowing one to solve conflicts commit by commit (which can be very useful if `<branch_name>` and `master` have diverged excessively). See `man git-rebase` for more details.


### Sending pull requests

The idea behind this is that you can patch or extend the code in some public repository and ask the maintainers to review your changes or additions and merge your code into theirs for others to take advantage of your work. Normally, you will not have push rights to the repository in question, so there is no other way, actually, than to submit a pull request. It may, however, also improve the collaborative code development workflow within a group, since changes will be made only in the own instances of each developer, and the maintainers (which could even be everybody else except the coder of the particular request) will be responsible for reviewing and merging the new code into the main repo, thus improving quality control.

A pull request presumes that the code you wish to submit is already deposited at some location the maintainer has access to (typically identified by a URL, such as a repository inside your own GitHub page), although a file path may work equally well. Of course, if you are using [GitHub](https://github.com), you can submit a pull request online. If you prefer the command line, though, you can generate one as follows:
```
git request-pull [-p] <upstream-base-for-your-changes> <url-where-to-find-your-pushed-code> [<last-commit-to-be-pulled>]
```
Here, specifying `-p` will also generate a `diff` in the pull request, and `<last-commit-to-be-pulled>` defaults to `HEAD`. It is probably useful to specify the latter in the following context. Assume that you pushed a local development branch directly onto the master branch of your remote. Then you can specify the master branch in the pull request while basing it on your local development branch using the syntax: `<local-devel-branch>:<remote-master-branch>`.

The output `git request-pull` generates can then be sent to the maintainers, along with a short description of what the series of commits you submit is doing (and, perhaps, why they should integrate it).

## Updating the Islandora core modules on the server

This should actually be easy, as long as one does not modify the code of the core modules (see "Updating untouched modules") --- modulo, of course, the then necessary adaptations in our own modules (especially [lib4ridora](https://github.com/Lib4RI/lib4ridora)). However, if modifications were made, the best approach seems to be to maintain a fork. We explain below how to set up such a fork and how to update modules once the fork is in place.

NOTE: It is _highly_ recommended, besides making a backup before you begin, to perform all updates on [`dev`](sftp://<dora-dev-server>) first and test extensively before updating [`prod`](sftp://<dora-prod-server>). Also, it might be a good idea to start with updating the system (including userland tools) and then all the Drupal modules. Finally, be aware that all the Islandora core modules should be updated synchronously.

**Remark:** It is probably a good idea to get acquainted with the rest of this little tutorial before attempting to follow the instructions in this section...

### Pre-update tasks

Before you update the modules, themes and libraries, it is a good idea to perform the following steps:
1. Backup the server (shut it down and take a snapshot)
2. Update the system and userland (e.g. [`php`](https://php.net))
3. Update the Drupal modules (@TODO: Make a step-by-step guide for this)
4. Shut down the `httpd` and `fedora` services:
    ```
    /usr/bin/sudo /sbin/service httpd stop
    /usr/bin/sudo /sbin/service fedora stop
    ```

Now update each module (`/var/www/html/sites/all/modules`), theme (`/var/www/html/sites/all/themes`) and library (`/var/www/html/sites/all/libraries`) according to the instructions below (in case they are `git` repositories and are not updated by Drupal)

### Updating untouched modules

A simple `git pull` inside the module's root directory on your server (`/var/www/html/sites/all/modules/<module_name>`) should do the trick. However, if `git status` shows that you are not on any branch, or there are other issues and weirdnesses, consider updating in the following way:
```
cd /var/www/html/sites/all/modules
yes | /usr/bin/sudo /bin/rm -r <module_name>
/usr/bin/sudo -u apache SSH_AUTH_SOCK="$SSH_AUTH_SOCK" /usr/bin/git clone git@github.com:<path_to_module_repo>.git <module_name>
```

### Updating modules featuring custom code --- the first time

The following steps can be performed ahead of the updating process, so that updating the server for those modules can be done similarly to updating untouched modules (see final step below)

1. Go to the Islandora module's GitHub page and _fork_ the repository into Lib4RI
2. On your fork's page ([`https://github.com/Lib4RI/<module_name>`](https://github.com/Lib4RI/&lt;module_name&gt;)), create a new branch (e.g. `7.x-lib4ri`), make it the default branch and, for good measure, protect the `7.x` branch (you can probably also safely deactivate "Wikis" and "Projects" for the fork)
3. _Locally_ (i.e. on your own machine), clone the fork and enter the local repo:
    ```
    git clone git@github.com:/Lib4RI/<module_name>.git <module_name>.lib4ri
    cd <module_name>.lib4ri
    ```
    (you will probably need to create an SSH-RSA-ID first and let [GitHub](https://github.com) know about it). `git branch` should now tell you that you are on `7.x-lib4ri`
4. Now set up the upstream remote, have the master branch track it and check out your own branch, again:
    ```
    git remote add upstream git@github.com:/Islandora/<module_name>.git
    git fetch --all
    git checkout -b 7.x upstream/7.x
    git checkout 7.x-lib4ri
    ```
5. In case you already have modified the module:
    * If you have uncommitted changes:
        1. Find the commit on which your changes are based (`git status`, `git log`, `git reflog` inside `/var/www/html/sites/all/modules/<module_name>` on your server should provide you with all the information you need; in case the code on your server is not a `git` repository, try guessing the commit by using the file dates on your server and confirm with `diff -ru`ing a local copy of your server's code against a `git checkout <guessed_hash>` of a local clone of the module)
        2. Reset to this commit: `git reset --hard <hash>`
        3. Copy the modified files into your local repository:
            ```
            scp -p <your_user_name>@<dora-prod-server>:/var/www/html/sites/all/modules/<module_name>/<path_to_file_1> <path_to_file_1>
            [...]
            scp -p <your_user_name>@<dora-prod-server>:/var/www/html/sites/all/modules/<module_name>/<path_to_file_n> <path_to_file_n>
            ```
            (be careful when copying recursively: the ".git" directory should _not_ be copied, whereas you might want other "dot-files")
        4. Commit the uncommitted changes (perhaps choose an appropriate commit date if your changes are old; also, do not forget to `git config --global` your `user.name` and `user.email` if you haven't done so, yet):
            ```
            git status
            git diff
            git add <modified_files>
            git status
            git diff --staged
            CDATE="<yyyy>-<mm>-<dd>T<hh>:<mm>:<ss>+<xxxx>"
            GIT_AUTHOR_DATE="$CDATE" GIT_COMMITTER_DATE="$CDATE" git commit -F- << EOF
            lib4ri customisations
            
            <description>
            EOF
            ```
        5. Merge the `upstream` master branch:
            ```
            git merge upstream/7.x
            ```
            (solve conflicts and commit, if necessary)
        6. Push the changes to your fork: `git push origin 7.x-lib4ri`
            
    * If your changes have already been committed on the server (<i>@unchecked</i>):
        It is probably possible to work this out directly in a local copy of your server's module (i.e. replacing step 3):
        1. Copy the server's module to your local machine and enter the repo:
            ```
            scp -pr <your_user_name>@<dora-prod-server>:/var/www/html/sites/all/modules/<module_name> <module_name>.lib4ri-prod
            cd <module_name>.lib4ri-prod
            ```
        2. Rename the master branch: `git branch -m 7.x 7.x-lib4ri`
        3. Set the remote of your branch to your Lib4RI repository:
            ```
            git remote set-url origin git@github.com:/Lib4RI/<module_name>.git
            ```
        4. Continue with step 4 above:
            ```
            git remote add upstream git@github.com:/Islandora/<module_name>.git
            git fetch --all
            git checkout -b 7.x upstream/7.x
            git checkout 7.x-lib4ri
            ```
        5. Merge the `upstream` master branch:
            ```
            git merge upstream/7.x
            ```
            (solve conflicts and commit, if necessary; also, do not forget to set yourself as the author and committer before this step)
        6. Push the changes to your fork: `git push origin 7.x-lib4ri`

    Once this is set up (or if there are no modifications yet), you can make changes, commit and push them to your Lib4RI fork as you would do with your own modules. Follow the instructions below on how to get the `upstream` updates into your fork
6. Delete the module on your server (`yes | /usr/bin/sudo /bin/rm -r /var/www/html/sites/all/modules/<module_name>`) and execute `/usr/bin/sudo -u apache /usr/bin/git clone git@github.com:/Lib4RI/<module_name>.git` inside `/var/www/html/sites/all/modules` to obtain a copy of your fork on the server

### Updating modules featuring custom code --- subsequent times

In the following, we assume that you are already keeping a fork of the module `<module_name>` according to the setup described in the previous subsection.

1. Enter the module's location (`cd /var/www/html/sites/all/modules/<module-name>`)
2. Check that you are on the right branch (`7.x-lib4ri`), that the remotes (`origin` and `upstream`) are set up correctly, and that there are no uncommitted changes (commit them, if necessary):
    ```
    git branch
    git config -l -f .git/config
    git reflog
    git status
    ```
3. Get up-to-date with the remotes:
    ```
    /usr/bin/sudo -u apache /usr/bin/git fetch --all
    ```
4. (<i>@optional</i>) Update the upstream code in your local repository (and go back to your custom branch):
    ```
    /usr/bin/sudo -u apache /usr/bin/git checkout 7.x
    /usr/bin/sudo -u apache /usr/bin/git pull upstream 7.x
    /usr/bin/sudo -u apache /usr/bin/git checkout 7.x-lib4ri
    ```
5. The following command shows you all the commits that are in your (local copy of your) fork but not upstream:
    ```
    git log --oneline upstream/7.x..7.x-lib4ri
    ```
   whereas the following gives all the commits that are upstream but not yet locally available:
    ```
    git log --oneline 7.x-lib4ri..upstream/7.x
    ```
   If the output of the second command is empty, then no update is needed (but you should make sure by comparing hashes).

   Note: If you do not wish to see your complete custom commit history, you can refine the first line using "`--since="<yyyy>-<mm>-<dd>T<hh>:<mm>:<ss>+<xxxx>"`", restricting, e.g., to the date of the last update, and/or you can specify "`--no-merges`" in order not to see any merge commit.
6. In case an update is due, issue
    ```
    /usr/bin/sudo -u apache /usr/bin/git merge upstream/7.x
    ```
   (solve conflicts and commit, if necessary --- do not forget to set author and committer appropriately: `/usr/bin/sudo -u apache GIT_AUTHOR_NAME="..." GIT_AUTHOR_EMAIL="..." GIT_COMMITTER_NAME="..." GIT_COMMITTER_EMAIL="..." /usr/bin/git commit` (Note: For some reason, `git commit` may ignore the `GIT_AUTHOR_...` variables. In this case use `/usr/bin/sudo -u apache GIT_COMMITTER_NAME="..." GIT_COMMITTER_EMAIL="..." /usr/bin/git commit --author="... <...>"`))
7. Push the changes to your fork: `/usr/bin/sudo -u apache SSH_AUTH_SOCK="$SSH_AUTH_SOCK" /usr/bin/git push origin 7.x-lib4ri`

### Post-update tasks

After you updated all the modules (and, where applicable, themes (`/var/www/html/sites/all/themes`) and libraries (`/var/www/html/sites/all/libraries`), too), you may want to do the following:
1. Go to the `html` folder and update the databases (for the main site and for the subsites):
    ```
    cd /var/www/html
    /usr/bin/sudo /usr/bin/drush updb
    /usr/bin/sudo /usr/bin/drush @sites updb
    ```
2. Clear all caches:
    ```
    /usr/bin/sudo /usr/bin/drush @sites cc all
    ```
3. Re-start the `fedora` and `httpd` services:
    ```
    /usr/bin/sudo /sbin/service fedora start
    /usr/bin/sudo /sbin/service httpd start
    ```
4. (<i>@optional</i>) On each subsite, verify [`<subsite-base-url>/update.php`](http://<dev_subsite-base-url>/update.php)
5. Test, test again, and, when you are done, go ahead and test some more. Try to perform all your routine jobs, as well as special cases. Go through the configuration pages to see what has changed. Also, you might want to see if the bugs in your bug list are still present. Finally, do some testing for good measure...

<br/>

> _This document is Copyright &copy; 2017 by d-r-p (Lib4RI) `<d-r-p@users.noreply.github.com>` and licensed under [CC&nbsp;BY&nbsp;4.0](https://creativecommons.org/licenses/by/4.0)._
