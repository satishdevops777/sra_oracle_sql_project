# ansible_role_sra_execute_oracle_sql_query
ansible_role_sra_execute_oracle_sql_query.

defaults/main.yml
---
# Default values                         # YAML start — this file defines default variables for the role

sql_query: "SELECT SYSDATE FROM DUAL"     # The default SQL query to run if none is provided. 
                                          # Here it returns the current Oracle database date/time.

oracle_host: ""                           # Oracle database hostname or IP. Must be supplied from SRA/SNOW input.
oracle_port: "1521"                      # Default Oracle Listener port (1521). Can be overridden.
oracle_service: ""                        # Oracle Service Name or SID (e.g., ORCL). Required for DB connection.

oracle_user: ""                           # Oracle username for authentication.
oracle_password: ""                       # Oracle password. (Can be replaced by AAP credential store.)

# Output file (temporary)
sql_output_file: "/tmp/oracle_sql_output.txt"
                                          # Location where SQL output will be temporarily saved
                                          # before being returned to CACF/SNOW.

# ServiceNow Callback (optional, if needed)
sn_callback_url: ""                       # If your flow supports callback -> SNOW REST API endpoint.
ritm_number: ""                           # The RITM number so CACF can attach or return results to the right request.


Explanation & notes

--- — YAML document start marker.

sql_query — default query; safe sample is SELECT SYSDATE FROM DUAL. This will run if no other value is provided.

oracle_host, oracle_port, oracle_service — DB connection parameters. oracle_port is defaulted to the common Oracle port 1521.

oracle_user, oracle_password — credentials; do not store plaintext passwords in defaults for production — use Ansible Vault, AAP credential store, or environment / secret manager.

sql_output_file — temporary file on the target host where sqlplus SPOOL output is written.

sn_callback_url, ritm_number — optional ServiceNow callback and request number to send results back.

files/config.yml
---
oracle:
  default_port: 1521
  client: "sqlplus"


Explanation

oracle: a parent object to group Oracle-specific static config.

default_port: 1521 — canonical numeric default port.

client: "sqlplus" — name of client to use (you later check for sqlplus).

files/extract_sra_data.yml
---
# Extract variables from ServiceNow payload sr_data

sql_query: "{{ sr_data['ITDORASQLQUERY'] | default('not_found') }}"
oracle_host: "{{ sr_data['ITDORAHOST'] | default('not_found') }}"
oracle_port: "{{ sr_data['ITDORAPORT'] | default('1521') }}"
oracle_service: "{{ sr_data['ITDORASID'] | default('not_found') }}"
oracle_user: "{{ sr_data['ITDORAUSER'] | default('not_found') }}"
oracle_password: "{{ sr_data['ITDORAPASS'] | default('not_found') }}"

flow_id: "{{ sr_data['Classification'] | default('ITDRESORA001') }}"
expected_start: "{{ sr_data['Expected Start'] | default('now') }}"


Explanation

This file maps values from an incoming ServiceNow payload variable sr_data into Ansible variables used by your role.

Each line uses Jinja2 lookup syntax {{ ... }} and default() filter if the key missing.

e.g. sql_query reads sr_data['ITDORASQLQUERY'], else 'not_found'.

oracle_port sets a default of string '1521' (note: earlier in config.yml it’s numeric 1521).

flow_id and expected_start map classification and expected start fields from the payload.

Important potential bug: you used sr_data['Expected Start'] (with a space) here, but in mapper_config.yml the key is ExpectedStart (no space). If your incoming payload actually uses Expected Start or ExpectedStart you must ensure consistency — mismatch will produce default('now') or not_found.

meta/main.yml
---
galaxy_info:
  role_name: sra_execute_oracle_sql_query
  author: Satish satti
  description: "Execute SELECT-only SQL queries on Oracle using sqlplus for SRA/AAP automation"
dependencies: []


Explanation

galaxy_info metadata used by Ansible Galaxy and readers.

role_name, author, description — basic role metadata.

dependencies: [] — no dependent roles.

tasks/main.yml
---
- name: Validate SQL query is SELECT-only
  assert:
    that:
      - sql_query is regex("^(?i)select.*")
    fail_msg: "Only SELECT statements are allowed."

- name: Check if sqlplus is installed
  command: which sqlplus
  register: sqlplus_check
  changed_when: false
  failed_when: sqlplus_check.rc != 0

