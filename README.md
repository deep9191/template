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
