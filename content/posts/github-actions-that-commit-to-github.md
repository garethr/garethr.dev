---
title: "Github Actions That Commit to Github"
date: 2019-09-14T16:22:54+01:00
---

I'm enjoying experimenting with GitHub Actions, and have now used the new beta for a number of projects and tasks. I've mainly
been interested in Actions for ad-hoc automation, either based on a schedule or on an event like pushing to master. Several of
those projects have involved having an Action generate some files and commit those back to GitHub. At the moment doing so requires
some jumping through hoops, so I thought it would be worthwhile documenting.

In my case I'm using the following in a few projects:

* Updating a Homebrew Formula when a new version of upstream software is released
* Generating a set of GitHub Actions and documentation from some input data

The following snippet commits back to GitHub, correctly authenticating with the repository and setting the user details for the commit.


```yaml
- name: Commit to repository
  env:
    GITHUB_TOKEN: ${{ secrets.github_token }}
    COMMIT_MSG: |
      <a commit message>
      skip-checks: true
  run: |
    # Hard-code user configuration
    git config user.email "<an email address>"
    git config user.name "<a name>"
    # Update origin with token
    git remote set-url origin https://x-access-token:${GITHUB_TOKEN}@github.com/${GITHUB_REPOSITORY}.git
    # Checkout the branch so we can push back to it
    git checkout master
    git add .
    # Only commit and push if we have changes
    git diff --quiet && git diff --staged --quiet || (git commit -m "${COMMIT_MSG}"; git push origin master)
```

It would be nice to have some sort of abstraction here, given that's quite a bit of code to achieve what we're wanting to do. It
would be nice for this to be build-in to GitHub Actions, as I can see this being something that's useful in lots of places. But
for the moment it's nice to know it's possible. It might even be nice to encapsulate this into an Action of it's own.

Thanks to [Steve Winton](https://twitter.com/stevewinton) for the original pointers that helped me get this working. Hopefully it comes in useful to others too.
