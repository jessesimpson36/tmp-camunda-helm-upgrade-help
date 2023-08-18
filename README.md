# Setup 8.1.4

Refer to 17876-upgrade.yaml and useingressnginx.yaml

```
helm install -f values.yaml \
             -f useingressnginx.yaml \
             cpt camunda/camunda-platform \
             --version 8.1.4
```

Referring to:

https://docs.camunda.io/docs/self-managed/platform-deployment/helm-kubernetes/upgrade/#v82

## Perform secrets extraction step

Run

```
export TASKLIST_SECRET=$(kubectl get secret "cpt-tasklist-identity-secret" -o jsonpath="{.data.tasklist-secret}" | base64 --decode)
export OPTIMIZE_SECRET=$(kubectl get secret "cpt-optimize-identity-secret" -o jsonpath="{.data.optimize-secret}" | base64 --decode)
export OPERATE_SECRET=$(kubectl get secret "cpt-operate-identity-secret" -o jsonpath="{.data.operate-secret}" | base64 --decode)
export CONNECTORS_SECRET=$(kubectl get secret "cpt-connectors-identity-secret" -o jsonpath="{.data.connectors-secret}" | base64 --decode)
export KEYCLOAK_ADMIN_SECRET=$(kubectl get secret "cpt-keycloak" -o jsonpath="{.data.admin-password}" | base64 --decode)
export KEYCLOAK_MANAGEMENT_SECRET=$(kubectl get secret "cpt-keycloak" -o jsonpath="{.data.management-password}" | base64 --decode)
export POSTGRESQL_SECRET=$(kubectl get secret "cpt-postgresql" -o jsonpath="{.data.postgres-password}" | base64 --decode)
export ZEEBE_SECRET=$(kubectl get secret --namespace "default" "cpt-zeebe-identity-secret" -o jsonpath="{.data.zeebe-secret}" | base64 -d)

```



## Perform Connectors step listed https://docs.camunda.io/docs/self-managed/platform-deployment/helm-kubernetes/upgrade/#v82

```
helm template cpt camunda/camunda-platform --version 8.2 --show-only charts/identity/templates/connectors-secret.yaml  > identity-connectors-secret.yaml

kubectl apply --namespace default -f identity-connectors-secret.yaml
```


## Keycloak steps exactly listed here: https://docs.camunda.io/docs/self-managed/platform-deployment/helm-kubernetes/upgrade/#keycloak


## Helm upgrade command


One possible complication is that ZEEBE_SECRET is not defined in 8.1.4. 
I also see that there are inconsistent references to POSTGRESQL_SECRET vs
POSTGRES_PASSWORD. Helm upgrade command recommends

```
export POSTGRES_PASSWORD=$(kubectl get secret --namespace "default" cpt-postgresql-web-modeler -o jsonpath="{.data.postgres-password}" | base64 -d)
```


I had to add the following lines to my upgrade command:

```
  --set global.postgresql.auth.postgresPassword=$POSTGRESQL_SECRET \
  --set global.postgresql.auth.password=$POSTGRESQL_SECRET \
```

making the full command:
```
helm upgrade cpt charts/camunda-platform/ \
  --values values.yaml \
  --set global.identity.auth.tasklist.existingSecret=$TASKLIST_SECRET \
  --set global.identity.auth.optimize.existingSecret=$OPTIMIZE_SECRET \
  --set global.identity.auth.operate.existingSecret=$OPERATE_SECRET \
  --set global.identity.auth.connectors.existingSecret=$CONNECTORS_SECRET \
  --set global.postgresql.auth.postgresPassword=$POSTGRESQL_SECRET \
  --set global.postgresql.auth.password=$POSTGRESQL_SECRET \
  --set identity.keycloak.auth.adminPassword=$KEYCLOAK_ADMIN_SECRET \
  --set identity.keycloak.auth.managementPassword=$KEYCLOAK_MANAGEMENT_SECRET \
  --set identity.keycloak.postgresql.auth.password=$POSTGRESQL_SECRET \
  --set global.identity.auth.zeebe.existingSecret=$ZEEBE_SECRET
```

And then get an error on the ZEEBE_SECRET since that does not exist.

