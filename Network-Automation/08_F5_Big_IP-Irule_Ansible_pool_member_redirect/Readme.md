

PoC Key Functionalities
Targeted Traffic Management: Applies to traffic for "/index.html*", ensuring precise control.
Dynamic Pool Routing: Determines traffic routing based on the 'failover_initiate' state.
Versatile Failover Management: Handles both immediate and timed failover scenarios.
Load Balancing: In load-balancing mode, traffic is alternated between the Blue and Green pools.


Scenarios Covered by the iRule
Session Persistence based on "Blue-Green" Cookie
If the request is under "/img/" and the "Blue-Green" cookie is set to "Blue", the request is directed to the Blue pool.
If the request is under "/img/" and the "Blue-Green" cookie is set to "Green", the request is directed to the Green pool.
Dynamic Load Balancing with User-Defined Percentages and Elapsed Time
If the request is under "/img/" and the "Blue-Green" cookie is not set, the iRule performs load balancing based on user-defined percentages and elapsed time.
During the failover duration, traffic is gradually shifted between the Blue and Green pools based on the elapsed time.
After the failover duration, traffic is distributed between the Blue and Green pools according to the user-defined percentages.
Fallback Handling for Unhealthy Pools
If both Blue and Green pools are down during load balancing, the iRule logs an error and sends a "503 Service Unavailable" response.
Setting "Blue-Green" Cookie in HTTP Response
When the "Blue-Green" header is present in the HTTP response, the iRule sets a corresponding "Blue-Green" cookie with appropriate attributes.


Traffic flow for Scenario
Scenario 1: Session Persistence based on "Blue-Green" Cookie

User -> VIP -> iRule checks for "Blue-Green" cookie under "/img/" -> Directs traffic to the corresponding pool (Blue or Green) -> Destination Server in the selected pool.
Scenario 2: Dynamic Load Balancing with Admin-Defined Percentages and Elapsed Time

User -> VIP -> iRule checks for "Blue-Green" cookie under "/img/" -> If cookie is not set, performs load balancing based on admin percentages-> During failover duration, gradually shifts traffic between Blue and Green pools based on elapsed time -> After failover duration, distributes traffic based on admin-defined percentages -> Destination Server in the selected pool.
Scenario 3: Fallback Handling for Unhealthy Pools

User -> VIP -> iRule checks for "Blue-Green" cookie under "/img/" -> If cookie is not set, performs load balancing -> If both Blue and Green pools are down, logs an error and sends a "503 Service Unavailable" response.
Scenario 4: Setting "Blue-Green" Cookie in HTTP Response

Destination Server -> iRule checks for "Blue-Green" header in the HTTP response -> If the header is present, sets a corresponding "Blue-Green" cookie with appropriate attributes -> Response sent to the User.

General Behavior for All Scenarios
The iRule processes requests under the "/img/" path and performs actions based on the presence and value of the "Blue-Green" cookie.
SSL is disabled on the server-side for the processed requests.
The iRule retrieves failover duration, start time, and pool percentages from the "BlueGreenConfigtest" data group.
The health threshold for the pools is set to 2 healthy members.
If the "Blue-Green" cookie is set, the iRule directs traffic to the corresponding pool (Blue or Green).
If the "Blue-Green" cookie is not set, the iRule performs load balancing based on user-defined percentages and elapsed time.
During the failover duration, traffic is gradually shifted between the Blue and Green pools based on the elapsed time.
After the failover duration, traffic is distributed between the Blue and Green pools according to the user-defined percentages.
If both Blue and Green pools are down during load balancing, the iRule logs an error and sends a "503 Service Unavailable" response.
When the "Blue-Green" header is present in the HTTP response, the iRule sets a corresponding "Blue-Green" cookie with appropriate attributes.
The iRule configuration can be dynamically updated using Ansible automation by modifying the "BlueGreenConfigtest" data group.
Summary
In "Blue" or "Green" scenarios, the iRule routes traffic based on the health of the specified and alternative pools, with an immediate or gradual shift to the primary pool depending on the failover_duration.
The iRule introduces session persistence based on the "Blue-Green" cookie for requests under the "/img/" path. It also incorporates Admin-defined percentages and elapsed time for load balancing when the cookie is not set. The iRule handles fallback scenarios when both pools are down and sets the "Blue-Green" cookie in the HTTP response when the corresponding header is present. The iRule's configuration can be dynamically updated using Ansible automation through the "BlueGreenConfigtest" data 

