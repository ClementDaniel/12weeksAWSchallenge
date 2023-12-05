# Deploy and scale Django applications on AWS App Runner
This tutorial covers how you can use AWS App Runner to deploy your application to an App Runner service. It shows how to deploy and scale a Django web application on AWS App Runner and how to securely connect to a managed database with Amazon Relational Database Service (Amazon RDS) for PostgreSQL.It walks through configuring the source code and deployment, the service build, and the service runtime.
## Django
Django is a high-level Python web framework that enables developers to build robust, scalable, and secure web applications quickly and efficiently. It follows the model-view-controller (MVC) architectural pattern and emphasizes the principle of "Don't Repeat Yourself" (DRY), promoting clean and reusable code.
Django is an open-source web application framework written in Python. It has many built-in features such as an object-relational mapper (ORM), URL routing, authentication system, templating system, and more, it is a popular choice for developing web applications and APIs.
## AWS App Runner
AWS App Runner makes it easy to deploy Django applications directly from a source code repository or container image. AWS App Runner provisions resources, automatically scales the number of containers up or down to meet the needs of your application, and load balances traffic to ensure high availability.
## Solution Architecture

![](/Week7/images/Project-overview.png)

The solution you are going to set up as part of this walkthrough comprises the following elements as shown in the architecture diagram:
* An AWS App Runner service running your Django application in an AWS-managed VPC.
* An RDS for PostgreSQL database instance running in your own customer-managed VPC. App Runner privately connects to the RDS instance using AWS PrivateLink.
* AWS Secrets Manager to securely store and access the database secret from App Runner.
* A GitHub repository from which the Django application source code will be deployed.

## Walkthrough
### Step 1 - Setting up a sample Django project
As a first step, create a project directory and set up a Python virtual environment using the pipenv.
```
mkdir django-apprunner
cd django-apprunner
pipenv install
```
Next, activate the virtual environment and install Django:
```
pipenv shell
pipenv install django
```
start a new Django project using the django-admin.
```
django-admin startproject myproject
cd myproject
```
Let’s test if our application runs correctly:
```
python manage.py runserver
```
If you visit http://127.0.0.1:8000 in a web browser, then you’ll see the default Django welcome screen:

![](/Week7/images/Django-welcome-screen.png)

### Step 2 - Preparing the application for deployment
Let’s install these two packages in our virtual environment: WSGI server such as gunicorn and WhiteNoise as a simple yet scalable option for serving static files.
```
pipenv install gunicorn==20.1.0 whitenoise==6.4.0
```
Output all installed packages in a requirements file:
```
pip freeze > requirements.txt
```
Before you can deploy to AWS App Runner, you need to make some changes to the Django **settings.py** file located in 
**django-apprunner/myproject/myproject/settings.py**
First, we need to update the ALLOWED_HOSTS to allow AWS App Runner to serve the Django application:

**settings.py**
```
ALLOWED_HOSTS = [".awsapprunner.com"]
```
Next, update and add the STATIC_URL and STATIC_ROOT settings for AWS App Runner and define WhiteNoise as staticfiles storage by adjusting the **STORAGES** setting:
```
STATIC_URL = "static/"
STATIC_ROOT = BASE_DIR / "staticfiles"

STORAGES = {
    "default": {
        "BACKEND": "django.core.files.storage.FileSystemStorage",
    },
    "staticfiles": {
        "BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage",
    },
}
```
Finally, add WhiteNoise to the middleware list:
```
MIDDLEWARE = [
    # ...
    "django.middleware.security.SecurityMiddleware",
    "whitenoise.middleware.WhiteNoiseMiddleware",
    # ...
]
```
### Step 3 - Deploying to AWS App Runner
**Configuring the deployment**
* AWS App Runner enables you to deploy from a GitHub repository or via a Docker image. In this walkthrough, you’ll use the code-based deployment from GitHub.
* In the case of AWS App Runner code-based deployments, you can define deployment configuration in the AWS Management Console or use a configuration file in your source code repository. When choosing the configuration file, any changes to the deployment options are tracked similarly to how changes to the source code are tracked.
* Create an **apprunner.yaml** file in the **django-apprunner/myproject** directory to use the configuration file approach:
  
