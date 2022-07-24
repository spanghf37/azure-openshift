Ajouter une interface réseau secondaire vers IPFIRE

Installer l'opérateur NMSTATE sur Openshift, et déployer une instance NMSTATE.

Puis appliquer une `NodeNetworkConfigurationPolicy` :

```yaml
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: enp5s0-additional-static-route-policy
spec:
  nodeSelector: 
    node-role.kubernetes.io/master: "" 
  maxUnavailable: 1
  desiredState:
    interfaces:
    - name: enp5s0
      description: Static routing on enp5s0
      type: ethernet
      state: up
      ipv4:
        address:
        - ip: 192.168.1.101
          prefix-length: 24
        enabled: true  
    routes:
      config:
      - destination: 10.0.0.0/16
        metric: 150
        next-hop-address: 192.168.1.253 #gateway IPFIRE RED
        next-hop-interface: enp5s0
        table-id: 254
# pas possible de le faire sur br-ex car nmstate ne peut ajuster de static route sur une interface adressée dynamiquement.
# il faut une interface secondaire configurée statiquement via nmstate.
```
