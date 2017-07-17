dnf install -y vim git etcd flannel docker wget


cd /root
wget https://kojipkgs.fedoraproject.org//packages/kubernetes/1.6.7/1.fc26/x86_64/kubernetes-client-1.6.7-1.fc26.x86_64.rpm
wget https://kojipkgs.fedoraproject.org//packages/kubernetes/1.6.7/1.fc26/x86_64/kubernetes-master-1.6.7-1.fc26.x86_64.rpm
wget https://kojipkgs.fedoraproject.org//packages/kubernetes/1.6.7/1.fc26/x86_64/kubernetes-node-1.6.7-1.fc26.x86_64.rpm

dnf install -y ./kubernetes-*

systemctl start etcd

FLANNEL_JSON=/etc/sysconfig/flannel-network.json
FLANNEL_NETWORK_CIDR="10.100.0.0/16"
FLANNEL_NETWORK_SUBNETLEN="24"
FLANNEL_BACKEND="vxlan"
PORTAL_NETWORK_CIDR="10.254.0.0/16"
cat > $FLANNEL_JSON <<EOF
{
  "Network": "$FLANNEL_NETWORK_CIDR",
  "Subnetlen": $FLANNEL_NETWORK_SUBNETLEN,
  "Backend": {
    "Type": "$FLANNEL_BACKEND"
  }
}
EOF

sed -i '/^FLANNEL_ETCD_ENDPOINTS=/ s/=.*/="http:\/\/127.0.0.1:2379"/' /etc/sysconfig/flanneld

FLANNELD_CONFIG=/etc/sysconfig/flanneld
FLANNEL_OPTIONS=""
sed -i '/FLANNEL_OPTIONS/'d $FLANNELD_CONFIG
cat >> $FLANNELD_CONFIG <<EOF
FLANNEL_OPTIONS="$FLANNEL_OPTIONS"
EOF
. $FLANNELD_CONFIG
curl -sf -L $ETCD_CURL_OPTIONS $FLANNEL_ETCD_ENDPOINTS/v2/keys${FLANNEL_ETCD_PREFIX}/config -X PUT --data-urlencode value@${FLANNEL_JSON}

mkdir -p /run/flannel/
cat > /run/flannel/docker <<EOF2
DOCKER_NETWORK_OPTIONS="--bip=\$FLANNEL_SUBNET --mtu=\$FLANNEL_MTU"
EOF2

systemctl start flanneld
systemctl start docker

cert_dir=/srv/kubernetes
mkdir -p ${cert_dir}
cd ${cert_dir}

KUBE_SERVICE_IP=$(echo $PORTAL_NETWORK_CIDR | awk 'BEGIN{FS="[./]"; OFS="."}{print $1,$2,$3,$4 + 1}')
MASTER_IP=137.138.7.21

# reference: https://coreos.com/kubernetes/docs/latest/openssl.html
cat << EOF > openssl.cnf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
[req_distinguished_name]
[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = ${KUBE_SERVICE_IP}
IP.2 = ${MASTER_IP}
EOF

openssl genrsa -out ca-key.pem 2048
openssl req -x509 -new -nodes -key ca-key.pem -days 10000 -out ca.pem -subj "/CN=kube-ca"

openssl genrsa -out apiserver-key.pem 2048
openssl req -new -key apiserver-key.pem -out apiserver.csr -subj "/CN=kube-apiserver" -config openssl.cnf
openssl x509 -req -in apiserver.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out apiserver.pem -days 365 -extensions v3_req -extfile openssl.cnf


groupadd kube_etcd
usermod -a -G kube_etcd etcd
usermod -a -G kube_etcd kube
chmod 550 "${cert_dir}"
chown -R kube:kube_etcd "${cert_dir}"


KUBE_API_ARGS="--tls-cert-file=/srv/kubernetes/apiserver.pem --tls-private-key-file=apiserver-key.pem --client-ca-file=/srv/kubernetes/ca.pem"
sed -i '/^KUBE_API_ARGS=/ s#\(KUBE_API_ARGS\).*#\1="'"${KUBE_API_ARGS}"'"#' /etc/kubernetes/apiserver

KUBE_CONTROLLER_MANAGER_ARGS="--service-account-private-key-file=/srv/kubernetes/apiserver-key.pem --root-ca-file=/srv/kubernetes/ca.pem"
sed -i '/^KUBE_CONTROLLER_MANAGER_ARGS=/ s#\(KUBE_CONTROLLER_MANAGER_ARGS\).*#\1="'"${KUBE_CONTROLLER_MANAGER_ARGS}"'"#' /etc/kubernetes/controller-manager


systemctl start kubelet
systemctl start kube-proxy
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler



DNS_CLUSTER_DOMAIN=cluster.local
DNS_SERVICE_IP=10.254.0.10

CORE_DNS=/root/kube-coredns.yaml
[ -f ${CORE_DNS} ] || {
    echo "Writing File: $CORE_DNS"
    mkdir -p $(dirname ${CORE_DNS})
    cat << EOF > ${CORE_DNS}
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        log stdout
        health
        kubernetes ${DNS_CLUSTER_DOMAIN} {
          cidrs ${PORTAL_NETWORK_CIDR}
        }
        proxy . /etc/resolv.conf
        cache 30
    }
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: coredns
  template:
    metadata:
      labels:
        k8s-app: coredns
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
        scheduler.alpha.kubernetes.io/tolerations: '[{"key":"CriticalAddonsOnly", "operator":"Exists"}]'
    spec:
      containers:
      - name: coredns
        image: coredns/coredns:007
        imagePullPolicy: Always
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: coredns
  clusterIP: ${DNS_SERVICE_IP}
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
EOF
}

kubectl create -f /root/kube-coredns.yaml --validate=false

kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml
