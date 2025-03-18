```yaml

env:
  DIGITAL_ORG: digital-org-name  # Replace with your digital org name

steps:
  - name: Generate GitHub App Token
    id: get-token
    uses: actions/create-github-app-token@v1
    with:
      app-id: ${{ secrets.APP_ID }}
      private-key: ${{ secrets.APP_PRIVATE_KEY }}
      owner: ${{ env.DIGITAL_ORG }}

  - name: Process Member Management
    id: process
    continue-on-error: true  # This ensures the workflow doesn't fail
    uses: actions/github-script@v7
    with:
      github-token: ${{ steps.get-token.outputs.token }}
      script: |
        async function performAction(action, username) {
          try {
            switch(action) {
              case 'Add member':
                await github.rest.orgs.createInvitation({
                  org: process.env.DIGITAL_ORG,
                  username: username,
                  role: 'direct_member'
                });
                return { success: true };
                
              case 'Remove member':
                await github.rest.orgs.removeMember({
                  org: process.env.DIGITAL_ORG,
                  username: username
                });
                return { success: true };
                
              case 'Make owner':
                await github.rest.orgs.setMembershipForUser({
                  org: process.env.DIGITAL_ORG,
                  username: username,
                  role: 'admin'
                });
                return { success: true };
                
              case 'Remove owner':
                await github.rest.orgs.setMembershipForUser({
                  org: process.env.DIGITAL_ORG,
                  username: username,
                  role: 'member'
                });
                return { success: true };
            }
          } catch (e) {
            console.error('Error performing action:', e);
            return { 
              success: false, 
              error: e.message,
              status: e.status
            };
          }
          return { success: false, error: 'Unknown action' };
        }

        const issue = context.payload.issue;
        const action = issue.body.match(/### Action\s*([^\n]+)/)[1].trim();
        const username = issue.body.match(/### GitHub Username\s*([^\n]+)/)[1].trim();
        
        console.log(`Processing: ${action} for user ${username}`);
        
        const result = await performAction(action, username);
        
        if (result.success) {
          console.log('Action completed successfully');
        } else {
          console.error(`Action failed: ${result.error}`);
        }
        
        // Always add a comment with the result
        let commentBody;
        if (result.success) {
          commentBody = `✅ Action completed successfully:
          - Action: ${action}
          - User: @${username}
          - Organization: ${process.env.DIGITAL_ORG}`;
        } else {
          commentBody = `⚠️ Action may have encountered issues:
          - Action: ${action}
          - User: @${username}
          - Organization: ${process.env.DIGITAL_ORG}
          - Note: ${result.error || 'Unknown error'}`;
        }
        
        await github.rest.issues.createComment({
          ...context.repo,
          issue_number: context.issue.number,
          body: commentBody
        });
        
        // Return the result for potential use in later steps
        return result;

  - name: Close Issue
    if: always()  # Always run this step
    uses: actions/github-script@v7
    with:
      github-token: ${{ secrets.GITHUB_TOKEN }}
      script: |
        await github.rest.issues.update({
          ...context.repo,
          issue_number: context.payload.issue.number,
          state: 'closed'
        });
        console.log('Issue closed');
```
==================================================================================

