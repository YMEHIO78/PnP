# Handful of Takaaful
Exploratory Research Informatics on the Islamic Insurance Market


This project consisted of doing qualitative research to observe the need for a financial analyst-friendly, Takaful-focused market research dashboard. To accomplish this, an automatically updating data infrastructure was needed. Below, is a simple breakdown:

Part I: Conception and Extraction+Transformation.


Our data was sourced from 2 sources:
World Bank API (Yearly Macroeconomic reports on financial and other, broadly exploratory metrics which could provide key insight for developing various Takaful offerings for various industries/niches.)
Business disclosure
Ease of Doing Business (score)
Foreign Direct Investment (As % of GDP)
GDP in USD 
Healthcare Expenditure (as % of GDP)
% of Internet users by country
Mobile cellular subscriptions (per 100 people)
Secure Internet servers (per 1 million people)

Islamic Financial Services Board (Quarterly Macoreconomic reports on key financial + Takaful metrics) gave us zoomed-in data on a few countries:
Bahrain
Brunei
Jordan
Malaysia
Nigeria
Saudi Arabia
Turkey
United Arab Emirates


Data from the Islamic Financial Services Board is to be manually downloaded from the website upon release (for reasons later listed), saved in an S3 bucket, and attempted to be loaded directly into PowerBI; this required using a custom connection method to PowerBI via an R script. This script used the aws.s3 package (as well as the aws.signature package, in case it would be needed). 
But firstly, data from the World Bank API was sourced via a pull request made to the API (in the form of a Python script) and would be saved in an Amazon S3 bucket. To automate this process, a Lambda function (PnPLambdaFunc1) was created to run a python script when triggered by a Cloudwatch rule which is set to make this API request every 90 days (one business quarter). However, due to the exceeding size of the wbgapi PiP for the memory limits set by Lambda (and no known workaround), the code was placed in a locally run Jupyter notebook; this was a feasible workaround as it held intact the intuitiveness for financial analysts, as they simply just hit “Run” to pump the data. 
Next, the data needs to be ingested from the S3 bucket into the respective tables in Redshift; where each CSV has a respective table within the Redshift Database. Setting up the Redshift infrastructure to receive this data consisted of firstly creating the compute resource (called a cluster) needed to host the database and then creating empty/receiving tables (by creating a schema for the tables). 
Furthermore, an Athena database was required to serve as an “intermediary/hosting DB” between the S3 and Redshift databases. Creating this Athena DB took not just setting up the DB instance but also creating the tables. This was done by creating a Glue Crawler which figured out the schemas for the tables and then created the tables in the Athena DB in the mold of said schema; and the final step in creating the table-to-table connection from our Athena Database to the Redshift database was by creating an AWS object called a “Connection” (yes, that is the actual name for it) between the S3 bucket and the corresponding tables in Redshift. When this was created and the data was piped through, all was functioning perfectly well until the endpoints for S3 and Redshift needed to connect, at which point I got the following error message: “VPC S3 endpoint validation; NAT gateway config”. 
As an alternative/workaround to this pipeline, a Lambda function was created to automate the execution of a SQL cookiecutter script which copies the data from S3 to the respective tables in Redshift. This required a similar workaround as in the first Lambda function: creating a (local) system directory and downloading the “psycopg2” and “requests” PiPs via the terminal and then zipping up and importing that directory into Lambda for the function’s script to run properly. However, upon running the necessary Unix code to download, I received the following error (see: X2: a pg_config executable error which indicates need for a system which has a PostgreSQL config). 
To maintain the intuitiveness of the system for financial analysts, a workaround Python script (which for access, required creating 2 new policies and a user with new access keys) was created in the local system as a Jupyter notebook. It simply needs nothing else from the end user but to be executed- the script uses the COPY commands to pump the data along the pipeline (from S3 into our Redshift tables). 

Lastly, pipelining the data from Redshift into PowerBI. This was done via the “Connection” interface in PowerBI. When it was fed the database connection details, I got the following error: (ODBC: ERROR [08S01] [Microsoft][Amazon Redshift] (10) Error occurred while trying to connect: [SQLState 08S01] timeout expired). This was strange as, when I went to the monitoring window for the DB/cluster, I saw that there was an outside connection made at the exact time that my PowerBI made the request.

Phase II: Dashboard transformations

After all dashboard connections were made, there were two strategies for the two data sources, respectively:





IFSB data: 
Changing the data types
Transposing the data (x,y) -> (y,x)
Filtering rows to key indicators:
Invest income/net earned contribution (General)
Invest income/net earned contribution (Family)
Investment Income (From Family Fund Only)
Investment Assets (of Family Fund Only)
Operating revenues/underwriting profit (General)
Operating revenues/underwriting profit (Family)
Net profit (after taxation/zakat) (General)
Net profit (after taxation/zakat) (Family)
Gross Domestic Product
Gross claims paid (General)
Surplus/deficit in the Participants’ Risk fund (General)
Surplus/deficit in the Participants’ Risk fund (Family)
Set 1st row as headers


WorldBank API data:
Changed data types
Set 1st rows as headers
Renamed all columns (across sheets) for 2020 and 2021 to “2020/2021[abbrev.forkeymetric]”



From this data, sheets were made for each country “in-common” between the data sources (Bahrain, Brunei, Jordan, Malaysia, Nigeria, Saudi Arabia, Turkey, United Arab Emirates) and a final sheet was created consisting of a map with each country of the world, colored in its borders, and a list of the countries tied to a table of key metrics; this way, when a country would be clicked on the map, the table would display that country’s (and only that country’s) key metrics, and vice-versa; when a country was clicked on the table, the map will automatically highlight the country and zoom in on it.


Here is a concluding, brief description of each file and its purpose:
RScript
R Script for attempting connection directly from S3 to PowerBI


PowerQuery script
PowerQuery scripts (one example/template for each datasource (IFSB & WBAPI sourced sheets))


LambdaFunction 1 script:
Python script for the 1st Lambda function (WBAPI -> S3) which, upon error+workaround, was adapted to a Jupyter Notebook

wbgapi terminal convo
Conversation with the terminal for downloading wbgapi package to the system

LambdaFunction2 script
Python script for the 2nd Lambda function which, upon its error+workaround, had to be adapted to a Jupyter NB.

SQL Copy Statement Cookie Cutter
Cookiecutter template for copying data from S3 into Redshift table.

psycopg2,requests policy JSON
the JSON policy which enables the psycopg2,requests PiPs to funciton in their python script.

psycopg2requeststerminalscript
Terminal code for downloading psycopg2request package to the system.

worldbankAPI policy JSON
the JSON code for the worldbankAPI policy






Notable Mentions: Users, Roles, Permissions, Policies.


The relations between Users, Roles, Permissions and Policies is best and clearly outlined pictographically as seen on the poster presentation. While many permissions were already created and I simply selected them for use when needed, 2 custom JSON permission were created from scratch to allow for the installed packages to function in their respective python script in order for the scripts to execute and act upon the objects in mention. 