General Behavior for All Scenarios
The iRule processes requests under the "/img/" path and performs actions based on the presence and value of the "Blue-Green" cookie.
SSL is disabled on the server-side for the processed requests.
The iRule retrieves failover duration, start time, and pool percentages from the "BlueGreenConfigtest" data group.
The health threshold for the pools is set to 2 healthy members.
If the "Blue-Green" cookie is set, the iRule directs traffic to the corresponding pool (Blue or Green).
If the "Blue-Green" cookie is not set, the iRule performs load balancing based on user-defined percentages and elapsed time.
During the failover duration, traffic is gradually shifted between the Blue and Green pools based on the elapsed time.
After the failover duration, traffic is distributed between the Blue and Green pools according to the user-defined percentages.
If both Blue and Green pools are down during load balancing, the iRule logs an error and sends a "503 Service Unavailable" response.
When the "Blue-Green" header is present in the HTTP response, the iRule sets a corresponding "Blue-Green" cookie with appropriate attributes.
The iRule configuration can be dynamically updated using Ansible automation by modifying the "BlueGreenConfigtest" data group.
Summary
In "Blue" or "Green" scenarios, the iRule routes traffic based on the health of the specified and alternative pools, with an immediate or gradual shift to the primary pool depending on the failover_duration.
The iRule introduces session persistence based on the "Blue-Green" cookie for requests under the "/img/" path. It also incorporates Admin-defined percentages and elapsed time for load balancing when the cookie is not set. The iRule handles fallback scenarios when both pools are down and sets the "Blue-Green" cookie in the HTTP response when the corresponding header is present. The iRule's configuration can be dynamically updated using Ansible automation through the "BlueGreenConfigtest" data 



F5 irule Example

when HTTP_REQUEST {
    switch -glob [string tolower [HTTP::uri]] {
        "/img/*" {
            # Check for the "Blue-Green" cookie only for requests under /img/
            set poolCookie [HTTP::cookie value "Blue-Green"]

            # Disable SSL on serverside if required
            SSL::disable serverside

            # Retrieve failover initiation, duration, start time, and percentages from data groups
            set failover_duration [class match -value -- "duration" equals BlueGreenConfigtest]
            set failover_start_time [class match -value -- "start_time" equals BlueGreenConfigtest]
            set Green_percentage [class match -value -- "Green_percentage" equals BlueGreenConfigtest]
            set Blue_percentage [class match -value -- "Blue_percentage" equals BlueGreenConfigtest]

            # Validate percentage values
            if { $Green_percentage + $Blue_percentage != 100 } {
                log local0. "Error: Green_percentage and Blue_percentage must add up to 100"
                # Handle the error condition, e.g., send a default response or return an error
                return
            }

            # Health threshold set to 2 healthy members
            set health_threshold 2

            # Normalize the query string to lower case for consistent comparison
            set query [string tolower [HTTP::query]]
            
            # Session persistence based on the "Pool" cookie and query string contains "color=blue"
            if { $poolCookie equals "Blue" || [string tolower [HTTP::query]] contains "color=blue" } {
                HTTP::header insert "Blue-Green" "blue"
                pool Blue
            } elseif { $poolCookie equals "Green" || [string tolower [HTTP::query]] contains "color=green" } {
                HTTP::header insert "Blue-Green" "green"
                pool Green
            } else {
                # Load Balancing logic with user-defined percentages and elapsed time
                    set current_time [clock seconds]
                    set elapsed_time [expr {$current_time - $failover_start_time}]

                    if { $elapsed_time < $failover_duration } {
                        set rand_val [expr {rand() * 100}]
                        if { $rand_val < (100 * $elapsed_time / $failover_duration) } {
                            if { [active_members Blue] >= $health_threshold } {
                                HTTP::header insert "Blue-Green" "blue"
                                pool Blue
                            } elseif { [active_members Green] >= $health_threshold } {
                                HTTP::header insert "Blue-Green" "green"
                                pool Green
                            } else {
                                # Handle scenario when both pools are down
                                log local0. "Error: Both Blue and Green pools are down"
                                # Send a default response or return an error
                                HTTP::respond 503 content "Service Unavailable"
                            }
                        } else {
                            if { [active_members Green] >= $health_threshold } {
                                HTTP::header insert "Blue-Green" "green"
                                pool Green
                            } elseif { [active_members Blue] >= $health_threshold } {
                                HTTP::header insert "Blue-Green" "blue"
                                pool Blue
                            } else {
                                # Handle scenario when both pools are down
                                log local0. "Error: Both Blue and Green pools are down"
                                # Send a default response or return an error
                                HTTP::respond 503 content "Service Unavailable"
                            }
                        }
                    } else {
                        set rand_val [expr {rand() * 100}]
                        if { $rand_val < $Blue_percentage } {
                            HTTP::header insert "Blue-Green" "blue"
                            pool Blue
                        } else {
                            HTTP::header insert "Blue-Green" "green"
                            pool Green
                        }
                    }
            }
        }
        default {
            # don't do anything...
        }
    }
}

