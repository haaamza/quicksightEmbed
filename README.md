# quicksightEmbed using Aws Lambda, Aws Cognito, Api Gateway
Here is the overlook for setting up the AWS QuickSight Dashboard and Publishing it.


You Will need:

1- AWS QuickSight Enterprise Edition Account
2- AWS Cognito
3- AWS IAM
4- AWS EC2 for deployment
5- A Domain with a valid SSL certificate as QuickSight won't work without SSL.




STEPS:
1- Create a analysis using AWS QuickSight.
2- Publish Your analysis as a Dashboard.
3- Create an App using AWS Cognito and a user Pool for the application.
4- Create a Role/Policy for the user pool to give them read access to the Dashboard.
5- Assign the Role/Policy to the user pool.
6- Use your app to register user using AWS Cognito.
7- Get the Embedded URL using AWS JavaScript SDK. (Every User will have unique URL)
8- User can login using app and see the Dashboard and interact with it.


Feel free to ask me any queries that you may have.

Best Regards,
Ameer Hamza




In order to generate Quicksight secure dashboard url, follow the below steps:



Step 1: Create a new Identity Pool. Go to https://console.aws.amazon.com/cognito/home?region=us-east-1 , click ‘Create new Identity Pool’

Give an appropriate name. Go to the Authentication Providers section, select Cognito. Give the User Pool ID(your User pool ID) and App Client ID (go to App Clients in userpool and copy id). Click ‘Create Pool’. Then click ‘Allow’ to create roles of the identity pool in IAM.



Step 2: Assign Custom policy to the Identity Pool Role (not the Auth or Unauth like in your case i think so   testuserpool1role)

Create a custom policy with the below JSON.

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": "quicksight:RegisterUser",
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Action": "quicksight:GetDashboardEmbedUrl",
            "Resource": "*",
            "Effect": "Allow"
        },
        {
            "Action": "sts:AssumeRole",
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
Note: if you want to restrict the user to only one dashboard, replace the * with the dashboard ARN name in quicksight:GetDashboardEmbedUrl,    (ARN is resourse number and can be get through the quicksight pannel)

then goto the roles in IAM. select the IAM role of the Identity pool and assign custom policy to the role.



Step 3: Configuration for generating the temporary IAM(STS) user

Login to your application with the user credentials. For creating temporary IAM user, we use Cognito credentials. When user logs in, Cognito generates 3 token IDs - IDToken, AccessToken, RefreshToken. These tokens will be sent to your application server.

For creating a temporary IAM user, we use Cognito Access Token and credentials will look like below.

const AWS = require('aws-sdk');
 AWS.config.region = 'us-east-1';     //region for your Cognito
       AWS.config.credentials = new AWS.CognitoIdentityCredentials({
           IdentityPoolId:"Identity pool ID",  //your cognito pool id
           Logins: {
               'cognito-idp.us-east-1.amazonaws.com/UserPoolID': AccessToken
           }
       });
For generating temporary IAM credentials, we call sts.assume role method with the below parameters.

var params = {
           RoleArn: "Cognito Identity role arn",    //you can have arn by selecting the identity role and seeing the details
           RoleSessionName: "Session name"  //any name
       };
sts.assumeRole(params, function (err, data) {
           if (err) console.log( err, err.stack); // an error occurred
           else {
               console.log(data);
})
You can add additional parameters like duration (in seconds) for the user. Now, we will get the AccessKeyId, SecretAccessKey and Session Token of the temporary user.

Step 4: Register the User in Quicksight

With the help of same Cognito credentials used in the Step 3, we will register the user in quicksight by using the quicksight.registerUser method with the below parameters

var params = {
                   AwsAccountId: “account id”,
                   Email: 'email',
                   IdentityType: 'IAM' ,
                   Namespace: 'default',
                   UserRole: ADMIN | AUTHOR | READER | RESTRICTED_AUTHOR | RESTRICTED_READER,
                   IamArn: 'Cognito Identity role arn',
                   SessionName: 'session name given in the assume role creation',
               };

quicksight.registerUser(params, function (err, data1) {
                   if (err) console.log("err register user”); // an error occurred
                   else {
                       // console.log("Register User1”);
                   }
               })
Now the user will be registered in quicksight.

Step5: Update AWS configuration with New credentials.

Below code shows how to configure the AWS.config() with new credentials generated Step 3.

AWS.config.update({

                   accessKeyId: AccessToken,
                   secretAccessKey: SecretAccessKey ,
                   sessionToken: SessionToken, 
                   "region": Region
                 });
Step6: Generate the EmbedURL for Dashboards:

By using the credentials generated in Step 3, we will call the quicksight.getDashboardEmbedUrl with the below parameters

var params = {
  AwsAccountId: "account ID",
  DashboardId: "dashboard Id",
  IdentityType: "IAM",
  ResetDisabled: true,
  SessionLifetimeInMinutes: between 15 to 600 minutes,
  UndoRedoDisabled: True | False
}

quicksight.getDashboardEmbedUrl(params,
  function (err, data) {
    if (!err) {
      console.log(data);
    } else {
      console.log(err);
    }
  });
Now, we will get the embed url for the dashboard.

Call the QuickSightEmbedding.embedDashboard from front end with the help of the above generated url. The result will be the dashboard embedded in your application with filter controls.
