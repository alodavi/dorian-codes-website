---
layout: post
title: "Code formatting: scalafmt and the git pre-commit hook"
date: 2018-09-20
---

When working in a team it is often useful to have a style guide. Why? Because for instance everyone has a particular coding style or formatting preferences. This leads either to a lack of consistency in the style of the whole project or to almost irrelevant discussions when it comes to review the code of another person.

That’s why formatting tools, like the one that we are using at ZyseMe, scalafmt, can boost the productivity of a team and allow developers to focus on problems that matter and are fun to solve.
What’s scalafmt

scalafmt is a code formatter for Scala. It can be run from the command line or as sbt task — making sure that your project has the right plugin. There is even an Intellij plugin.

It comes with a standard configuration that can be changed simply by adding a .scalafmt.conf file in the root directory of your project. All you need to do is decide the rules you want scalafmt to apply to your code and then — in case you installed the scalafmt cli — run

    scalafmt

And this is how scalafmt automatically formats your project for you.

## What’s a pre-commit hook

In the git Version Control System hooks are actions that are executed by git when particular events occur. When running `git init`, a directory containing the hooks is created in the `.git` hidden folder. By default the scripts in `./git/hooks` are not active and end with `.sample`. They can be activated by changing their ending to correspond to a script format — usually it is a shell script but it can be also python, ruby, perl etc…

A `pre-commit hook` in particular is a special command that is executed before committing. More information can be found in the [git documentation](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks).

## How to set a scalafmt git pre-commit hook for your project

The goal of this `pre-commit` hook is:

* To check compilation
* To check formatting

If one of the two is failing, git doesn’t allow to commit.

In the root directory of the project you should create a `hook` folder that has the following structure:

    .
    └── hook
        ├── README.md
        ├── pre-commit-hook-install.sh
        └── pre-commit-hook.sh

Where `pre-commit-hook.sh` is the script that contains the instructions for the hook itself, `pre-commit-hook-install.sh` manages the local installation of the hook and the `README.md` contains the relative documentation.

### Step 1: create a script to install the pre-commit hook on your local git folder

By design hooks can only be installed locally, so if you want everybody on the team to be able to use and edit it, you need a script that sets up everything automatically.

First you have to ensure that the scalafmt cli is installed:

    brew install --HEAD olafurpg/scalafmt/scalafmt

