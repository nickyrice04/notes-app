# Lab 5 Evidence

Nicky Rice, MIS 547

This file has the evidence for Lab 5. The screenshots are saved in the `evidence` folder, and every block of output below was copied from my terminal while the kind cluster was running.

## GitHub Actions run

Below is a successful run of the Build and publish image workflow. It was triggered by a push to main and finished in 42 seconds.

![Successful GitHub Actions run](evidence/actions-run.png)

## Docker Hub tags

Below is the tags page for nickyrice/notes-app on Docker Hub. From the screenshot we can see that both the sha tag and latest have a linux/amd64 and a linux/arm64 image. This is important because my Mac is Apple Silicon, so the kind node needs the arm64 version.

![Docker Hub tags page showing amd64 and arm64](evidence/dockerhub-tags.png)

## kubectl get all,pvc

Below is the output of `kubectl get all,pvc` from my running cluster. It shows 1 db pod, 2 web pods, both Services, and the PVC in the Bound state. The web ReplicaSet with 0 pods is left over from the rolling update in Experiment 4.

```
NAME                      READY   STATUS    RESTARTS   AGE
pod/db-6c5c8947cd-n9lgk   1/1     Running   0          4m30s
pod/web-cbf7bb59c-hkffq   1/1     Running   0          2m3s
pod/web-cbf7bb59c-s9jjm   1/1     Running   0          2m10s

NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/db    ClusterIP   10.96.116.43    <none>        5432/TCP   5m46s
service/web   ClusterIP   10.96.255.104   <none>        80/TCP     5m24s

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/db    1/1     1            1           5m46s
deployment.apps/web   2/2     2            2           5m24s

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/db-6c5c8947cd    1         1         1       5m46s
replicaset.apps/web-68dbbfdcbd   0         0         0       2m35s
replicaset.apps/web-cbf7bb59c    2         2         2       5m24s

NAME                            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/db-data   Bound    pvc-8013923e-31e6-4007-8363-8b5d3182b3d0   1Gi        RWO            standard       <unset>                 5m46s
```

## Experiment 2 - Data persistence

First I created a note through the port-forward.

```
$ curl -X POST http://localhost:8000/notes -H "Content-Type: application/json" -d '{"body": "I should survive a pod deletion"}'
{"body":"I should survive a pod deletion","id":2}
```

Then I deleted the database pod and waited for the new one to be ready.

```
$ kubectl delete pod -l app=db
pod "db-6c5c8947cd-6f24k" deleted from notes-lab namespace
$ kubectl get pods -l app=db
NAME                  READY   STATUS    RESTARTS   AGE
db-6c5c8947cd-n9lgk   1/1     Running   0          6s
```

After that I listed the notes again.

```
$ curl http://localhost:8000/notes
[{"body":"hello from kubernetes","created_at":"2026-09-27T14:24:54.626807+00:00","id":1},{"body":"I should survive a pod deletion","created_at":"2026-09-27T14:25:17.239446+00:00","id":2}]
```

Both notes are still there, even though the pod that saved them is gone. The reason is that Postgres writes its data to the db-data PersistentVolumeClaim, not to the pod itself. When Kubernetes made the new db pod, it mounted that same volume, so the data came right back.

## Experiment 3 - Load balancing across replicas

For this one I scaled web up to 4 replicas and called the Service 8 times from inside the cluster. When I used `--rm -it` like the lab shows, the terminal attached late and cut off the first few responses. Because of this, I ran the curl pod without attaching and then read its logs, which gave me all 8.

```
$ kubectl scale deployment web --replicas=4
$ kubectl run curl --restart=Never --image=curlimages/curl -- sh -c 'for i in 1 2 3 4 5 6 7 8; do curl -s http://web/; echo; done'
$ kubectl logs curl
{"message":"Hello from the notes app, now on Kubernetes!","served_by":"web-cbf7bb59c-hkffq","service":"notes-app"}
{"message":"Hello from the notes app, now on Kubernetes!","served_by":"web-cbf7bb59c-s9jjm","service":"notes-app"}
{"message":"Hello from the notes app, now on Kubernetes!","served_by":"web-cbf7bb59c-hkffq","service":"notes-app"}
{"message":"Hello from the notes app, now on Kubernetes!","served_by":"web-cbf7bb59c-8jdd4","service":"notes-app"}
{"message":"Hello from the notes app, now on Kubernetes!","served_by":"web-cbf7bb59c-hkffq","service":"notes-app"}
{"message":"Hello from the notes app, now on Kubernetes!","served_by":"web-cbf7bb59c-hkffq","service":"notes-app"}
{"message":"Hello from the notes app, now on Kubernetes!","served_by":"web-cbf7bb59c-8jdd4","service":"notes-app"}
{"message":"Hello from the notes app, now on Kubernetes!","served_by":"web-cbf7bb59c-8jdd4","service":"notes-app"}
```

From the output we can see 3 different served_by values, that being hkffq, s9jjm, and 8jdd4. The Service picks a pod at random for each request, so with only 8 requests one of the 4 pods happened to not get any. After this I scaled web back down to 2.

## Rollout history

Below is the output of `kubectl rollout history deployment/web` right after the rolling update to the `sha-c751beb` image.

```
deployment.apps/web
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

Revision 1 is the original `latest` rollout and revision 2 is the sha tag. After running `kubectl rollout undo`, the history looks like this.

```
deployment.apps/web
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```

Revision 1 turned into revision 3, because a rollback is really just a new rollout of the old settings. Important to note is that the greeting did not change back after the rollback. That's because my push had also moved latest to the same build as sha-c751beb. So even though the Deployment went back to latest, latest was already the new code. This is a good example of why latest is a moving target and the sha tag is the one that actually pins a version.
