# RHCSA-Practice-Build-a-Log-Checking-Bash-Script
RHCSA Practice — Build a Log-Checking Bash Script
Today’s focused objective: Create a simple Bash script that accepts a command-line argument, validates input with an if statement, processes command output, and returns a meaningful exit status.
Estimated time: 45 minutes
Lab system: RHEL 9 or RHEL 10
Privileges required: None
Safety: This lesson works only under /tmp/rhcsa-script-lab. It does not modify services, networking, storage, boot configuration, authentication, SELinux, or firewall settings.
This lesson combines your earlier practice with files, permissions, redirection, grep, and regular expressions. Simple shell scripting is part of the current RHCSA EX200 objectives. Official RHCSA EX200 objectives
1. The concept in plain language — 8 minutes
A shell script is a text file containing commands that Bash executes in sequence.
A basic script begins with a shebang:
#!/bin/bash

This tells Linux to execute the file using Bash.
Positional parameters
When you run:
./check-log.sh inventory.log

Bash makes these values available:
Variable	Meaning
$0	Script’s name
$1	First argument
$2	Second argument
$#	Number of arguments
$?	Exit status of the most recent command


In this example:
$0 = ./check-log.sh
$1 = inventory.log
$# = 1

Always place double quotation marks around variables containing paths:
"$1"

This prevents filenames containing spaces from being split into separate words.
Exit statuses
Linux commands communicate success or failure using a number:
- 0 — success
- 1–255 — failure or another defined condition
A useful administrator script should:
1. Validate its input.
2. Perform one clear task.
3. Display a useful result.
4. Return an appropriate exit status.
2. Prepare the lab — 5 minutes
Safety warning
The first command recursively deletes only an earlier copy of this temporary lab. Confirm that the path is exactly /tmp/rhcsa-script-lab.
rm -rf /tmp/rhcsa-script-lab
mkdir -p /tmp/rhcsa-script-lab/{logs,reports,scripts}
cd /tmp/rhcsa-script-lab
pwd

Expected location:
/tmp/rhcsa-script-lab

Create two sample application logs:
printf '%s\n' \
'2026-09-20 18:00:01 INFO  InventoryApp started' \
'2026-09-20 18:01:10 WARN  Database response time exceeded 800ms' \
'2026-09-20 18:02:15 ERROR Database connection failed' \
'2026-09-20 18:02:20 INFO  Retrying database connection' \
'2026-09-20 18:02:25 INFO  Database connection successful' \
'2026-09-20 18:05:30 ERROR API returned HTTP 503' \
> logs/inventory.log

printf '%s\n' \
'2026-09-20 18:00:01 INFO  BillingApp started' \
'2026-09-20 18:05:00 INFO  Payment service connected' \
'2026-09-20 18:10:00 INFO  Health check passed' \
> logs/billing.log

Inspect them:
cat logs/inventory.log
cat logs/billing.log

3. Guided practice: create the first script — 7 minutes
Create the script with this complete copy-and-paste block:
cat > scripts/check-log.sh <<'EOF'
#!/bin/bash

echo "Script name: $0"
echo "Log file supplied: $1"
echo "Number of arguments: $#"
EOF

How this works:
- cat > scripts/check-log.sh redirects the block into a new file.
- <<'EOF' starts a here-document.
- Everything before the final EOF becomes file content.
- Quoting the first 'EOF' prevents your current shell from expanding $0, $1, and $# while creating the script.
Inspect the result:
cat -n scripts/check-log.sh

cat -n displays line numbers, which helps when troubleshooting syntax errors.
Check its initial permissions:
ls -l scripts/check-log.sh

The script will probably not have execute permission yet.
Add execute permission for the owner:
chmod u+x scripts/check-log.sh

Verify:
stat -c '%A %a %n' scripts/check-log.sh

Run it with one argument:
./scripts/check-log.sh logs/inventory.log

