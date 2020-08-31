# Commiticons
![Workflow Status](https://img.shields.io/github/workflow/status/amorpheuz/commiticons/Generate%20Commiticon) ![License](https://img.shields.io/github/license/amorpheuz/commiticons) [![Twitter Follow](https://img.shields.io/twitter/follow/amorpheuz)](https://twitter.com/amorpheuz)

Bring your commits to life with unique Avatars for each of them! 

> "Why should users have all the fun?"

The concept of identicons has been around for long and many websites use it to generate a default avatar for users. Since all git commits already have unique hashes, why not generate identicons for them? I configured a small (but interesting) workflow to automatically generate unique avatars for your commits. I call these avatars **Commiticons**, identicons but for commits! 😬🎉

_A submission for the [DEV: GitHub Actions For Open Source!](https://dev.to/devteam/announcing-the-github-actions-hackathon-on-dev-3ljn) hackathon! Read more about commiticons on my blog post [Commiticons: Hashes are boring, Identify commits with Avatars via Actions](https://dev.to/amorpheuz/commiticons-hashes-are-boring-identify-commits-with-avatars-via-actions-5d88)._

### Demo
![](https://github.com/Amorpheuz/Commiticons/blob/main/commiticons-demo.gif)

## Table of Contents

- [Workflow File](#workflow-file)
- [Core Functionality](#core-functionality)
- [How does .github/workflows/commiticons.yml work?](#how-does-githubworkflowscommiticonsyml-work)
- [Credits](#credits)

## Workflow File

The workflow file for Commiticons can be found at [.github/workflows/commiticons.yml](https://github.com/Amorpheuz/Commiticons/blob/main/.github/workflows/commiticons.yml).

## Core Functionality

Commiticons are a concept, they are unique avatars for a specific commit. How the creation process is implemented is up to the developer to decide! Here is a basic way of implementing them:

1. Go to a GitHub Repo of your choice, where you have the required permissions, and create a `new workflow` from the Actions Tab. Select the `set up a workflow yourself` option.

2. Copy-paste the following template to the new file that gets created:

    > Edit the `name:` (name of the workflow) and `branches:` (branch for which commiticons are created) fields as per your repository requirements.

    ```yml
    name: CI

    on:
      push:
        branches: [ main ]

    jobs:
      build:
        runs-on: ubuntu-latest

        steps:

        - name: Add Commiticon as Commit Comment
          run: |
            curl \
            -X POST \
            -H "Accept: application/vnd.github.v3+json" \
            -H 'authorization: Bearer ${{ secrets.GITHUB_TOKEN }}' \
            $GITHUB_API_URL/repos/$GITHUB_REPOSITORY/commits/$GITHUB_SHA/comments \
            -d "{\"body\":\"![Commiticon](https://avatars.dicebear.com/api/human/$GITHUB_SHA.svg?h=250)\"}"
    ```
3. Once all the changes are done, commit the actions to the repo. You are done! 🎉 

Switch to the `Actions` tab to see the workflow in action. Once the workflow is complete, go to the latest commit of your repo in order to see the Commiticon as a comment!

## How does [.github/workflows/commiticons.yml](https://github.com/Amorpheuz/Commiticons/blob/main/.github/workflows/commiticons.yml) work?

1. First, I check-out my repo with [actions/checkout](https://github.com/actions/checkout) since I want to save the Commiticons to my repo.
   ```yml
   - name: Checkout
     uses: actions/checkout@v2
   ```

2. Next, I want my Commiticons to be accompanied by a joke to make them even more fun. So, I use the infamous [icanhazdadjoke API](https://icanhazdadjoke.com/api) to grab one for me via cURL.
   I also must perform a few clean-up steps on them to make it simple for me to push them to the GitHub as a commit comment later. They are:

   1. I clean up any new lines, `\n` or `\r\n`, in the joke to prevent problems when I later push it to GitHub with `cURL`. The [sed](https://www.gnu.org/software/sed/manual/sed.html) command replaces `\n` with the `<br>` tag. Next, the [tr](https://en.wikipedia.org/wiki/Tr_(Unix)) command deletes `\r` which is present in case of Windows Line Endings. Read more about [\r](https://en.wikipedia.org/wiki/Carriage_return#Computers), [\n](https://en.wikipedia.org/wiki/Newline#In_programming_languages) and [the problem of CRLF](https://www.hanselman.com/blog/CarriageReturnsAndLineFeedsWillUltimatelyBiteYouSomeGitTips.aspx).
   
   ```yml
   - name: Get a dad-joke
     run: |
       echo "::set-env name=TEMP::$(curl -H "Accept: text/plain" https://icanhazdadjoke.com/ | sed -r ':a;N;$!ba;s/\n/<br>/g' | tr -d '\r')"
   ```

   2. Next, I replace all " in the text with ', via tr, in order to avoid clashes with the " with the commands in the cURL command. The $TEMP is the env variable that saves output of previous command.
   
   ```yml
    - name: Escape double-quotes
      run: |
        echo "::set-env name=DAD_JOKE::$(echo "$TEMP" | tr \" \')"
   ```

   These commands are enclosed within echo and set-env to save them as Environment variables for future steps.

3. Now, I use the [GitHub REST API v3](https://docs.github.com/en/rest) to push them as a commit comment. Here are some more details about how to [Create a commit comment](https://docs.github.com/en/rest/reference/repos#create-a-commit-comment). The request is also [authenticated by using GITHUB_TOKEN](https://docs.github.com/en/actions/configuring-and-managing-workflows/authenticating-with-the-github_token).

   ```yml
   - name: Create commit comment
     run: |
       curl \
       -X POST \
       -H "Accept: application/vnd.github.v3+json" \
       -H 'authorization: Bearer ${{ secrets.GITHUB_TOKEN }}' \
       $GITHUB_API_URL/repos/$GITHUB_REPOSITORY/commits/$GITHUB_SHA/comments \
       -d "{\"body\":\"| ![Commiticon](https://avatars.dicebear.com/api/human/$GITHUB_SHA.svg?h=250) | $DAD_JOKE |\n|:-:|:-:|\"}"
   ```
   
   The process of generating the Commiticons is handled by [Dicebear Avatars](https://avatars.dicebear.com/), a hidden gem that I was glad I came across! They are the ❤ and soul of Commiticons.

4. Finally, I save the svg image to my repo which is already present here courtesy of actions/checkout in first step!

   I also need to push it to my repo via git. You might be going "Wait what? Isn't this already GitHub?". It's a Gotcha that had me surprised when I was new to Actions too! In simple terms, actions are their own computers which run automatically by following a script which we provide them (our workflow files).

   ```yml
   - name: Write Commiticon
     run: |
       curl https://avatars.dicebear.com/api/human/$(echo $GITHUB_SHA).svg?h=250 --output .commiticons/$(echo $GITHUB_SHA).svg
   
   - name: Push Changes
     run: |
       git config user.name Amorpheuz
       git config user.email connect@amorpheuz.dev
       git add .commiticons/
       git commit -m "Generated Commiticon for: $GITHUB_SHA"
       git push
   ```
   
## Credits

Here are all the projects that make this workflow possible, A huge shout out to all of them:

- [Dicebear Avatars](https://github.com/DiceBear/avatars) for the lovely avatars. 🖼
- [Icanhazdadjoke](https://icanhazdadjoke.com/) for the jokes! 🤣
- [actions/checkout](https://github.com/actions/checkout) for making repo interactions a breeze. 🏄🏽‍♂️
- The `sed`, `cURL` and `tr` commands. 👨🏽‍💻
- [GitHub Docs](https://docs.github.com/en) for all the knowledge! 🧠

_If you are confused about something, have some ideas, or just want to chat; reach out to me via my Twitter [@Amorpheuz](https://twitter.com/amorpheuz) or [open an Issue](https://github.com/Amorpheuz/Commiticons/issues/new)!_