In case you’re not a mac user follow the [instructions in the scalafmt documentation](https://scalameta.org/scalafmt/docs/installation.html).

Then you have to create a link and copy paste the code in `project-root/hook/pre-commit-hook.sh` to `.git/hooks/pre-commit-hook.sh`:

    #create the file if not exist
    touch ../.git/hooks/pre-commit

    #delete the file
    rm ../.git/hooks/pre-commit

    #create a file link
    ln -s pre-commit-hook.sh ../.git/hooks/pre-commit

    #copy paste the code in the hook folder
    cp pre-commit-hook.sh ../.git/hooks/pre-commit-hook.sh

### Step 2: create the pre-commit-hook script

I got some help from this gist to create the pre-commit-hook script. Moreover I used color codes for better readability. Each code has a meaning:

    #COLOR CODES:
    #tput setaf 3 = yellow -> Info
    #tput setaf 1 = red -> warning/not allowed commit
    #tput setaf 2 = green -> all good!/allowed commit

The script is roughly divided in 5 parts:

#### checking for non staged changes:

    git diff --quiet
    hadNoNonStagedChanges=$?

    if ! [ $hadNoNonStagedChanges -eq 0 ]
    then
        echo "* Stashing non-staged changes"
        git stash --keep-index -u > /dev/null
    fi

#### checking for compilation:

    (cd $DIR/; sbt test:compile)
    compiles=$?

    echo "* Compiles?"

    if [ $compiles -eq 0 ]
    then
        echo "* Yes"
    else
        echo "* No"
    fi

#### checking for formatting:


    (cd $DIR/; scalafmt)
    git diff --quiet
    formatted=$?

    echo "* Properly formatted?"

    if [ $formatted -eq 0 ]
    then
        echo "* Yes"
    else
        echo "* No"
        echo "The following files need formatting (in stage or commited):"
        git diff --name-only
        echo ""
        echo "Please run 'scalafmt' to format the code."
        echo ""
    fi

#### undoing formatting:

    git stash --keep-index > /dev/null
    git stash drop > /dev/null

Keep in mind that the goal of the pre-commit hook is not to format the code in your place, but to check if the code is properly formatted.

#### taking action basing on the results of the previous operations:

##### 1. if there are not staged changes, stash them:

    if ! [ $hadNoNonStagedChanges -eq 0 ]
    then
        echo "* Scheduling stash pop of previously stashed non-staged changes for 1 second after commit."
        sleep 1 && git stash pop --index > /dev/null & # sleep and & otherwise commit fails when this leads to a merge conflict
    fi

##### 2. if the code compiles and is properly formatted end program with no errors

    if [ $compiles -eq 0 ] && [ $formatted -eq 0 ]
    then
        echo "... done. Proceeding with commit."
        echo ""
        exit 0

##### 3. if the code compiles but is not formatted end program with error:

    elif [ $compiles -eq 0 ]
    then
        echo "... done."
        echo "CANCELLING commit due to NON-FORMATTED CODE."
        echo ""
        exit 1

##### 4. if the code doesn’t compile end the program with error:

    else
        echo "... done."
        echo "CANCELLING commit due to COMPILE ERROR."
        echo ""
        exit 2
    fi

Putting everything together and adding the color codes according to the different situations the [final scripts can be derived](https://gist.github.com/alodavi/c42da82b888869bf935242b1160e05b7#file-pre-commit-hook-install-sh).

In case the code compiles and is properly formatted, before committing you should see something like this:

<img class="responsive-img" src="/assets/img/2018/pre-commit-hook.png">

### Step 3: Set up the hook

Having the scripts in your project-root/hook folder per se doesn’t make the pre-commit hook effective. You need to give execute permission to the pre-commit-hook-install script:

    chmod +x pre-commit-hook-install.sh

Then run the install script:

    ./pre-commit-hook-install.sh

This will create a link with your local .git/hooks folder. If you now go there you will see that there are two new files: pre-commit and pre-commit-hook.sh As default bash files are not executable, so you will need to give them permission to execute with the chmod command.

    chmod +x pre-commit
    chmod +x pre-commit-hook.sh

Now your pre-commit hook should be active.

Please keep in mind that you need to have homebrew installed on your local machine to allow the pre-commit-hook-install.sh to install the scalafmt cli locally.

#### What to do if the code is not correctly formatted?

Just run:

    scalafmt

add the changes and try to commit again.

#### What if you want to commit not formatted code?

you can still do it running the command:

    git commit -m "your commit message here" --no-verify

Now that you’re all set, you should instruct your project collaborators on how to set up the pre-commit hook so that the whole team has the same workflow. Ideally you should document such process in a written form — like the README.md file.

## Updating the git pre-commit hook

Changes to the pre-commit hook script aren’t automatically installed locally. Indeed every time you change something you need to run the command again to make your changes effective:

    .\pre-commit-hook-install.sh

After you’ve successfully updated the script, remember to:

* make your project collaborators aware of the changes, so that they can update their local pre-commit-hook as well.
* update the README file.

## Conclusions

Using this solution at ZyseMe has made the development workflow smoother and improved the developer experience. Now the code looks better and more consistent with the styling rules we have established.

Overall, we have more control over our code style and avoid overthinking it.

I hope this article will help you to reach the same level of satisfaction. Happy committing!


From this [article](https://medium.com/zyseme-technology/code-formatting-scalafmt-and-the-git-pre-commit-hook-3de71d099514) originally posted through [ZyseMe](https://medium.com/zyseme-technology).
