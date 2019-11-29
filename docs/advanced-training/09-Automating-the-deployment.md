# 08 - Automating the deployment

Deploying our application was easy, right? But we don't want to do that
manually every time we develop new features or fix bugs. We already have
a pipeline that automatically tests & builds our image for every commit to the
master branch, why don't we also automatically renew our deployment? As you've learned in the theory part about staging & branching, it is a good idea to test your application in a test environment before deploying it into production.

Before we automate our deployment, let's remove the manual deployment that we created earlier, because we won't need that anymore. Execute the following commands to delete all application resources we created earlier:

```bash
cd ~/devops-training/devops-training-application/deploy
kubectl delete -f ingress.yaml
kubectl delete -f application.yaml
```

Your task will be to extend the `.gitlab-ci.yml` with another pipeline
step that automatically deploys our application into our test environment. We will extend the variables block with two variables. One will hold the name of our Kubernetes cluster, the other one will hold your prefix (your initials or whatever you chose):

```yaml
variables:
  DOCKER_TLS_CERTDIR: ''
  REGION: eu-central-1
  REPOSITORY_URL: <your-ECR-repository>
  KUBERNETES_CLUSTER: <your-kubernetes-cluster>
  PREFIX: <prefix>
```

You will need the `kubectl` CLI for that, which you can install in the `before_script` step:

**_devops-training-application/.gitlab-ci.yml:_**

```yaml
before_script:
  - apk add --no-cache curl jq python py-pip gettext
  - pip install awscli
  # Add the lines below to make the CLI available in our build container
  - curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl && mv kubectl /usr/local/bin && chmod +x /usr/local/bin/kubectl
  - curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.12.7/2019-03-27/bin/linux/amd64/aws-iam-authenticator && chmod +x ./aws-iam-authenticator && mkdir -p $HOME/bin && cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$HOME/bin:$PATH
```

The deploy stage will roughly look like this:

```yaml
deploy_test:
  stage: deploy
  environment:
    name: test
  script:
    # Your deploy commands go here
  except:
    - master
```

Inside the `script` of this stage you will need to execute two commands. Can you guess which ones?

??? note "The two commands..." 1. You need to update the kubectl config to be able to interact with
the cluster. You can use the `aws eks update-kubeconfig...` command that we used earlier. 2. You need to apply the Kubernetes resources, as we did manually in
the last part. Every time we run our pipeline, we build a new image of our application with a new image tag. We want to deploy exactly this version of our image to Kubernetes. That's why we need to get the image tag into our `application.yaml`. You will find another file called `voting-application/deploy/application.yaml.tmpl` which you can use to replace the correct image tag before deploying. Remember that we tagged our image with the environment variable `$CI_COMMIT_SHORT_SHA`? Now we want to deploy exactly this version of our application, so we need to enter the exact value of this environment variable. Luckily there is a handy tool called `envsubst` which takes a file as input and substitutes all environment variables with their corresponding values. We can combine it like this with our `kubectl apply` command:

      ```bash
      - cat deploy/application.yaml.tmpl | envsubst "`env | awk -F = '{printf \" \\\\$%s\", $1}'`" | kubectl apply -f -
      ```

Try to figure out how the deploy step should look like and open the
solution when you're finished.

??? note "Solution: `.gitlab-ci.yml` and `application.yaml.tmpl`"

    **_devops-training-application/.gitlab-ci.yml:_**

    ```yaml
    image: docker:latest

    variables:
      DOCKER_TLS_CERTDIR: ''
      REGION: eu-central-1
      REPOSITORY_URL: <insert your repository url>
      KUBERNETES_CLUSTER: <insert your Kubernetes cluster name>
      PREFIX: <your-prefix>

    services:
      - docker:dind

    before_script:
      - apk add --no-cache curl jq python py-pip gettext
      - pip install awscli
      - curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl && mv kubectl /usr/local/bin && chmod +x /usr/local/bin/kubectl
      - curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.12.7/2019-03-27/bin/linux/amd64/aws-iam-authenticator && chmod +x ./aws-iam-authenticator && mkdir -p $HOME/bin && cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$HOME/bin:$PATH

    stages:
      - test
      - build
      - pull_push
      - deploy
      - smoke_test

    test:
      stage: test
      image: node:12.1-alpine
      script:
        - cd ./app && npm install && npm test
      except:
        - master

    build:
      stage: build
      script:
        - $(aws ecr get-login --no-include-email --region $REGION)
        - docker build -t $REPOSITORY_URL:$CI_COMMIT_SHORT_SHA -t $REPOSITORY_URL:latest ./app
        - docker push $REPOSITORY_URL:$CI_COMMIT_SHORT_SHA
        - docker push $REPOSITORY_URL:latest
      except:
        - master

    deploy_test:
      stage: deploy
      environment:
        name: test
      script:
        - aws eks update-kubeconfig --name $KUBERNETES_CLUSTER --region $REGION
        - cat deploy/application.yaml.tmpl | envsubst "`env | awk -F = '{printf \" \\\\$%s\", $1}'`" | kubectl apply -f -
      except:
        - master
    ```

    We store the name of our Kubernetes cluster in a variable called
    `KUBERNETES_CLUSTER` in line 7. Don't forget to set the name of your Kubernetes cluster there. We then add a new step called `deploy_test` during which we update the kubeconfig and apply our Kubernetes resources.
    Simple, isn't it?

