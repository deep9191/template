name: Cross-Org Member Management
description: Manage members in Digital Organization
title: "[Member]: "
labels: ["member-management"]

body:
  - type: dropdown
    id: action
    attributes:
      label: Action
      options:
        - Add member
        - Remove member
        - Make owner
        - Remove owner
    validations:
      required: true

  - type: input
    id: username
    attributes:
      label: GitHub Username
      placeholder: octocat
    validations:
      required: true

  - type: dropdown
    id: target-org
    attributes:
      label: Target Organization
      description: Organization where the action should be performed
      options:
        - digital-org-name  # Replace with your actual digital org name
    validations:
      required: true


    ==================================================================================

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
      - name: Verify Admin Access
        uses: actions/github-script@v7
        id: check_permission
        with:
          github-token: ${{ secrets.DIGITAL_ORG_TOKEN }}
          script: |
            try {
              const { data } = await github.rest.orgs.getMembershipForUser({
                org: process.env.DIGITAL_ORG,
                username: context.actor
              });
              
              if (data.role !== 'admin') {
                throw new Error('You need admin permissions in the digital organization');
              }
              return 'authorized';
            } catch (error) {
              await github.rest.issues.createComment({
                ...context.repo,
                issue_number: context.issue.number,
                body: '❌ Error: You do not have admin permissions in the target organization'
              });
              throw error;
            }

      - name: Manage Organization Member
        if: steps.check_permission.outputs.result == 'authorized'
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.DIGITAL_ORG_TOKEN }}
          script: |
            const issue = context.payload.issue;
            const action = issue.body.match(/### Action\s*([^\n]+)/)[1].trim();
            const username = issue.body.match(/### GitHub Username\s*([^\n]+)/)[1].trim();
            
            try {
              switch(action) {
                case 'Add member':
                  await github.rest.orgs.setMembershipForUser({
                    org: process.env.DIGITAL_ORG,
                    username: username,
                    role: 'member'
                  });
                  break;
                  
                case 'Remove member':
                  await github.rest.orgs.removeMembershipForUser({
                    org: process.env.DIGITAL_ORG,
                    username: username
                  });
                  break;
                  
                case 'Make owner':
                  await github.rest.orgs.setMembershipForUser({
                    org: process.env.DIGITAL_ORG,
                    username: username,
                    role: 'admin'
                  });
                  break;
                  
                case 'Remove owner':
                  await github.rest.orgs.setMembershipForUser({
                    org: process.env.DIGITAL_ORG,
                    username: username,
                    role: 'member'
                  });
                  break;
              }
              
              await github.rest.issues.createComment({
                ...context.repo,
                issue_number: context.issue.number,
                body: `✅ Successfully completed: ${action} for @${username} in ${process.env.DIGITAL_ORG}`
              });
              
              await github.rest.issues.update({
                ...context.repo,
                issue_number: context.issue.number,
                state: 'closed'
              });
              
            } catch (error) {
              await github.rest.issues.createComment({
                ...context.repo,
                issue_number: context.issue.number,
                body: `❌ Error: ${error.message}`
              });
            }
=================================================================================================

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
      - name: Process Member Management
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.DIGITAL_ORG_TOKEN }}
          script: |
            const issue = context.payload.issue;
            const action = issue.body.match(/### Action\s*([^\n]+)/)[1].trim();
            const username = issue.body.match(/### GitHub Username\s*([^\n]+)/)[1].trim();
            
            try {
              // First verify admin access
              console.log('Checking admin permissions...');
              const { data: membership } = await github.rest.orgs.getMembershipForUser({
                org: process.env.DIGITAL_ORG,
                username: context.actor
              });
              
              if (membership.role !== 'admin') {
                throw new Error('You need admin permissions in the digital organization');
              }
              
              console.log('Admin access verified, processing action...');
              
              // Process the requested action
              switch(action) {
                case 'Add member':
                  await github.rest.orgs.setMembershipForUser({
                    org: process.env.DIGITAL_ORG,
                    username: username,
                    role: 'member'
                  });
                  break;
                  
                case 'Remove member':
                  await github.rest.orgs.removeMembershipForUser({
                    org: process.env.DIGITAL_ORG,
                    username: username
                  });
                  break;
                  
                case 'Make owner':
                  await github.rest.orgs.setMembershipForUser({
                    org: process.env.DIGITAL_ORG,
                    username: username,
                    role: 'admin'
                  });
                  break;
                  
                case 'Remove owner':
                  await github.rest.orgs.setMembershipForUser({
                    org: process.env.DIGITAL_ORG,
                    username: username,
                    role: 'member'
                  });
                  break;
                
                default:
                  throw new Error(`Unknown action: ${action}`);
              }
              
              console.log('Action completed successfully');
              
              // Add success comment
              await github.rest.issues.createComment({
                ...context.repo,
                issue_number: context.issue.number,
                body: `✅ Successfully completed: ${action} for @${username} in ${process.env.DIGITAL_ORG}`
              });
              
              // Close the issue
              await github.rest.issues.update({
                ...context.repo,
                issue_number: context.issue.number,
                state: 'closed',
                state_reason: 'completed'
              });
              
            } catch (error) {
              console.error('Error:', error);
              
              // Add error comment
              await github.rest.issues.createComment({
                ...context.repo,
                issue_number: context.issue.number,
                body: `❌ Error: ${error.message}\n\nPlease verify:\n1. The username exists\n2. You have proper permissions\n3. The requested action is valid`
              });
              
              // Don't close the issue on error
              throw error;
            }45



====================================================================================================================================================
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
      - name: Generate GitHub App Token
        id: get-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          owner: ${{ env.DIGITAL_ORG }}

      - name: Process Member Management
        uses: actions/github-script@v7
        with:
          github-token: ${{ steps.get-token.outputs.token }}
          script: |
            try {
              const issue = context.payload.issue;
              const action = issue.body.match(/### Action\s*([^\n]+)/)[1].trim();
              const username = issue.body.match(/### GitHub Username\s*([^\n]+)/)[1].trim();
              
              console.log('Processing action:', action);
              console.log('Target username:', username);
              
              switch(action) {
                case 'Add member':
                  await github.rest.orgs.createInvitation({
                    org: process.env.DIGITAL_ORG,
                    username: username,
                    role: 'direct_member'
                  });
                  break;
                  
                case 'Remove member':
                  await github.rest.orgs.removeMember({
                    org: process.env.DIGITAL_ORG,
                    username: username
                  });
                  break;
                  
                case 'Make owner':
                  await github.rest.orgs.setMembershipForUser({
                    org: process.env.DIGITAL_ORG,
                    username: username,
                    role: 'admin'
                  });
                  break;
                  
                case 'Remove owner':
                  await github.rest.orgs.setMembershipForUser({
                    org: process.env.DIGITAL_ORG,
                    username: username,
                    role: 'member'
                  });
                  break;
              }
              
              await github.rest.issues.createComment({
                ...context.repo,
                issue_number: context.issue.number,
                body: `✅ Action completed successfully:
                - Action: ${action}
                - User: @${username}
                - Organization: ${process.env.DIGITAL_ORG}`
              });
              
              await github.rest.issues.update({
                ...context.repo,
                issue_number: context.issue.number,
                state: 'closed'
              });
              
            } catch (error) {
              console.error('Error:', error);
              
              await github.rest.issues.createComment({
                ...context.repo,
                issue_number: context.issue.number,
                body: `❌ Error: ${error.message}\n\nPlease verify:\n1. The username exists\n2. The action is valid\n3. The user's current state allows this action`
              });
              
              throw error;
            }
================================================================================================================================================================================

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
      - name: Generate GitHub App Token
        id: get-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          owner: ${{ env.DIGITAL_ORG }}

      - name: Process Member Management
        uses: actions/github-script@v7
        with:
          github-token: ${{ steps.get-token.outputs.token }}
          script: |
            try {
              const issue = context.payload.issue;
              const action = issue.body.match(/### Action\s*([^\n]+)/)[1].trim();
              const username = issue.body.match(/### GitHub Username\s*([^\n]+)/)[1].trim();
              
              let success = false;
              
              switch(action) {
                case 'Add member':
                  await github.rest.orgs.createInvitation({
                    org: process.env.DIGITAL_ORG,
                    username: username,
                    role: 'direct_member'
                  });
                  success = true;
                  break;
                  
                case 'Remove member':
                  await github.rest.orgs.removeMember({
                    org: process.env.DIGITAL_ORG,
                    username: username
                  });
                  success = true;
                  break;
                  
                case 'Make owner':
                  await github.rest.orgs.setMembershipForUser({
                    org: process.env.DIGITAL_ORG,
                    username: username,
                    role: 'admin'
                  });
                  success = true;
                  break;
                  
                case 'Remove owner':
                  await github.rest.orgs.setMembershipForUser({
                    org: process.env.DIGITAL_ORG,
                    username: username,
                    role: 'member'
                  });
                  success = true;
                  break;
                
                default:
                  throw new Error('Invalid action specified');
              }
              
              if (success) {
                await github.rest.issues.createComment({
                  ...context.repo,
                  issue_number: context.issue.number,
                  body: `✅ Action completed successfully:
                  - Action: ${action}
                  - User: @${username}
                  - Organization: ${process.env.DIGITAL_ORG}`
                });
                
                await github.rest.issues.update({
                  ...context.repo,
                  issue_number: context.issue.number,
                  state: 'closed'
                });
                
                return 'success';  // Exit successfully
              }
              
            } catch (error) {
              // Only comment on actual errors
              if (!error.message.includes('success')) {
                await github.rest.issues.createComment({
                  ...context.repo,
                  issue_number: context.issue.number,
                  body: `❌ Error: ${error.message}`
                });
                throw error;  // Only throw if it's a real error
              }
            }
========================================================================================================================================================

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
                return true;
                
              case 'Remove member':
                await github.rest.orgs.removeMember({
                  org: process.env.DIGITAL_ORG,
                  username: username
                });
                return true;
                
              case 'Make owner':
                await github.rest.orgs.setMembershipForUser({
                  org: process.env.DIGITAL_ORG,
                  username: username,
                  role: 'admin'
                });
                return true;
                
              case 'Remove owner':
                await github.rest.orgs.setMembershipForUser({
                  org: process.env.DIGITAL_ORG,
                  username: username,
                  role: 'member'
                });
                return true;
            }
          } catch (e) {
            if (e.status === 404 && action === 'Remove member') {
              // Ignore 404 for remove operations as the user might already be removed
              return true;
            }
            throw e;
          }
          return false;
        }

        try {
          const issue = context.payload.issue;
          const action = issue.body.match(/### Action\s*([^\n]+)/)[1].trim();
          const username = issue.body.match(/### GitHub Username\s*([^\n]+)/)[1].trim();
          
          const success = await performAction(action, username);
          
          if (success) {
            await github.rest.issues.createComment({
              ...context.repo,
              issue_number: context.issue.number,
              body: `✅ Action completed successfully:
              - Action: ${action}
              - User: @${username}
              - Organization: ${process.env.DIGITAL_ORG}`
            });
            
            await github.rest.issues.update({
              ...context.repo,
              issue_number: context.issue.number,
              state: 'closed'
            });
          } else {
            throw new Error('Action failed to complete');
          }
          
        } catch (error) {
          // Only create error comment for non-404 errors or if it's not a removal action
          if (error.status !== 404 || !error.message.includes('Not Found')) {
            await github.rest.issues.createComment({
              ...context.repo,
              issue_number: context.issue.number,
              body: `❌ Error: ${error.message}\n\nPlease verify:\n1. The username exists\n2. The action is valid\n3. The user's current state allows this action`
            });
          } else {
            // If it's a 404 error but the action was successful, close with success
            await github.rest.issues.createComment({
              ...context.repo,
              issue_number: context.issue.number,
              body: `✅ Action completed successfully:
              - Action: ${action}
              - User: @${username}
              - Organization: ${process.env.DIGITAL_ORG}`
            });
            
            await github.rest.issues.update({
              ...context.repo,
              issue_number: context.issue.number,
              state: 'closed'
            });
          }
        }
