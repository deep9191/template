```yaml

name: Repository Archive IssueOps

on:
  issues:
    types: [opened, edited]
  issue_comment:
    types: [created, edited]

jobs:
  process-archive-request:
    runs-on: ubuntu-latest
    # Only run on issues, not PRs
    if: ${{ github.event.issue && !github.event.issue.pull_request }}
    
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
      
      - name: Generate GitHub App token
        id: generate-token
        uses: tibdex/github-app-token@v1
        with:
          app_id: ${{ secrets.APP_ID }}
          private_key: ${{ secrets.APP_PRIVATE_KEY }}
      
      - name: Setup GitHub CLI
        run: |
          echo "${{ steps.generate-token.outputs.token }}" > gh_token.txt
          gh auth login --with-token < gh_token.txt
          rm gh_token.txt
        env:
          GH_TOKEN: ${{ steps.generate-token.outputs.token }}
      
      - name: Get issue information
        id: issue-info
        run: |
          # Get issue details
          ISSUE_NUMBER="${{ github.event.issue.number }}"
          ISSUE_BODY=$(gh issue view $ISSUE_NUMBER --json body -q .body)
          
          # If this is a comment event, get the comment body
          if [[ "${{ github.event_name }}" == "issue_comment" ]]; then
            COMMENT_ID="${{ github.event.comment.id }}"
            COMMENT_BODY=$(gh api repos/${{ github.repository }}/issues/comments/$COMMENT_ID --jq .body)
          else
            COMMENT_BODY=""
          fi
          
          # Check for delete command in comment or issue body
          DELETE_COMMAND_REGEX="^/delete-repo[[:space:]]+([[:alnum:]_-]+)"
          
          if [[ "$COMMENT_BODY" =~ $DELETE_COMMAND_REGEX ]]; then
            REPO_TO_DELETE="${BASH_REMATCH[1]}"
            IS_DELETE_COMMAND="true"
          elif [[ "${{ github.event.action }}" == "opened" ]] && [[ "${{ contains(github.event.issue.labels.*.name, 'archive-request') }}" == "true" ]]; then
            # Parse form data from issue template
            REPO_TO_DELETE=$(echo "$ISSUE_BODY" | grep -A 1 "### Repository Name" | tail -n 1 | xargs)
            if [[ -n "$REPO_TO_DELETE" ]]; then
              IS_DELETE_COMMAND="true"
            else
              IS_DELETE_COMMAND="false"
            fi
          else
            IS_DELETE_COMMAND="false"
            REPO_TO_DELETE=""
          fi
          
          echo "is-delete-command=$IS_DELETE_COMMAND" >> $GITHUB_OUTPUT
          echo "repo-to-delete=$REPO_TO_DELETE" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ steps.generate-token.outputs.token }}
      
      - name: Verify admin permissions
        if: ${{ steps.issue-info.outputs.is-delete-command == 'true' }}
        id: verify-permissions
        run: |
          REPO_TO_DELETE="${{ steps.issue-info.outputs.repo-to-delete }}"
          REQUESTER="${{ github.event.issue.user.login }}"
          ISSUE_NUMBER="${{ github.event.issue.number }}"
          
          # Check if user has admin permissions
          PERMISSION=$(gh api repos/${{ github.repository_owner }}/$REPO_TO_DELETE/collaborators/$REQUESTER/permission --jq .permission || echo "error")
          
          if [[ "$PERMISSION" == "admin" ]]; then
            echo "is-admin=true" >> $GITHUB_OUTPUT
          else
            echo "is-admin=false" >> $GITHUB_OUTPUT
            
            # Add comment about insufficient permissions
            if [[ "$PERMISSION" == "error" ]]; then
              gh issue comment $ISSUE_NUMBER --body "Error verifying permissions. Please make sure the repository exists and is accessible."
            else
              gh issue comment $ISSUE_NUMBER --body "@$REQUESTER You don't have admin permissions on the repository \`$REPO_TO_DELETE\`. Only repository admins can archive repositories."
            fi
          fi
        env:
          GH_TOKEN: ${{ steps.generate-token.outputs.token }}
      
      - name: Process archive request
        if: ${{ steps.issue-info.outputs.is-delete-command == 'true' && steps.verify-permissions.outputs.is-admin == 'true' }}
        id: archive-repo
        run: |
          REPO_TO_DELETE="${{ steps.issue-info.outputs.repo-to-delete }}"
          SOURCE_ORG="${{ github.repository_owner }}"
          ARCHIVE_ORG="DigitalArchive"  # Replace with your actual archive org name
          ISSUE_NUMBER="${{ github.event.issue.number }}"
          
          # Add comment that we're processing the request
          gh issue comment $ISSUE_NUMBER --body "Processing archive request for repository \`$REPO_TO_DELETE\`..."
          
          # Check if repo with same name exists in archive org
          TARGET_REPO_NAME="$REPO_TO_DELETE"
          REPO_EXISTS=$(gh api repos/$ARCHIVE_ORG/$TARGET_REPO_NAME --silent || echo "not_found")
          
          if [[ "$REPO_EXISTS" != "not_found" ]]; then
            # Generate random suffix
            SUFFIX="-$(cat /dev/urandom | tr -dc 'a-z0-9' | fold -w 4 | head -n 1)"
            TARGET_REPO_NAME="${REPO_TO_DELETE}${SUFFIX}"
            
            # Log the name change
            gh issue comment $ISSUE_NUMBER --body "A repository with the name \`$REPO_TO_DELETE\` already exists in the archive organization. Will use name \`$TARGET_REPO_NAME\` instead."
          fi
          
          # Transfer the repository
          TRANSFER_RESULT=$(gh api -X POST repos/$SOURCE_ORG/$REPO_TO_DELETE/transfer \
            -f new_owner="$ARCHIVE_ORG" \
            -f new_name="$TARGET_REPO_NAME" || echo "transfer_failed")
          
          if [[ "$TRANSFER_RESULT" == "transfer_failed" ]]; then
            gh issue comment $ISSUE_NUMBER --body "Error transferring repository. Please check if the repository exists and you have the necessary permissions."
            exit 1
          fi
          
          # Archive the repository in its new location
          gh api -X PATCH repos/$ARCHIVE_ORG/$TARGET_REPO_NAME -f archived=true
          
          # Get the new repository URL
          ARCHIVED_REPO_URL="https://github.com/$ARCHIVE_ORG/$TARGET_REPO_NAME"
          
          # Add comment with the result
          gh issue comment $ISSUE_NUMBER --body "Repo \`$REPO_TO_DELETE\` has been deleted. Archive can be found at $ARCHIVED_REPO_URL"
          
          # Close the issue
          gh issue close $ISSUE_NUMBER
        env:
          GH_TOKEN: ${{ steps.generate-token.outputs.token }}



```