Once you've updated your pipeline definition, feel free to commit the code to your repository to try your automatic deployment to your test environment.

```bash
cd ~/devops-training/devops-training-application/
git add .
git commit -m "Automating the deployment"
git push
```

Switch to your Gitlab project and wait until your pipeline has finished all three steps. What has happened? Gitlab has executed the unit tests, build our image, pushed it to our ECR repository and deployed the application. The deployment is in a Kubernetes namespace called `test`, so let's see the services that we created in there. Run the following commands from your terminal:

```bash
kubectl get svc -n test
```

Output:

```bash
NAME                TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)        AGE
frontend            LoadBalancer   172.20.191.187   a4a798b62bf3c11e9bfe60672e92a85e-302946498.eu-central-1.elb.amazonaws.com   80:31057/TCP   3m46s
redis               ClusterIP      172.20.224.32    <none>                                                                      6379/TCP       3m45s
redis-monitor-svc   ClusterIP      172.20.174.238   <none>                                                                      9121/TCP       3m45s
```

Copy the URL of your LoadBalancer and open it in a new browser tab to check that your application is working. You know what you did right there? A manual smoke test. You opened the URL of the service in your browser and checked if you get the right content back, which suggests that the service is working. But why would you do that manually? Let's add another step into our pipeline, which automates just that: a smoke test.

## Smoke test

A smoke test is pretty basic. We will just `curl` the index page of our application and in case we receive an HTTP status code of 200, everything is fine. We have to make two things sure, before we can call our index page:

1. The application is successfully deployed. Luckily there is a command that we can use to check just that. Run that from your terminal and you should see that the deployment was successfully rolled out:

```bash
kubectl rollout status deploy frontend -n test
```

2. The LoadBalancer has to be ready to route traffic to our application. AWS LoadBalancers take a short amount of time before they get ready after creation. That's why we will delay this pipeline step by 2 minutes, so our LoadBalancer has some time to become ready.

The last thing we need for our smoke test is the actual URL of the LoadBalancer. Previously, we executed `kubectl get svc` and then copied the LoadBalancer URL from the output. There is a handy command which we can use to get just the LoadBalancer URL which we can then pass to the `curl` command:

```bash
kubectl get svc frontend -n test -o json | jq -r .status.loadBalancer.ingress[0].hostname
```

Go ahead and try it.

Now we have to fit the pieces together and build our smoke test. Add the following to the end of your existing pipeline definition:

**_voting-application/.gitlab-ci.yml:_**

```yaml
smoke_test_staging:
  stage: smoke_test
  environment:
    name: test
  script:
    - aws eks update-kubeconfig --name $KUBERNETES_CLUSTER --region $REGION
    - kubectl rollout status deploy frontend -n $CI_ENVIRONMENT_SLUG
    # If the service is deployed, the curl command will show an HTTP status code 200 and return with 0, suggesting that the deployment was successful
    - curl -I $(kubectl get ingress application-ingress -n $CI_ENVIRONMENT_SLUG -o json | jq -r ".spec.rules[0].host")
  when: delayed
  start_in: 2 minutes
  except:
    - master
```

If you commit and push your code, you can see the smoke test in action:

```bash
cd ~/devops-training/devops-training-application/
git add .
git commit -m "Implemented a smoke test"
git push
```
