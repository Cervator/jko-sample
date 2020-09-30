## Sample repo for the Jenkins Kubernetes Operator (JKO)

This is a quick sample setup for the JKO, following the [official documentation](https://jenkinsci.github.io/kubernetes-operator/docs/) with a few minor tweaks and fixes.

For the standard part of the setup get a cluster ready with `kubectl` able to run commands against it and [execute the following to install](https://jenkinsci.github.io/kubernetes-operator/docs/installation/):

* `kubectl apply -f https://raw.githubusercontent.com/jenkinsci/kubernetes-operator/master/deploy/crds/jenkins_v1alpha2_jenkins_crd.yaml`
  * This creates the custom resource (CR) in Kubernetes that defines what a "Jenkins" is
* `kubectl apply -f https://raw.githubusercontent.com/jenkinsci/kubernetes-operator/master/deploy/all-in-one-v1alpha2.yaml`
  * This installs the JKO pod that knows how to look for instances of the Jenkins CR then making the needed k8s objects to actually create and run masters
    * This could be considered a form of GitOps, as the operator pod watches a resource then makes the cluster converge to match. Normally this is a Git repo outside the cluster, however, having to apply changes into the cluster in the first place to have the changes read there somewhat violates the spirit of _True_ GitOps.

## Sample masters

Installing the JKO does not actually get you any Jenkins masters. Those are what you define in the new Jenkins CR for Kubernetes. Beyond the base setup of a master you can also define plugin lists, seed jobs for [Jenkins Job DSL](https://github.com/jenkinsci/job-dsl-plugin, ), [JCasC](https://github.com/jenkinsci/configuration-as-code-plugin), init scripts, and much more. Two simple examples are included at the root of this repo.

* Example team A: This master loads the basic Job DSL scripts from the JKO example along with a second Job DSL repo containing some utility jobs. These in turn depend on two more plugins than the base.
* Example team B: This master is about the same, again loads the base Job DSL from the JKO, but then a different second repo with some sample build jobs.

Both masters otherwise are fairly standard and come out of the JKO box with a very nice setup, including a Kubernetes Cloud for creating build agents in.

One potential bug: The default setup comes up with the masters on a `ClusterIP` which is not reachable outside the cluster. You can uncomment the two final lines to switch their service types to `LoadBalancer` which will cause a cloud hosting provider to create publicly reachable LBs. However this may _not_ work on the first try for some reason - that may be fixed in the more [active fork under RedHat](https://github.com/redhat-developer/jenkins-operator) where a more sophisticated ingress setup is in place. As a workaround creating the master first with `ClusterIP` then applying a second time to get a `LoadBalancer` seems to work, at least on Google Cloud.

Another bug appears to be plugin version conflicts due to something not being version-pinned somewhere. The example files have had their plugins updated explicitly to the latest plugin versions as of 2020-09-29. There are blocks both for `basePlugins` and `plugins` - exact distinction needs a bit more poking around :-)

### Setting up the masters

To continue from the installation to also create a master try the following:

* `kubectl apply -f jenkins_team_a.yaml`
  * This creates a CR in the cluster of type "Jenkins" which the JKO pod will spot and read to create the k8s objects for the master. Give it a minute or two. Check the operator's pod logs if needed
* `kubectl get secret jenkins-operator-credentials-example-a -o 'jsonpath={.data.user}' | base64 -d`
  * This will write the username of a newly created admin user to the terminal. In this case `example-a` is the name of the Jenkins from `metadata.name` in the master's YAML file
* `kubectl get secret jenkins-operator-credentials-example-a -o 'jsonpath={.data.password}' | base64 -d`
  * And this is how you get the password. Everything via Kubernetes secrets, sweet!

At this point you should be tunnel via `kubectl` to the new master if you want to check it out, logging in with the user/pass retrieved above. You can switch the service to use a `LoadBalancer` by:

* Edit `jenkins_team_a.yaml` and uncomment the two lines at the end for `service.type`
* `kubectl apply -f jenkins_team_a.yaml` again

Note that if you end up entirely deleting your master to recreate it (`kubectl delete -f jenkins_instance.yaml`) you may have to first recreate with the `service.type` commented out again, then re-apply

### Observing Job DSL in action

If you create both the example masters you should be able to see them come online with different jobs.

* Master A points to https://github.com/Cervator/GitOpsUtilityJobs as indicated near the bottom of its YAML file in the `seedJobs` section - this makes a few utility jobs in a GitOpsUtility folder
  * The utility jobs include use of two additional plugins, which have been added to the YAML file for the master in the `plugins` section
* Master B points to https://github.com/Cervator/job-dsl-gradle-example which creates some sample build jobs that may even run (and succeed?) before you even manage to get into the master to look!
* Both masters will also have jobs from the JKO example, from https://github.com/jenkinsci/kubernetes-operator/tree/master/cicd