**apprunner.yaml**
```
version: 1.0
runtime: python3
build:
  commands:
    build:
      - pip install -r requirements.txt
run:
  runtime-version: 3.8.16
  command: sh startup.sh
  network:
    port: 8000
```
To start our application, AWS App Runner must run a number of commands, such as **collectstatic** for serving static files and **gunicorn** for starting the WSGI server. Create a **startup.sh** file in the **django-apprunner/myproject** directory to list these commands:

**startup.sh**
```
#!/bin/bash
python manage.py collectstatic && gunicorn --workers 2 myproject.wsgi
```
Create a .gitignore file in the django-apprunner/myproject directory:

**.gitingnore**
```
*.log
*.pot
*.pyc
__pycache__
db.sqlite3
media
staticfiles
```
Initialize a new Git repository in the **django-apprunner/myproject** directory and push it to GitHub. Follow the below link for instructions on working with GitHub.
https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github

**My GitHub repo:** https://github.com/Sudarkodi-Muthiah-repo/Django_on_aws_apprunner
### step 4 -Creating and deploying the AWS App Runner service
The following diagram outlines the steps for creating an App Runner service:
![](/Week7/images/create-service-workflow.png)
1. Navigate to the **AWS App Runner service** in the AWS Management Console and choose **Create an App Runner service**
   ![](/Week7/images/Apprunner1.png)
2. For Source, choose **Source code repository**
   ![](/Week7/images/Apprunner2.png)
3. Under **Connect to GitHub**, choose **Add new** and follow the instructions to connect to your GitHub account.
   ![](/Week7/images/Apprunner3.png)
   ![](/Week7/images/Apprunner4.png)
   ![](/Week7/images/Apprunner5.png)
4. For **Deployment settings**, choose **Automatic**
   ![](/Week7/images/Apprunner6.png)
5. Choose Next.
6. For **Build settings**, choose Use a configuration file. AWS App Runner automatically picks up the **apprunner.yaml** file stored in our repository.
    ![](/Week7/images/Apprunner7.png)
7. Choose Next.
8. For **Service name**, enter **django-apprunner**
    ![](/Week7/images/Apprunner8.png)
9. Leave the remaining settings as per default for now and choose Next.
10. Review the configuration and choose **Create & deploy**.
    AWS App Runner will now pull your application source code from GitHub to build a container image and deploy it.
    ![](/Week7/images/Apprunner9.png)
    ![](/Week7/images/Apprunner10.png)
    
Once your service reaches the status Running, choose the **default domain** for your service to see the deployed website.

![](/Week7/images/Apprunner11.png)

### Step 5 - Connecting a PostgreSQL database with Amazon RDS
**Setting up an Amazon RDS for PostgreSQL database**
1. Go to the Amazon RDS console, and choose Create database.
2. To choose a database creation method, choose Easy Create. For Engine type, select PostgreSQL.
   
![](/Week7/images/Apprunner_new_db1.png)

3. For DB instance size, select Free tier. 
4. For DB instance identifier, input django-apprunner-db.
5. Leave the initial username as postgres and set an initial password.

![](/Week7/images/Apprunner_new_db2.png)

6. Expand View default settings for Easy create and confirm the settings are the same as on the following image:
   
![](/Week7/images/Apprunner_new_db3.png)

7. Choose Create database.

![](/Week7/images/Apprunner_DB2.png)
 
### Step 6 - Preparing the PostgreSQL database for Django
To connect to your PostgreSQL database from the Django application, you should create a new database user and database just for Django. 
To do so, start at Amazon Elastic Compute Cloud (Amazon EC2) instance with an Amazon Linux 2023 AMI in the default VPC and with the default security group (the same you used for the database instance).

![](/Week7/images/Apprunner_DB3.png)