Expected pattern:
Script name: ./scripts/check-log.sh
Log file supplied: logs/inventory.log
Number of arguments: 1

4. Guided practice: validate the argument — 8 minutes
Replace the script with this improved version:
cat > scripts/check-log.sh <<'EOF'
#!/bin/bash

if [[ $# -ne 1 ]]; then
    echo "Usage: $0 LOG_FILE"
    exit 2
fi

log_file="$1"

if [[ ! -f "$log_file" ]]; then
    echo "ERROR: File not found: $log_file"
    exit 3
fi

echo "PASS: Valid log file received: $log_file"
exit 0
EOF

Understanding the first condition
if [[ $# -ne 1 ]]; then

This means:
- if begins a conditional decision.
- [[ ... ]] performs a Bash test.
- $# is the number of supplied arguments.
- -ne means numerically “not equal.”
- The script expects exactly one argument.
- then begins the commands executed when the condition is true.
- fi ends the if block.
Understanding the file test
if [[ ! -f "$log_file" ]]; then

This means:
- -f tests whether the path is a regular file.
- ! reverses the test.
- The condition is true when the regular file does not exist.
Test the failure paths
Run it without an argument:
./scripts/check-log.sh
echo "Exit status: $?"

Expected:
Usage: ./scripts/check-log.sh LOG_FILE
Exit status: 2

Run it with a nonexistent file:
./scripts/check-log.sh logs/missing.log
echo "Exit status: $?"

Expected:
ERROR: File not found: logs/missing.log
Exit status: 3

Run it correctly:
./scripts/check-log.sh logs/inventory.log
echo "Exit status: $?"

Expected:
PASS: Valid log file received: logs/inventory.log
Exit status: 0

5. Guided practice: process command output — 10 minutes
Replace the script with its operational version:
cat > scripts/check-log.sh <<'EOF'
#!/bin/bash

if [[ $# -ne 1 ]]; then
    echo "Usage: $0 LOG_FILE"
    exit 2
fi

log_file="$1"

if [[ ! -f "$log_file" ]]; then
    echo "ERROR: File not found: $log_file"
    exit 3
fi

error_count=$(grep -c ' ERROR ' "$log_file")
warning_count=$(grep -c ' WARN ' "$log_file")

echo "Log Analysis Report"
echo "File: $log_file"
echo "Errors: $error_count"
echo "Warnings: $warning_count"

if [[ $error_count -gt 0 ]]; then
    echo "Status: ATTENTION REQUIRED"
    exit 1
else
    echo "Status: HEALTHY"
    exit 0
fi
EOF

Make sure it remains executable:
chmod 750 scripts/check-log.sh

Understanding command substitution
error_count=$(grep -c ' ERROR ' "$log_file")

The $() construction:
1. Runs the command inside the parentheses.
2. Captures its standard output.
3. Stores that output in error_count.
The same process stores the warning count in warning_count.
Understanding the numeric comparison
if [[ $error_count -gt 0 ]]; then

-gt means numerically “greater than.”
Other useful numeric comparisons include:
Test	Meaning
-eq	Equal
-ne	Not equal
-gt	Greater than
-ge	Greater than or equal
-lt	Less than
-le	Less than or equal


Test a log containing errors
./scripts/check-log.sh logs/inventory.log
echo "Exit status: $?"

Expected output:
Log Analysis Report
File: logs/inventory.log
Errors: 2
Warnings: 1
Status: ATTENTION REQUIRED
Exit status: 1

An exit status of 1 is intentional here. The script ran correctly but reported that the monitored application requires attention.
Test a healthy log
./scripts/check-log.sh logs/billing.log
echo "Exit status: $?"

Expected:
Log Analysis Report
File: logs/billing.log
Errors: 0
Warnings: 0
Status: HEALTHY
Exit status: 0

6. Redirect the report — 2 minutes
Run the script and save its output:
./scripts/check-log.sh logs/inventory.log > reports/inventory-status.txt
script_status=$?

Because the InventoryApp log contains errors, script_status should contain 1.
Verify both output and status:
cat reports/inventory-status.txt
echo "Saved exit status: $script_status"

This is important: capture $? immediately after the command. Running another command first replaces the previous exit status.
7. Verify script syntax — 2 minutes
Use Bash’s syntax checker:
bash -n scripts/check-log.sh
echo "Syntax-check exit status: $?"

Expected:
Syntax-check exit status: 0

bash -n reads the script without executing its operational commands. It catches many structural mistakes, such as missing fi statements or unmatched quotation marks.
It does not prove that the script’s logic is correct, so you must still test its behavior.
8. Independent challenge — 6 minutes
Do not copy the guided script. Build this second script independently.
Create:
scripts/check-config.sh

Requirements
The script must:
1. Use /bin/bash.
2. Require exactly one argument.
3. Treat that argument as a configuration-file path.
4. Display a usage message and exit with status 2 if the argument count is incorrect.
5. Display an error and exit with status 3 if the path is not a regular file.
6. Search the supplied file for:
environment=production

7. If the setting exists:
   - Display Production configuration confirmed
   - Exit with status 0
8. If the setting does not exist:
   - Display Production configuration not found
   - Exit with status 1
9. Be executable by the owner.
10. Pass bash -n syntax verification.
Create these test files:
printf '%s\n' \
'application=InventoryApp' \
'environment=production' \
'database=db01.example.test' \
> reports/production.conf

printf '%s\n' \
'application=InventoryApp' \
'environment=development' \
'database=db-dev.example.test' \
> reports/development.conf

Test your script against:
- reports/production.conf
- reports/development.conf
- reports/missing.conf
- No argument
9. Independent verification
After writing the script, these tests should pass:
bash -n scripts/check-config.sh \
  && echo "PASS: syntax is valid" \
  || echo "FAIL: syntax error"

Production test:
./scripts/check-config.sh reports/production.conf
test $? -eq 0 \
  && echo "PASS: production file returned 0" \
  || echo "FAIL: unexpected production result"

Development test:
./scripts/check-config.sh reports/development.conf
test $? -eq 1 \
  && echo "PASS: development file returned 1" \
  || echo "FAIL: unexpected development result"

Missing-file test:
./scripts/check-config.sh reports/missing.conf
test $? -eq 3 \
  && echo "PASS: missing file returned 3" \
  || echo "FAIL: unexpected missing-file result"

Argument-count test:
./scripts/check-config.sh
test $? -eq 2 \
  && echo "PASS: missing argument returned 2" \
  || echo "FAIL: unexpected argument result"

Permission test:
test -x scripts/check-config.sh \
  && echo "PASS: script is executable" \
  || echo "FAIL: script is not executable"

10. Knowledge check
Answer these without looking back:
1. What is the purpose of #!/bin/bash?
2. What do $0, $1, and $# represent?
3. Why should "$1" normally be enclosed in double quotes?
4. What does [[ -f "$1" ]] test?
5. What does [[ ! -f "$1" ]] test?
6. What does -ne mean?
7. What does -gt mean?
8. What does $(command) do?
9. What does exit 0 communicate?
10. When must you capture $??
11. What does bash -n script.sh check?
12. Why is syntax verification not a replacement for operational testing?
13. Which permission is required to run a script as ./script.sh?
14. Why might a monitoring script deliberately return exit status 1?
I would consider this objective mastered when you can independently write a script that validates its argument, tests for a file, captures command output, makes a decision, and returns meaningful exit statuses.
Safe cleanup
Safety warning
The following command recursively removes the temporary lab. Confirm the full path is exactly /tmp/rhcsa-script-lab.
cd /tmp
rm -rf /tmp/rhcsa-script-lab

Verify:
test ! -e /tmp/rhcsa-script-lab \
  && echo "PASS: script lab removed safely" \
  || echo "CHECK: script lab still exists"

Expected:
PASS: script lab removed safely