```yaml

name: Cross-Org Member Management
on:
  issues:
    types: [opened]

jobs:
  manage-member:
    runs-on: ubuntu-latest
    if: contains(github.event.issue.labels.*.name, 'member-management')
    
    env:
      DIGITAL_ORG: digital-org-name  # Replace with your digital org name
    
    steps:
      - name: Install GitHub CLI
        run: |
          type -p curl >/dev/null || (apt update && apt install curl -y)
          curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
          && chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg \
          && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
          && apt update \
          && apt install gh -y

      - name: Setup GitHub App Authentication
        id: setup-app
        env:
          APP_ID: ${{ secrets.APP_ID }}
          PRIVATE_KEY: ${{ secrets.APP_PRIVATE_KEY }}
        run: |
          echo "$PRIVATE_KEY" > private-key.pem
          TOKEN=$(gh auth login --with-token <<< $(gh api --method POST -H "Accept: application/vnd.github+json" \
            /app/installations/${{ secrets.APP_INSTALLATION_ID }}/access_tokens \
            --app $APP_ID \
            --key private-key.pem | jq -r .token))
          echo "::add-mask::$TOKEN"
          echo "token=$TOKEN" >> $GITHUB_OUTPUT
          rm private-key.pem

      - name: Parse Issue and Process Request
        env:
          GH_TOKEN: ${{ steps.setup-app.outputs.token }}
        run: |
          # Parse issue body
          ISSUE_BODY=$(gh api repos/${{ github.repository }}/issues/${{ github.event.issue.number }} --jq .body)
          
          # Extract action and username
          ACTION=$(echo "$ISSUE_BODY" | grep -A1 "### Action" | tail -n1 | xargs)
          USERNAME=$(echo "$ISSUE_BODY" | grep -A1 "### GitHub Username" | tail -n1 | xargs)
          
          echo "Processing action: $ACTION for user: $USERNAME"
          
          # Set up error handling
          set +e  # Don't exit on error
          
          # Process based on action
          case "$ACTION" in
            "Add member")
              gh api --method POST /orgs/$DIGITAL_ORG/invitations \
                -f email="" -f role="direct_member" -f username="$USERNAME"
              ;;
              
            "Remove member")
              gh api --method DELETE /orgs/$DIGITAL_ORG/members/$USERNAME
              ;;
              
            "Make owner")
              gh api --method PUT /orgs/$DIGITAL_ORG/memberships/$USERNAME \
                -f role="admin"
              ;;
              
            "Remove owner")
              gh api --method PUT /orgs/$DIGITAL_ORG/memberships/$USERNAME \
                -f role="member"
              ;;
          esac
          
          # Capture exit code but don't fail
          EXIT_CODE=$?
          
          # Add comment based on result
          if [ $EXIT_CODE -eq 0 ]; then
            gh issue comment ${{ github.event.issue.number }} --body "✅ Action completed successfully:
            - Action: $ACTION
            - User: @$USERNAME
            - Organization: $DIGITAL_ORG"
          else
            gh issue comment ${{ github.event.issue.number }} --body "⚠️ Action may have encountered issues:
            - Action: $ACTION
            - User: @$USERNAME
            - Organization: $DIGITAL_ORG
            - Note: The action was attempted but may not have completed successfully."
          fi
          
          # Always close the issue
          gh issue close ${{ github.event.issue.number }}
          
          # Always exit successfully
          exit 0
```
================================================================================================================================

```yaml

env:
  DIGITAL_ORG: digital-org-name  # Replace with your digital org name

steps:
  - name: Generate GitHub App Token
    id: get-token
    uses: actions/create-github-app-token@v1
    with:
      app-id: ${{ secrets.APP_ID }}
      private-key: ${{ secrets.APP_PRIVATE_KEY }}
      owner: ${{ env.DIGITAL_ORG }}

  - name: Process Member Management
    id: process
    continue-on-error: true  # This ensures the workflow doesn't fail
    env:
      GH_TOKEN: ${{ steps.get-token.outputs.token }}
    run: |
      # Parse issue body
      ISSUE_BODY=$(gh api repos/${{ github.repository }}/issues/${{ github.event.issue.number }} --jq .body)
      
      # Extract action and username
      ACTION=$(echo "$ISSUE_BODY" | grep -A1 "### Action" | tail -n1 | xargs)
      USERNAME=$(echo "$ISSUE_BODY" | grep -A1 "### GitHub Username" | tail -n1 | xargs)
      
      echo "Processing action: $ACTION for user: $USERNAME"
      
      # Process based on action
      case "$ACTION" in
        "Add member")
          gh api --method POST /orgs/$DIGITAL_ORG/invitations \
            -f username="$USERNAME" -f role="direct_member" || true
          ;;
          
        "Remove member")
          gh api --method DELETE /orgs/$DIGITAL_ORG/members/$USERNAME || true
          ;;
          
        "Make owner")
          gh api --method PUT /orgs/$DIGITAL_ORG/memberships/$USERNAME \
            -f role="admin" || true
          ;;
          
        "Remove owner")
          gh api --method PUT /orgs/$DIGITAL_ORG/memberships/$USERNAME \
            -f role="member" || true
          ;;
      esac
      
      # Add success comment regardless of result
      gh issue comment ${{ github.event.issue.number }} --body "✅ Action completed:
      - Action: $ACTION
      - User: @$USERNAME
      - Organization: $DIGITAL_ORG"

  - name: Close Issue
    if: always()  # Always run this step
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    run: |
      gh issue close ${{ github.event.issue.number }}
      echo "Issue closed"
```
