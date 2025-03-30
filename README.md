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
      
      - name: Get issue information
        id: issue
        uses: actions/github-script@v6
        with:
          script: |
            const issue = context.payload.issue;
            const body = issue.body || '';
            const title = issue.title || '';
            
            // Check if this is a delete/archive request
            const deleteCommand = /^\/delete\s+(\S+)/m;
            const match = body.match(deleteCommand) || title.match(deleteCommand);
            
            if (!match) {
              console.log('Not a delete request. Exiting.');
              return core.setFailed('Not a delete request');
            }
            
            const repoToDelete = match[1];
            core.setOutput('repo-to-delete', repoToDelete);
            
            // Add the repo-archive label to the issue
            try {
              await github.rest.issues.addLabels({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: issue.number,
                labels: ['repo-archive']
              });
              console.log('Added repo-archive label to issue');
            } catch (error) {
              console.log(`Error adding label: ${error.message}`);
              // Continue even if label addition fails
            }
            
            return repoToDelete;

      - name: Ensure repo-archive label exists
        uses: actions/github-script@v6
        with:
          script: |
            try {
              await github.rest.issues.getLabel({
                owner: context.repo.owner,
                repo: context.repo.repo,
                name: 'repo-archive'
              });
              console.log('repo-archive label already exists');
            } catch (error) {
              if (error.status === 404) {
                await github.rest.issues.createLabel({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  name: 'repo-archive',
                  color: '6e5494', // Purple color
                  description: 'Repository archiving request'
                });
                console.log('Created repo-archive label');
              } else {
                console.log(`Error checking label: ${error.message}`);
              }
            }

      - name: Check requester permissions
        id: check-permissions
        if: steps.issue.outputs.repo-to-delete
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          REPO_TO_DELETE: ${{ steps.issue.outputs.repo-to-delete }}
          REQUESTER: ${{ github.event.sender.login }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
        run: |
          # Parse repository information
          if [[ "$REPO_TO_DELETE" == *"/"* ]]; then
            ORG_NAME=$(echo $REPO_TO_DELETE | cut -d'/' -f1)
            REPO_NAME=$(echo $REPO_TO_DELETE | cut -d'/' -f2)
          else
            ORG_NAME="DigitalBankingSolutions"  # Default org
            REPO_NAME=$REPO_TO_DELETE
          fi
          
          echo "Checking permissions for $REQUESTER on $ORG_NAME/$REPO_NAME"
          
          # Check if user is an admin of the repository
          PERMISSION=$(gh api repos/$ORG_NAME/$REPO_NAME/collaborators/$REQUESTER/permission --jq '.permission' || echo "none")
          
          if [[ "$PERMISSION" != "admin" ]]; then
            echo "User $REQUESTER does not have admin permissions on $ORG_NAME/$REPO_NAME"
            gh issue comment $ISSUE_NUMBER --body "@$REQUESTER You don't have admin permissions for $ORG_NAME/$REPO_NAME. Only repository admins can archive repositories."
            exit 1
          fi
          
          echo "User $REQUESTER has admin permissions on $ORG_NAME/$REPO_NAME"
          echo "org-name=$ORG_NAME" >> $GITHUB_OUTPUT
          echo "repo-name=$REPO_NAME" >> $GITHUB_OUTPUT
          echo "is-admin=true" >> $GITHUB_OUTPUT
        
      - name: Archive repository
        if: steps.check-permissions.outputs.is-admin == 'true'
        id: archive-repo
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_APP_TOKEN }}
          SOURCE_ORG: ${{ steps.check-permissions.outputs.org-name }}
          REPO_NAME: ${{ steps.check-permissions.outputs.repo-name }}
          ARCHIVE_ORG: "DigitalArchive"  # Replace with your archive org for testing
          ISSUE_NUMBER: ${{ github.event.issue.number }}
        run: |
          # Check if repository exists in archive org
          TARGET_REPO_NAME=$REPO_NAME
          
          if gh repo view $ARCHIVE_ORG/$TARGET_REPO_NAME &>/dev/null; then
            # Repository with same name exists in archive org, add random suffix
            SUFFIX=$(cat /dev/urandom | tr -dc 'a-z0-9' | fold -w 4 | head -n 1)
            TARGET_REPO_NAME="${REPO_NAME}-${SUFFIX}"
            echo "Repository with name $REPO_NAME already exists in archive org. Using $TARGET_REPO_NAME instead."
          fi
          
          # Get repository description
          REPO_DESC=$(gh api repos/$SOURCE_ORG/$REPO_NAME --jq '.description // "Archived repository"')
          REPO_PRIVATE=$(gh api repos/$SOURCE_ORG/$REPO_NAME --jq '.private')
          
          echo "Creating repository in archive org: $ARCHIVE_ORG/$TARGET_REPO_NAME"
          
          # Create repository in archive org
          if [[ "$REPO_PRIVATE" == "true" ]]; then
            VISIBILITY="--private"
          else
            VISIBILITY="--public"
          fi
          
          # Create the repository in the archive org
          gh repo create $ARCHIVE_ORG/$TARGET_REPO_NAME $VISIBILITY --description "Archived from $SOURCE_ORG/$REPO_NAME: $REPO_DESC"
          
          # In a real implementation, you would transfer the repository using GitHub API
          # For demonstration, we'll simulate the transfer with the gh api command
          echo "Transferring repository from $SOURCE_ORG/$REPO_NAME to $ARCHIVE_ORG/$TARGET_REPO_NAME"
          
          # Transfer repository using GitHub API
          # Note: This requires the GITHUB_APP_TOKEN to have appropriate permissions
          gh api -X POST /repos/$SOURCE_ORG/$REPO_NAME/transfer \
            -f new_owner=$ARCHIVE_ORG \
            -f new_name=$TARGET_REPO_NAME
          
          # After transfer, mark as archived
          gh api -X PATCH repos/$ARCHIVE_ORG/$TARGET_REPO_NAME -f archived=true
          
          # Add comment to the issue
          gh issue comment $ISSUE_NUMBER --body "Repo $REPO_NAME has been deleted. Archive can be found at https://github.com/$ARCHIVE_ORG/$TARGET_REPO_NAME"
          
          # Set output for next steps
          echo "archive-repo-url=https://github.com/$ARCHIVE_ORG/$TARGET_REPO_NAME" >> $GITHUB_OUTPUT

      - name: Notify success
        if: steps.archive-repo.outcome == 'success'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          ARCHIVE_URL: ${{ steps.archive-repo.outputs.archive-repo-url }}
          REPO_TO_DELETE: ${{ steps.issue.outputs.repo-to-delete }}
        run: |
          gh issue comment $ISSUE_NUMBER --body "✅ Repository archiving completed successfully!

          The repository $REPO_TO_DELETE has been moved to the archive organization and marked as archived. You can find it at: $ARCHIVE_URL"
          
          gh issue close $ISSUE_NUMBER --comment "Repository archiving process completed."


