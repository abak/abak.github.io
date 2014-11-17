---
layout: post
title: PyGit2 in pacbackup
---

My main machine runs under [Arch Linux](https://www.archlinux.org/). Given the fact that it is a rolling release distro, updating can sometimes bring problems. The fact that it is DIY also means that I can mess things up and make my system unbootable. So I'm developping [pacbackup](https://github.com/abak/pacbackup), a utility that backs-up the list of all installed packages on my systems, and provides a way to restore them after a catastrophic failure, from a completely fresh install.


---


One of the features I wanted for pacbackup is tha ability to version my package list in order to restore it at a former point in time, in the event that a particular piece of software caused incompatibilities (I'm looking at you GPU drivers). 

So I looked into [pygit2](http://www.pygit2.org/), the python wrappers for libgit2. All in all, it's a great and powerful library. The possiblity are limitless, as every aspect of git data model is exposed. And that's the main issue. Every aspect of the underlying data model is exposed. It's a really thin abstraction layer[^documentation].

One shortcoming of pygit2 is that the provided wrappers are very thin, and do not abstract the underlying git data model to provide common tasks.

For pacbackup, I needed to perform the following : 
  
  * Initialize an empty git repository :
  ````
  git init
  ````

  * Commit the locally changed files
  ````
  git add . && git commit -m "some commit message"
  ````
  * push the commits to a remote
  ````
  git push remote_name branch_name
  ````


##git init

Initializing an empty repo is fairly straigthforward : 

````
pygit2.init_repository(path, False)
````
Where the last argument indicates whether the repository is [bare](http://www.saintsjd.com/2011/01/what-is-a-bare-git-repository/) or not.


##git push


##git add . && git commit

Staging files and committing thw=em is by far the worse offender, as you need to understand git's data model, much more than when simply using git CLI.


### General case

First you need to get the repository object, and check its status, because no check will be performed automagically, which will lead to empty commits. That's not necesarilly a bad thing, but it might not be what you want.

    repo = pygit2.Repository(path)
    st = repo.status()

The ````st```` variable is a dict of all the paths to unclean files (either locally modified, untracked, etc.). You want to check its contents before the following.

Then you will need to fetch the current index of the working directory, and add any locally modified files[^local_paths] you want to commit to it, and save the result : 

    index = repo.index
    index.read()
    index.add(local_path)
    index.write()

Now comes the time to actually craft the commit. The method ````Repositor.create_commit```` serves this purpose. It requires :

  * The [references](http://git-scm.com/book/en/v2/Git-Internals-Git-References) to update
  * the author of the changes
  * the committer of the changes
  * the commit message
  * the treeish object to append to the references
  * the parents of the new commit

The author, committer and message are fairly straightforward, the ````pygit2.Signature```` class can be used to handle author/committer data :

    message = today + " - Automated Package List Backup"
    comitter = pygit2.Signature('PacBackup '+__version__, '')


In my use case, the only reference that needs updating is the head of the master branch.
The treeish to add can be obtained from the index : 

    tree = index.write_tree()

Finally, the parents (in my case, the only one) is the head commit of the master branch :

    parents = [repo.head.get_object().hex]


Putting everything together, to commit all locally modified files on top of head : 

    repo = pygit2.Repository(self.container)
    st = repo.status()

    if st:
      index = repo.index
      index.read()
      index.add(local_path)

      index.write()
      tree = index.write_tree()

      today = datetime.date.today().strftime("%B %d, %Y")
      message = today + " - Automated Package List Backup"
      comitter = pygit2.Signature('PacBackup '+__version__, '')

      parents = [repo.head.get_object().hex]

      sha = repo.create_commit('refs/heads/master',
       comitter, comitter, message, 
       tree,
       parents)

A whopping 20 lines.

### The case of the initial commit

The initial commit deserves some special love since, by definition, it doesn't have any parents. The call to ````Repository.create_commit```` needs to be modified as follows : 

    message = "Initial Commit - Automated Package List Backup"
    comitter = pygit2.Signature('PacBackup '+__version__, '')
    sha = repo.create_commit('HEAD', 
      comitter, comitter, message, 
      tree, 
      [])



[^documentation]: To the point that I recommend to use the [C documentation](https://libgit2.github.com/libgit2/#HEAD) instead of [python one](http://www.pygit2.org/).

[^local_paths]: As you would with git CLI, you must use here the path to the file you want committed, relative to the root of the repo. As of pygit2 0.21.4, this path will simply be appended to the absolute path to the repo.

