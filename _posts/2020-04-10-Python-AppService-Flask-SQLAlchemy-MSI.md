---
title: "Linux AppService Python (Flask-SQLAlchemy) using MSI"
author_name: "Ben Ohana"
tags:
    - Python
    - Flask
    - pyodbc
categories:
    - App Service # Azure App Service on Linux, Azure App Service on Windows, Function App, Azure VM, Azure SDK
    - Python # Python, Java, PHP, Nodejs, Ruby, .NET Core
    - Flask
    - MS-SQL
    - How-To
header:
    teaser: "/assets/images/flask-logo.png" # There are multiple logos that can be used in "/assets/images" if you choose to add one.
# If your Blog is long, you may want to consider adding a Table of Contents by adding the following two settings.
toc: true
toc_sticky: true
date: 2020-04-10 09:11:00
---

## Python Flask connects to SQL using sqlalchemy

In this guide I'll describe how to leverage Flask-SQLAlchemy framwork that relies on ODBC Driver 17 for database connection **all** using [Manage Identity](https://docs.microsoft.com/en-us/azure/app-service/app-service-web-tutorial-connect-msi).

## First Step - Configuration of Azure resources

   1. Set up a system assigned identity
   ![assigned identity in the portal:](https://ghost-azure9135.azurewebsites.net/content/images/2020/03/image-1.png)
   2. Firewall for thse SQL server should allow other Azure resources to access the database.
   ![Azure Firewall:](https://ghost-azure9135.azurewebsites.net/content/images/2020/03/image-2.png)
   3. Set Active Directory Admin for SQL server:
   ![Azure AD setup:](https://ghost-azure9135.azurewebsites.net/content/images/2020/03/image-3.png)
   4. give our App Service (I named it "az-dash-lnx") access rights to the table.

    CREATE USER [az-dash-lnx] FROM EXTERNAL PROVIDER;
    ALTER ROLE db_datareader ADD MEMBER [az-dash-lnx];
    ALTER ROLE db_datawriter ADD MEMBER [az-dash-lnx];
    ALTER ROLE db_ddladmin ADD MEMBER [az-dash-lnx];
    GO

After our App Service and SQL database have been set up, we can now access the database in Python.

## Second Step - Deploy Python App Service

1) Azure App Service offers two option to run Python runtime with pyodbc  (actually 3 but Windows is deprecated)
[Built in images](https://github.com/Azure-App-Service/python) or [Custom container](https://docs.microsoft.com/en-us/azure/app-service/containers/configure-custom-container), here is a DockerFile for the ladder.

```python
# mssql-python3.6-pyodbc
FROM ubuntu:16.04
# apt-get and system utilities
RUN apt-get update && apt-get install -y \
    curl apt-utils apt-transport-https debconf-utils gcc build-essential g++-5\
    && rm -rf /var/lib/apt/lists/*
# adding custom MS repository
RUN curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add -
RUN curl https://packages.microsoft.com/config/ubuntu/18.04/prod.list > /etc/apt/sources.list.d/mssql-release.list
# install libssl - required for sqlcmd to work on Ubuntu 18.04
RUN apt-get update && apt-get install -y libssl1.0.0 libssl-dev
# install SQL Server drivers
RUN apt-get update && ACCEPT_EULA=Y apt-get install -y msodbcsql17 unixodbc-dev
# install SQL Server tools
RUN apt-get update && ACCEPT_EULA=Y apt-get install -y mssql-tools
RUN echo 'export PATH="$PATH:/opt/mssql-tools/bin"' >> ~/.bashrc
RUN /bin/bash -c "source ~/.bashrc"
# python libraries
RUN apt-get update && apt-get install -y \
    python3-pip python3-dev python3-setuptools \
    --no-install-recommends \
    && rm -rf /var/lib/apt/lists/*
# install necessary locales
RUN apt-get update && apt-get install -y locales \
    && echo "en_US.UTF-8 UTF-8" > /etc/locale.gen \
    && locale-gen
RUN pip3 install --upgrade pip
# install SQL Server Python SQL Server connector module - pyodbc
RUN pip3 install pyodbc
RUN pip3 install setuptools
RUN pip3 install imutils
RUN pip3 install flask_restful
RUN pip3 install flask_sqlalchemy
RUN pip3 install numpy
RUN pip3 install flask_httpauth
RUN pip3 install Flask-JWT
RUN pip3 install flask_babel
RUN pip3 install werkzeug
RUN pip3 install requests
RUN pip3 install flask_cors
RUN pip3 install urllib3
RUN pip3 install bs4
RUN pip3 install flask_jwt_extended
RUN pip3 install python-csv
RUN pip3 install uuid
RUN pip3 install scipy
RUN pip3 install flask_httpauth
RUN pip3 install colorgram.py
RUN pip3 install opencv-python
RUN pip3 install pillow
RUN pip3 install colorthief
# install additional utilities
RUN apt-get update && apt-get install gettext nano vim -y
# add sample code
RUN mkdir /sample
ADD . /sample
WORKDIR /sampler
ENTRYPOINT ["python3"]
CMD ["run.py"]
```

2) libraries leveraged.

```python
#libs
   import os
   import requests
   import struct
   import pyodbc
   import sqlalchemy
   import urllib
   def get_bearer_token(resource_uri, token_api_version):
   token_auth_uri = f"{msi_endpoint}?resource={resource_uri}&api-version={token_api_version}"
   head_msi = {'Secret':msi_secret}
   resp = requests.get(token_auth_uri, headers=head_msi)
   access_token = resp.json()['access_token']
   return access_token
```

This function will be used later to get the access token

3) Environment variables "MSI_ENDPOINT" and "MSI_SECRET" are created by the App Service when you turn on managed identities. The server address and the database must of course also be specified:

```python
    msi_endpoint = os.environ["MSI_ENDPOINT"]
    msi_secret = os.environ["MSI_SECRET"]
    connstr="Driver={ODBC Driver 17 for SQL Server};Server=tcp:az-sqlserver-az.database.windows.net,1433;Database=az-titanicdb-jma";
```

4) Retrieve  access token and convert it to struct:

