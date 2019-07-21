# Install HelloWorld Application

This tutorial walks you through an example to hot deploy a sample java application using advanced statefulset.
The sample java app used is from this [repo](https://tomcat.apache.org/tomcat-7.0-doc/appdev/sample/).

## Install the HelloWorld application with YAML files

Below creates a java application using advanced statefulset.

```
kubectl apply -f https://raw.githubusercontent.com/furykerry/kruise/sidecar-tutorial/docs/tutorial/v1/tomcat-sts-for-hotdeply-demo.yaml
kubectl apply -f https://raw.githubusercontent.com/furykerry/kruise/sidecar-tutorial/docs/tutorial/v1/tomcat-svc-with-samplewar.yaml
```

Several things to note in the `tomcat-sts-for-hotdeply-demo.yaml`

```yaml
      containers:
      - name: tomcat
        image: tomcat:8.0
        volumeMounts:
        # mount point of shared volume in tomcat
        - mountPath: /usr/local/tomcat/webapps/samplewar
          name: share-volume
        ports:
        - name: http-server
          containerPort: 3000
      - name: data
        image: openkruise/guestbook:warv1
        # keep data container always running 
        command: ["/bin/sh", "-c", "tail -f /dev/null"]
        lifecycle:
          postStart:
            exec:
              # cp .war from data container to tomcat container
              command:
              - "/bin/sh"
              - "-c"
              - "if [ ! -f .only-copy-once ]; then unzip -x  /webapps/sample.war -o -d /shared/webapps/samplewar; fi; touch .only-copy-once "
        volumeMounts:
         # mount point of shared volume in war container
        - mountPath: /shared/webapps/samplewar
          name: share-volume
      volumes:
      # shared volumes for tomcat and war data container
      - name: share-volume
        emptyDir: {}
```

Now the app has been installed.

## Verify HelloWorld Started

Check the HelloWorld apps are started. `statefulset.apps.kruise.io` or shortname `sts.apps.kruise.io` is the resource kind.
`app.kruise.io` postfix needs to be appended due to naming collision with Kubernetes native `statefulset` kind.
 Verify that all pods are READY.

```
kubectl get sts.apps.kruise.io

NAME                       DESIRED   CURRENT   UPDATED   READY   AGE
tomcat-with-samplewar      1         1         1         1       10m
```

## View the HelloWorld app

You can now view the HelloWorld apps on browser.

* **Local Host:**
    If you are running Kubernetes locally, to view the HelloWorld, navigate to `http://localhost:8080` for the hello world app

* **Remote Host:**
    To view the hello world apps on a remote host, locate the external IP of the application in the **IP** column of the `kubectl get services` output.
    For example, run

```
kubectl get svc

NAME                     TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                         AGE
tomcat-with-samplewar    LoadBalancer   172.28.9.5      47.111.36.134    8080:32024/TCP                  11m

```

`47.111.36.134` is the external IP.
Visit `http://47.111.36.134:8080/samplewar/index.html` for the HelloWorld page.

## hot update HelloWorld to the new image

First, check the running pods.

```
NAME                            READY   STATUS    RESTARTS   AGE     IP             NODE                        NOMINATED NODE
tomcat-with-samplewar-0         2/2     Running   1          18m     172.27.2.218   cn-hangzhou.192.168.0.198   <none>
```

Run this command to update the statefulset to use the new image.

```
kubectl edit sts.apps.kruise.io demo-v1-guestbook-kruise
```

What this command does is that it changes the image version to `v2` .The YAML diff details are shown below:

```yaml
spec:
    ...
      containers:
      - name: data
-       image: openkruise/guestbook:warv1
+       image: openkruise/guestbook:warv2
  podManagementPolicy: Parallel  # allow parallel updates, works together with maxUnavailable
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      # Do in-place update if possible, currently only image update is supported for in-place update
      podUpdatePolicy: InPlaceIfPossible
      # Allow parallel updates with max number of unavailable instances equals to 2
      maxUnavailable: 3
```

Check the statefulset, find the statefulset has 1 pods updated

```
kubectl get sts.apps.kruise.io

NAME                       DESIRED   CURRENT   UPDATED   READY   AGE
tomcat-with-samplewar      1         1         1         1       19m
```

Check the pods again. `tomcat-with-samplewar-0` are updated with `RESTARTS` showing `1`,
IPs remain the same, `CONTROLLER-REVISION-HASH` are updated from `tomcat-with-samplewar-7c947b5f94` to `tomcat-with-samplewar-756b69898c`

```
kubectl get pod -L controller-revision-hash -o wide | grep samplewar

NAME                            READY   STATUS    RESTARTS   AGE     IP             NODE                        NOMINATED NODE   CONTROLLER-REVISION-HASH
tomcat-with-samplewar-0         2/2     Running   1          20m     172.27.2.218   cn-hangzhou.192.168.0.198   <none>           tomcat-with-samplewar-756b69898c```
```

Now upgrade all the pods, run

Describe a pod and find that the events show the original container is killed and new container is started. This verifies `in-place` update

```
kubectl describe pod tomcat-with-samplewar-0

...
  Type    Reason     Age                From                                Message
  ----    ------     ----               ----                                -------
  Normal  Scheduled  24m                default-scheduler                   Successfully assigned default/tomcat-with-samplewar-0 to cn-hangzhou.192.168.0.198
  Normal  Pulling    24m                kubelet, cn-hangzhou.192.168.0.198  pulling image "tomcat:8.0"
  Normal  Pulled     24m                kubelet, cn-hangzhou.192.168.0.198  Successfully pulled image "tomcat:8.0"
  Normal  Created    24m                kubelet, cn-hangzhou.192.168.0.198  Created container
  Normal  Started    24m                kubelet, cn-hangzhou.192.168.0.198  Started container
  Normal  Pulling    24m                kubelet, cn-hangzhou.192.168.0.198  pulling image "openkruise/guestbook:warv1"
  Normal  Pulled     24m                kubelet, cn-hangzhou.192.168.0.198  Successfully pulled image "openkruise/guestbook:warv1"
  Normal  Killing    21m                kubelet, cn-hangzhou.192.168.0.198  Killing container with id docker://data:Container spec hash changed (933922600 vs 1244952363).. Container will be killed and recreated.
  Normal  Pulling    21m                kubelet, cn-hangzhou.192.168.0.198  pulling image "openkruise/guestbook:warv2"
  Normal  Created    21m (x2 over 24m)  kubelet, cn-hangzhou.192.168.0.198  Created container
  Normal  Started    21m (x2 over 24m)  kubelet, cn-hangzhou.192.168.0.198  Started container
  Normal  Pulled     21m                kubelet, cn-hangzhou.192.168.0.198  Successfully pulled image "openkruise/guestbook:warv2"
```

The pods should also be in `Ready` state, the `InPlaceUpdateReady` will be set to `False` right before in-place update and to `True` after update is complete

```yaml
Readiness Gates:
  Type                 Status
  InPlaceUpdateReady   True
Conditions:
  Type                 Status
  InPlaceUpdateReady   True  # Should be False right before in-place update and True after update is complete
  Initialized          True
  Ready                True  # Should be True after in-place update is complete
  ContainersReady      True
  PodScheduled         True
```

## Uninstall app


deleting the application using below commands:

```
kubectl delete sts.apps.kruise.io tomcat-with-samplewar
kubectl delete svc tomcat-with-samplewar
```
