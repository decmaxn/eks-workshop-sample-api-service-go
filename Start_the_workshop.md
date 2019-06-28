# Launch Cloud9 in your closest region:
1. Oregon
1. Ireland
1. Ohio  - in my case
1. Singapore

# Create an IAM role for your Workspace
role:eksworkshop-admin with AdministratorAccess policy and assign it to Cloud9 intance:	

in my case:e:   ecd-dev-bastion-role

# Install kubectl
```
sudo curl --silent --location -o /usr/local/bin/kubectl https://storage.googleapis.com/kubernetes-release/release/v1.13.7/bin/linux/amd64/kubectl
sudo chmod +x /usr/local/bin/kubectl
sudo yum -y install jq gettext
for command in kubectl jq envsubst
  do
    which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
  done
```

# Clone the Service Repos
```
cd ~/environment
git clone https://github.com/brentley/ecsdemo-frontend.git
git clone https://github.com/brentley/ecsdemo-nodejs.git
git clone https://github.com/brentley/ecsdemo-crystal.git
```
# Update IAM settings for your Workspace
in my case, since the cloud9 instance is a bastion host connected with ssh, not created by cloud9, so I don't have this settings.
```
Return to your workspace and click the sprocket, or launch a new tab to open the Preferences tab
Select AWS SETTINGS
Turn off AWS managed temporary credentials
Close the Preferences tab 
```

# Create an SSH key
```
ssh-keygen
aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material file://~/.ssh/id_rsa.pub
```