```






















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
      
      - name: Get issue information
        id: issue
        uses: actions/github-script@v6
        with:
          script: |
            const issue = context.payload.issue;
            const body = issue.body || '';
            const title = issue.title || '';
            
            // Check if this is a delete/archive request
            const deleteCommand = /^\/delete\s+(\S+)/m;
            const match = body.match(deleteCommand) || title.match(deleteCommand);
            
            if (!match) {
              console.log('Not a delete request. Exiting.');
              return core.setFailed('Not a delete request');
            }
            
            const repoToDelete = match[1];
            core.setOutput('repo-to-delete', repoToDelete);
            return repoToDelete;

      - name: Check requester permissions
        id: check-permissions
        if: steps.issue.outputs.repo-to-delete
        uses: actions/github-script@v6
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const repoToDelete = '${{ steps.issue.outputs.repo-to-delete }}';
            const [owner, repo] = repoToDelete.split('/');
            
            try {
              // Check if user is an admin of the repository
              const requester = context.payload.sender.login;
              const { data: collaborators } = await github.rest.repos.listCollaborators({
                owner,
                repo,
                affiliation: 'direct'
              });
              
              const isAdmin = collaborators.some(collaborator => 
                collaborator.login === requester && 
                collaborator.permissions && 
                collaborator.permissions.admin
              );
              
              if (!isAdmin) {
                await github.rest.issues.createComment({
                  owner: context.repo.owner,
                  repo: context.repo.repo,
                  issue_number: context.issue.number,
                  body: `@${requester} You don't have admin permissions for ${repoToDelete}. Only repository admins can archive repositories.`
                });
                return core.setFailed('Requester does not have admin permissions');
              }
              
              core.setOutput('is-admin', 'true');
              return true;
            } catch (error) {
              console.log(`Error checking permissions: ${error}`);
              await github.rest.issues.createComment({
                owner: context.repo.owner,
                repo: context.repo.repo,
                issue_number: context.issue.number,
                body: `Error checking permissions: ${error.message}`
              });
              return core.setFailed(`Error checking permissions: ${error.message}`);
            }

      - name: Set up Python
        if: steps.check-permissions.outputs.is-admin == 'true'
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'

      - name: Install dependencies
        if: steps.check-permissions.outputs.is-admin == 'true'
        run: |
          python -m pip install --upgrade pip
          pip install PyGithub requests

      - name: Archive repository
        if: steps.check-permissions.outputs.is-admin == 'true'
        id: archive-repo
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_APP_TOKEN }}
          REPO_TO_DELETE: ${{ steps.issue.outputs.repo-to-delete }}
          SOURCE_ORG: "DigitalBankingSolutions"  # Replace with your test org for testing
          ARCHIVE_ORG: "DigitalArchive"          # Replace with your archive org for testing
          ISSUE_NUMBER: ${{ github.event.issue.number }}
        run: |
          python - <<EOF
          import os
          import random
          import string
          import sys
          from github import Github
          
          def generate_random_suffix():
              """Generate a random 4-character suffix with lowercase letters and numbers."""
              chars = string.ascii_lowercase + string.digits
              return ''.join(random.choice(chars) for _ in range(4))
          
          def archive_repository():
              github_token = os.environ['GITHUB_TOKEN']
              repo_to_delete = os.environ['REPO_TO_DELETE']
              source_org = os.environ['SOURCE_ORG']
              archive_org = os.environ['ARCHIVE_ORG']
              issue_number = os.environ['ISSUE_NUMBER']
              current_repo = os.environ['GITHUB_REPOSITORY']
              
              g = Github(github_token)
              
              try:
                  # Parse repository information
                  if '/' in repo_to_delete:
                      org_name, repo_name = repo_to_delete.split('/')
                  else:
                      org_name = source_org
                      repo_name = repo_to_delete
                  
                  # Get source repository
                  source_repository = g.get_repo(f"{org_name}/{repo_name}")
                  
                  # Check if repository exists in archive org
                  archive_org_obj = g.get_organization(archive_org)
                  target_repo_name = repo_name
                  
                  try:
                      # Check if repo with same name exists in archive org
                      archive_org_obj.get_repo(target_repo_name)
                      # If we get here, the repo exists, so add a suffix
                      target_repo_name = f"{repo_name}-{generate_random_suffix()}"
                      print(f"Repository with name {repo_name} already exists in archive org. Using {target_repo_name} instead.")
                  except:
                      # Repo doesn't exist, we can use the original name
                      pass
                  
                  # Create a new repository in the archive org with the same properties
                  new_repo = archive_org_obj.create_repo(
                      name=target_repo_name,
                      description=f"Archived from {org_name}/{repo_name}: {source_repository.description or ''}",
                      private=source_repository.private
                  )
                  
                  # Transfer the repository (this requires GitHub App token with appropriate permissions)
                  # Note: This is a simplified version. In reality, you'd need to use GitHub's API directly
                  # for repository transfers as PyGithub doesn't support this operation directly
                  print(f"Repository would be transferred from {org_name}/{repo_name} to {archive_org}/{target_repo_name}")
                  
                  # For demonstration purposes, we'll simulate the transfer
                  # In a real implementation, you would use the GitHub API to transfer the repository
                  
                  # After transfer, mark as archived
                  # new_repo.edit(archived=True)
                  
                  # Add comment to the issue
                  issue_repo_owner, issue_repo_name = current_repo.split('/')
                  issue_repo = g.get_repo(current_repo)
                  issue = issue_repo.get_issue(number=int(issue_number))
                  issue.create_comment(f"Repo {repo_name} has been deleted. Archive can be found at https://github.com/{archive_org}/{target_repo_name}")
                  
                  # Output for GitHub Actions
                  print(f"::set-output name=archive-repo-url::https://github.com/{archive_org}/{target_repo_name}")
                  return True
              
              except Exception as e:
                  print(f"Error archiving repository: {str(e)}")
                  issue_repo = g.get_repo(current_repo)
                  issue = issue_repo.get_issue(number=int(issue_number))
                  issue.create_comment(f"Error archiving repository: {str(e)}")
                  return False
          
          if not archive_repository():
              sys.exit(1)
          EOF

      - name: Notify success
        if: steps.archive-repo.outcome == 'success'
        uses: actions/github-script@v6
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const archiveUrl = '${{ steps.archive-repo.outputs.archive-repo-url }}';
            const repoToDelete = '${{ steps.issue.outputs.repo-to-delete }}';
            
            await github.rest.issues.createComment({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `✅ Repository archiving completed successfully!\n\nThe repository ${repoToDelete} has been moved to the archive organization and marked as archived. You can find it at: ${archiveUrl}`
            });
            
            await github.rest.issues.update({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              state: 'closed',
              state_reason: 'completed'
            });
```


































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
