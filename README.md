# Certificate Expiry Notification
This repo helps in developing a CICD process to keep track of certificates expiring in Mulesoft application

This article is targeted for customers, who have purchased the certificate from CA and are interested in having the custom capability, to alert the certificate expiration date to the responsible team for renewing and managing the certificate accordingly. The outcome of this document will be more helpful for the L2/L3 support and maintenance teams to follow up the certificate renewal with the internal or external teams.

Set up a Jenkins pipeline that will perform basic things like build, unit test, deploy, and **most importantly check if there is a certificate available in the application source, and then find the certificate expiry date using a password available in the Jenkins file**.

It is assumed that your CICD pipeline is able to deploy the application to Cloudhub runtime manager. The application in this repository has a sample certificate that is not used, but we will be using this certificate’s detail to send alerts. 

**Our current CICD pipeline is pretty simple having the usual build, test and deploy stages. Now, let’s add a new stage to do the following:**

1. Check certificate availability in source code
2. If certificates are found, then get the expiry details using Keytool command
3. Send the Keytool command output to Mulesoft application using CURL command

**Our Mulesoft application will receive the Keytool output from Jenkins CICD using HTTP call to perform the below:**

1. Parse the Keytool text output to get the required details in JSON format
2. Store the details in *persistent* Objectstore v2 using a unique key that fits your use case (use of non-persistent data can cause data loss)
3. Pull data weekly from *persistent* Objectstore v2 to find the certificates expiring in the next 30 days
4. Send alert using Cloudhub connector
5. On Anypoint platform, create an alert to trigger an email using Cloudhub custom alerts; such as the one below

**As previously mentioned, there are only 2 main components introduced to achieve the above notification: 
**
1. New stage to the CICD pipeline
2. Mule application to store certificate expiry details and send alerts

**New Stage in CICD**

This stage consists of scripts in ***Jenkinsfile*** available in this repository. Please go through the new stage named "**Certificate Check**" to understand it in more detail (current line number 30-53). Comments are added to each line to explain its functionality and any changes required in the future.

**Consider the below changes in the new CICD stage:**

1. Change URL in line number 36 to point it to your Mulesoft alerts-api
2. This script assumes that only 1 certificate is available, so you will notice *files[0].path. * In the case of multiple files, you will need to loop through all of them one by one. Consider putting lines #43 to 53 in that loop
3. Headers like application name, environment name etc. are hardcoded in the curl command; these headers need to be picked dynamically from Jenkinsfile
4. Add security header once you apply policy to Mulesoft API
5. Use the Keystore password from Jenkins credentials in line number 46, currently hardcoded to make it easier to run the stage


**Mule application**

This application runs on Mule 4.3 runtime, and it is the same application available in this repository . Kindly refer to src/main/mule folder to understand these 2 flows which *extensively use Dataweave, Objectstore and Cloudhub Connector*. Dataweave plays an important role in converting the Keytool text output to JSON. Please go through objectstore FAQ (https://docs.mulesoft.com/object-store/osv2-faq) and cloudhub connector rate limit (https://docs.mulesoft.com/runtime-manager/alerts-on-runtime-manager#:~:text=When%20using%20CloudHub%20Connector%20to,workers%20assigned%20to%20the%20application.) before using this solution.

Please ensure you use RAML and API policies to secure this application. It might be preferable to use a relational database instead of Objectstore and Email instead of Cloudhub connector. The use of Objectstore and Cloudhub connectors are dependent on your existing toolsets.

**Consider the below changes in Mulesoft application:**

1. Create RAML for the HTTP endpoint, Publish it to Exchange, Import it in API Manager and apply security policies
2. Add autodiscovery to the code to use the policies applied in API manager
3. Secure the endpoints with HTTPS
4. Replace Objectstore to database if available for Central IT team
5. Replace Cloudhub connector with Splunk HTTP or Email when required
6. Disable HTTP wire logging in log4j.xml line number 22
7. Create a custom alert in your runtime environment following this document (https://docs.mulesoft.com/runtime-manager/custom-application-alerts)
8. Update the property file with your Cloudhub account details to trigger alerts

**Please note, that the above guide is just an example to accomplish certificate expiry alerts, and there are multiple ways to achieve the same result. Please do not deem to above code as production-ready, as it may contain bugs and might have stability issues. Thorough testing is strongly recommended, before considering it in production.**
