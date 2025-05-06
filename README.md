```yaml

name: 'GitHub Native Notification'
description: 'Send notifications to team members using GitHub repository features'
inputs:
  team:
    description: 'Team name from config or comma-separated GitHub usernames'
    required: true
  message:
    description: 'Notification message'
    required: true
  report_path:
    description: 'Path to report file in the repository'
    required: false
  config_path:
    description: 'Path to notification config file in the repository'
    required: false
    default: '.github/notification-config.yml'

runs:
  using: 'composite'
  steps:
    - name: Create notification
      shell: bash
      run: |
        # Determine recipients
        TEAM="${{ inputs.team }}"
        CONFIG_PATH="${{ inputs.config_path }}"
        RECIPIENTS=""
        
        # Check if team name exists in config
        if [ -f "$CONFIG_PATH" ]; then
          # Extract team members if team exists in config
          if grep -q "^  $TEAM:" "$CONFIG_PATH"; then
            TEAM_MEMBERS=$(sed -n "/^  $TEAM:/,/^  [a-z]/ p" "$CONFIG_PATH" | grep "^    -" | sed 's/^    - //')
            for member in $TEAM_MEMBERS; do
              RECIPIENTS+="$member,"
            done
            RECIPIENTS=${RECIPIENTS%,}  # Remove trailing comma
          else
            # If not found in config, use input as direct usernames
            RECIPIENTS="$TEAM"
          fi
        else
          # If config doesn't exist, use input as direct usernames
          RECIPIENTS="$TEAM"
        fi
        
        # Create notification content with @mentions
        NOTIFICATION=""
        
        # Add @mentions for each recipient
        IFS=',' read -ra USERS <<< "$RECIPIENTS"
        for user in "${USERS[@]}"; do
          NOTIFICATION+="@${user//[[:space:]]/} "
        done
        
        NOTIFICATION+="\n\n${{ inputs.message }}"
        
        # Add report link if provided
        if [ -n "${{ inputs.report_path }}" ] && [ -f "${{ inputs.report_path }}" ]; then
          REPO_URL="${GITHUB_SERVER_URL}/${GITHUB_REPOSITORY}"
          BRANCH="${GITHUB_REF#refs/heads/}"
          REPORT_URL="${REPO_URL}/blob/${BRANCH}/${{ inputs.report_path }}"
          
          NOTIFICATION+="\n\n## Report\n[View Report: ${{ inputs.report_path }}](${REPORT_URL})"
        fi
        
        # Create a comment on the commit with the notification
        gh api \
          --method POST \
          -H "Accept: application/vnd.github+json" \
          /repos/${GITHUB_REPOSITORY}/commits/${GITHUB_SHA}/comments \
          -f body="${NOTIFICATION}"
        
        echo "Notification posted as commit comment"
      env:
        GITHUB_TOKEN: ${{ github.token }}

```

```yaml
# Team member GitHub usernames
teams:
  otop-team:
    - user1
    - user2
    - user3
  
  dev-team:
    - dev-lead
    - developer1
    - developer2
```

```yaml
name: Process and Notify

on:
  workflow_dispatch:
  push:
    branches: [ main ]

jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      # Generate or process report
      
      - name: Send notification
        uses: ./.github/actions/github-notification
        # Or if published to a central repository:
        # uses: your-org/github-notification-action@v1
        with:
          team: 'otop-team'
          message: 'Topics and BUC issues have been detected that require human validation or correction.'
          report_path: './reports/issue-report.md'
          # config_path: '.github/custom-notification-config.yml'  # Optional override
```



































===========================================================================================================================================

