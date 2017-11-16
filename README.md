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


<br/>

> _This document is Copyright &copy; 2017 by d-r-p (Lib4RI) `<d-r-p@users.noreply.github.com>` and licensed under [CC&nbsp;BY&nbsp;4.0](https://creativecommons.org/licenses/by/4.0)._
