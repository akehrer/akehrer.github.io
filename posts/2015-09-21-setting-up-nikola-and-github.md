.. title: Setting up Nikloa for Personal Github Sites
.. slug: setting-up-nikola-and-github
.. date: 2015-09-21 15:01:00 UTC-04:00
.. tags: python,nikola,github
.. category: 
.. link: 
.. description: 
.. type: text

After a bit of a struggle I finally moved my personal Github site over from Jekyll to [Nikola](https://getnikola.com/). I'm documenting what I did here for my future self and in case it's useful to anyone else.

# Github setup
- Create `source` branch in addition to `master`, all of the Nickola stuff will happen here. The `master` branch is where the static site will be served from and the output from `nikola build` will be put there using `ghp-import`. 
- Checkout the `source` branch and initialize your Nikola site (see [Nikola getting started][nkstart])
- You can now do all the things you need to do with Nikola (pages, posts, theme, [etc.](https://getnikola.com/handbook.html))


# Syncing site
I am having trouble getting `nikola github_deploy` working with my setup (Python 3.4, Window 7, in a virtualenv) so here is how I am doing it manually.

- Make sure [`ghp-import`](https://github.com/davisp/ghp-import) is installed 
    - If you are getting `TypeError: Type str doesn't support the buffer API` check [here] for the fix.
- Use `ghp-import.py -n -b master output`, this will
    - Update the `master` branch with what's in `./output/`
    - Include a `.nojekyll` file to signal to Github that `master` is not Jekyll based
- If you can use `nikola deploy` you will probably need to edit the following:
``` python
GITHUB_SOURCE_BRANCH = 'source'  # changed from 'master'
GITHUB_DEPLOY_BRANCH = 'master'  # changed from 'gh-pages'
```
    
    
Thanks to John Foster and his [johntfoster.github.io](https://github.com/johntfoster/johntfoster.github.io) repository for the example.
    
[nkstart]: https://getnikola.com/getting-started.html
[here]: https://github.com/w1ld/ghp-import/commit/0b575dfcdd459594518c66e9635d6d155397c219