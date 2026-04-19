# Longhorn Installation

[Reference DOcumentation](https://longhorn.io/docs/)

## Prerequisite

'''bash
apt-get install -y open-iscsi nfs-common
'''

## Install with Helm

'''bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn \
    --namespace longhorn-system \
    --create-namespace \
    --version 1.11.1
'''


### Confirm installation

'''bash
kubectl -n longhorn-system get pod
'''

## Customization

'''bash
kubectl -n longhorn-system apply -f longhorn.yaml
'''