SSH into the instance, install PostgreSQL, and connect to your Amazon RDS instance:
```
sudo yum install postgresql15 -y
psql -h <Your RDS endpoint> -p 5432 -U postgres -W
```

![](/Week7/images/db_creation.png)

Create a new django database and user. Make sure to replace <Secure password> with an actual password. While still in the psql shell, switch to the newly created Django database and update permissions for the public schema with the following commands. You’ll be asked to re-enter the password of the postgres user:

![](/Week7/images/db_user_creation.png)


Your database is now ready to be used with Django.
### Step 7 - Configuring the Django sample application for PostgreSQL
Install the psycopg package as a Python database adapter for PostgreSQL. Also, install dj-database-url so that you can easily set up the database connection form an environment variable:
```
pip install psycopg2-binary==2.9.6 dj-database-url==1.3.0
```
Update the requirements file to include these new dependencies:
```
pip freeze > requirements.txt
```
Let’s update the **settings.py** to use PostgreSQL as well. First, adapt the imports as follows:
```
from pathlib import Path
import json
import dj_database_url
from os import environ
```
Next, replace the DATABASES setting so that SQLite can be used in development, but PostgreSQL is used if a DATABASE_SECRET is passed as environment variable:
```
if "DATABASE_SECRET" in environ:
    database_secret = environ.get("DATABASE_SECRET")
    db_url = json.loads(database_secret)["DATABASE_URL"]
    DATABASES = {"default": dj_database_url.parse(db_url)}
else:
    DATABASES = {"default": dj_database_url.parse("sqlite:///db.sqlite3")}
```
Finally, instruct AWS App Runner to run database migrations each time your application starts by editing startup.sh:
```
#!/bin/bash
python manage.py migrate && python manage.py collectstatic && gunicorn --workers 2 myproject.wsgi
```
### Step 8 - Securely store the database secret in AWS Secrets Manager
AWS Secrets Manager helps you manage, retrieve, and rotate database credentials, API keys, and other secrets throughout their lifecycles. AWS App Runner allows us to inject secrets from AWS Secrets Manager during application runtime as environment variables. 

1. Open the Secrets Manager console at https://console.aws.amazon.com/secretsmanager/.

2. Choose **Store a new secret**

    ![](/Week7/images/Apprunner_new_SM1.png)

4. On the Choose secret type page, do the following:

    a. For Secret type, choose **Other type of secret**.
    b. For the secret’s key, input **DATABASE_URL**. For the value, define the database URL following the schema supported by dj-database-url as follows:
   ```
   postgres://django:<Secure password>@<RDS endpoint>/django
   ```
    c. Choose Next.
   ![](/Week7/images/Apprunner_new_SM2.png)
   
   ![](/Week7/images/Apprunner_new_SM3.png)
   
6. On the Configure secret page, do the following:

    Enter a descriptive Secret name and Description. Secret names must contain 1-512 Unicode characters.
7. On the Review page, review your secret details, and then choose **Store**.

Secrets Manager returns to the list of secrets. If your new secret doesn't appear, choose the refresh button.

 ![](/Week7/images/Apprunner_new_SM4.png)
 

AWS Secrets Manager relies on AWS IAM to secure access to secrets. Therefore, you need to provide AWS App Runner the necessary permissions to access your newly created secret. AWS App Runner uses an instance role to provide permissions to AWS service actions that your service’s compute instances need. Follow this guide to create a new AWS IAM role in the AWS Management Console.
**To create a role using a custom trust policy (console)**
1. Open the IAM console at https://console.aws.amazon.com/iam/.

2. Choose Roles and then choose Create role.

3. Choose the Custom trust policy role type.
   ![](/Week7/images/Apprunner_new_IAMRole1.png)
   
4. In the Custom trust policy section, enter or paste the custom trust policy for the role.Add a trust policy that declares the AWS App Runner service principal tasks.apprunner.amazonaws.com as a trusted entity to the role:
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "tasks.apprunner.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
 ![](/Week7/images/Apprunner_new_IAMRole2.png)
