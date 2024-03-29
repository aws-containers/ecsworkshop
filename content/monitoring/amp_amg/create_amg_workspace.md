---
title: "Create AMG workspace"
draft: false
weight: 9
---

#### Prerequisite

AMG requires AWS SSO enabled in your account. AWS SSO is used as the authentication provider to sign into the AMG workspace.

##### Follow the steps below to enable AWS SSO in your account

- Sign in to the AWS Management Console with your AWS Organizations management account credentials.
- Open the [AWS SSO console](https://console.aws.amazon.com/singlesignon).
- Choose `Enable AWS SSO`.

If you have not yet set up AWS Organizations, you will be prompted to create an organization. Choose `Create AWS organization` to complete this process.

Now go ahead and create a new AWS SSO user that we will use to provide access to the AMG workspace later.

- Choose `Users` on the left side of AWS SSO console and click `Add user`
  ![Create SSO user](/images/sso1.png)

- Provide the following required information on next screen
  ![SSO user details](/images/sso2.png)

- Choose `Next:Groups`

- Click `Add user` in lower right

#### Create AMG workspace

Go to the [AMG console](https://console.aws.amazon.com/grafana/home), click on **Create workspace** and provide a workspace name as shown below, click `Next`, 
![Create AMG workspace](/images/amg1.png).

Choose `Service managed` in the `Configure Settings` page and click `Next`. Choosing this option will allow the wizard to automatically provision the permissions for you based on the AWS services we will choose later on.

In the `Service managed permission settings` screen, you can choose to configure Grafana to monitor resources in the same account where you are creating the workspace or allow Grafana to reach into multiple AWS accounts by choosing the `Organization` option and providing the necessary OU IDs.

![Configure accounts](/images/amg2.png)

We will simply leave the option to `Current account` and select all the Data sources and the Notification channels. Click `Next`

![Review amg screen](/images/amg3.png)

In the Review screen, take a look at the options and click on `Create workspace`

#### Add Users

Once the AMG workspace turns to `ACTIVE`, click on `Assign user` and select the SSO user created in previously. Click `Assign user`

![Assign user](/images/amg4.png)

By default, all newly assigned users are added as `Viewers` that only provides read-only permissions on Grafana. To make the user as Administrator, select the user under `Users` and select `Make admin`. Now you should see that the user is an Administrator.

![Assign user](/images/amg5.png)