when HTTP_RESPONSE {
    set headerValue [HTTP::header value "Blue-Green"]
    if {$headerValue ne ""} {
        switch $headerValue {
            "blue" {
                # Ensure proper cookie format and insertion
                HTTP::cookie insert name "Blue-Green" value "Blue"
                # Set the cookie expiration time to 10 seconds from now
                HTTP::cookie expires "Blue-Green" 10
            }
            "green" {
                # Ensure proper cookie format and insertion
                HTTP::cookie insert name "Blue-Green" value "Green"
                # Set the cookie expiration time to 10 seconds from now
                HTTP::cookie expires "Blue-Green" 10
            }
            default {
                # Handle unexpected header values or implement a default action
            }
        }
    } else {
        # Explicitly taking no action for missing "Blue-Green" header
    }
}

Ansible Task Details

The Ansible playbook contains tasks for setting the BlueGreenFailoverStartTime and modifying the BlueGreenConfigtest data group. This automation ensures that changes to the iRule's behavior are consistent and error-free.



Key Ansible Tasks

Set Failover Start Time: Captures the current Unix timestamp as the start time for failover.
Modify Data Group: Updates the BlueGreenConfigtest with values for duration Blue_percentage, Green_percentage, and start_time.
Variables in Ansible Tower

failoverduration
Green_percentage
Blue_percentage


Ansible Data Group Modification Example


Task file for Blue/Green bleed over/failover
Variables defined in ansible tower - failover_initiate, failoverduration
Datagroup is BlueGreenConfigtest being referenced in irule irule-test-blue-green
    - name: Check if Green_percentage and Blue_percentage sum exceeds 100
      fail:
        msg: "The sum of the Blue and Green Pool Percentage cannot exceed 100!"
      when: (Green_percentage | int) + (Blue_percentage | int) > 100

    - name: Set BlueGreenFailoverStartTime to current Unix timestamp
      set_fact:
        failover_start_time: "{{ lookup('pipe', 'date +%s') }}"

    - name: Convert the Variable Min from minute to Seconds
      set_fact:
        failoverduration: "{{ min * 60 }}"

    - name: Modify BlueGreenConfigtest Data Group
      bigip_data_group:
        name: "BlueGreenConfigtest"
        internal: yes
        type: "string"
        records:
          - key: "duration"
            value: "{{ failoverduration }}"
          - key: "start_time"
            value: "{{ failover_start_time }}"
          - key: "Green_percentage"
            value: "{{ Green_percentage }}"
          - key: "Blue_percentage"
            value: "{{ Blue_percentage }}"
        state: present
        provider: "{{ provider }}"
		
Variables file for Blue/Green bleed over/failover

Variables defined in ansible tower - failover_initiate, failoverduration
Datagroup is BlueGreenConfigtest being referenced in irule irule-test-blue-green

F5_node: "{{ F5_nodes | extract_ip }}"

    provider:

      server: "{{ F5_node }}"

      user: "{{ user_username }}"

      password: "{{ user_pass }}"

      validate_certs: no

Main .yml file

Variables defined in ansible tower - failover_initiate, failoverduration
Datagroup is BlueGreenConfigtest being referenced in irule irule-test-blue-green
- name: Blue Green Testing

  hosts: "{{ F5_nodes | extract_ip }}"

  connection: local

  gather_facts: no

  vars_files:

    - roles/svc_ansnetwork.yml

  roles:

    - role: F5_Blue_Green_Test

Conclusion
The integration of Ansible automation with the F5 iRule enhances our network's flexibility and resilience. This setup allows us to manage traffic efficiently during deployment and failover processes, ensuring minimal disruption and maintaining high availability.