```yaml
name: Repository Archive Request
description: Request to archive a repository by moving it to the Digital Archive organization
title: "[ARCHIVE] Repository archive request for: "
labels: ["archive-request"]
assignees: []

body:
  - type: markdown
    attributes:
      value: |
        ## Repository Archive Request
        
        This form will help you request archiving a repository by moving it from the current organization to the Digital Archive organization.
        
        **Note:** You must have admin permissions on the repository to request archiving.

  - type: input
    id: repository-name
    attributes:
      label: Repository Name
      description: The name of the repository you want to archive
      placeholder: e.g., my-project-repo
    validations:
      required: true

  - type: textarea
    id: reason
    attributes:
      label: Reason for Archiving
      description: Please provide a brief explanation for why this repository should be archived
      placeholder: This project is no longer active and should be archived for historical reference.
    validations:
      required: true

  - type: checkboxes
    id: confirmation
    attributes:
      label: Confirmation
      description: Please confirm the following before submitting
      options:
        - label: I have admin permissions on this repository
          required: true
        - label: I understand that this repository will be moved to the Digital Archive organization
          required: true
        - label: I understand that the repository will be marked as archived after the move
          required: true

  - type: markdown
    attributes:
      value: |
        ## Command
        
        After submitting this issue, the workflow will process your request automatically.
        Alternatively, you can trigger the archiving process by adding the following command in a comment:
        
        ```
        /delete-repo repository-name
        ```
```
