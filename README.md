These are templates and resources I used to deploy Red Hat (Formerly CoreOS) Quay on Openshift 3.11

Please be aware these installation steps are subject to change any time, and are probably expected to more suite Quay to Openshift. 

These steps are to install Quay Enterprise for a LAB. This is not HA, and this does not use an object based storage solution.

It is assumed this is ran on an Openshift cluster with dynamic storage available, a default storageclass assigned, and access to the Redhat Postgresql 9.6 imagestream/template. Installing the openshift-examples via openshift-ansible will make this available. They can also be manually imported.

It is also assumed that the user has completed part 4.5 of the Quay on Openshift documentation, adding Quay anthentication. It is assumed this was executed as root and the info lives in /root/.docker/config.json


This setup is not suitable for production workloads. That is in the works and at the mercy of my free time.

Create all of the boring stuff:

```
oc create -f quay-enterprise-namespace.yml -n quay-enterprise
oc create -f quay-enterprise-config-secret.yml -n quay-enterprise
oc create secret generic coreos-pull-secret --from-file=".dockerconfigjson=/root/.docker/config.json" --type='kubernetes.io/dockerconfigjson' -n quay-enterprise
oc create -f quay-servicetoken-role-k8s1-6.yaml -n quay-enterprise
oc create -f quay-servicetoken-role-binding-k8s1-6.yaml -n quay-enterprise
oc adm policy add-scc-to-user anyuid system:serviceaccount:quay-enterprise:default
oc create -f quay-enterprise-redis.yml -n quay-enterprise
oc create -f quay-enterprise-app-rc.yml -n quay-enterprise
oc create -f quay-enterprise-service-clusterip.yml -n quay-enterprise
```
At this point: You should see the `quay-enterprise-app` and `quay-enterprise-redis` pods running or creating. If you get an image pull error, you most likely misconfigured the pull secret.


Next, we deploy postgresl-persistent. In this example I use the Red Hat Catalog:

```
oc new-app --template=postgresql-persistent --param=DATABASE_SERVICE_NAME=quaydb --param=POSTGRESQL_USER=quayuser --param=POSTGRESQL_PASSWORD=quaypass --param=POSTGRESQL_DATABASE=quaydb -n quay-enterprise 
oc rsh <postgres-pod>
echo "SELECT * FROM pg_available_extensions" | /opt/rh/rh-postgresql96/root/usr/bin/psql
echo "CREATE EXTENSION pg_trgm" | /opt/rh/rh-postgresql96/root/usr/bin/psql
echo "SELECT * FROM pg_extension" | /opt/rh/rh-postgresql96/root/usr/bin/psql
echo "ALTER USER quayuser WITH SUPERUSER;" | /opt/rh/rh-postgresql96/root/usr/bin/psql
exit
```

At this point you need to expose your service. This depends on your load balancer, if port 80 isn't opened at all you'll need to do some secured route settings, `oc expose` or `oc create route --service=..`

//TODO: Finish the webui config portion

