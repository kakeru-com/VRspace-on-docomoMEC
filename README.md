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

export TURN_DOMAIN=`oc get svc coturn -o jsonpath='{.status.loadBalancer.ingress[*].hostname}'`
## Azureをしようしている場合はexport TURNIP=`oc get svc coturn -o jsonpath='{.status.loadBalancer.ingress[*].ip}'`のみ
export TURNIP=`dig $TURN_DOMAIN | grep -v ";" | grep $TURN_DOMAIN  | awk '{print $5}'`
TURNIPをセットできたら、以下の環境変数もセットします。

export MAILADDR=<your mail address>
export BASE_DOMAIN=<your base domain>
export STUN_LIST=`echo $TURNIP:3489 | base64`
export TURN_LIST=`echo ${OPENVIDU_USERNAME}:${OPENVIDU_SERVER_SECRET}@$TURNIP:3489 | base64`
coturn、kurento、openvidu-serverをdeployします。

gomplate -f manifest/openvidu/coturn-deployment.yaml | envsubst | oc apply -f -
gomplate -f manifest/openvidu/kms-deployment.yaml | envsubst | oc apply -f -
gomplate -f manifest/openvidu/openvidu-server-deployment.yaml | envsubst | oc apply -f -
Routeを作成します。

gomplate -f manifest/openvidu/openvidu-server-route.yaml | envsubst | oc apply -f -
export OPENVIDU_SERVER_URL=`oc get route openvidu-server -o jsonpath='{.status.ingress[*].host}'`
https://${OPENVIDU_SERVER_URL}へアクセスし、TURNサーバ経由でビデオ配信ができることを確認してください。

VRSpace

GitHubをforkした https://github.com/yd-ono/vrspace を用いてコンテナイメージを作成します。 オリジナルはlocalhostで実行する前提になっており、以下の箇所を修正しています。

vrspace/babylon/video-test.js

OPENVIDU_SERVER_URLとOPENVIDU_SERVER_SECRETを修正
vrspace/babylon/sound-test.js

OPENVIDU_SERVER_URLとOPENVIDU_SERVER_SECRETを修正
vrspace/server/src/main/resources/application.propertiesの

openvidu.publicurl
openvidu.secret
sketchfab.redirectUri
spring.security.oauth2.client.registration.facebook.redirect-uri
spring.security.oauth2.client.registration.google.redirect-uriを修正
自分のGitHubへforkした場合は以下の環境変数を変更してください。

export VRSPACE_GIT_URL=<forkしたリポジトリのURL>
forkしていない場合は、以下の通り環境変数を設定してください。

export VRSPACE_GIT_URL=https://github.com/yd-ono/vrspace
続いて、以下の環境変数を設定します。

export VRSPACE_SERVER_URL=vrspace.${BASE_DOMAIN}
あとは、OpenShiftでコンテナイメージをBuildしてデプロイするだけです。 もちろん、ローカルのPCなどでBuildしても良いです。

Buildする前に以下の通りService Accountを作っておきましょう。

oc new-project vrspace
oc create sa vrspace
oc adm policy add-scc-to-user anyuid -z vrspace
oc adm policy add-scc-to-user privileged -z vrspace
※SCCは上記で設定しないと動きませんでした…セキュリティレベルが物凄く低下するので他の方法を調査中です…

Buildします。

gomplate -f manifest/vrspace/Dockerfile | envsubst | oc new-build --dockerfile=- --to=vrspace -
最後に、DeployしてRouteを設定します。

gomplate -f manifest/vrspace/vrspace-deployment.yaml | envsubst | oc apply -f -
cat manifest/vrspace/vrspace-route.yaml | envsubst | oc apply -f -
https://${VRSPACE_SERVER_URL}/babylon/avatar-selection.html へアクセスします。
