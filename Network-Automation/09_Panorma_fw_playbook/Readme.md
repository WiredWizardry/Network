PANORAMA SECURITY RULE DEPLOYMENT
=================================

This document explains how to use an Ansible playbook that automates the creation of a security rule on Palo Alto Networks Panorama. Inputs are collected via an AWX survey. The playbook handles service object creation, rule configuration, and—if requested—commits and pushes the change to your managed firewalls.

---------------------------------
Prerequisites
---------------------------------

You will need:

* An AWX or Ansible Tower instance with this playbook in its project repository.
* Network connectivity from the AWX host to your Panorama management IP or DNS name on the API port (default 443).
* A valid Panorama API key generated via the Panorama GUI or API.
* Python 3 on the AWX control node, plus these libraries (installed automatically by the playbook’s pre-task):
    * pan-os-python version 1.7.3 or newer
    * pan-python version 0.17.0 or newer
    * xmltodict version 0.12.0 or newer

---------------------------------
AWX Survey Fields
---------------------------------

The survey must include the following variables:

- panorama_host
	Type: string
	Default: none
	What It Does: IP address or hostname for Panorama’s management interface.

- panorama_api_key
	Type: string
	Default: none
	What It Does: API key for authenticating to Panorama.

- panorama_port
	Type: integer
	Default: 443
	What It Does: TCP port for the Panorama API if it differs from 443.

- device_group_name
	Type: string
	Default: none
	What It Does: Name of the Panorama device group where the security rule will be placed.

- rule_name_input
	Type: string
	Default: none
	What It Does: The unique name you want to assign to the security rule.

- rule_description
	Type: string
	Default: Security rule created by AWX
	What It Does: A human-readable description to store alongside the rule.

- source_zone
	Type: string
	Default: trust
	What It Does: Panorama zone from which traffic will be allowed.

- destination_zone
	Type: string
	Default: untrust
	What It Does: Panorama zone to which traffic will be allowed.

- source_ip_list
	Type: string
	Default: any
	What It Does: Comma-separated list of source IP addresses or network objects. Use “any” to allow all sources.

- dest_ip_list
	Type: string
	Default: any
	What It Does: Comma-separated list of destination IP addresses or network objects. Use “any” to allow all.

- service_list
	Type: string
	Default: (empty)
	What It Does: Comma-separated list of service definitions, each in the form name/protocol/port.

- commit_changes
	Type: choice
	Default: no
	What It Does: Set to 'yes' to commit and push the configuration; set to 'no' to preview without committing.

---------------------------------
Service List Format
---------------------------------

You specify custom services by listing entries separated by commas. Each entry must use the pattern: <service-base-name>/<protocol>/<port>
For example: http/tcp/80, https/tcp/443, dns/udp/53

The playbook will:
* Split on commas and trim whitespace.
* For every valid entry:
    * Construct a service object named “base-name-protocol-port” (for example, http-tcp-80).
    * Set the protocol to tcp or udp.
    * Use the numeric port.
    * Store the base name as the object’s description.
* Create any missing service objects in the specified device group.

If you leave this field empty or set it to “any,” then the playbook will use the built-in Panorama “any” service.

---------------------------------
How the Playbook Works
---------------------------------

1. Disables fact gathering to speed up execution, since no host facts are needed.
2. Installs required Python libraries for the paloaltonetworks.panos collection.
3. Reads AWX survey inputs into Ansible variables. It builds:
    * The connection dictionary (panos_provider) with host, API key, and port.
    * Lists of source and destination IPs, defaulting to the Panorama special object “any.”
    * A parsed list of custom service dictionaries.
    * A final list of service names, falling back to ['any'] if none are defined.
    * A Boolean flag (do_commit) that tracks whether you chose to commit and push.
4. Validates that both the device group and the rule name are present. The run fails fast if either is missing.
5. Ensures service objects exist by looping over the parsed list and creating any that are not already present.
6. Displays a configuration summary so you can verify all values before changes are made.
7. Creates or updates the security rule in the specified device group using the panos_security_rule module. The rule will:
    * Allow traffic from the specified source zone(s) and IP(s)
    * Allow traffic to the specified destination zone(s) and IP(s)
    * Match any application
    * Match the computed service list
    * Log at session end
    * Be placed in the pre-rulebase
8. Commits to Panorama if you set commit_changes to 'yes'.
9. Pushes the configuration to the managed firewalls in the device group when the commit is successful.
10. Prints a final summary indicating success or failure, whether changes were applied, which services were used, and whether the commit occurred.

---------------------------------
Running the Playbook
---------------------------------

1. Add this playbook to your AWX project and link it to a job template.
2. Configure the AWX survey according to the fields listed above. Provide clear help text so engineers know what each field does.
3. Launch the job. Review the initial preview in the job output.
4. Confirm that the inputs look correct. If you opted not to commit, you can inspect the Panorama candidate config without making any changes.
5. When you are ready, set commit_changes to 'yes' and run again to apply and push the rule.

---------------------------------
Additional Tips
---------------------------------

* Choose descriptive, project-oriented rule names (for example, ALLOW_WEB_TO_DMZ) so that other engineers can immediately understand intent.
* Use the preview mode (commit_changes set to 'no') to build confidence before affecting production devices.
* Keep service definitions consistent with naming conventions in your organization.
* You can extend the playbook to handle application groups, schedule objects, tags or other Panorama features by adding new survey fields and tasks.

This README should give engineers sufficient background to configure, test, and maintain the playbook without referring back to the source code.