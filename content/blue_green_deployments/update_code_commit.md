+++
title = "Update CodeCommit Repository"
chapter = false
weight = 5
+++

#### Edit the `index.html` to change the `background-color` to `green`

```bash
cd ~/environment/nginx-sample/
vim index.html
```

```html
<head>
  <title>Demo Application</title>
</head>
<body style="background-color: green;">
  <h1 style="color: white; text-align: center;">
    Demo application - hosted with ECS
  </h1>
</body>
```

#### Push the code to the CodeCommit repository
```bash
git add .
git commit -m "Changed background to green"
git push
``` 

#### Navigate to CodePipeline
* The code push has triggered the pipeline execution
* We have three stages in the pipeline
    * **Source**
        * Pull down the code from the git repository
        * Package the code for the build stage
    * **Build**
        * Build the Docker image from the code
        * Push to the ECR repository
        * Package the artifacts for the deploy stage 
    * **Deploy**
        * Initiate the blue/green deployment for the ECS service

![Navigate-to-CodePipeline](/images/blue-green-navigate-to-codepipeline.gif)

* Next we will review the deployment
