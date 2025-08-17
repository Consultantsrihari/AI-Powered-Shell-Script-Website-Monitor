# AI-Powered Website Monitor (Shell Script Project)

[![GitHub Repository](https://img.shields.io/badge/GitHub-View%20on%20GitHub-blue?logo=github)](https://github.com/Consultantsrihari/microservices-cicd-aws)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-blue?logo=linkedin)](https://www.linkedin.com/in/VenkataSriHari)
[![Website](https://img.shields.io/badge/Website-Visit-green?logo=google-chrome)](https://Techcareerhubs.com)

A robust, configurable, and intelligent website monitoring tool written entirely in Bash. This project is designed for DevOps engineers and students to learn advanced shell scripting, automation, and AI integration in a practical, real-world scenario. The script monitors a list of websites for downtime. If a site is down, it sends a detailed email alert which includes an AI-generated analysis of the error and suggested troubleshooting steps.

---

## Features

- **Uptime Monitoring**: Checks a list of websites for HTTP status codes and connection errors.
- **External Configuration**: Manage the list of websites to monitor in a simple `websites.conf` file.
- **Secrets Management**: All sensitive data (API keys, email credentials) is stored in a `.env` file and is not hardcoded.
- **Email Alerts**: Sends immediate, detailed email notifications upon failure.
- **AI-Powered Analysis**: Integrates with the OpenAI (ChatGPT) API to provide expert-level explanations and troubleshooting steps directly in the alert message.
- **Detailed Logging**: Keeps a running log of all checks and actions in `logs/monitor.log`.
- **Automation Ready**: Designed to be run automatically as a cron job.

---

## Prerequisites

Ensure you have the following installed on your Linux system:

- `curl`: Command-line tool for making HTTP requests (usually pre-installed).
- `jq`: A lightweight command-line JSON processor.
    - Debian/Ubuntu: `sudo apt-get install jq`
    - CentOS/RHEL: `sudo yum install jq`
- `ssmtp`: A simple tool to send emails via an SMTP server.
    - Debian/Ubuntu: `sudo apt-get install ssmtp`
    - CentOS/RHEL: 
      ```
      sudo yum install epel-release
      sudo yum install ssmtp
      ```
- **OpenAI API Key**: (Optional, for AI analysis) Get one from the [OpenAI Platform](https://platform.openai.com/).

---

## Project Setup and Configuration

### 1. Create the Project Structure

```bash
# Create the main project directory and enter it
mkdir website-monitor && cd website-monitor

# Create the core script file and make it executable
touch monitor.sh
chmod +x monitor.sh

# Create the configuration and secrets files
touch websites.conf
touch .env
touch .gitignore
```

---

### 2. Configure ssmtp for Email Alerts

You need to configure `ssmtp` to connect to your email provider's SMTP server.

```bash
sudo nano /etc/ssmtp/ssmtp.conf
```

Paste the following configuration (replace the placeholder values with your own):

```
# Config file for sSMTP sendmail

# The person who gets all mail for userids < 1000
root=your-email@gmail.com

# The place where the mail goes. The actual machine name is required.
mailhub=smtp.gmail.com:587

# Where will the mail seem to come from?
RewriteDomain=gmail.com

# The full hostname of your server
Hostname=your-server-hostname

# Use SSL/TLS encryption
UseSTARTTLS=YES

# Username and password for SMTP authentication
AuthUser=your-email@gmail.com
AuthPass=YOUR_GMAIL_APP_PASSWORD

# Allow the 'From:' header to be set by the script
FromLineOverride=YES
```

> **IMPORTANT (for Gmail users):**
> - You cannot use your regular password. You must generate an "App Password" from your Google Account security settings. [Learn how here.](https://support.google.com/accounts/answer/185833?hl=en)
> - Secure this file as it contains credentials:
>
> ```bash
> sudo chmod 640 /etc/ssmtp/ssmtp.conf
> sudo chown root:mail /etc/ssmtp/ssmtp.conf
> ```

---

### 3. Populate the Project Files

#### `monitor.sh` (Main Script)

```bash
#!/bin/bash

# ================================================================
# AI-Powered Website Monitor v3.1 (Email Alerts)
# ================================================================

# --- Set script directory context ---
cd "$(dirname "$0")"

# --- Load Environment Variables ---
if [ -f .env ]; then
    export $(grep -v '^#' .env | xargs)
else
    echo "Error: .env file not found. Please create it and add your secrets."
    exit 1
fi

# --- Configuration & Initialization ---
CONFIG_FILE="websites.conf"
LOG_DIR="logs"
LOG_FILE="${LOG_DIR}/monitor.log"

# Create logs directory if it doesn't exist
mkdir -p "${LOG_DIR}"

# --- Functions ---

log() {
    # Log to both console and file
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "${LOG_FILE}"
}

get_ai_suggestion() {
    local url=$1
    local error_code=$2
    local error_type=$3 # "HTTP" or "Connection"

    if [ "${AI_ANALYSIS_ENABLED}" != "true" ] || [ -z "${OPENAI_API_KEY}" ]; then
        echo "AI analysis disabled or API key not set."
        return
    fi

    log "Querying AI for analysis on ${url} (${error_type} ${error_code})..."

    if [ "${error_type}" == "HTTP" ]; then
        prompt="A website monitor check for the URL '${url}' failed with HTTP status code ${error_code}. As a DevOps expert, briefly explain what this status code likely means in one sentence and suggest 2-3 immediate, practical troubleshooting steps in a bulleted list."
    else
        prompt="A website monitor check for the URL '${url}' failed with a cURL connection error (exit code ${error_code}). As a DevOps expert, briefly explain what this type of error likely means (e.g., DNS, firewall) in one sentence and suggest 2-3 immediate, practical troubleshooting steps in a bulleted list."
    fi

    json_payload=$(printf '{"model": "gpt-3.5-turbo", "messages": [{"role": "user", "content": "%s"}], "temperature": 0.5, "max_tokens": 150}' "$(echo "$prompt" | sed 's/"/\\"/g')")

    response=$(curl -s -X POST https://api.openai.com/v1/chat/completions \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer ${OPENAI_API_KEY}" \
      -d "${json_payload}")

    suggestion=$(echo "${response}" | jq -r '.choices[0].message.content')

    if [ -n "${suggestion}" ] && [ "${suggestion}" != "null" ]; then
        echo -e "\n\n-----------------------------------\n🤖 AI DevOps Assistant Suggestion:\n-----------------------------------\n${suggestion}"
    else
        echo "Could not retrieve AI suggestion. API response might have an error."
    fi
}

send_email_alert() {
    local subject=$1
    local body=$2

    if [ -z "${TO_EMAIL}" ]; then
        log "ERROR: TO_EMAIL is not set in .env. Cannot send email."
        return
    fi

    log "Sending email alert to ${TO_EMAIL}..."

    (
        echo "From: Website Monitor <${FROM_EMAIL}>"
        echo "To: ${TO_EMAIL}"
        echo "Subject: ${subject}"
        echo "Content-Type: text/plain; charset=UTF-8"
        echo ""
        echo -e "${body}"
    ) | ssmtp "${TO_EMAIL}"

    if [ $? -eq 0 ]; then
        log "Email alert sent successfully."
    else
        log "ERROR: Failed to send email alert using ssmtp."
    fi
}

check_site() {
    local url=$1

    response=$(curl -s -o /dev/null -w "%{http_code}" --connect-timeout 5 --max-time 10 "${url}" 2>&1)
    curl_exit_code=$?

    if [ $curl_exit_code -ne 0 ]; then
        subject="🚨 Website Down: ${url}"
        body="Failed to connect to the site.\n\nError Details:\n- cURL exit code: ${curl_exit_code}\n- Message: ${response}\n\nThis could be a DNS, network, or firewall issue."
        ai_suggestion=$(get_ai_suggestion "${url}" "${curl_exit_code}" "Connection")
        send_email_alert "${subject}" "${body}${ai_suggestion}"
    elif [ "${response}" -ge 400 ]; then
        http_code=${response}
        subject="🚨 Website Alert: ${url} (HTTP ${http_code})"
        body="The site is responding with a client or server error.\n\nError Details:\n- URL: ${url}\n- HTTP Status Code: ${http_code}"
        ai_suggestion=$(get_ai_suggestion "${url}" "${http_code}" "HTTP")
        send_email_alert "${subject}" "${body}${ai_suggestion}"
    else
        log "SUCCESS: ${url} is UP. Status code: ${response}."
    fi
}

# --- Main Script Logic ---
if [ ! -f "${CONFIG_FILE}" ]; then
    log "Error: Configuration file not found at ${CONFIG_FILE}"
    exit 1
fi

log "--- Monitor script started ---"
while IFS= read -r site || [[ -n "$site" ]]; do
    # Ignore empty lines and lines starting with '#' (comments)
    if [[ -n "$site" ]] && [[ ! "$site" =~ ^# ]]; then
        check_site "$site"
    fi
done < "${CONFIG_FILE}"
log "--- Monitor script finished ---"
```

---

#### `websites.conf`

Add all the websites you want to monitor here, one URL per line.

```
# Production Servers
https://google.com
https://github.com

# Staging Environment
https://api.your-staging-server.com

# Test Failure Cases (these should trigger alerts)
https://httpstat.us/503
https://non-existent-domain-12345.com
```

---

#### `.env`

Store all your secrets here. **Do not commit this file to Git.**

```
# --- Alerting Configuration ---

# Your email address to receive alerts
TO_EMAIL="your-alert-email@example.com"

# A "From" address for the email (can often be the same as your mail hub user)
FROM_EMAIL="your-smtp-user@gmail.com"

# --- OpenAI Configuration (Optional) ---
# Set to "true" to enable AI analysis on failures.
AI_ANALYSIS_ENABLED="true"

# Your secret API key from OpenAI
OPENAI_API_KEY="sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

---

#### `.gitignore`

This file tells Git to ignore sensitive files and logs.

```
# Ignore environment files containing secrets
.env

# Ignore log files and directory
logs/
```

---

## How to Use

### Manual Execution

To run the monitor manually, navigate to the project directory and execute the script:

```bash
./monitor.sh
```

The output will be printed to your console and also appended to `logs/monitor.log`. If any site is down, an email alert will be sent.

---

### Automation with Cron

To run the monitor automatically, schedule it as a cron job. Open your user's crontab file for editing:

```bash
crontab -e
```

Add the following line to the file to run the script every 5 minutes. You **must use the absolute path** to your script.

```
# Run the website monitor every 5 minutes and log output
*/5 * * * * /home/user/path/to/website-monitor/monitor.sh
```

(Replace `/home/user/path/to/website-monitor/` with the actual, absolute path to your project folder. You can find it by running `pwd` inside the folder.)

---

## Expected Alert Email

When a site fails, you will receive an email that looks like this:

**Subject:**  
🚨 Website Alert: https://httpstat.us/503 (HTTP 503)

**Body:**  
The site is responding with a client or server error.  
Error Details:  
- URL: https://httpstat.us/503  
- HTTP Status Code: 503

🤖 **AI DevOps Assistant Suggestion:**  
The HTTP 503 Service Unavailable error indicates that the server is temporarily unable to handle the request due to being overloaded or down for maintenance. Here are immediate troubleshooting steps:

- **Check Server Load**: Log into the server and check the CPU, memory, and disk I/O usage using tools like `top`, `htop`, or `iostat`.
- **Review Server Logs**: Examine the web server's error logs (e.g., `/var/log/nginx/error.log` or `/var/log/apache2/error.log`) for specific error messages that occurred at the time of the failure.
- **Verify Application Status**: Ensure the backend application or service that the web server relies on is running correctly. Restart the service if necessary.

---

## 🤝 Contributing

1. Fork and clone the repo.
2. Make changes in a feature branch.
3. Open a pull request.

---

## 👤 Author

**VenkataSriHari**

*   LinkedIn: [Venkata sri Hari](https://www.linkedin.com/in/venkatasrihari/)
*   GitHub: `@Consultantsrihari`
*   Website: [TechCareerHubs](https://techcareerhubs.com/).


## License

This project is licensed under the MIT License.
