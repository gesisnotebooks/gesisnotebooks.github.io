# Troubleshooting

### Calico pod is not ready - not able to provision new pods

A recurring issue on the cluster leads to the loopback interface to go down on `spko-css-app03`(worker node) server. There is an [open issue](https://github.com/projectcalico/calico/issues/4257) on the calico issue tracker about this.

If there is a sudden drop in number of pods running on the cluster this issue is usually the first suspect.

To check if this is the issue, first check the status of the calico pod running on `spko-css-app03` using the following command on `svko-ilcm03`(Control Plane node).
```
$ kubectl get pods -n kube-system -o wide | grep calico
```

The output will either show the output to be `Running` but not ready with several restarts or in `CrashLoopBackOff` state.

The next step is to go to `spko-css-app03` and bring up the loopback interface.

- On `spko-css-app03` run `ip address show lo`, the state of the loopback device should be `DOWN`.
- (For a sanity check) ping the server itself, it should fail.
- The calicoctl process should be down.
```
$ sudo calicoctl node status
Calico process is not running.
```

To fix this issue:
- on `spko-css-app03`: enable loopback interface `ip link set dev lo up` and use `ip address show lo` to check the state again, it should show `UNKNOWN`.
- on `spko-css-app03`: pinging the server should work now.
- on `svko-ilcm03`: Delete the calico pod which is running on `spko-css-app03` to restart it.
```
$ kubectl get pods -n kube-system -o name --no-headers --selector  \
k8s-app=calico-node --field-selector spec.nodeName=spko-css-app03  \
| xargs kubectl delete -n kube-system
```
will fetch the calico pod running on `spko-css-app03` and delete it.
- on `svko-ilcm03`: The new calico pod on `spko-css-app03` will be restarted, to check the progress
```
$ kubectl get pods -n kube-system -o wide --selector k8s-app=calico-node --watch
```
- on `spko-css-app03`: Now the status of calico on the worker node should be up again.
```
$ sudo calicoctl node status
Calico process is running.

IPv4 BGP status
+-----------------+-------------------+-------+----------+-------------+
| PEER ADDRESS    |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+-----------------+-------------------+-------+----------+-------------+
| XXX.XXX.XXX.XXX | node-to-node mesh | up    | 05:43:39 | Established |
| XXX.XXX.XXX.XXX | node-to-node mesh | up    | 05:43:41 | Established |
+-----------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```