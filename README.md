# CloudFormation Template: Demo for Migrating a Web Server with Application Migration Service
This guide comprises a CloudFormation template and demo instructions on how to migrate a web server to a new EC2 instance using Application Migration Service (MGN).

We will deploy a 'source server', then install the Application Migration Service agent on the source server and configure MGN to launch and cutover to a new instance.

While this template does deploy an instance (our source server) to AWS, it is also representative of having a server hosted in any number of other manners, including on-premise.

## The Guide
1. Ensure you have an AWS account to deploy into. Configure your terminal so you can run AWS CLI commands in this account.
2. Create an EC2 key pair (`RSA` and `.pem`) in the console with the name `mgn_demo`. If you use a different name, pass this as the `Ec2KeyPairName` parameter in step 6.
   > While we do not intend to SSH to the instance in this guide, it can be useful to have SSH access for debugging purposes. System Manager is also configured to gain access to the EC2 terminal using Session Manager.
3. Set up a user for the MGN agent. Follow this guide and get your access key ID and secret access key: https://docs.aws.amazon.com/mgn/latest/ug/credentials.html
4. Pass in key ID and secret access key values using the supplied  `MgnUserAccessKeyId` and `MgnUserSecretAccessKey` parameters in step 6.
5. Configure the Application Migration Service in the console:
   * **Staging area subnet:** Select the VPC and Subnet prefixed `TargetVpc*` in your account.
   * Leave all other settings as default values
6. Deploy the stack:
   ```bash
   aws cloudformation create-stack \
   --stack-name mgn-demo \
   --capabilities CAPABILITY_NAMED_IAM \
   --template-body file://mgn_demo.yml \
   --parameters \
   ParameterKey=Ec2KeyPairName,ParameterValue=<keypair-name> \
   ParameterKey=MgnUserAccessKeyId,ParameterValue=<AccessKeyId> \
   ParameterKey=MgnUserSecretAccessKey,ParameterValue=<SecretAccessKey>
   ```
7. Wait for the instance to appear as a source server in the Application Migration Service console and complete the initialization. You'll know it's completed once the `Next step` is `Launch test instance`. MGN can take 20-30 minutes to get to this stage.
   > You can look at the user data in the CloudFormation template to see how this happens automatically. We install and start Apache to simulate a web server, and then we install the MGN agent and pass AWS credentials to it in order to connect to the MGN console in our account.
8.  Once the source server has `Next step` as `Launch test instance` in the MGN console, modify the launch template for the source server in the MGN console (select source server > `Launch settings` > `EC2 Launch Template` > `Modify`):
   * **Amazon machine image (AMI):** _Don't include in launch template_
   * **Instance type:** `t2.micro` (change this from the default `c4.large`)
   * **Key pair (login):** _Don't include in launch template_
   * **Network settings (Networking platform):** _Virtual Private Cloud (VPC)_
   * **Network settings (Security groups):** leave blank (we will select the security group in the `Network interfaces` section)
   * **Storage (volumes):** leave as default `Volume 1`
   * **Resource tags:** leave as default (MGN adds some tags)
   * **Network interfaces (subnet):** `TargetVPC-subnet`
   * **Network interfaces (security groups):** `mgn-demo-TargetEc2SecurityGroup`
   * **Network interfaces (Auto-assign public IP):** `Enable`
   * **Advanced details:** No changes
9.  Set the launch template's default version to be the new version you just created (creating a new version of a launch template does _not_ automatically set that new version to be the default version; you need to do this manually after creating the new version).
10.  Back in the MGN console, you should see the launch template for your source server automatically update to show the new instance type, security group and subnet. Choose `Test and Cutover` > `Launch test instances` > `Launch` to create a new test instance. This step can take a while (10-20 minutes). You can click on the "job" to view the job log as the instance launches.
11. You will eventually see the test instance has been launched. Head to `http://<new instance IP address>` and check that you can see the Apache welcome page. This shows that the server has successfully replicated.
12. On the source server page in the MGN console, head to `Test and Cutover` > `Mark as "Ready for cutover"`. Leave the checkbox checked (to terminate the test instance) and press `Continue` to mark the instance as `Ready for cutover`.
13. You may need to modify the launch template again at this point. Change the version back to the version that you created earlier, where the instance type was `t2.micro`.
14. When you are ready to complete the cutover, click on `Test and Cutover` > `Launch cutover instances` > `Launch`.
    > You may receive the error `One or more of the Source Servers included in API call are currently being processed by a Job`. This means that the instance is not yet ready for cutover as the test instance is still being terminated. Wait a couple of minutes, then try again.
    You can click on the cutover job ID to view the progress of launching the cutover instance.
15. Once the instance is launched, finalize the cutover by heading to `Test and Cutover` > `Finalize cutover` > `Finalize`. 
16. The final step is to terminate the old instance and check the website hosted by the new instance. We can now say we have successfully migrated an instance using the Application Migration Service!

## Troubleshooting
### Deploy New Stack
   ```bash
   aws cloudformation create-stack \
   --stack-name mgn-demo \
   --capabilities CAPABILITY_NAMED_IAM \
   --template-body file://mgn_demo.yml \
   --parameters \
   ParameterKey=Ec2KeyPairName,ParameterValue=<keypair-name> \
   ParameterKey=MgnUserAccessKeyId,ParameterValue=<AccessKeyId> \
   ParameterKey=MgnUserSecretAccessKey,ParameterValue=<SecretAccessKey>
   ```

### Update Stack
   ```bash
   aws cloudformation update-stack \
   --stack-name mgn-demo \
   --capabilities CAPABILITY_NAMED_IAM \
   --template-body file://mgn_demo.yml \
   --parameters \
   ParameterKey=Ec2KeyPairName,ParameterValue=<keypair-name> \
   ParameterKey=MgnUserAccessKeyId,ParameterValue=<AccessKeyId> \
   ParameterKey=MgnUserSecretAccessKey,ParameterValue=<SecretAccessKey>
   ```

### Delete Stack
```bash
aws cloudformation delete-stack --stack-name mgn-demo
```

### Specify Parameter Values on Command Line
```bash
aws cloudformation create-stack --stack-name mgn-demo --capabilities CAPABILITY_NAMED_IAM --template-body file://mgn_demo.yml ParameterKey=theparametername,ParameterValue=theparametervalue
```
