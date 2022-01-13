---
title: Split out a Git Repo and Keep it's Commit History
date: 2022-01-12T00:00:00+
draft: false
categories: misc
tags: ["git", "misc"]
---

One thing I've been tasked with working on recently is splitting out microservices from a monorepo into their own separate repos. While this may sound trivial, one thing that is essential is to keep the git history associated with these services - which is more complicated than it looks. There's a native tool built into git, `git filter-branch` - but it is quite finiky. 

Thankfully, a tool exists which offers a one command solution - [git filter repo](https://github.com/newren/git-filter-repo). In fact, the [man page for git filter-branch](https://git-scm.com/docs/git-filter-branch) even reccomends it be used as the tool of choice. 

### A Note for MacOS Users

You may receive an error such as 

```
fatal: cannot lock ref 'refs/heads/branchname': Unable to create '/Users/your_user_name/your_repo/.git/refs/heads/your_branch.lock': File exists.

Another git process seems to be running in this repository, e.g.
an editor opened by 'git commit'. Please make sure all processes
are terminated then try again. If it still fails, a git process
may have crashed in this repository earlier:
remove the file manually to continue.
git update-ref failed; see above
```

For me, this was because there were two branches with the [same name, but with different casing](https://github.com/newren/git-filter-repo/issues/48). Since MacOS ships with a default of a non-case sensitive filesystem, I needed to run this in a Linux VM. 

### Prerequisites

Per the docs, `git filter-repo` requires:

> git >= 2.22.0 at a minimum; some features require git >= 2.24.0 or later

However, I just update to the newest version of git. If you can't update to this natively (I had issues on Ubuntu 18.04), you can do the following:

```
sudo add-apt-repository ppa:git-core/ppa
sudo apt update
sudo apt install git
```

Next, install `git filter repo` - if you're on macOS you can simply `brew install git filter repo`, if you're on Linux, depending on your distro, you may have to [build from source](https://github.com/newren/git-filter-repo/blob/main/INSTALL.md) - here's a slightly modified version of the instructions taken from the `INSTALL.md`

```
git clone https://github.com/newren/git-filter-repo
cd git-filter-repo
cp -a git-filter-repo $(git --exec-path)
cp -a git-filter-repo.1 $(git --man-path)/man1 && mandb
cp -a git-filter-repo.html $(git --html-path)
ln -s $(git --exec-path)/git-filter-repo \
    $(python -c "import site; print(site.getsitepackages()[-1])")/git_filter_repo.py
```

### Performing The Initial Split

First, I recommend cloning a fresh copy of your repo as we don't want to mess up any version you may actually be working in

```
git clone https://github.com/${your_repo} -o your_repo_filter_clone
```

Next, in the freshly cloned repo, you can simply run 

```
git filter-repo --subdirectory-filter ${/YOUR/BROKEN/OUT/DIR}
```

This command takes everything in `/your/broken/out/dir`, puts it in the root directory, rewriting the commits so that the only thing that now exists in this repo are the contents of the directory, along with *only* the commit history of that directory. Neat!

Now all that's left to do is set up your remote instance, but before we do that...

### If you need to rename directories

It’s best to do this before committing anything to remote repositories – you can do this by simply using 

```
git filter-repo --path-rename ${original_path}:${new_path}
```

For example, as I break out these microservices, I'd like to restructure them according to the [standard go project layout](https://github.com/golang-standards/project-layout) - here's an instance of me renaming the `app` directory to what should be the `internal` directory:

```
git filter-repo --path-rename app/my-app.go:internal/my-app.go
```

This renames, and rewrites the history for app/smoke-test.go under internal/smoke-test.go

### Syncing your new repo back to the remote 

To sync the repo back to GitHub, or whatever you're using, you can simply set the remote. `git filter-repo` removes the remote origin, so you shouldn't be able to commit back to the repo you cloned it from by accident:

```
 git remote add origin https://github.com/${YOUR_ORG}/${YOUR_NEW_REPO}.git
 git branch -M main
 git push
```
