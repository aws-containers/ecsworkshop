+++
title = "Review blue deployment"
chapter = false
weight = 4
+++

#### Open the service in your browser

We can access the deployed blue version via the load balancer url. Let's open it in our browser.

Run the following command to get the url:

```bash
echo "http://$ALB_DNS"
```

#### Result

- The endpoint will show the version of the application that you are deployed (which will be blue).

![blue-deployment](/images/blue-green-deployment-1.png)

* Now it's time to update the CodeCommit repository for the green deployment
