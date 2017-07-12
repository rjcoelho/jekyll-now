---
layout: post
title: Using Git Keyword Expansion!
---

### Git Keyword Expansion

It is possible to apply transformations to files under git on checkout:
![smudge filter](https://git-scm.com/book/en/v2/images/smudge.png)

And on adding to staging:
![clean filter](https://git-scm.com/book/en/v2/images/clean.png)

This is similar to what RCS's keyword expansion. See [Git Attributes](https://git-scm.com/book/en/v2/Customizing-Git-Git-Attributes) and [git-rcs-keywords](https://github.com/turon/git-rcs-keywords).

First lets extract author's email ```git log --pretty=format:"%aN" -1 -- FILE```, of the last commit of a given file. See [Git Pretty Format](https://git-scm.com/docs/pretty-formats).

Then write a script that replaces ```$Author$``` with the actual commit date
```
perl -pe "s/\\\$Author\\\$/\\\$Author, `git log --pretty=format:"%ae" -1 -- $1`\\\$/"
```
And undoes it:
```
perl -pe "s/\\\$Author[^\\\$]*\\\$/\\\$Author\\\$/"
```

Finally declare the clean/smudge filter:
```
git config filter.author.clean 'perl -pe "s/\\\$Author[^\\\$]*\\\$/\\\$Author\\\$/"'
git config filter.author.smudge 'perl -pe "s/\\\$Author\\\$/\\\$Date, `git log --pretty=format:"%cD" -1 -- $1`\\\$/" %f'
```
And on which files it should be applied by editing ```.gitattributes```:
```
*.txt filter=author
```

Now lets test it
```
echo "*.txt filter=author" > .gitattributes
echo "A simple file from \$Author\$." > test.txt
git add .gitattributes test.txt
git commit -m "Testing date expansion in Git"
rm test.txt
git checkout test.txt
cat test.txt
```

### Include gitconfig from repo

Git configuratio is loaded from ```~/.gitconfig``` (also called global) and locally per-repo in ```$GIT_DIR/config```. But not all sections are store on the locally, for example the filters aren't.

We can however use git configuration's ```include``` to include other files in the configuration, and add those files to the git repo.

For example, create a ```.gitconfig``` with:
```
[filter "author"]
    clean = perl -pe \"s/\\\\\\$Author[^\\\\\\$]*\\\\\\$/\\\\\\$Author\\\\\\$/\"
    smudge = perl -pe \"s/\\\\\\$Author\\\\\\$/\\\\\\$Date, `git log --pretty=format:\"%cD\" -1`\\\\\\$/\"
[alias]
    pull-force = "!git fetch ; git reset --hard @{u}"
    co = checkout
```
And add it the the repo:
```
git commit -am "Add .gitconfig"
```

Then when someone clones the repo all they need to do to include it on configuration is:
```
git config --local include.path ".gitconfig"
```
And check that your configuration are being used:
```
git config -l | grep filter
```

### Using git init templates

It is possible to override the default template directory ```/usr/local/share/git-core/template``` that will be copied to the $GIT_DIR after it is created when we call ```git init```. You can it on an existing cloned repository. This directory can be used to include git configurations (hooks, aliases, ...) in all repositories. See [Git Init](https://git-scm.com/docs/git-init).

Create a ```~/.git_template``` and add it to the user's global configuration:
```
git config --global init.templatedir '~/.git_template'
mkdir -p ~/.git_template/hooks
````
Alternativally you can add the following configuration snippet in your git configuration:
```
[init]
    templatedir = ~/.git_template
```

Now lets create [ctags](http://ctags.sourceforge.net/) on hook checkout, by creating a ```~/.git_template/hooks/ctags``` file and marking it as executable, see [Effortless Ctags with Git](http://tbaggery.com/2011/08/08/effortless-ctags-with-git.html) and [Git Hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks):
```
#!/bin/sh
set -e
PATH="/usr/local/bin:$PATH"
dir="`git rev-parse --git-dir`"
trap 'rm -f "$dir/$$.tags"' EXIT
git ls-files | \
  ctags --tag-relative -L - -f"$dir/$$.tags" --languages=-javascript,sql
mv "$dir/$$.tags" "$dir/tags"
````

And using it on ```~/.git_template/hooks/post-checkout``` hook:
```
#!/bin/sh
.git/hooks/ctags >/dev/null 2>&1 &
````

Now any new repositories you create or clone will be immediately indexed with Ctags and set up to re-index every time you check out.

----
****