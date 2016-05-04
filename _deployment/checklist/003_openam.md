---
title: OpenAM Installation Checklist
permalink: "deployment/checklist/openam.html"
layout: "document"
categories: ["deployment", "checklist"]
---

## Overview

| Item | Description |
|:-----|:------------|
| Purpose | Provide authentication for TDS components |
| Communicates With | Progman, Permissions, ART, Proctor, Teacher Hand-Scoring System,TestSpecBank |
| Repository Location | [https://bitbucket.org/sbacoss/openam12_release](https://bitbucket.org/sbacoss/openam12_release) |
| Additional Documentation | [SBAC OpenAM Installation](https://bitbucket.org/sbacoss/openam12_release/src/66cc50368d39bce89d22913d5be74eb96d1d0b90/sbacOpenAM-Installation-04152015.pdf?at=default), [SBAC SSO Design](https://bitbucket.org/sbacoss/openam12_release/src/66cc50368d39bce89d22913d5be74eb96d1d0b90/SBAC_SSO_Design-v2.0-04132015.pdf?at=default) |

## Instructions

### Create AWS Instance
* Create server instance to host OpenAM software
  * AWS instance type must be at least **t2.medium**
* Create an AWS security group with the following ports for inbound TCP traffic (can be done during instance creation):
  * 22
  * 1689
  * 4444
  * 8005
  * 8080

### Create Load Balancer (***OPTIONAL***)
* Create a load balancer (additional information [here](https://docs.aws.amazon.com/ElasticLoadBalancing/latest/DeveloperGuide/elb-getting-started.html?icmpid=docs_elb_console)) for the server instance that will host OpenAM:
  * Choose a meaningful name
  * Choose the appropriate VPC ([Virtual Private Cloud](http://docs.aws.amazon.com/AmazonVPC/latest/UserGuide/VPC_Introduction.html))
    * For any environment besides Production, "My Default VPC" should be sufficient.  For a Production environment, consult your system administrator(s)
  * Leave "Create an internal load balancer" unchecked
  * Leave "Enable advanced VPC configuration" unchecked
    * HTTP: forward port 80 to port 8080
    * HTTPS: forward port 443 to port 8080
  * Listener configuration:
  * Create a security group for the load balancer:
    * Choose a meaningful name
    * Inbound: 443 from anywhere (0.0.0.0/0)
  * Health check:
    * Ping protocol: HTTP
    * Ping port: 80
    * Ping path: `/auth/isAlive.jsp`
    * Advanced Details can be left unchanged
  * Instance: [{: color=red}choose instance that was created during the first step]
  * ***OPTIONAL:***  Add tags to describe this load balancer
  * Add a record set to AWS [Route 53](http://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-creating.html?console_help=true):
    * Choose a meaningful name
    * Type: CNAME
    * TTL: 300 seconds (default value)
    * Value: [DNS Name of the load balancer]
    * Routing Policy: Simple

### Install OpenAM on AWS Instance
* Update package manager:
  * `sudo apt-get update`
  * `sudo apt-get upgrade -y`
* Install packages to satisfy dependencies:
  * `sudo apt-get install -y unzip mercurial software-properties-common ntp`
* Add repository and install Java 6 JDK using Oracle Java Installer:
  * `sudo add-apt-repository ppa:webupd8team/java -y`
  * `sudo apt-get update`
  * `sudo apt-get install -y oracle-java7-installer`
* Clone the openam12_release repository from Smarter Balanced BitBucket to the server:
  * `hg clone https://[your BitBucket user or team name]/sbacoss/openam12_release`
* Copy SBAC OpenAM installer and content to the /opt directory:
  * `sudo cp -R openam12_release/sbacInstaller/* /opt`
* Update the `/opt/installOpenAM.sh` script to use the correct Java version:
  * change `JAVAVERSION="1.7.0_76"` to `JAVAVERSION="1.7.0_80"`
* Run the OpenAM installer:
  * `cd /opt`
  * `sudo ./installOpenAM.sh`
* Verify only one instance of OpenAM is running:
  * `ps -ef | grep openam`
* If more than one process is returned by the previous command, kill them and restart OpenAM:
  * `sudo kill -9 [all process ids for openam]`
    * Example: `sudo kill -9 31980 30502`
  * Restart OpenAM:
    * `sudo su`
    * `su - openam -c /opt/tomcat/bin/startup.sh`
    * `exit`

#### Update OpenDJ Configuration
* Navigate to *Access Control* -> click on the **sbac** link -> *Authentication* tab -> click on **LDAP** link
* Remove the existing value from the LDAP server:
  * Highlight value in list of LDAP servers and click **Remove**
* Add the correct OpenDJ LDAP server (that was set up previously) in the **New Value** field and click **Add**
* Update the **LDAP Bind Password** to use the correct OpenDJ password for the value specified in **LDAP Bind DN**
  * Starting value of **LDAP Bind DN** is `cn=SBAC Admin`
* Click **Back to Authentication** button
* Click the *Data Stores* tab -> click on **OpenDJ** link
* Remove the existing value from the LDAP server:
  * Highlight value in list of LDAP servers and click **Remove**
* Add the correct OpenDJ LDAP server (that was set up previously) in the **New Value** field and click **Add**
* Update the **LDAP Bind Password** to use the correct OpenDJ password for the value specified in **LDAP Bind DN**
  * Starting value of **LDAP Bind DN** is `cn=SBAC Admin`
* Click **Back to Data Stores**
* Click **Back to Access Control**
* Navigate to *Configuration* -> click on **LDAP** link
* Remove the existing value from the LDAP server:
  * Highlight value in list of LDAP servers and click **Remove**
* Add the correct OpenDJ LDAP server (that was set up previously)
* Update the **LDAP Bind Password** to use the correct OpenDJ password for the value specified in **LDAP Bind DN**
  * Starting value of **LDAP Bind DN** is `cn=SBAC Admin`

### Change OAuth Client Agent Configuration
* Navigate to *Access Control* -> click on **sbac** link -> click on *Agents* tab -> click on *OAuth 2.0/OpenID Connect Client* tab
* Click the link of the agent to edit
* Change the value in the **Client Password** field to the desired password
* Change the value in the **Client Password (confirm)** field to match the value in the **Client Password** field.
* Click **Save**
* For each agent listed:
  * Click the **Inheritance Settings** button
  * Update the **Agent Property Inheritance** settings so that only the following are checked:
    * Default Scope(s)
    * ID Token Signed Response Algorithm
    * Scope(s)
* Click **Save**

### Verify OpenAM Installation

#### Log Into Admin Console
* Navigate to https://[FQDN or IP address of OpenAM server or load balancer]/auth/console?realm=/
  * Example:  https://sso-dev.sbtds.org/auth/console?realm=/
* Log in with valid credentials
  * Default user name is **amadmin**, password is whatever was chosen during installation

#### Verify OpenDJ Connectivity
* Log into OpenAM Admin console
* Navigate to *Access Control* -> click on **sbac** link -> *Subjects*
* A list of user accounts created in the OpenDJ instance (e.g. the prime user account) should be displayed
* **NOTE:** If there are a large number of user accounts in the OpenDJ installation, viewing the subjects could take a long time

#### Verify OAuth Access Token Retrieval
* Run the following `cURL` command:

~~~~
curl -i -X POST \
   -H "Content-Type:application/x-www-form-urlencoded" \
   -d "grant_type=password" \
   -d "username=[Email address of user account that exists in OpenDJ]" \
   -d "password=[Password for user account that exists in OpenDJ]" \
   -d "client_id=[OAuth Client ID from OpenAM]" \
   -d "client_secret=[Client ID secret from OpenAM]" \
 'https://[FQDN or IP address of OpenAM server]/auth/oauth2/access_token?realm=%2Fsbac'
~~~~

* Example:

~~~~
curl -i -X POST \
   -H "Content-Type:application/x-www-form-urlencoded" \
   -d "grant_type=password" \
   -d "username=prime.user@example.com" \
   -d "password=[redacted]" \
   -d "client_id=pm" \
   -d "client_secret=[redacted]" \
 'https://sso-deployment-oauth-test.sbtds.org/auth/oauth2/access_token?realm=%2Fsbac'
~~~~

* Example response:

~~~~
{
    "scope": "cn givenName mail sbacTenancyChain sbacUUID sn",
    "expires_in": 35999,
    "token_type": "Bearer",
    "refresh_token": "ed48e54b-b951-4fe5-bb23-ea8d2d215613",
    "access_token": "82d62be8-136d-4eeb-8f94-68e13b19fc5f"
}
~~~~

### Helpful OpenAM Tips

#### View OAuth 2.0 Profile Information
* Navigate to *Access Control* -> click on **sbac** link -> *Agents* -> *OAuth 2.0/OpenID Connect Client*
* Click on the link to one of the Agents listed (e.g. **pm**)
* Click **Export Configuration** button to view the profile's details (including password)
  * Example:

~~~~
com.forgerock.openam.oauth2provider.clientType=Confidential
com.forgerock.openam.oauth2provider.contacts[0]=
com.forgerock.openam.oauth2provider.description[0]=
com.forgerock.openam.oauth2provider.name[0]=
com.forgerock.openam.oauth2provider.redirectionURIs[0]=
com.forgerock.openam.oauth2provider.responseTypes[0]=code
com.forgerock.openam.oauth2provider.responseTypes[1]=token
com.forgerock.openam.oauth2provider.responseTypes[2]=id_token
com.forgerock.openam.oauth2provider.responseTypes[3]=code token
com.forgerock.openam.oauth2provider.responseTypes[4]=token id_token
com.forgerock.openam.oauth2provider.responseTypes[5]=code id_token
com.forgerock.openam.oauth2provider.responseTypes[6]=code token id_token
userpassword=[redacted]
~~~~

#### Turn on Debugging in OpenAM
* Navigate to *Configuration* -> *Sites and Servers*
* Click on item in **Servers** list
* Under **Debugging**:
  * Change **Debug Level** to **Message** (the most detailed level)
  * Change **Merge debug files** to **On** (all components write to same debug file)
* Debug log file can be found at `/opt/openam/auth/debug/debug.out`

#### Command-Line Utilities for OpenAM
* [ssoadm Command](https://wikis.forgerock.org/confluence/display/openam/ssoadm-agents)
* Path to `ssoadm` is `/opt/openamtools/auth/bin` (assuming the steps in this checklist have been followed)

### Troubleshooting

#### OAuth 2.0 Authentication Failure
* If the `cURL` command to test the OAuth configuration returns a 400 - Bad Request error with an "invalid_client" message in the response, run the following commands (while SSH'd into the AWS instance):
  * `cd /opt/openamtools/auth/bin`
  * `sudo ./ssoadm update-agent -e /sbac -b [name of agent] -a com.forgerock.openam.oauth2provider.clientType=Confidential -u amadmin -f ../../pwd.txt`
    * Example:  `sudo ./ssoadm update-agent -e /sbac -b pm -a com.forgerock.openam.oauth2provider.clientType=Confidential -u amadmin -f ../../pwd.txt`

[back to Deployment Checklists](index.html)