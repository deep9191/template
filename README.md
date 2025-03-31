```yaml

      - name: Process repository
        if: steps.check-permissions.outputs.is-admin == 'true'
        id: process-repo
        env:
          TEST_TOKEN: ${{ steps.generate-test-token.outputs.token }}
          BANKING_TOKEN: ${{ steps.generate-banking-token.outputs.token }}
          SOURCE_ORG: ${{ steps.check-permissions.outputs.org-name }}
          REPO_NAME: ${{ steps.check-permissions.outputs.repo-name }}
          BANKING_ORG: "DigitalBankingSolutions"  # Replace with your banking solutions org
          ISSUE_NUMBER: ${{ github.event.issue.number }}
        run: |
          # Get current date in YYYYMMDD format
          TODAY=$(date +"%Y%m%d")
          
          # Get repository details before deleting
          echo "Getting repository details for $SOURCE_ORG/$REPO_NAME"
          # Use the test token for source org operations
          export GITHUB_TOKEN=$TEST_TOKEN
          
          REPO_DESC=$(gh api repos/$SOURCE_ORG/$REPO_NAME --jq '.description // "Repository"')
          REPO_PRIVATE=$(gh api repos/$SOURCE_ORG/$REPO_NAME --jq '.private')
          REPO_TOPICS=$(gh api repos/$SOURCE_ORG/$REPO_NAME/topics --jq '.names | join(",")' || echo "")
          REPO_HOMEPAGE=$(gh api repos/$SOURCE_ORG/$REPO_NAME --jq '.homepage // ""')
          
          # Set visibility flag for repo creation
          if [[ "$REPO_PRIVATE" == "true" ]]; then
            VISIBILITY="--private"
          else
            VISIBILITY="--public"
          fi
          
          # Step 1: Create new repository in banking solutions org with date suffix
          # Use banking token for banking org operations
          export GITHUB_TOKEN=$BANKING_TOKEN
          
          NEW_REPO_NAME="${REPO_NAME}-${TODAY}"
          echo "Creating new repository in banking solutions org: $BANKING_ORG/$NEW_REPO_NAME"
          
          # Create the repository
          gh repo create $BANKING_ORG/$NEW_REPO_NAME $VISIBILITY --description "$REPO_DESC"
          
          # Set topics if any
          if [[ ! -z "$REPO_TOPICS" ]]; then
            echo "Setting topics: $REPO_TOPICS"
            # Properly format the topics as a JSON array
            if [[ "$REPO_TOPICS" == *","* ]]; then
              # Multiple topics
              FORMATTED_TOPICS="[\"$(echo $REPO_TOPICS | sed 's/,/\",\"/g')\"]"
            elif [[ ! -z "$REPO_TOPICS" ]]; then
              # Single topic
              FORMATTED_TOPICS="[\"$REPO_TOPICS\"]"
            else
              # No topics
              FORMATTED_TOPICS="[]"
            fi
            echo "Formatted topics: $FORMATTED_TOPICS"
            gh api -X PUT repos/$BANKING_ORG/$NEW_REPO_NAME/topics --input - <<EOF
            {
              "names": $FORMATTED_TOPICS
            }
EOF
          fi
          
          # Set homepage if any
          if [[ ! -z "$REPO_HOMEPAGE" && "$REPO_HOMEPAGE" != "null" ]]; then
            echo "Setting homepage: $REPO_HOMEPAGE"
            gh api -X PATCH repos/$BANKING_ORG/$NEW_REPO_NAME -f homepage="$REPO_HOMEPAGE"
          fi
          
          # Step 2: Clone the original repository
          echo "Cloning repository: $SOURCE_ORG/$REPO_NAME"
          export GITHUB_TOKEN=$TEST_TOKEN
          git clone https://x-access-token:${GITHUB_TOKEN}@github.com/$SOURCE_ORG/$REPO_NAME.git source-repo
          cd source-repo
          
          # Get all branches and tags
          git fetch --all
          git fetch --tags
          
          # Step 3: Push to the new repository with branch protection bypass
          echo "Pushing data to new repository: $BANKING_ORG/$NEW_REPO_NAME"
          export GITHUB_TOKEN=$BANKING_TOKEN
          
          # Add the new repository as a remote
          git remote add banking https://x-access-token:${GITHUB_TOKEN}@github.com/$BANKING_ORG/$NEW_REPO_NAME.git
          
          # Get list of all branches
          BRANCHES=$(git branch -r | grep -v '\->' | sed 's/origin\///')
          
          # Push each branch individually to the new repository
          for branch in $BRANCHES; do
            echo "Processing branch: $branch"
            git checkout -b $branch origin/$branch || git checkout $branch
            
            # Check if this is a protected branch (main, master, develop)
            if [[ "$branch" == "main" || "$branch" == "master" || "$branch" == "develop" ]]; then
              echo "This is a protected branch: $branch"
              # For protected branches, we'll use the admin token to bypass branch protection
              # The --force flag bypasses branch protection rules when used with an admin token
              git push banking $branch --force
            else
              # For non-protected branches, push normally
              git push banking $branch
            fi
          done
          
          # Push all tags
          git push banking --tags
          
          cd ..
          
          # Step 4: Delete the repository from test org
          # Use test token for test org operations
          export GITHUB_TOKEN=$TEST_TOKEN
          
          echo "Deleting repository from test org: $SOURCE_ORG/$REPO_NAME"
          gh api -X DELETE /repos/$SOURCE_ORG/$REPO_NAME
          
          # Add comment to the issue
          gh issue comment $ISSUE_NUMBER --body "Repository processing completed:
          
          - Original repo \`$SOURCE_ORG/$REPO_NAME\` has been deleted
          - New repository created with all data at: https://github.com/$BANKING_ORG/$NEW_REPO_NAME"
          
          # Set outputs for next steps
          echo "new-repo-url=https://github.com/$BANKING_ORG/$NEW_REPO_NAME" >> $GITHUB_OUTPUT
```


```yaml
      - name: Notify success
        if: steps.process-repo.outcome == 'success'
        env:
          GITHUB_TOKEN: ${{ steps.generate-test-token.outputs.token }}
          ISSUE_NUMBER: ${{ github.event.issue.number }}
          NEW_REPO_URL: ${{ steps.process-repo.outputs.new-repo-url }}
          REPO_TO_DELETE: ${{ steps.issue.outputs.repo-to-delete }}
        run: |
          gh issue comment $ISSUE_NUMBER --body "✅ Repository processing completed successfully!

          - Original repository \`$REPO_TO_DELETE\` has been deleted
          - New repository has been created with all data at: $NEW_REPO_URL"
          
          gh issue close $ISSUE_NUMBER --comment "Repository deletion and recreation process completed."
```
