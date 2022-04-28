---
title: Github workflow to publish a repo to a Databricks workspace
tags:
  - github
  - databricks
  - ci-cd
---

Here is a quick code snippet of a Github workflow that publishes (or updates, if it already exists) a repository to a Databricks workspace.

I assume you already configured your repository provider in the Databricks workspace: <https://docs.databricks.com/repos/index.html>

```yaml
# ------------------------------------------------------------
# GITHUB WORKFLOW TO PUBLISH/UPDATE A GIT REPO IN DATABRICKS WORKSPACE
# 
# In this workflow we publish a new repo or update an existing one
# in a given Databricks workspace.
# The workflow also allows to set 2 environments:
# - production
# - development
# The two environments can point to two different DB workspaces
# The workflow is triggered by commits and, depending on the branch,
# it uses one of the two environments.
# More specifically, the branch name defined in the var
# `PROD_BRANCH_NAME` dictates which branch is considered as production.
# 
# We assume the following secrets are defined:
# - DATABRICKS_HOST_PROD : Production workspace url
# - DATABRICKS_TOKEN_PROD : Token to access the production workspace
# - DATABRICKS_HOST_DEV : Development workspace url
# - DATABRICKS_TOKEN_DEV : Token to access the development workspace
# ------------------------------------------------------------

name: Databricks

# NOTE: a succesfull PR (i.e. that is approved and merged), generates
# a commit on the target branch, so we don't need to specify `pull_request`
on:
  push:
    # branches:
    #   - main
  
env:
  # Name of the branch that represents the production code
  # Usually it's `main` or `master`
  PROD_BRANCH_NAME: main
  # Path in the Databricks workspace where to store the repo
  DATABRICKS_REPO_PATH: "/Repos/folder/name"
  # Repo url
  DATABRICKS_REPO_URL: https://github.com/repo_url
  # See https://docs.databricks.com/dev-tools/cli/repos-cli.html#create-a-repo
  DATABRICKS_REPO_PROVIDER: gitHub

jobs:
  databricks:
    
    runs-on: ubuntu-latest
    
    steps:
      
      # ---------------------------------------
      # SET ENV VARS FOR DIFFERENT ENVIRONMENTS
      # We select the values for `host` and `token` depending on the branch.
      # If we are in `main` (or the value set in `PROD_BRANCH_NAME`), then we
      # retrieve the production values, otherwise we take the development ones.
      # This is achieved using the two conditional steps below.
      # As an example, in the condition
      # `${{ github.ref_name == env.PROD_BRANCH_NAME }}`
      # `github.ref_name` might have the `development` if we performed a commit in that branch.
      # https://docs.github.com/en/actions/learn-github-actions/contexts#github-context
        
      - name: Sets env vars for Production
        if: ${{ github.ref_name == env.PROD_BRANCH_NAME }}
        run: |
          echo "DATABRICKS_HOST=${{ secrets.DATABRICKS_HOST_PROD }}" >> $GITHUB_ENV
          echo "DATABRICKS_TOKEN=${{ secrets.DATABRICKS_TOKEN_PROD }}" >> $GITHUB_ENV
          
      - name: Sets env vars for Development
        if: ${{ github.ref_name != env.PROD_BRANCH_NAME }}
        run: |
          echo "DATABRICKS_HOST=${{ secrets.DATABRICKS_HOST_DEV }}" >> $GITHUB_ENV
          echo "DATABRICKS_TOKEN=${{ secrets.DATABRICKS_TOKEN_DEV }}" >> $GITHUB_ENV
      
      # ---------------------------------------
      # INSTALL DATABRICKS-CLI AND CONFIGURE IT

      - name: Install databricks-cli
        run: python -m pip install databricks-cli
        shell: bash
    
      - name: Set `.databrickscfg`
        run: | 
          cat > ~/.databrickscfg <<EOF 
          [DEFAULT] 
          host = ${{ env.DATABRICKS_HOST }}
          token = ${{ env.DATABRICKS_TOKEN }}
          EOF
      
      # ---------------------------------------
      # TEST DATABRICKS-CLI
      
      - name: List all repos
        run: databricks repos list
      
      # ---------------------------------------
      # PUBLISH SELECTED REPO
      #
      # Code breakdown:
      # - ERR=0
      #   We initialize the var `ERR` to 0. This var represents if the
      #   following command executes successfully (0) or not (1)
      # - `databricks repos get --path ${{ env.DATABRICKS_REPO_PATH }}`
      #   Tries to get details about repo with path `DATABRICKS_REPO_PATH`.
      #   If it does not exist, it fails with exist status -1.
      # - ... || ERR=1
      #   this part is executed if the previous command fails, in which
      #   case we assign 1 to ERR
      #   https://stackoverflow.com/questions/65754507/bash-pipe-get-the-exit-status-of-previous-process-in-pipeline
      # - `if [...]; then ...; else ...; fi`
      #   if-then-else statement, https://www.geeksforgeeks.org/bash-scripting-if-statement/
      # - `ERR -eq 0`
      #   Checks if `ERR` is equal to 0
      # - `echo "::set-output name=repo_exists::1"`
      #   Add to the outputs of the step the output `repo_exists` with value 1
      - name: Check for repo existence
        id: check_repo_existence
        run: |
          ERR=0
          databricks repos get --path ${{ env.DATABRICKS_REPO_PATH }} || ERR=1
          if [ $ERR -eq 0 ]; then echo "Repo exists"; else echo "Repo does not exist"; fi
          if [ $ERR -eq 0 ]; then echo "::set-output name=repo_exists::1"; else echo "::set-output name=repo_exists::0"; fi

      # Check if the step `check_repo_existence` output `` is 0,
      # meaning that the repo `DATABRICKS_REPO_PATH` does not exist
      # https://docs.github.com/en/actions/learn-github-actions/contexts#steps-context
      # In that case, we create the repository and switch to the branch
      # that triggered this workflow.
      # We use `${{ github.ref_name }}` to get the name of the branch
      # https://docs.github.com/en/actions/learn-github-actions/contexts#github-context
      - name: Repo does not exist, create it
        if: steps.check_repo_existence.outputs.repo_exists == 0
        run: |
          echo "Repo does not exist!"
          databricks repos create --url ${{ env.DATABRICKS_REPO_URL }} --provider ${{ env.DATABRICKS_REPO_PROVIDER }} --path ${{ env.DATABRICKS_REPO_PATH }}
          databricks repos update --path ${{ env.DATABRICKS_REPO_PATH }} --branch ${{ github.ref_name }}
      
      # If the repo already exists, we update it.
      # We use `${{ github.ref_name }}` to get the name of the branch
      # https://docs.github.com/en/actions/learn-github-actions/contexts#github-context
      - name: Repo exists, update it
        if: steps.check_repo_existence.outputs.repo_exists == 1
        run: |
          echo "Repo exists!"
          databricks repos update --path ${{ env.DATABRICKS_REPO_PATH }} --branch ${{ github.ref_name }}
```