- name: Execute Oracle SQL query
  shell: |
    sqlplus -S {{ oracle_user }}/{{ oracle_password }}@'{{ oracle_host }}:{{ oracle_port }}/{{ oracle_service }}' <<EOF
    SET PAGESIZE 50000
    SET LINESIZE 32767
    SET FEEDBACK OFF
    SET HEADING OFF
    SET TRIMSPOOL ON
    SPOOL {{ sql_output_file }}
    {{ sql_query }};
    SPOOL OFF
    EXIT;
    EOF
  register: sql_result
  changed_when: true

- name: Read SQL query output
  slurp:
    src: "{{ sql_output_file }}"
  register: raw_output

- name: Decode SQL output
  set_fact:
    decoded_output: "{{ raw_output.content | b64decode }}"

- name: Print SQL output (visible in job log + sent back to CACF/SN)
  debug:
    msg: "{{ decoded_output }}"

- name: Send output back to ServiceNow RITM
  uri:
    url: "{{ sn_callback_url }}"
    method: POST
    headers:
      Content-Type: "application/json"
    body_format: json
    body:
      ritm: "{{ ritm_number }}"
      status: "SUCCESS"
      output: "{{ decoded_output }}"
  when: sn_callback_url != ""


Line-by-line explanation and notes

- name: Validate SQL query is SELECT-only — task name.

assert: — Ansible module that enforces condition(s).

that: — list of assertions. sql_query is regex("^(?i)select.*") uses the regex test; (?i) makes it case-insensitive. This checks the sql_query starts with select (prevents non-SELECT statements).

Caveat: this is a simple check — it can be bypassed with leading comments/whitespace or multi-statement queries. Consider normalizing whitespace and explicitly forbidding semicolons with another check if needed.

fail_msg: — message shown if assert fails.

- name: Check if sqlplus is installed — ensures client present.

command: which sqlplus — runs which to find sqlplus.

register: sqlplus_check — saves result.

changed_when: false — this task should not mark host changed.

failed_when: sqlplus_check.rc != 0 — mark failed if command exit code non-zero (i.e. not found).

- name: Execute Oracle SQL query — runs the query.

shell: | — multi-line shell command block. Details inside:

sqlplus -S {{ oracle_user }}/{{ oracle_password }}@'{{ oracle_host }}:{{ oracle_port }}/{{ oracle_service }}'

-S = silent mode (suppresses banner, but still prints query output).

Security issue: passing oracle_password on command line exposes it to ps listing and logs; better to use Oracle wallet, environment variables, or credential stores.

Connection string format: user/password@'host:port/service'. The single quotes help with @ and slashes in the shell.

SET PAGESIZE 50000 — increase pagesize to reduce page breaks.

SET LINESIZE 32767 — max line length for output (reduce wrapping/truncation).

SET FEEDBACK OFF — disables "X rows selected" messages.

SET HEADING OFF — disables column headings (you may prefer headings on).

SET TRIMSPOOL ON — trims trailing spaces in spool file.

SPOOL {{ sql_output_file }} — redirect session output into file defined in sql_output_file.

{{ sql_query }}; — the SQL statement run. The trailing semicolon is required by SQL*Plus to execute the statement.

SPOOL OFF — close spool.

EXIT; — exit SQL*Plus.

register: sql_result — stores the shell task result (stdout/stderr/rc).

changed_when: true — mark this task as changed (you could be more nuanced depending on rc).

- name: Read SQL query output — read the file produced by spool.

slurp: — Ansible module to read a file and return it base64-encoded (works across platforms).

src: "{{ sql_output_file }}" — source file path.

register: raw_output — store slurp result.

- name: Decode SQL output — decode base64 into usable string.

set_fact: — set a fact variable.

decoded_output: "{{ raw_output.content | b64decode }}" — decode the content field returned by slurp (which is base64) into decoded_output.

- name: Print SQL output (visible in job log + sent back to CACF/SN) — show the output in the Ansible job log.

debug: msg: "{{ decoded_output }}" — prints decoded output.

- name: Send output back to ServiceNow RITM — optional callback to ServiceNow.

uri: — Ansible HTTP client module.

url: "{{ sn_callback_url }}" — endpoint to call.

method: POST — HTTP method.

headers: content type JSON.

body_format: json — send JSON body.

body: — payload containing ritm, status, and output.

when: sn_callback_url != "" — only run if callback URL supplied.

