# Polling Triggers in OpenShift Pipeline
By Daein Park

[Webhooks](https://docs.github.com/en/webhooks-and-events/webhooks/about-webhooks) can be triggered whenever specific events on GitHub send change information to [OpenShift Pipelines](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/understanding-openshift-pipelines.html) for integrating your CI/CD workflow. In the process, you should set up your server to receive and manage the payload from the Internet by [adding triggers to a pipeline](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/creating-applications-with-cicd-pipelines.html#adding-triggers_creating-applications-with-cicd-pipelines). If your pipeline is in a restricted network and does not allow access from the Internet (Github), it is necessary to consider poll-based change detection for integrating the GitHub in the Internet instead.

Poll-based change detection is a kind of polling trigger, which is an event that periodically makes a call to your git repository to look for new changes. This article will take you through how to configure polling triggers in OpenShift Pipelines ([Tekton](https://tekton.dev/)) in a restricted network.

## Workflow of Pull-Based Triggers
Basically, a polling trigger is a pull-based workflow, while webhooks are push-based. In other words, polling triggers  periodically initiate an event over an interval to determine if there is any change, whereas webhooks respond to a push of changes from the repository.

If you have a private GitLab repository in your disconnected network, you can configure the polling triggers for your pipeline CI/CD using [Repository Mirroring](https://docs.gitlab.com/ee/user/project/repository/mirror/) (which mirrors all changes over an interval with another repository);Webhooks are usually triggered by a push event. The following diagram shows the workflow based on GitLab:

![Polling Triggers Diagram](https://github.com/bysnupy/blog/blob/master/kubernetes/polling-trigger-diagram.png)

However, GitLab is overkill for some use cases, so you should implement polling triggers using [CronJob](https://docs.openshift.com/container-platform/4.13/nodes/jobs/nodes-nodes-jobs.html#nodes-nodes-jobs-creating-cron_nodes-nodes-jobs) with bash scripts. The diagram above [Option 2] illustrates this.

## Polling Triggers Using CronJob
I prepared a sample application for testing pipeline run through [Creating CI/CD solutions for applications using OpenShift Pipelines](https://docs.openshift.com/container-platform/4.13/cicd/pipelines/creating-applications-with-cicd-pipelines.html) as follows. In this section, I skipped a secret token configuration in Trigger YAML for simpler tests. In a restricted network, security would not be a problem.

```console
// Create a project for the sample application
$ oc new-project pipelines-tutorial

// Creating pipeline tasks
$ oc create -f https://raw.githubusercontent.com/openshift/pipelines-tutorial/pipelines-1.9/01_pipeline/01_apply_manifest_task.yaml
$ oc create -f https://raw.githubusercontent.com/openshift/pipelines-tutorial/pipelines-1.9/01_pipeline/02_update_deployment_task.yaml

// Assembling a pipeline
$ oc create -f https://raw.githubusercontent.com/openshift/pipelines-tutorial/pipelines-1.9/01_pipeline/04_pipeline.yaml

// Adding triggers to a pipeline
$ oc create -f https://raw.githubusercontent.com/openshift/pipelines-tutorial/pipelines-1.9/03_triggers/01_binding.yaml
$ oc create -f https://raw.githubusercontent.com/openshift/pipelines-tutorial/pipelines-1.9/03_triggers/02_template.yaml
$ oc create -f - <<EOF
apiVersion: triggers.tekton.dev/v1beta1
kind: Trigger
metadata:
  name: vote-trigger
spec:
  serviceAccountName: pipeline
  interceptors:
    - ref:
        name: "github"
      params:
        - name: "eventTypes"
          value: ["push"]
  bindings:
    - ref: vote-app
  template:
    ref: vote-app
EOF
$ oc create -f https://raw.githubusercontent.com/openshift/pipelines-tutorial/pipelines-1.9/03_triggers/04_event_listener.yaml

// Running a pipeline
$ tkn pipeline start build-and-deploy \
    -w name=shared-workspace,volumeClaimTemplateFile=https://raw.githubusercontent.com/openshift/pipelines-tutorial/pipelines-1.9/01_pipeline/03_persistent_volume_claim.yaml \
    -p deployment-name=pipelines-vote-ui \
    -p git-url=https://github.com/openshift/pipelines-vote-ui.git \
    -p IMAGE='image-registry.openshift-image-registry.svc:5000/$(context.pipelineRun.namespace)/pipelines-vote-ui' \
    --use-param-defaults

$ oc get pod -n pipelines-tutorial
NAME                                               READY   STATUS      RESTARTS   AGE
build-and-deploy-run-tbks8-apply-manifests-pod     0/1     Completed   0          3m38s
build-and-deploy-run-tbks8-build-image-pod         0/1     Completed   0          4m11s
build-and-deploy-run-tbks8-fetch-repository-pod    0/1     Completed   0          4m23s
build-and-deploy-run-tbks8-update-deployment-pod   0/1     Completed   0          3m24s
el-vote-app-68fcd9675f-hdw6j                       1/1     Running     0          10m
pipelines-vote-ui-676bccc85-m4rsp                  1/1     Running     0          3m20s
```

After confirming that the sample application **pipelines-vote-ui** is working well, replace the above GitHub repository with the one you have forked from the front end [pipelines-vote-ui](https://github.com/openshift/pipelines-vote-ui/tree/pipelines-1.9) for your tests (in my case, https://github.com/bysnupy/pipelines-vote-ui). I will apply a change to my repository for checking whether the pipeline kicks off or not in the following demonstration.

Let’s look at the details of the following resources to implement simple polling triggers with scripts. Simply comparing the previous and current revision sha256 hash values using the git command is the main logic to determine triggering the pipeline, and the latest sha256 hash would be stored in a PV volume for the next checks. The CronJob requests a push event using the curl command to an EventListener on behalf of the GitHub repository Webhook for triggering a pipeline run.

This approach allows us to reuse existing TriggerBinding and TriggerTemplate resources. Regardless, you can consider using [tkn cli](https://docs.openshift.com/container-platform/4.13/cli_reference/tkn_cli/op-tkn-reference.html) instead of the curl command in CronJob for your needs. The tkn cli can customize more than the curl command in your pipeline flow.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: repolist-pvc
  namespace: pipelines-tutorial
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: polling-triggers
  namespace: pipelines-tutorial
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: polling-triggers
            volumeMounts:
            - name: repolist
              mountPath: /repolist
            image: registry.redhat.io/rhel8/toolbox:latest
            env:
            - name: REPONAME
              value: pipelines-vote-ui
            - name: REPOBASEURL
              value: https://github.com/bysnupy
            - name: REPOBRANCH
              value: master
            - name: REPOURL
              value: https://github.com/bysnupy/pipelines-vote-ui
            - name: BASEDIR
              value: /repolist
            - name: EVENTLISTENERSVC
              value: "http://el-vote-app.pipelines-tutorial.svc.cluster.local:8080"
            - name: JSONTEMPLATE
              value: '{"object_kind": "push","event_name": "push","head_commit": {"id": "GITHUBREPOREV"},"repository": {"name": "GITHUBREPONAME","url": "GITHUBREPOURL"}}'
            command:
            - /bin/sh
            args:
            - -c
            - |
              set -eu

              # revision initialization
              _current_revision=$(git ls-remote --heads ${REPOBASEURL}/${REPONAME} ${REPOBRANCH} | awk '{print $1}')
              _prev_revision=${_current_revision}

              # if there is not existing previous revision data file, creating new revision data file with current revision
              test -f ${BASEDIR}/${REPONAME}.sha256 && _prev_revision=$(cat ${BASEDIR}/${REPONAME}.sha256) || echo ${_current_revision} > ${BASEDIR}/${REPONAME}.sha256

              # generating JSON data
              _jsondata=$(echo ${JSONTEMPLATE} | sed -e "s=GITHUBREPOREV=${_current_revision}=" -e "s=GITHUBREPONAME=${REPONAME}=" -e "s=GITHUBREPOURL=${REPOURL}=")

              # check if there are any changes through comparing previous and current revisions.
              # If there are any changes, trigger a new pipeline using curl and json data.
              test "${_current_revision}" != "${_prev_revision}" && echo ${_current_revision} > ${BASEDIR}/${REPONAME}.sha256 &&
              curl -s -X POST -H 'Content-Type: application/json' -H 'X-GitHub-Event: push' \
              -d "${_jsondata}" ${EVENTLISTENERSVC} ||
              echo "No changes"
          restartPolicy: Never
          volumes:
          - name: repolist
            persistentVolumeClaim:
              claimName: repolist-pvc
```
For more details of Github webhook HTTP POST payloads for modification of curl options, please refer to [Webhook events and payloads](https://docs.github.com/en/webhooks-and-events/webhooks/webhook-events-and-payloads).

## Triggering a Pipeline Run Using Polling Triggers
When a change event occurs in the Git repository, the aforementioned polling triggers CronJob checks it periodically. After becoming aware of new data, send an event payload (JSON data) to the privately exposed EventListener service using the curl command. The EventListener service of the application processes the payload and passes it to the relevant TriggerBinding and TriggerTemplate resource pairs. The TriggerBinding resource extracts the parameters and the TriggerTemplate resource uses these parameters and specifies the way the resources must be created. This may rebuild and redeploy the application.
In this section, you push an empty commit to the front-end pipelines-vote-ui repository, which then triggers the pipeline run.

```console
// Check the polling triggers pod logs before changes
$ oc get pod -n pipelines-tutorial
 NAME                                 READY   STATUS              RESTARTS   AGE
el-vote-app-68fcd9675f-hdw6j         1/1     Running             0          135m
pipelines-vote-ui-54fcb9658d-tkqjp   1/1     Running             0          9m29s
polling-triggers-28174084-hrdth      0/1     Completed           0          33s

$ oc logs -f polling-triggers-28174084-hrdth
No changes

// Clone your forked Git repository pipelines-vote-ui, and push an empty commit
$ git clone https://github.com/bysnupy/pipelines-vote-ui.git -b master
$ git commit -m "empty-commit" --allow-empty && git push origin master

// Check if the new pipeline run would be triggered by the polling triggers
$ oc get pod -n pipelines-tutorial
 NAME                                 READY   STATUS              RESTARTS   AGE
el-vote-app-68fcd9675f-hdw6j         1/1     Running             0          136m
pipelines-vote-ui-54fcb9658d-tkqjp   1/1     Running             0          9m59s
polling-triggers-28174084-hrdth      0/1     Completed           0          63s
polling-triggers-28174085-s8hzh      0/1     ContainerCreating   0          3s

$ oc logs -f polling-triggers-28174085-s8hzh
{"eventListener":"vote-app","namespace":"pipelines-tutorial","eventListenerUID":"b71d09d8-245f-4126-9c3b-53f399c7c24c","eventID":"169d6f94-29c1-4f65-92fe-1985d6faa3fb"}

$ oc get pod -n pipelines-tutorial
NAME                                                        READY   STATUS      RESTARTS   AGE
build-deploy-pipelines-vote-ui-8bxfv-fetch-repository-pod   0/1     Init:0/2    0          23s <--- New pipeline tasks started
el-vote-app-68fcd9675f-hdw6j                                1/1     Running     0          136m
pipelines-vote-ui-54fcb9658d-tkqjp                          1/1     Running     0          10m
polling-triggers-28174084-hrdth                             0/1     Completed   0          89s
polling-triggers-28174085-s8hzh                             0/1     Completed   0          29s
```

## Wrap Up
In this post, we have demonstrated the polling triggers using CronJob and verified how the process works. You can also customize the logic for your own use cases, for instance, directly running the pipeline without EventListener using the [tkn command](https://docs.openshift.com/container-platform/4.13/cli_reference/tkn_cli/op-tkn-reference.html). Using your own implementation,  you can do anything that meets your needs. I hope this has helped to deepen your knowledge of the OpenShift Pipeline.
