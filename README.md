# VRspace

環境


# 1. LET's Encrypt

wget https://raw.githubusercontent.com/tnozicka/openshift-acme/master/deploy/cluster-wide/clusterrole.yaml

wget https://raw.githubusercontent.com/tnozicka/openshift-acme/master/deploy/cluster-wide/serviceaccount.yaml

wget https://raw.githubusercontent.com/tnozicka/openshift-acme/master/deploy/cluster-wide/issuer-letsencrypt-live.yaml

wget https://raw.githubusercontent.com/tnozicka/openshift-acme/master/deploy/cluster-wide/deployment.yaml

kubectl create -f clusterrole.yaml

kubectl create -f serviceaccount.yaml

kubectl create -f issuer-letsencrypt-live.yaml

kubectl create -f deployment.yaml


# OpenVidu

ビデオ会議や音声チャットなどを有効にするためにはOpenViduの構築が必要です。 まず、以下の環境変数をセットします。

export OPENVIDU_USERNAME=<任意のアカウント名>
export OPENVIDU_SERVER_SECRET=<任意のパスワード>
kubectl create namespace openvidu
kubectl create sa openvidu -n openvidu
kubectl create clusterrolebinding openvidu-cluster-admin --clusterrole=cluster-admin --serviceaccount=openvidu:openvidu
kubectl apply -f manifest/openvidu/redis-deployment.yaml -n openvidu
kubectl apply -f manifest/openvidu/coturn-service.yaml -n openvidu


coturnはtype:LoadBalancerのServiceを使用します。 AWS上にdeployする場合、NLBが新規作成され、DNSレコードが更新されるまで1分ほどかかります。

以下の環境変数TURNIPが取得できるまで待ちます。

export TURN_DOMAIN=$(kubectl get svc coturn -o jsonpath='{.status.loadBalancer.ingress[*].hostname}')

export TURNIP=$(kubectl get svc coturn -o jsonpath='{.status.loadBalancer.ingress[*].ip}')


export MAILADDR=<your mail address>

export BASE_DOMAIN=<your base domain>

export STUN_LIST=$(echo "$TURNIP:3489" | base64)

export TURN_LIST=$(echo "${OPENVIDU_USERNAME}:${OPENVIDU_SERVER_SECRET}@$TURNIP:3489" | base64)


gomplate -f manifest/openvidu/coturn-deployment.yaml | envsubst | kubectl apply -f -
gomplate -f manifest/openvidu/kms-deployment.yaml | envsubst | kubectl apply -f -
gomplate -f manifest/openvidu/openvidu-server-deployment.yaml | envsubst | kubectl apply -f -

gomplate -f manifest/openvidu/openvidu-server-route.yaml | envsubst | kubectl apply -f -

export OPENVIDU_SERVER_URL=$(kubectl get route openvidu-server -o jsonpath='{.status.ingress[*].host}')


# VRSpace

export VRSPACE_GIT_URL=https://github.com/yd-ono/vrspace

export VRSPACE_SERVER_URL=vrspace.${BASE_DOMAIN}

kubectl create sa vrspace
kubectl adm policy add-scc-to-user anyuid -z vrspace
kubectl adm policy add-scc-to-user privileged -z vrspace


gomplate -f manifest/vrspace/Dockerfile | envsubst | docker build -t vrspace -

gomplate -f manifest/vrspace/vrspace-deployment.yaml | envsubst | kubectl apply -f -


cat manifest/vrspace/vrspace-route.yaml | envsubst | kubectl apply -f -


https://${VRSPACE_SERVER_URL}/babylon/avatar-selection.html へアクセスします。
