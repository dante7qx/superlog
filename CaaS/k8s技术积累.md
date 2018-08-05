## k8s 技术积累

### 一. Secrets

​	k8s Secret 用来存储少量敏感的数据，例如数据库、Gitlab的用户、密码。这样可以不在 Image 中去保留敏感信息，并且可以在多个 Pod 中共享这些 Secret。因为 k8s 使用 JWT 的方式进行加密，所以需要加密的数据必须先进行 base64 编码。参考 https://kubernetes.io/docs/concepts/configuration/secret

```shell
## 编码
echo -n "root" | base64			# => cm9vdA==
echo -n "123456" | base64		# => MTIzNDU2
## 解码
echo "cm9vdA==" | base64 -D
echo "MTIzNDU2" | base64 -D
```
- ***kubectl create secret.yaml***
```yaml
## secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlabsecret
  namespace: dante
type: Opaque
data:
  username: cm9vdA==
  password: MTIzNDU2

## data 是 map
## key 字母、数字或 _ - .
## value 必须用 base64 编码
```

- **在 Pod 中使用**
  - 环境变量

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dante-healthcheck
  namespace: dante
spec:
  containers:
  - name: dante-boot-docker
    image: dante/springboot-docker
    imagePullPolicy: Never
    ports:
    - containerPort: 8080
    livenessProbe:
      httpGet:
        path: /docker
        port: 8080
      initialDelaySeconds: 30
      timeoutSeconds: 4
    env:
      - name: hello_msg
        valueFrom:
          secretKeyRef:
            name: gitlabsecret
            key: username
      - name: hello_x_info
        valueFrom:
          secretKeyRef:
            name: gitlabsecret
            key: password
```

