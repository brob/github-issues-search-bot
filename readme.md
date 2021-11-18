# GitHub Recommended Issues Bot

When users leave an Issue on a repository, getting timely feedback is critical. If their Issue is actually a duplicate or something that other contributors/users are talking about, it would be good for them to know this.

This bot uses Algolia to index and perform searches against a repository's Issue history to leave a selection of possibly related Issues on a newly opened Issue.

## Usage

The workflow depends on three different Actions. 

1. [An action to search Algolia Records](https://github.com/marketplace/actions/get-algolia-issue-records)
2. [An action to leave a comment based on the search results](https://github.com/marketplace/actions/create-or-update-comment)
3. [An action to ingest the inciting Issue into the Algolia Index](https://github.com/marketplace/actions/create-or-update-algolia-index-record)

```yaml
# This is a basic workflow to help you get started with Actions
name: related-issues

# Controls when the workflow will run
on:
  # Triggers the workflow on push or pull request events but only for the main branch
  issues:
    types: 
      - opened

# A workflow run is made up of one or more jobs that can run sequentially or in parallel
jobs:
  get-related-issues:
    permissions: 
      issues: write
    runs-on: ubuntu-latest
    steps:
      - id: search
        name: Search based on issue title
        uses: brob/algolia-issue-search@v1.0
        with: 
          app_id: ${{ secrets.ALGOLIA_APP_ID }}
          api_key: ${{ secrets.ALGOLIA_API_KEY }}
          index_name: ${{ github.event.repository.name }}
          issue_title: ${{ github.event.issue.title }}
      - name: Create or Update Comment
        uses: peter-evans/create-or-update-comment@v1.4.5
        with:
          # GITHUB_TOKEN or a repo scoped PAT.
          token: ${{ github.token }}
          # The number of the issue or pull request in which to create a comment.
          issue-number: ${{ github.event.issue.number }}
          # The comment body.
          body: |
            # While you wait, here are related issues:
            ${{ steps.search.outputs.issues_list }}
            just a test of post comment
      - name: Add Algolia Record
        id: ingest
        uses: chuckmeyer/add-algolia-record@v0

        with:
          app_id: ${{ secrets.ALGOLIA_APP_ID }}
          api_key: ${{ secrets.ALGOLIA_API_KEY }}
          index_name: ${{ github.event.repository.name }}
          record: |
            {
              "title": "${{ github.event.issue.title }}", 
              "url": "${{ github.event.issue.html_url }}", 
              "labels": "${{ github.event.issue.labels }}",
              "objectID": "${{ github.event.issue.number }}"
            }
```