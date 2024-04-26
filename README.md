# Training exercise 1 - The real cluster

## Getting access

### Mail...

Your group will receive an email containing the _kubeconfig_ file granting you
admin access to a virtual cluster

Since everyone gets a group number your access endpoint will be:
_https://swwao.courses.orbit.au.dk/grp-X_ where _X_ is your group number.

All applications, both those that have already been installed as well as those
that you are to install will reside, url wise, at a sub url from this point.

NB! All ingress routes must contain this url, otherwise your routes will not
work. This in turn means that any application you create must have a base url
being configurable.

### Kubeconfig

Move the kubeconfig file you receive from your instructor to the directory ~/.kube/

Reload _OpenLens_ and verify that you actually have access to a new
cluster. Select the cluster and have a look around.

However for _kubectl_ you either have to merge this config file into
_~/.kube/config_ or set the environment variable _KUBECONFIG_ accorddingly.

Assuming that your kubeconfig.yaml resides in your .kube directory then in:
- Windows (change søren to your name)
> $Env:KUBECONFIG="c:\users\søren\.kube\kubeconfig.yaml"

- Linux/Mac
export KUBECONFIG=~/.kube/kubeconfig.yaml



As a secondary object, change the default cluster context for _kubectl_ such
that performing a
> kubectl apply -f somedeployment.yaml 
will actually be applied to your newly acquired cluster.

### Applications

Inspect your newly acquired cluster and see which applications and thus
_Deployments_ that have already been installed.

Make a note of these.


#### ArgoCD

ArgoCD a k8s controller has been preinstalled at the url
https://swwao.courses.orbit.au.dk/grp-X/argocd Login using the username _admin_ and the
password _adminpassword_. You might wanna change this password.

#### Grafana (and Prometheus)

These have been installed as well. You can login using the username _admin_ and
password _prom-operator_. You might wanna change this password.



# Training exercise 2 - Migrating to the real virtual cluster

In the previous lecture you created two different setups. One using
_ConfigMaps_, _Secrets_ and a _Deployment_ with a single _Persistent Volume
Claim_ and another using a _StatefulSet_ and multiple _Persistent Volume
Claims_.

These will be used as a basis for the following exercises.

For training mirgrating both will be a _very_ good exercise, however this takes
time... so you might opt to skip one...

## Namespace

You have two setups for each of these ensure that they are placed in their own
namespace.

## Ingress

Modify the _Ingress_ part for each setup and have each have their own url base
path.

## Resources

Consider the amount of resources (CPU & Memory) that you believe will be
adequate for each _Pod_ in each of the sets.

Add these requirements to the respective _Pod_ specs.

### Things to heed or consider
- Why is this actually necessarily in this context?
- why do you recon this is an important thing to always consider?
- What happens if
  - CPU request/limit is exceeded?
  - Memory request/limit is exceeded?

## Apply & Verify

Apply each setup to your newly acquired cluster. 

Check that each application has their respectively _Deployments/StatefulSet_ up
and running as you presume. This also includes the needed _Persistent Volume
Claims_, _Ingress_, _Services_ etc...

Upon having this verified check that the applications work at their respective
urls.

# Training exercise 3 - GitOps - ArgoCD

## Intro

The goal of this exercise is to have _ArgoCD_ watch a repository on
_gitlab.au.dk_ in which you have yaml files that describe your application
setup.

Changes to these files should therefore, within a short timeperiod, be reflected
in your cluster. If, for instance, you change the image specified in the _Pod_
_spec_ of your _Deployment_ then this image should be fetched and deployment and
thus started in your cluster.

For this to be achieved several steps have to be completed.

## k8s yaml setup repository

Create a git repository that contains the setup, e.g. yaml files, describing how
your application is to be deployed in your cluster.

A reasonable starting point would be one of the applications migrated earlier in
exercise 2.

### Container registry access @ gitlab.au.dk

Heads up - The _imagePullSecrets_ added in the previous lecture to the _Pod_
spec must likewise be employed here, otherwise your cluster won't be able to
fetch and deploy it.

Placing this in a repos is not the best, however we will accept it for the time being. For this to work we need to extract the Secret's contents and place it in a yaml file. This can be achieved like this: 
> kubectl get secrets my-registry-credentials -o yaml -n app > mysecret.yaml

Do note that you should remove the following lines containing:
- creationTimestamp
- resourceVersion
- uid

before you add/commit the file!

## ArgoCD
### Accessing your Git repos @ gitlab.au.dk