Since Zeebe does not exist and the documentation does not reference Zeebe, I
will perform the same steps we did for the connectors secret:

```
helm template cpt camunda/camunda-platform --version 8.2 --show-only charts/identity/templates/zeebe-secret.yaml > identity-zeebe-secret.yaml

kubectl apply --namespace default -f identity-zeebe-secret.yaml
```


Then perform the secrets extraction step once more
```
export TASKLIST_SECRET=$(kubectl get secret "cpt-tasklist-identity-secret" -o jsonpath="{.data.tasklist-secret}" | base64 --decode)
export OPTIMIZE_SECRET=$(kubectl get secret "cpt-optimize-identity-secret" -o jsonpath="{.data.optimize-secret}" | base64 --decode)
export OPERATE_SECRET=$(kubectl get secret "cpt-operate-identity-secret" -o jsonpath="{.data.operate-secret}" | base64 --decode)
export CONNECTORS_SECRET=$(kubectl get secret "cpt-connectors-identity-secret" -o jsonpath="{.data.connectors-secret}" | base64 --decode)
export KEYCLOAK_ADMIN_SECRET=$(kubectl get secret "cpt-keycloak" -o jsonpath="{.data.admin-password}" | base64 --decode)
export KEYCLOAK_MANAGEMENT_SECRET=$(kubectl get secret "cpt-keycloak" -o jsonpath="{.data.management-password}" | base64 --decode)
export POSTGRESQL_SECRET=$(kubectl get secret "cpt-postgresql" -o jsonpath="{.data.postgres-password}" | base64 --decode)
export ZEEBE_SECRET=$(kubectl get secret --namespace "default" "cpt-zeebe-identity-secret" -o jsonpath="{.data.zeebe-secret}" | base64 -d)

```

And then try upgrade:
```
helm upgrade cpt charts/camunda-platform/ \
  --values values.yaml \
  --version 8.2.10 \
  --set global.identity.auth.tasklist.existingSecret=$TASKLIST_SECRET \
  --set global.identity.auth.optimize.existingSecret=$OPTIMIZE_SECRET \
  --set global.identity.auth.operate.existingSecret=$OPERATE_SECRET \
  --set global.identity.auth.connectors.existingSecret=$CONNECTORS_SECRET \
  --set global.postgresql.auth.postgresPassword=$POSTGRESQL_SECRET \
  --set global.postgresql.auth.password=$POSTGRESQL_SECRET \
  --set identity.keycloak.auth.adminPassword=$KEYCLOAK_ADMIN_SECRET \
  --set identity.keycloak.auth.managementPassword=$KEYCLOAK_MANAGEMENT_SECRET \
  --set identity.keycloak.postgresql.auth.password=$POSTGRESQL_SECRET \
  --set global.identity.auth.zeebe.existingSecret=$ZEEBE_SECRET

```


This results in the following error:
```
Error: UPGRADE FAILED: rendered manifests contain a resource that already exists. Unable to continue with update: Secret "cpt-connectors-identity-secret" in namespace "default" exists and cannot be imported into the current release: invalid ownership metadata; annotation validation error: missing key "meta.helm.sh/release-name": must be set to "cpt"; annotation validation error: missing key "meta.helm.sh/release-namespace": must be set to "default"
```

Looks like some missing annotation. Lets go ahead and add that into the
connectors and the zeebe secret that we created:

connectors will look like:

```
apiVersion: v1
data:
  connectors-secret: MUN5U1kxaDFNMQ==
kind: Secret
metadata:
  annotations:
    ...
    meta.helm.sh/release-name: cpt
    meta.helm.sh/release-namespace: default
    ...
```

And the same for zeebe.




After performing these steps, the upgrade command was able to complete
successfully. However, I am noticing an error in keycloak regarding a
duplicate key constraint error:

```
keycloak 2023-08-14 16:58:13,631 ERROR [org.keycloak.quarkus.runtime.cli.ExecutionExceptionHandler] (main) ERROR: ERROR: duplicate key value violates unique constraint "constraint_3c"
keycloak   Detail: Key (client_id, name)=(07a20e46-b43f-497e-b9ed-38438fa25788, post.logout.redirect.uris) already exists. [Failed SQL: (0) INSERT INTO public.client_attributes (CLIENT_ID, NAME, VALUE) VALUES ('07a20e46-b43f-497e-b9ed-38438fa25788', 
'post.logout.redirect.uris', '+')]
keycloak 2023-08-14 16:58:13,631 ERROR [org.keycloak.quarkus.runtime.cli.ExecutionExceptionHandler] (main) ERROR: ERROR: duplicate key value violates unique constraint "constraint_3c"                                                                   
keycloak   Detail: Key (client_id, name)=(07a20e46-b43f-497e-b9ed-38438fa25788, post.logout.redirect.uris) already exists.   
```

Using the password and port-forward we retrieved earlier, we should be able to connect to the database and take a look at this table.

```
postgres=# \c bitnami_keycloak
bitnami_keycloak=# select * from public.client_attributes;

              client_id               | value |                    name                    
--------------------------------------+-------+--------------------------------------------
 ff291767-d8ea-4bac-97ab-2077f7a19b5b | S256  | pkce.code.challenge.method
 b08f2c9f-7f91-4a03-99d3-40275492973f | S256  | pkce.code.challenge.method
 7718ba85-7c88-4be4-a78f-e409b52d795d | S256  | pkce.code.challenge.method
 e94815c6-c098-48be-8634-42dc3004ca04 | S256  | pkce.code.challenge.method
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | id.token.as.detached.signature
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | saml.assertion.signature
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | saml.force.post.binding
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | saml.multivalued.roles
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | saml.encrypt
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | oauth2.device.authorization.grant.enabled
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | backchannel.logout.revoke.offline.tokens
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | saml.server.signature
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | saml.server.signature.keyinfo.ext
 75b15c95-e0fd-484c-a010-8d6805488c0b | true  | use.refresh.tokens
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | exclude.session.state.from.auth.response
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | oidc.ciba.grant.enabled
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | saml.artifact.binding
 75b15c95-e0fd-484c-a010-8d6805488c0b | true  | backchannel.logout.session.required
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | client_credentials.use_refresh_token
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | saml_force_name_id_format
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | require.pushed.authorization.requests
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | saml.client.signature
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | tls.client.certificate.bound.access.tokens
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | saml.authnstatement
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | display.on.consent.screen
 75b15c95-e0fd-484c-a010-8d6805488c0b | false | saml.onetimeuse.condition
 75b15c95-e0fd-484c-a010-8d6805488c0b | +     | post.logout.redirect.uris
 07a20e46-b43f-497e-b9ed-38438fa25788 | +     | post.logout.redirect.uris
 c34f450d-fce3-451d-91c8-32b4f3215535 | +     | post.logout.redirect.uris
 d6ee50ef-9f14-4719-b168-f148ca65c68c | +     | post.logout.redirect.uris
 f205d7b2-ceda-4f30-87fe-59435df9b6fc | +     | post.logout.redirect.uris
 e0c2cc50-8952-4e34-9386-df725ff8f02c | +     | post.logout.redirect.uris
(32 rows)

```


It appears that there are indeed duplicate entries for this (07a20e46-b43f-497e-b9ed-38438fa25788, post.logout.redirect.uris) row. So lets delete this row, and try once more. 

```
bitnami_keycloak=# delete from public.client_attributes where client_id='07a20e46-b43f-497e-b9ed-38438fa25788' and name='post.logout.redirect.uris';
```

Now lets delete the keycloak pod to see if it can start up without that client
id.


hmm... similar errors. lets try deleting all the post.logout.redirect.uris


```
bitnami_keycloak=# delete from public.client_attributes where name='post.logout.redirect.uris';
```


Keycloak now starts up successfully. Now I'm going to restart each of the pods
that entered a CrashLoopBackOff

One minor odd thing is that Optimize and zeebe-gateway for some reason want to
use 2 replicas instead of one.


To resolve the optimize issue, I replaced the command in the optimize
deployment to sh -c 'sleep 9999999' and then exec'd into the container and ran
`./optimize.sh --upgrade`.

exited successfully, and now optimize readinessProbe passes.


Optimize webapp is now available!