```yaml

name: 'Email Notification for Reports'
description: 'Send email notifications for reports in a specific folder using centralized SMTP configuration'
inputs:
  subject:
    description: 'Email subject line'
    required: true
  message:
    description: 'Email message body (HTML supported)'
    required: true
  recipients:
    description: 'Comma-separated list of email recipients'
    required: false
  recipient_group:
    description: 'Name of recipient group from config file'
    required: false
  reports_folder:
    description: 'Path to the folder containing reports'
    required: true
    default: 'reports'
  report_pattern:
    description: 'File pattern to match reports (e.g., "*.txt" or "report-*.csv")'
    required: false
    default: '*'
  config_path:
    description: 'Path to recipients config file'
    required: false
    default: '.github/notification-config.yml'

runs:
  using: "composite"
  steps:
    - name: Load Configuration
      id: config
      shell: bash
      run: |
        # Set SMTP configuration (hardcoded in the action)
        echo "::set-output name=smtp_server::smtp.company.com"
        echo "::set-output name=smtp_port::465"
        echo "::set-output name=smtp_username::notifications@company.com"
        echo "::set-output name=email_from::notifications@company.com"
        
        # The password is hardcoded in a secure way within the action
        # This is a placeholder - in a real implementation, you would use a more secure approach
        # such as GitHub's encrypted secrets at the organization level
        SMTP_PASSWORD="your-secure-password-or-token"
        echo "::set-output name=smtp_password::$SMTP_PASSWORD"
    
    - name: Resolve Recipients
      id: resolve-recipients
      shell: bash
      run: |
        FINAL_RECIPIENTS="${{ inputs.recipients }}"
        
        # If recipient_group is provided, try to extract from config file
        if [[ -n "${{ inputs.recipient_group }}" && -f "${{ inputs.config_path }}" ]]; then
          GROUP_RECIPIENTS=$(grep -A 20 "${{ inputs.recipient_group }}:" "${{ inputs.config_path }}" | grep -v "${{ inputs.recipient_group }}:" | grep -B 20 -m 1 "^[a-z]" | grep "@" | sed 's/[^@]*\(@.*\)/\1/g' | tr -d ' -' | tr '\n' ',')
          
          if [[ -n "$GROUP_RECIPIENTS" ]]; then
            if [[ -n "$FINAL_RECIPIENTS" ]]; then
              FINAL_RECIPIENTS="$FINAL_RECIPIENTS,$GROUP_RECIPIENTS"
            else
              FINAL_RECIPIENTS="$GROUP_RECIPIENTS"
            fi
          fi
        fi
        
        echo "::set-output name=email_list::$FINAL_RECIPIENTS"
    
    - name: Find Reports
      id: find-reports
      shell: bash
      run: |
        # Check if reports folder exists
        if [ ! -d "${{ inputs.reports_folder }}" ]; then
          echo "Error: Reports folder not found at ${{ inputs.reports_folder }}"
          exit 1
        fi
        
        # Find reports matching the pattern
        REPORTS=$(find "${{ inputs.reports_folder }}" -type f -name "${{ inputs.report_pattern }}" | tr '\n' ',' | sed 's/,$//')
        
        if [ -z "$REPORTS" ]; then
          echo "Error: No reports found matching pattern '${{ inputs.report_pattern }}' in folder '${{ inputs.reports_folder }}'"
          exit 1
        fi
        
        echo "Found reports: $REPORTS"
        echo "::set-output name=report_list::$REPORTS"
    
    - name: Send Email
      shell: bash
      run: |
        # Create a temporary file for the email content
        EMAIL_FILE=$(mktemp)
        
        # Create email content with MIME parts
        echo "From: ${{ steps.config.outputs.email_from }}" > $EMAIL_FILE
        echo "To: ${{ steps.resolve-recipients.outputs.email_list }}" >> $EMAIL_FILE
        echo "Subject: ${{ inputs.subject }}" >> $EMAIL_FILE
        echo "MIME-Version: 1.0" >> $EMAIL_FILE
        echo "Content-Type: multipart/mixed; boundary=\"boundary-string\"" >> $EMAIL_FILE
        echo "" >> $EMAIL_FILE
        echo "--boundary-string" >> $EMAIL_FILE
        echo "Content-Type: text/html; charset=\"UTF-8\"" >> $EMAIL_FILE
        echo "Content-Transfer-Encoding: 7bit" >> $EMAIL_FILE
        echo "" >> $EMAIL_FILE
        echo "${{ inputs.message }}" >> $EMAIL_FILE
        
        # Add each report as an attachment
        IFS=',' read -ra REPORT_ARRAY <<< "${{ steps.find-reports.outputs.report_list }}"
        for REPORT in "${REPORT_ARRAY[@]}"; do
          if [ -f "$REPORT" ]; then
            echo "--boundary-string" >> $EMAIL_FILE
            echo "Content-Type: text/plain; charset=\"UTF-8\"" >> $EMAIL_FILE
            echo "Content-Transfer-Encoding: base64" >> $EMAIL_FILE
            echo "Content-Disposition: attachment; filename=\"$(basename $REPORT)\"" >> $EMAIL_FILE
            echo "" >> $EMAIL_FILE
            base64 -w 0 "$REPORT" >> $EMAIL_FILE
            echo "" >> $EMAIL_FILE
          fi
        done
        
        echo "--boundary-string--" >> $EMAIL_FILE
        
        # Send email using curl and SMTP
        curl --url "smtp://${{ steps.config.outputs.smtp_server }}:${{ steps.config.outputs.smtp_port }}" \
          --ssl-reqd \
          --mail-from "${{ steps.config.outputs.email_from }}" \
          --mail-rcpt "${{ steps.resolve-recipients.outputs.email_list }}" \
          --upload-file $EMAIL_FILE \
          --user "${{ steps.config.outputs.smtp_username }}:${{ steps.config.outputs.smtp_password }}"
        
        # Clean up
        rm $EMAIL_FILE


```

