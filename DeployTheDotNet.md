# Build and Deploy your .Net Core App to Amazon Linux 2 EC2 in in AWS

## Goal - Facilitate Learning

This is a work in progress. I have a specific goal - this is not meant to be the ideal or enterprise-ready solution for your .Net Core App. This is meant as a quick guide for a new developer to get their .Net Core App into AWS with a minimal stack, just the core pieces.

## Automate the App Build

The main reason I recommend this, is to build an artifact that can easily be downloaded from the AWS server. This minimizes access requirements to the server.

Example app https://aka.ms/dotnet-hello-world

### Github Actions

This simple GHA workflow will run a dotnet build and publish on a simple .Net Core app. The publish command here assumes the dotnet 6.x SDK/Runtime is installed on the Linux server, which is easy to install. If the build was triggered by a _release_ tag, the published files will be zipped up and attached to the release.  Assets attached to a release are **publicly retrievable via curl/wget from any computer that can reach github**.

Note: You do have to set the repository or organization secret _GH_TOKEN_ to a Personal Access token with Content and Workflow permissions to use the release action-gh-release action.

Feel free to use this GitHub Action workflow as a starting point.

Create this file in your application Git repo: .github/workflows/build.yml:
```
name: Build .NET Core App

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
  release:
    types: [published]

jobs:
  build:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: 6.0.x
    - name: Restore dependencies
      run: dotnet restore
    - name: Build
      run: dotnet build --no-restore
    - name: Test
      run: dotnet test --no-build --verbosity normal
    - name: Publish
      run: dotnet publish -r linux-x64 -o site;zip -r Application.zip site
    - name: Release
      env:
       GITHUB_TOKEN: ${{ secrets.GH_TOKEN }}
      uses: softprops/action-gh-release@v1
      if: startsWith(github.ref, 'refs/tags/')
      with:
       files: Application.zip
```

## Setup an EC2 Instance in AWS

The operating system used in this scenario is Amazon Linux 2. Start an EC2 instance that meets the following criteria:

### Settings
- Operating System or AMI: Amazon Linux 2

- Network should only use a Private network VPC (does not need a public IP)

- Security groups only need access Inbound to port 80.

Note: Use Session Manager to login - this does not require SSH access.

### Install software on the EC2

Login to the EC2 instance you created with Session Manager. 

For a dotnet application with target runtime 6.

```
# Enable access to microsoft RPMS
sudo rpm -Uvh https://packages.microsoft.com/config/centos/7/packages-microsoft-prod.rpm

# Install a dotnet SDK, Runtime and the Nginx web server
sudo yum install aspnetcore-runtime-6.0  dotnet-sdk-6.0 amazon-linux-extras -y
sudo amazon-linux-extras install nginx1

# start Nginx
sudo systemctl start nginx.service


```

## Setup AWS Application Load Balancer

Create a new ALB

- Put it in the same VPC as the EC2 instance

- Map it to external VPCs (DMZ) so the ALB will have an external IP

- create a new Target Group (TG)

- Add listener(s) that listens on HTTP and/or HTTPS and forwards to the new TG

- Add security group(s) needed to allow inbound HTTP or HTTPS access and outbound HTTP to the EC2 instance or VPC.

- Register the EC2 instance to the Target Group


### Configure Nginx

Setup Nginx to redirect all traffic to the dotnet application.  Login to the EC2 instance with Session Manager and create the following file.  You could ignore this and use port 5000 as the port between the EC2 instance and the ALB. However, it is usually best to put a web server in front of the dotnet server.

Replace **ALB_HOSTNAME** with the ALB hostname, which you can get from AWS once the ALB is created.  You could also replace the `server {}` section in the /etc/nginx.conf for work for all connections.

Note: the ALB_HOSTNAME will need to match the public DNS name we use on the ALB.

sudo vi /etc/nginx/conf.d/dotnet.conf:
```
server {
    listen       80;
    server_name ALB_HOSTNAME;

    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_cache_bypass $http_upgrade;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_ignore_client_abort off;
        proxy_intercept_errors on;

        client_max_body_size 50m;
    }
}
```

## Setup the Database (optional)

Create the database you need. Save the connection info to use in your application settings.

For security groups, go with automatic, set it up to only allow your EC2 instance inbound. It should even show your EC2 instance in a drop-down in that section of the settings. This is why the EC2 instance is setup first.

## Run the .net app

Download your application file to the EC2 instance via wget or curl. If you zipped up your published files, in the GitHub Action above, and uploaded it to the Git Repo releases section, you would use something like this:

Note: you will need the database URL and connection info above, if you will use that. An example postgres connection string would be:

### Custom settings

`Server=127.0.0.1;Port=5432;Database=moviebate;User Id=myUsername;Password=myPassword;`

Example of settings (values populated via IConfiguration in your app) to add in the appsettings.json may be an api key, secret and DB connection string:

```
  "APIKey": "010101010",
  "APISecret": "*&@$^@!&#^",
  "ConnectionStrings": {
    "MovieBateDB": "Server=127.0.0.1;Port=5432;Database=your_app_db;"
  }
  ```

### Setup the app
```
# Doesn't really matter where you download or run this from. I will use /app/site
sudo mkdir /app
sudo chmod 775 /app
cd /app
curl -sO https://github.com/your_name/your_app/releases/download/v1.0/your_app.zip

# In our example, the app zip file unzips into a /site directory
unzip your_app.zip

# update any configuration files or set environment variables
# nano, ex and vi editors are often available to use
vi /app/site/appsettings.json # add any settings (hint: if you used user-secrets - that stuff)


# If the dotnet app needs database setup, install dotnet-ef
dotnet tool install -g dotnet-ef --version 6.0.25
# add ~/.dotnet/tools/ to your path temporarily
. /etc/profile.d/dotnet-cli-tools-bin-path.sh
dotnet ef database update

nohup dotnet site/your_app.dll &
```

The exact files, names and directories are arbitrary. The end goal is to use `dotnet <file>.dll` to run the app. It will automatically start and listen on port 5000.

#### References

https://cloudkatha.com/how-to-install-nginx-on-amazon-linux-2-instance/

https://docs.servicestack.net/deploy-netcore-to-amazon-linux-2-ami#install.net-6.0
