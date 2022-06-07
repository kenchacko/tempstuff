# AWS stuff
## EC2
Launch two Ubuntu 20.04 (or prevailing LTS version) instances of types `t3.medium` and `t3.micro`. Ensure same subnet/VPC/etc.

the `t3.micro` instance will be the **Jenkins Instance**.
the `t3.medium` instance will be the **Conjur Instance**

Ensure the security group for both instances allow for :22 [TCP] (SSH) and :8080 [TCP] (Web servers) inbound traffic.

Your default username on these instances is `ubuntu`, remember that.

## IAM
[Create a user](https://us-east-1.console.aws.amazon.com/iam/home#/users$new?step=details) with **Access Key** access, and the `AmazonEC2ReadOnlyAccess` policy attached.
![image](https://user-images.githubusercontent.com/9132059/171922129-c36caa7d-9720-4544-9acb-c3d1774b3fc6.png)

Save the _Access Key ID and Secret Access Key_ somewhere safe, we will need it later!


# Conjur OSS
Connect to the `t3.medium` **Conjur Instance**

Run the following:
* Create docker user group: `sudo groupadd docker`
* Add current user (`ubuntu`) to group: `sudo usermod -aG docker ubuntu`
* Install conjur:`curl -fsSL https://cybr.rocks/conjur-install | bash -s`
  * (the script is from [here](https://github.com/infamousjoeg/conjur-install))

## connect to client
`sudo docker exec -it ubuntu_client_1 bash`

## admin account
Account name: `admin`

Run `conjur user rotate_api_key` to generate API key, save it somewhere.

## tutorial
based on [App and Secrets Enrollment Tutorial: Organizing Policy Management](https://www.conjur.org/get-started/tutorials/enrolling-application/)

**everything here assumes you have executed** `sudo docker exec -it ubuntu_client_1 bash` **and are in the client shell**

### create conjur.yml
* Create directory `policies` in `/root/` (run `mkdir /root/policies`)
* open a text editor for `conjur.yml` in that folder (run `nano /root/policies/conjur.yml`
 
```yml
- !policy
  id: db

- !policy
  id: frontend
```

Load it
`conjur policy load --replace root /root/policies/conjur.yml`

### db.yml

Create file `db.yml`  in the same directory, contents:
```yml
# Declare the secrets which are used to access the database
- &variables
  - !variable password

# Define a group which will be able to fetch the secrets
- !group secrets-users

- !permit
  resource: *variables
  # "read" privilege allows the client to read metadata.
  # "execute" privilege allows the client to read the secret data.
  # These are normally granted together, but they are distinct
  #   just like read and execute bits on a filesystem.
  privileges: [ read, execute ]
  roles: !group secrets-users
```

Load it
`conjur policy load db /root/policies/db.yml`

### set a value to variable: password
Value we are going to use: **THE IAM SECRET ACCESS KEY FROM EARLIER**

Setting it:
`conjur variable values add db/password [AWS IAM Secret Access Key]`

Viewing it after:
`conjur variable value db/password`

### frontend.yml (the application)

Same directory as db.yml and conjur.yml. contents:
```yml
- !layer

- !host frontend-01

- !grant
  role: !layer
  member: !host frontend-01
```

Load it:
`conjur policy load frontend /root/policies/frontend.yml`

It should create a role. Output like so: (api key will differ)
```json
{
  "created_roles": {
    "quick-start:host:frontend/frontend-01": {
      "id": "quick-start:host:frontend/frontend-01",
      "api_key": "1kw55sg2tab1ya201pqqm1bvrrrq3tvv7n13dyfad020jha0p1b5qb5y"
    }
  },
  "version": 1
}
```

Run the following to test, replace api key with your own from above command. This allows you to execute the command `conjur variable value db/password` as the `frontend-01` role:
```sh
CONJUR_AUTHN_LOGIN=host/frontend/frontend-01 \
CONJUR_AUTHN_API_KEY=1kw55sg2tab1ya201pqqm1bvrrrq3tvv7n13dyfad020jha0p1b5qb5y \
conjur variable value db/password
```

expect:
`error: 404 Not Found`

You shouldn’t be able to see any objects (empty list) either with:
```
CONJUR_AUTHN_LOGIN=host/frontend/frontend-01 \
CONJUR_AUTHN_API_KEY=1kw55sg2tab1ya201pqqm1bvrrrq3tvv7n13dyfad020jha0p1b5qb5y \
conjur list
```

### Update db.yml policy with new entitlements
Append this to the bottom
```yml
# Entitlements
- !grant
  role: !group secrets-users
  member: !layer /frontend
```

Load it:
`conjur policy load db /root/policies/db.yml`

Verify the group `secret-users` now has access to `/frontend`
run: `conjur role members group:db/secrets-users`
Expect: `"quick-start:layer:frontend”` to be part of the list

To verify what roles can access `db/password`, do this:
run: `conjur resource permitted_roles variable:db/password execute`

Expect `quick-start:host:frontend/frontend-01` (or something similar i cant remember) to be one of the roles listed.

And finally check to see if you can now view the password from the `frontend-01` role:
```sh
CONJUR_AUTHN_LOGIN=host/frontend/frontend-01 \
CONJUR_AUTHN_API_KEY=1kw55sg2tab1ya201pqqm1bvrrrq3tvv7n13dyfad020jha0p1b5qb5y \
conjur variable value db/password
```

expect: `IVBgTDFCeEhxfjhELS50OH1dNjNCZnRyIkliQmUibXN7Y1hCKV9gXHItMDggWg==`

# Jenkins
Connect to the `t3.micro` **Jenkins Instance**

## Installing Docker
First, install some pre-requisites:
* `sudo apt update`
* `sudo apt install apt-transport-https ca-certificates curl software-properties-common openjdk-8-jre`

Add Docker's GPG key:
`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`

Add Docker's repository to APT:
`sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"`

Then, update APT again:
`sudo apt update`

Set docker to install from docker repo:
`apt-cache policy docker-ce`

Install Docker and Docker-compose:
`sudo apt install docker-ce docker-compose`

Add current user (`ubuntu`) to Docker group:
`sudo usermod -aG docker ubuntu`

Restart the instance:
`sudo reboot`

## Running Jenkins on Docker
Create a folder at `~/` called `jenkins`
`mkdir ~/jenkins`

Enter the folder
`cd ~/jenkins`

Create a Dockerfile:
`nano Dockerfile`

Paste the following in the Dockerfile:
```
FROM jenkins/jenkins:lts

USER root
RUN apt update -y && \
    apt install -y python3-pip

RUN pip3 install awscli --upgrade
```

Save.

Create a file `docker-compose.yml`
`nano docker-compose.yml`

Paste the followingL
```
version: '3.8'
services:
  jenkins:
    build: .
    privileged: true
    user: root
    ports:
     - 8080:8080
     - 50000:50000
    container_name: jenkins
    volumes:
     - ./jenkins_config:/var/jenkins_home
     - /var/run/docker.sock:/var/run/docker.sock
```

Create a folder `jenkins_config`:
`mkdir jenkins_config`

Run the server
`docker-compose up -d`

Then, follow only **Step 4** from here:
https://www.digitalocean.com/community/tutorials/how-to-install-jenkins-on-ubuntu-20-04

Note: to get your initialAdminPassword, do one of the following commands:
* `sudo cat jenkins_config/secrets/initialAdminPassword`
* `docker-compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword`

## Adding Conjur integration
### Installing plugin
Install Jenkins `conjur-credentials` plugin: https://plugins.jenkins.io/conjur-credentials/ _(click on top right "How to install", follow GUI steps)_

### adding credentials to Jenkins
Go to http://jenkinsIP:8080/credentials/store/system/domain/_/newCredentials
(Replace jenkinsIP with your Jenkins host)

Set
* **Kind**: Username with password
* **Username**: `host/frontend/frontend-01`
* **Password**: `1kw55sg2tab1ya201pqqm1bvrrrq3tvv7n13dyfad020jha0p1b5qb5y` (API key for frontend-01 role, update as necessary)
* **ID**: `conjur-login`

### adding variable to jenkins
Go to http://jenkinsIP:8080/credentials/store/system/domain/_/newCredentials
(Replace jenkinsIP with your Jenkins host)

Set 
* **Kind**: Conjur Secret Credential
* **Variable Path**: `db/password`
* **ID**: `DB_PASSWORD`

### creating a pipeline
Go to http://jenkinsIP:8080/view/all/newJob
(Replace jenkinsIP with your Jenkins host)

Select **Pipeline**, press **OK**

Go down to _Conjur Appliance_ configuration:
* **Inherit from parent?**: _Disabled_
* **Account**: `quick-start`
* **Appliance URL**: http://**[Conjur Host IP]**:8080
* **Conjur Auth Credential**: Select the credential `host/frontend/frontend-01` in the drop down.
* **Conjur SSL Certificate**: _-none-_

Go down to _Pipeline_ configuration:
* **Definition**: _Pipeline script_
* **Script**:
```java
node {
  stage('Work') {
    withCredentials([conjurSecretCredential(credentialsId: 'DB_PASSWORD', variable: 'AWS_SECRET_ACCESS_KEY')]) {
      echo 'describe EC2'
      def output = sh(
        script: "AWS_ACCESS_KEY_ID=[REPLACE WITH AWS ACCESS KEY ID] aws ec2 describe-regions --region ap-southeast-1 --output json",
        label: "Describe all EC2 regions",
        returnStdout: true
      )
      print(output)

    }
  }
  stage('Results') {
    echo 'Finished!'
  }
}
```
**Replace** `[REPLACE WITH AWS ACCESS KEY ID]` **with AWS Access Key ID from earlier**

End
---
You should be done. Run the build, see what happens.
