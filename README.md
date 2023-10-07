# URL Shortener Load Testing

## Overview
I was tasked with deploying a new version of the URL Shortener application and ensuring it could handle a load test of 14,000 concurrent users. The initial test exposed some performance issues under high load. I took steps to remediate these issues and re-tested on a larger instance type from the m4 instance family to successfully handle the required user load.

## Initial Deployment and Configuration
- Verified AWS CLI was installed locally
  - It was not installed, so installed with: `sudo apt install awscli`
  - Checked version with `aws --version` to confirm successful install
- Configured AWS CLI with `aws configure`
- Ensured application was running
- Added logging to application:
  - Imported logging module
  - Added `logging.basicConfig` to log to `app.log` file
 
<img width="1083" alt="blitz2_img5" src="https://github.com/belindadunu/Blitz2/assets/139175163/e91f5bc2-cbc7-44d1-b48f-5ba934f10097">

  - Committed and pushed logging changes to repo
- **Purpose of Nginx and Gunicorn:**
  - Nginx is a proxy server, Gunicorn runs the Flask app
  - Running Nginx in front protects the application
  - Application data saved in JSON file for now but further data persistence needs to be implemented
  - Added an additional layer of security by closing Gunicorn port and having Nginx handle incoming traffic

## Initial Test
- Deployed application on t2.medium EC2 instance
  - The t2 instance family provides burstable CPU performance which can be problematic for sustaining high loads
- Configured Nginx with 8 worker processes and connections increased to 2000
<img width="381" alt="blitz2_img13" src="https://github.com/belindadunu/Blitz2/assets/139175163/b68a19e1-961a-4386-a657-51cf93f1a847">

- Enabled Gzip compression in Nginx for faster load times
<img width="1276" alt="blitz2_img14" src="https://github.com/belindadunu/Blitz2/assets/139175163/776e1eb8-a0cb-49a2-bfd2-089b69c8b76e">

- Wrote a bash script to simulate load: `sudo nice -n -20 stress-ng --cpu 2 &`
  - This script consumes 2 CPU cores to simulate user load
<img width="1027" alt="blitz2_img6" src="https://github.com/belindadunu/Blitz2/assets/139175163/da2df2ed-53be-416f-bbfe-79be5c50bc24">

- QA engineer kicked off 14,000 concurrent requests to stress test the application
<img width="1203" alt="blitz2_img7" src="https://github.com/belindadunu/Blitz2/assets/139175163/6a79850a-d1de-41ee-8ce8-5f179fd2cb6f">


### Initial Test Results
- Application served 13,992 requests successfully
- 8 users received errors and were unable to access the application
<img width="980" alt="blitz2_img8" src="https://github.com/belindadunu/Blitz2/assets/139175163/dc1b1241-cc25-4752-8563-49b188f015a4">

- CPU usage spiked to 99%:
<img width="758" alt="blitz2_img10" src="https://github.com/belindadunu/Blitz2/assets/139175163/b4a45f56-42b2-4225-8de1-e1a33c4d8706">

- Logging showed request timeouts and 500 errors
<img width="1420" alt="blitz2_img9" src="https://github.com/belindadunu/Blitz2/assets/139175163/9b9a5480-9d93-4362-bca9-7536c68d2018">

## Diagnosis and Remediation
- Reviewed logs and monitoring data
<img width="1186" alt="blitz2_img11" src="https://github.com/belindadunu/Blitz2/assets/139175163/fa7a0840-0dae-4cb6-a1f3-996888d4b2d7">
<img width="1422" alt="blitz2_img12" src="https://github.com/belindadunu/Blitz2/assets/139175163/2fc9748c-342d-4741-a867-5699527fcd95">

- Determined the t2.medium instance did not have adequate resources to handle the load
- Considered m4, c4, and r4 instance families and selected m4.xlarge as best fit
- Created a t2.micro instance to test instance type change script:

```bash
#!/bin/bash

# Vars
INSTANCE_ID=""
NEW_TYPE=""

# Help message
echo "Usage: change_instance.sh -i <instance-id> -t <new-instance-type>"
echo "Example: change_instance.sh -i i-012345678910abcd -t t2.micro"

# Prompt the user for instance ID
read -p "Enter instance ID: " INSTANCE_ID

# Prompt the user for new instance type
read -p "Enter new instance type: " NEW_TYPE

# Get arguments
# based on https://www.geeksforgeeks.org/getopts-command-in-linux-with-examples/
while getopts ":i:t:" option; do
  case $option in
    i) INSTANCE_ID=$OPTARG;; # Handle the -i option (INSTANCE_ID)
    t) NEW_TYPE=$OPTARG;; # Handle the -t option (NEW_TYPE)
    *) show_help # Handle any other unknown options or missing argument errors
   ;;
  esac
done

# Stop instance
aws ec2 stop-instances --instance-ids $INSTANCE_ID

# Wait for instance to reach the stopped state
aws ec2 wait instance-stopped --instance-ids $INSTANCE_ID

# Change instance type
aws ec2 modify-instance-attribute --instance-id $INSTANCE_ID --instance-type "{"Value": "$NEW_TYPE"}"

# Start instance
aws ec2 start-instances --instance-ids $INSTANCE_ID

echo "Instance $INSTANCE_ID changed to type $NEW_TYPE"
```

- Tested script and changed instance to t2.medium
- Verified new instance type with `aws ec2 describe-instance-types`
- Migrated application to m4.xlarge instance

## Final Load Test Results
- Reran test with 14,000 concurrent users on m4.xlarge instance
- Application served 100% of traffic without crashes or degradation
- CPU usage stabilized around 40%
- No request timeouts or 500 errors

## Conclusion
The t2.medium instance lacked the resources to handle the target load. Migrating to m4.xlarge provided the necessary CPU, memory, and network capacity to successfully serve 14,000 concurrent users. The application is now optimized for the traffic demands based on the load test results.

## Instance Sizing Guide
When selecting an EC2 instance type to handle a certain load, there are a few key specifications to consider:

- **vCPUs:** The number of virtual CPUs or compute cores an instance has. More CPUs enable handling more parallel computing processes and traffic.
- **RAM:** The amount of working memory available to applications. More RAM allows working with more data in memory without slowdowns.
- **Gbps:** The maximum network bandwidth or data transfer rate. Higher Gbps means faster data transfers over the network.

### Rules of Thumb:
- Estimate 50-100 kbps bandwidth per user on average
- Estimate 1 Mbps per 2-3 users average for web traffic
- Allow for 2-3x projected bandwidth for spikes and growth

To handle 14,000 users with 50 kbps average bandwidth:

- 14,000 users x 50 kbps = 700 Mbps = 0.7 Gbps minimum required
- An instance with 10 Gbps bandwidth has plenty of headroom

When sizing the instance:

- Consider the projected bandwidth based on the number of users and average usage
- Allow for growth by selecting an instance well above the minimum required
- Prioritize having adequate CPU cores and RAM for your application

For this application, the m4.xlarge with 4 vCPUs and 16GB RAM provided the right resources to handle the load.

## References
- [EC2 Instance Types](https://aws.amazon.com/ec2/instance-types/)
- [EC2 Network Bandwidth](https://aws.amazon.com/ec2/networking/)