```python
token = bytes(get_bearer_token("https://database.windows.net/", "2017-09-01"), "UTF-8")
exptoken = b"";
for i in token:
exptoken += bytes({i});
exptoken += bytes(1);
tokenstruct = struct.pack("=i", len(exptoken)) + exptoken;
```

## Examples

Database query in ODBC, the number 1256 corresponds to the attribute "SQL_COPT_SS_ACCESS_TOKEN":

```python
conn = pyodbc.connect(connstr, attrs_before = { 1256:tokenstruct });
cursor = conn.cursor()
cursor.execute("SELECT TOP (1000) [id],[Survived],[Pclass],[Name],[Sex],[Age],[sibling_or_spouse],[parents_or_children],[Fare] FROM [dbo].[titanic_passanger]")
row = cursor.fetchone()
while row:
print (str(row[2]) + " " + str(row[3]))
row = cursor.fetchone()
```

Query in SQLAlchemy

```python
params = urllib.parse.quote(connstr)
engine = sqlalchemy.create_engine('mssql+pyodbc:///?odbc_connect={}'.format(params) ,connect_args={'attrs_before': { 1256:tokenstruct}})
conn = engine.connect()
result = conn.execute("SELECT TOP (10) [id],[Survived],[Pclass],[Name],[Sex],[Age],[sibling_or_spouse],[parents_or_children],[Fare] FROM [dbo].[titanic_passanger]")
for row in result:
print (str(row[2]) + " " + str(row[3]))
conn.close()
```

For Flask-SQLAlchemy, a configuration file config.py can contain these  settings

```python
class BaseConfig:
    SQLALCHEMY_DATABASE_URI = 'mssql+pyodbc:///?odbc_connect={}'.format(params)
    SQLALCHEMY_ENGINE_OPTIONS = {'connect_args': {'attrs_before': {1256:tokenstruct}}}
    SQLALCHEMY_TRACK_MODIFICATIONS = False
    DEBUG = False
    TESTING = False
```

As you can see code is avoiding credential leak and address of our SQL server is still contained in the code, however this information can also be saved outside the program code.
This code should also work in a similar form in order to grant an Azure function database access by means of managed identities [see link](https://azure.microsoft.com/en-us/blog/simplifying-security-for-serverless-and-web-apps-with-azure-functions-and-app-service/).

5) To test if ODBC is included in the underlying image, [SSH in to container](https://docs.microsoft.com/en-us/azure/app-service/containers/configure-custom-container#enable-ssh) and run:

```bash
sudo su
wget https://gallery.technet.microsoft.com/ODBC-Driver-13-for-Ubuntu-b87369f0/file/154097/2/installodbc.sh
sh installodbc.sh
```


Additional Samples can be found [Here](https://www.az.run/app-service-linux-python-to-sql/)


| Table | Test | Description |
|----|----|----|
|Some|Test|Data|
|Row|Number|2|