Additional notes & improvements

Remove or securely handle oracle_password to avoid exposure. Passing credentials in command-line is insecure.

Consider oracle_home/ORACLE_SID environment variables or sqlplus /@tns alternatives.

Consider cleaning up sql_output_file after sending results.

Consider capturing sql_result.rc and failing gracefully if non-zero.

slurp returns bytes base64 — correct decoding handled.

.yamllint
extends: default

rules:
  line-length:
    max: 120
  truthy:
    check-keys: false


Explanation

Linter config for yamllint.

extends: default — inherit default rules.

line-length.max: 120 — allow up to 120 columns per line.

truthy.check-keys: false — disables warnings for keys like on/off/yes/no being used as booleans (not checking keys for truthy values).

ansible_role_sra_execute_oracle_sql_query.yml
---
- name: sra_execute_oracle_sql_query
  hosts: all
  gather_facts: true
  become: false

  vars_files:
    - files/config.yml
    - files/extract_sra_data.yml

  tasks:
    - name: Include main role tasks
      include_tasks: tasks/main.yml


Explanation

This is a playbook that runs the role logic as a play.

hosts: all — run on all hosts in inventory (typical for local automation you might use localhost).

gather_facts: true — collect facts about hosts.

vars_files: includes files/config.yml and files/extract_sra_data.yml to load ServiceNow payload mappings and static config.

include_tasks: tasks/main.yml — include the task list you defined (this simply runs the tasks file). This is equivalent to using the role from roles: but kept here as a simple play.

mapper_config.yml
---
version: "1.0"

# Maps ServiceNow variables -> Ansible variables

variable_mapping:
  ITDORASQLQUERY: "sql_query"
  ITDORAHOST: "oracle_host"
  ITDORAPORT: "oracle_port"
  ITDORASID: "oracle_service"
  ITDORAUSER: "oracle_user"
  ITDORAPASS: "oracle_password"

  Classification: "flow_id"
  ExpectedStart: "expected_start"

  RITM: "ritm_number"
  CALLBACKURL: "sn_callback_url"

defaults:
  oracle_port: 1521
  flow_id: "ITDRESORA001"

required:
  - ITDORASQLQUERY
  - ITDORAHOST
  - ITDORASID
  - Classification
  - ExpectedStart


Explanation

This file documents how incoming ServiceNow variable names map into the Ansible variable names used in the role.

variable_mapping keys are the ServiceNow field names, values are the Ansible variable names.

defaults provides fallback defaults for mappings if missing.

required lists required ServiceNow fields that must be present.

Important notes

As earlier, extract_sra_data.yml used the key 'Expected Start' (with a space) while mapper_config.yml expects ExpectedStart (no space). Ensure your ServiceNow payload naming is consistent with mapper_config.yml OR adjust extract_sra_data.yml. This mismatch will cause missing data or defaults to be used.

CALLBACKURL vs sn_callback_url mapping must match the payload key (case sensitivity depends on how payload is passed).

playbook.yml
---
- name: Run Oracle SQL Query from SRA
  hosts: all
  gather_facts: true
  become: false
  vars_files:
    - files/config.yml
    - files/extract_sra_data.yml

  roles:
    - ansible_role_sra_execute_oracle_sql_query


Explanation

Another top-level playbook that uses the role by including it in roles: (this is the conventional way to use a role).

vars_files again loads config and extraction mapping into the play context before the role runs.

Final quick checklist / recommended fixes

Secrets: do not pass oracle_password on command line. Use Ansible Vault, AAP credential storage, or use environment-based secure authentication / wallet.

SQL validation: the regex check is helpful but minimal — strip leading comments/whitespace and ensure no semicolons or multiple statements if you want stronger protection.

Key name mismatch: fix Expected Start vs ExpectedStart mismatch between extract_sra_data.yml and mapper_config.yml (or standardize the payload).

Error handling: check sql_result.rc and sql_result.stderr to fail or mark status "FAILED" in callback if needed.

Remove temporary file: consider file: state=absent path={{ sql_output_file }} after sending results to avoid leaving secrets on disk.

Avoid which + command module for presence: you can use ansible.builtin.package or stat or command -v sqlplus — current method works but be mindful of cross-platform differences.

Consider Oracle modules: for more advanced integration, there are community modules (like community.oracle) or using Python drivers — though sqlplus is fine for quick SELECTs.