5. Resolve any security warnings, errors, or general warnings generated during policy validation, and then choose Next.

6. Select the check box next to the custom trust policy you created.
7. Choose Next.
8. Set a permissions boundary. Limit the permissions afforded by the role to only the secret you created in AWS Secrets Manager identified by its Amazon Resource Name (ARN).
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "secretsmanager:GetSecretValue",
            "Resource": "<Secret ARN>"
        }
    ]
}
```


9. For Role Name, Enter role name.
10. For Description, enter a description for the new role.
     ![](/Week7/images/Apprunner_new_IAMRole3.png)
11. Review the role and then choose Create role.

### Step 9 - Updating the AWS App Runner configuration
1. Select your **django-apprunner** service in the AWS App Runner console. Choose the Configuration
Next to Configure service, choose **Edit**.

![](/Week7/images/Apprunner_config2.png)

2. Under **Security**, for Instance role, select the **AppRunnerAccessRole** AWS IAM role you created in the previous step.

![](/Week7/images/Apprunner_config3.png)

3. Under **Networking**, for Outgoing network traffic, create a new VPC connector by choosing **Add new**.

![](/Week7/images/Apprunner_config4.png)

4. Select the **default VPC** and its subnets as well as the default security group. 

![](/Week7/images/Apprunner_config5.png)

6. Choose **Save changes**.
   
Your AWS App Runner service enters the **Operation in progress** state while the new configuration is being applied.

![](/Week7/images/Apprunner_config7.png)

Once the service is back in the Running state, you can update **apprunner.yaml** to reference the secret stored in AWS Secrets Manager via its ARN (i.e., replace the ARN with your own value):
**apprunner.yaml**
```
version: 1.0
runtime: python3
build:
  commands:
    build:
      - pip install -r requirements.txt
run:
  runtime-version: 3.8.16
  command: sh startup.sh
  network:
    port: 8000
  secrets:
    - name: DATABASE_SECRET
      value-from: "arn:aws:secretsmanager:eu-west-1:111122223333:secret:my-django-database-secret-kh2vEL"
```
Finally, push all updates to GitHub. Because you have automated deployments from GitHub enabled, AWS App Runner redeploys your service and your Django application connects to the PostgreSQL database.

### Considerations for scaling Django on AWS App Runner
You have successfully deployed your Django application to an autoscaling compute layer with AWS App Runner and connected to a managed PostgreSQL database with Amazon RDS for PostgreSQL. 
AWS App Runner automatically scales compute resources for your AWS App Runner service up and down based on its autoscaling configuration. This configuration defines the minimum and maximum number of provisioned instances for your service as the min size and max size. AWS App Runner increases and decreases the number of instances within this range based on the maximum number of concurrent requests per instance, the max concurrency. When the number of concurrent requests exceeds this limit, AWS App Runner scales up the service.

You can adjust the per-instance CPU and memory configuration for your AWS App Runner service from 0.25 vCPUs and 0.5 GB of RAM to 4 vCPUs and 12 GB of RAM, respectively. To run your application efficiently on AWS App Runner, adjust both the CPU and memory configuration and the max concurrency to fit your workload’s requirements and use load testing to validate your setup.

AWS App Runner uses AWS Fargate as the underlying compute engine to deploy instances of your application.

### Cleaning up
To avoid incurring charges, delete any resources you created as part of this walkthrough that you no longer need:

1. Delete the AWS App Runner service by selecting the **django-apprunner** service in the AWS App Runner console. Choose Actions and Delete service. Confirm deletion.
2. Delete the Amazon RDS database instance by selecting the **django-apprunner-db** instance in the Amazon RDS console. Choose Actions and Delete. Confirm deletion.
3. Delete the AWS Secrets Manager secret by selecting the **my-django-database-secret** secret in the AWS Secrets Manager console. Choose Actions and Delete         secret. Confirm deletion.
4. Delete the AWS App Runner IAM role by selecting the **AppRunnerAccessRole** role in the IAM console. Choose Actions and Delete role. Confirm deletion.