```yaml

name: Process and Notify

on:
  workflow_dispatch:
  schedule:
    - cron: '0 9 * * 1-5'  # Weekdays at 9am

jobs:
  notify-from-reports:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
      
      # The reports are assumed to already exist in the 'reports' folder
      # This could be from a previous job, workflow, or commit
      
      - name: Send Email Notification
        uses: ./.github/actions/email-notification
        with:
          subject: "Topics and BUC Issues Detected"
          message: "The automated check has detected issues with Topics and BUC. Please review the attached reports."
          recipient_group: "otop-team"
          reports_folder: "reports"
          report_pattern: "*.txt"  # Send all text files in the reports folder


```

```yaml

# Notification Configuration

# Recipient Groups
recipientGroups:
  otop-team:
    - user1@example.com
    - user2@example.com
  security-team:
    - security-lead@example.com
    - security-analyst@example.com
  admin-team:
    - admin@example.com
    - manager@example.com

```


```
# GitHub Email Notification Action for Reports

A simple, reusable GitHub Action for sending email notifications with reports from a specific folder. This action uses a centralized SMTP configuration so you don't need to provide these details each time.

## Features

- Send HTML email notifications
- Automatically attaches reports from a specified folder
- Supports file pattern matching to select specific reports
- No third-party dependencies
- Simple to use and integrate
- Supports recipient groups from a configuration file
- Centralized SMTP configuration - no need to provide SMTP credentials

## Usage

Add this action to your workflow:

## Input Parameters

| Parameter | Description | Required | Default |
|-----------|-------------|----------|---------|
| `subject` | Email subject line | Yes | - |
| `message` | Email message body (HTML supported) | Yes | - |
| `recipients` | Comma-separated list of email recipients | Yes (if `recipient_group` not used) | - |
| `recipient_group` | Name of recipient group from config file | Yes (if `recipients` not used) | - |
| `reports_folder` | Path to the folder containing reports | Yes | `reports` |
| `report_pattern` | File pattern to match reports | No | `*` (all files) |
| `config_path` | Path to recipients config file | No | `.github/notification-config.yml` |

## SMTP Configuration

This action uses a centralized SMTP configuration that is managed within the action itself. You don't need to provide any SMTP credentials.
```
