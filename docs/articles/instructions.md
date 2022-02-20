---
title: GitHub Pages + MkDocs + GitHub Actions = The lazy Dev dream for publishing!
description: How I created this website.
---

## "Hello MkDocs!" website

- create empty repo `{git username}.github.io`
  For example, in my case it's `stawo.github.io`
- create the `.gitignore` file
- from the web interface, create the file `.gitignore`, just add a space in it (otherwise you cannot save it), and commit it.  
  This step is useful to initiate the repo and avoid having to do it locally on the computer later on.
- (optional, but strongly suggested) create conda env with `mkdocs` installed and activate it  
  `conda create --name stawo.github.io python=3.10 mkdocs-material`
  `conda activate stawo.github.io`
  - if you don't have/want `conda`, just run `pip install mkdocs`
- clone repo on computer and access repo root folder.
  In my case, open the terminal, go to my repositories folder, and run:  
  `git clone https://github.com/stawo/stawo.github.io.git`  
  `cd stawo/stawo.github.io.git`
- in the teminal, run the command `mkdocs new .`, as we want `mkdocs` to setup the root folder
  Test it by running `mkdocs serve`
- build the website by running `mkdocs build`. This generates the folder `site/` with many files in it.
- Add `site/` to `.gitignore`, so that we can build locally with `mkdocs` and avoid aving the generated file in git.
  Test it by running `git status` and see that the generated files in the folder `site/` do not appear. You should see only `docs/` and `mkdocs.yml`
- create folder `.github/workflows` and add file `build_site.yml` with following code (from https://github.com/marketplace/actions/deploy-mkdocs):
  ```
  name: Publish site via GitHub Pages
    on:
    push:
        branches:
        - main

    jobs:
    build:
        name: Deploy docs
        runs-on: ubuntu-latest
        steps:
        - name: Checkout main
            uses: actions/checkout@v2

        - name: Deploy docs
            uses: mhausenblas/mkdocs-deploy-gh-pages@master
            ## Or use mhausenblas/mkdocs-deploy-gh-pages@nomaterial to build without the mkdocs-material theme
            env:
            GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    ```
- commit changes.
  This will trigger the CICD pipeline.
  If all goes fine, the branch `gh-pages` will be created with the mkdocs build of our website.  
  Check the progress of the pipeline on the repo website under `Actions`
- go on github website of the repo, go to `Settings->Pages` and select the branch `gh-pages` as the source
- the website will be visible at the address `https://{git username}.github.io/`
  For example, in my case it's `https://stawo.github.io/`.  
  **Note**: it takes a little bit of time for GitHub to pick up changes, so don't worry if you get an error, just retry to refresh the page.

Resources:
- https://squidfunk.github.io/mkdocs-material
- https://www.mkdocs.org/
- https://github.com/marketplace/actions/deploy-mkdocs
- 

## Power-up: Blog!

https://liang2kl.github.io/mkdocs-blogging-plugin/

- install plugin  
  ```
  pip install mkdocs-blogging-plugin
  ```
- set up plugin  
  `mkdocs.yml` should look like this:  
  ```
  site_name: My Docs
  
  site_url: https://stawo.github.io/

  plugins:
    - search
    - blogging:
      dirs:
        - articles
  ```
  - re-eable search bar
    https://squidfunk.github.io/mkdocs-material/setup/setting-up-site-search/#built-in-search
- create folder
- fix cicd
  - create file `requirements.txt` and add the line:  
    ```
    mkdocs-blogging-plugin
    ```
  - modify `build_site.yml`

## Power-up: Write first content

## Power-up: customize theme

Resources:
- https://squidfunk.github.io/mkdocs-material/

## Power-up: add linter
