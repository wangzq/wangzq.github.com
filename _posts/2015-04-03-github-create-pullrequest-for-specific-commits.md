---
layout: post
category: notes
tags : [github]
---

Following is an example using github, but should be applicable for general Git usage.

1. I cloned from a github repository; assume it is `https://github.com/my/repo` forked from `https://github.com/origin/repo`
2. I made a few changes and pushed it over to my own master branch
3. I fixed a bug and pushed it over to my own master branch
4. Now I want to send a PullRequest for the bug I fixed above to the original repo I cloned from, if I use github UI to create a PR I will see changes both from step 2 and 3 since I didn't used a separate branch to fix the bug.

The solution is like following:

	Git remote add upstream https://github.com/origin/repo
	Git remote update
	Git checkout -b upstream upstream/master
	Git cherry-pick <sha hash of commit>
	Git push origin upstream

Now I can switch to the `upstream` branch in my github UI and create a PR.

Reference:

 * http://stackoverflow.com/questions/5256021/send-a-pull-request-on-github-for-only-latest-commit