Create or reuse a token making it possible to access a repository at
gitlab.au.dk. In particular it must be possible to access (in read/write mode)
the _k8s yaml setup_ repository just created. Furthermore ensure that the token
created actually may push to the repository!! (If unsure see Project->Settings->Repository->Protected branches)

Use the token and add the repository to _ArgoCD_ - See:
https://argo-cd.readthedocs.io/en/stable/user-guide/private-repositories/

Its easy to do via the ArgoCD Web GUI.


### Add your Application

NB! Your cluster shouldn't have the application running from a previous exercise
at this point, so if it is, then delete it!

In the ArgoCD Web GUI add an application. Use the repository just added as source.

See slides for help!


### Verify

Having added your application, you should now see it being started up.

Inspect the different views in _ArgoCD_ and see how your application pods are
placed on which nodes.

## GitLab CI/CD

All elements are now ready to connect the final dots... MEANING that having
completed this exercise you should have a system that upon creating a git tag in
your _Application repos_ ensues that the following steps are carried out and the
deployed application in the cluster has been updated within minutes.
- create a new application image
- updated the yaml files describing the _Deployments_ and has _ArgoCD_
automatically updating your deployment such that your updated application runs
on the cluster...
- deployed to the cluster - ArgoCD's job

It is assumed that you have the following otherwise this needs to be done before you carry on!

### Application repos

You have to have your application, that is to run in your cluster, placed in a
repository. Futhermore it must have a CI/CD pipeline that creates an image upon
adding a tag to the repos.

The best approach would obviously be that you have your own application with a
working associated pipeline, however if that is not the case you can use
https://gitlab.au.dk/swwao/demo-projs/typescript-mongodb-crud as a starting
point.

In the latter case fork the repository so you have it as your own.

### Preparation for the new deployment stage

For this to work, several things need to be in order.

#### Access to the repository containing the k8s yaml setup 

The _build bot_ must have read/write access to the yaml git repository. 

1. In the _k8s yaml setup git repos_ create an access token that has read/write access
2. In your application add two CI/CD environment variables that contain _build bot's_ username as well as the access token password. The idea is that these should not be in plain text in the log files.

The _build bot's_ username can be found on the members list in your project. It
will have name similar to this _project\_9142\_bot\_32132131223134_. The number will be
different. 

The environment variables should be denote _Masked_ and _Expanded_ when they are
created.

#### Needed tools

You need access to two different tools.
1. git to checkout, add commit push the new changes to the _k8s yaml setup repository_
2. The ability to modify yaml files from the command line. Several approaches exists ranging from _sed_ to _yq_

Simple possible solution for each:
S1: Use _alpine/git_
S2: Download _yq_ unto the _S1_ container
> export VERSION=v4.33.1
> wget https://github.com/mikefarah/yq/releases/download/${VERSION}/${BINARY}.tar.gz -O - | tar xz && mv ${BINARY} /usr/bin/yq

If you really feel up to it, then create a new repository containing a
_Dockerfile_ depending on the _alpine/git_ and download and install _yq_. IF so,
then this is the image you depend upon in the _deployment-stage_


Either approach works, however there are obviously different approaches 

### Deployment stage

Several things have to be carried out:

1. Configurering git user / email
2. Clone the _k8s yaml setup repository_
   - The user and password should be the two environment variables added earlier. Remember their name, not contents!!
3. Use _yq_ or whatever tool to choose to change the image specified in the _Pod's_ spec. Remember to figure out the correct image name. See image build stage. (Hint: Run the command in a terminal with a relevant yaml until you figure out how this should be done!)
4. Add the changes to github
5. Commit them with a reasonable comment
6. push them onto master

Finally remember this stage should only be run insofar that the git repository has been tagged!

# Training exercise 4 - Nginx in Grafana

## Dashboard

Login Grafana and go browse the available dashboards. Create a new folder called
"Overview" (or whatever). 

### Nginx

To get an overview of what happens stats wise in your Nginx ingree import the dashboard having the id 9614.

Now you have access to monitor your very own _Nginx_ server!

### Virtual cluster wide overview

A simple yet powerful dashboard that provides you with an overview of your entire cluster can be imported using the id _15758_.

## Monitoring own Application - BONUS


At some point you might actually need to know what your application spends its
time on. As a bonus exercise by inspired by the repos
https://github.com/RisingStack/example-prometheus-nodejs and add this capability
to your application. Following it through will mean that you via Grafana can
create graphs showing metrics from your application.
