### Déploiement de NGINX Kubernetes Ingress Controller

On va déployer NGINX Kubernetes Ingress Controller avec le chart Helm officiel.

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

```
helm install my-ingress-nginx ingress-nginx/ingress-nginx --version 4.2.0 
```

Sous Openshift, ajouter un RoleBinding sur scc-privileged pour le ServiceAccount de NGINX et des Jobs de l'Admission Webhook (qui vont générer les certificats pour le webhook).

Ensuite, ajouter une Route Openshift (dans le Namespace de l'ingress-nginx) qui va correspondre à l'URL de Kibana (ici : cafe.example.com) :

```yaml
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
    app.kubernetes.io/version: 1.3.0
spec:
  host: cafe.example.com
  to:
    kind: Service
    name: ingress-nginx-controller
    weight: 100
  port:
    targetPort: https
  tls:
    termination: passthrough
    insecureEdgeTerminationPolicy: Redirect
  wildcardPolicy: None

```

### Déploiement de oauth2-proxy (NameSpace de Kibana)

On va déployer oauth2-proxy avec le chart Helm officiel.

```
helm repo add oauth2-proxy https://oauth2-proxy.github.io/manifests
```

Créer un fichier *oauth2-proxy-values.yaml* pour ajuster la configuration :

```yaml
#Compte-tenu de la taille potention des Tokens Keycloak, il est conseillé d'utiliser REDIS plutôt qu'un sessionStorage de type
# cookie (cf. documentation oauth2-proxy : 
# https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/overview#configuring-for-use-with-the-nginx-auth_request-directive)
sessionStorage:
  type: redis

redis:
  enabled: true
  
extraArgs:
  # il faut utiliser le provider oidc et non keycloak d'oauth2-proxy. En effet, le provider keycloak de transmet pas
  # correctement le Prefered_Username (requis pour l'Ingress Kibana / NGINX).
  provider: 'oidc'
  client-id: 'kibana-oauth2-proxy'
  client-secret: 'M9tHvjJhbfShvGBVMX9Q54Pe9VcFcKvN'
  oidc-issuer-url: 'https://keycloak-keycloak.apps-crc.testing/auth/realms/master'

  # éventuellement SSL insecure pour phase de test. Puis gérer TLS de bout en bout.
  ssl-insecure-skip-verify: true
  
  # ajuster la redirect-url
  redirect-url: https://cafe.example.com/oauth2/callback

  # pour générer cookie-secret: https://oauth2-proxy.github.io/oauth2-proxy/docs/configuration/overview#generating-a-cookie-secret
  # python -c 'import os,base64; print(base64.urlsafe_b64encode(os.urandom(32)).decode())'
  cookie-secret: 'HilAW97PR7CCZDYOQpIQnqgS91WfKVVyCNmBSs5Wfpw='
  # ajuster le cookie-domain
  cookie-domain: 'cafe.example.com'
  # ajuster le whitelist-domain (ici il faut celui de Keycloak qui sera appelé dans l'URL de logout de Kibana - cf Ingress Kibana).
  whitelist-domain: 'keycloak-keycloak.apps-crc.testing'
  cookie-samesite: none
  cookie-secure: true

  set-xauthrequest: true

  session-store-type: redis
  # ajuster l'URL de REDIS
  redis-connection-url: 'redis://my-oauth2-proxy-redis-master.eck.svc.cluster.local:6379/0'
  
  request-logging: true
  auth-logging: true
  standard-logging: true
  silence-ping-logging: true
```

Puis déployer avec Helm :

```
helm install \
        my-oauth2-proxy oauth2-proxy/oauth2-proxy \
        --version 6.2.2 \
        --values oauth2-proxy-values.yaml
```

### **KIBANA - configuration pour oauth2-proxy**

Les paramètres suivants sont requis pour le bon fonctionnement du login avec oauth2-proxy :

```yaml
spec:
  config:
    elasticsearch.requestHeadersWhitelist:
      - es-security-runas-user
      - authorization
    xpack.monitoring.elasticsearch.requestHeadersWhitelist:
      - es-security-runas-user
      - authorization
```

Il faut commencer par déployer Kibana avec l'opérateur ECK :

```yaml
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: kibana-sample
  namespace: eck
spec:
  config:
    elasticsearch.requestHeadersWhitelist:
      - es-security-runas-user
      - authorization
    xpack.monitoring.elasticsearch.requestHeadersWhitelist:
      - es-security-runas-user
      - authorization
  count: 1
  elasticsearchRef:
    name: elasticsearch-sample
  enterpriseSearchRef: {}
  http:
    service:
      metadata: {}
      spec: {}
    tls:
      selfSignedCertificate:
        disabled: true
  monitoring:
    logs: {}
    metrics: {}
  podTemplate:
    spec:
      containers:
        - name: kibana
          resources:
            limits:
              cpu: 1
              memory: 4Gi
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
  version: 8.3.1


```

Puis déployer deux objets Ingress pour Kibana et pour oauth2-proxy (requis).

Il faut d'abord générer le Base64 *user:password* de l'utilisateur *nginx* créé dans Kibana. Puis insérer cette valeur ici :

```
proxy_set_header Authorization "Basic <Base64 de user:password>";
```

Exemple de manifest Ingress de Kibana :

```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: kibana-ingress
  namespace: eck
  annotations:
    nginx.ingress.kubernetes.io/auth-response-headers: X-Auth-Request-Preferred-Username
    nginx.ingress.kubernetes.io/auth-signin: 'https://$host/oauth2/start?rd=$escaped_request_uri'
    # modifier auth-url par l'url de l'oauth2-proxy. Celle-ci doit être 
    # atteignable par le pod nginx-ingress-controller qui va contrôler 
    # qu'il peut atteindre cette url.
    nginx.ingress.kubernetes.io/auth-url: 'http://my-oauth2-proxy.eck.svc.cluster.local/oauth2/auth'
    # modifier les url de logout. Attention : Keycloak va contrôler l'url
    # de redirection (legacy-logout-redirect-uri), et valider le SSL.
    # Vérifier donc que le TrustStore de Keycloak contient le CA de
    # cette URL.
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header 'es-security-runas-user' $authHeader0;
      proxy_set_header Authorization "Basic bmdpbng6c2VjcmV0cGFzc3dvcmQ=";
      location /logout {
          rewrite ^/logout(.*)$ /oauth2/sign_out?rd=https%3A%2F%2Fkeycloak-keycloak.apps-crc.testing%2Fauth%2Frealms%2Fmaster%2Fprotocol%2Fopenid-connect%2Flogout?https://keycloak-keycloak.apps-crc.testing/auth/realms/master/protocol/openid-connect/logout?legacy-logout-redirect-uri=https%3A%2F%2Fcafe.example.com redirect;
      }
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - cafe.example.com
      secretName: cafe-secret
  rules:
    - host: cafe.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: kibana-sample-kb-http
                port:
                  number: 5601

```

Exemple de manifest Ingress de oauth2-proxy :

```yaml
kind: Ingress
apiVersion: networking.k8s.io/v1
metadata:
  name: kibana-oauth2-proxy
  namespace: eck
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - cafe.example.com
      secretName: cafe-secret
  rules:
    - host: cafe.example.com
      http:
        paths:
          - path: /oauth2
            pathType: Prefix
            backend:
              service:
                name: my-oauth2-proxy
                port:
                  number: 80

```

Se connecter avec Kibana avec la route habituelle créée par l'ECK (hors oauth2-proxy), puis utiliser le username *elastic* pour :

- Dans Stack Management - Sécurity, créer un utilisateur qui aura l'autorisation de se connecter à Kibana (exemple : *developer*). Ajuster ses privilèges.

- Puis créer un rôle *nginx* et ajouter l'utilisateur *developer* dans Run As privileges. Il n'est pas nécessaire d'accorder d'autres droits à ce rôle.

-  Créer un utilisateur *nginx*, Full name *Service Account*, avec les Privileges du rôle *nginx*. Créer un mot de passe pour cet utilisateur. Ajuster alors dans l'Ingress Kibana la valeur de :
  
  ```
  proxy_set_header Authorization "Basic <Base64 de user:password>";
  ```




