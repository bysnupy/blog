# CHAP Secret YAML sample

## sample
~~~
apiVersion: v1
data:
  node.session.auth.password: xxxBASE64xxx
  node.session.auth.username: xxxBASE64xxx
kind: Secret
metadata:
  name: chap-for-iscsi
  namespace: default
type: kubernetes.io/iscsi-chap
~~~
