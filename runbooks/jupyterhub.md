# JupyterHub Runbook

This document provides information on how to administer our JupyterHub
deployment.

## Key Locations

| Env           | Namespace                                                                                                      | GitHub Repo                                                           |
| ------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| MOC-ZERO Prod | [opf-jupyterhub](https://console-openshift-console.apps.zero.massopen.cloud/project-details/ns/opf-jupyterhub) | https://github.com/operate-first/apps/tree/master/odh/base/jupyterhub |

## Dashboards

Links to the monitoring dashboards for this service.

| Dashboard Description | Dashboard URL                                                                                        |
| --------------------- | ---------------------------------------------------------------------------------------------------- |
| JupyterHub SLI/SLO    | https://grafana-route-opf-monitoring.apps.zero.massopen.cloud/d/BfSK2f1Mz/jupyterhub-sli-slo?orgId=1 |
| JupyterHub Usage      | https://grafana-route-opf-monitoring.apps.zero.massopen.cloud/d/YuYfkHYMk/jupyterhub-usage?orgId=1   |

## Prerequisites

Our JupyterHub instance is managed by the Open Data Hub operator.

## Deployment

The JupyterHub server is deployed using Kustomize and ArgoCD. As such, any changes should be made
in the necessary files in git and redeployed (rather than manually editing
OpenShift objects directly).

## Upkeep and Administration

JupyterHub includes an Admin tool that lets us manage users. In production,
this tool can be reached [here](https://jupyterhub-opf-jupyterhub.apps.zero.massopen.cloud/hub/admin). Note that the members of the `operate-first` group have been added as `jupyterhub_admins` [here](https://github.com/operate-first/apps/blob/master/odh/base/jupyterhub/overrides/jupyterhub/base/jupyterhub-configmap.yaml). The admin tool will let you stop/start,
delete, and access a user's notebook server and can be very helpful for
addressing issues that JupyterHub users may encounter.

## Common Problems

The following is a list of common issues we've encountered with JupyterHub and
how to fix them.

### Smoke Test

Verify that the service is `up` and `available` by checking if you can access/login to the service [endpoint](https://jupyterhub-opf-jupyterhub.apps.zero.massopen.cloud/). Verify that the page loads, that you can log in, and
that you can spin up a notebook.

### Insufficient disk space for notebook pod

When a user runs out of disk space on their notebook pod, the pod will fail to start and give little indication to the user about why that's happening.
Follow steps [here](https://github.com/operate-first/blueprint/blob/main/docs/userstories/jupyterhub/analyze_storage.feature) to determine if the user has, in fact, used up their storage quota.

If the user has run out of storage space, follow the steps [here](https://github.com/operate-first/blueprint/blob/main/docs/userstories/jupyterhub/increase_pvc_size_jh.feature) to increase their PVC size.

### Custom Image not listed in Spawner UI

Custom JupyterHub notebook images add libraries, and kernels to JupyterHub which users can spawn and use. The steps followed to create these custom images are documented [here](https://github.com/operate-first/blueprint/blob/main/docs/userstories/jupyterhub/add_imagestream_to_jh.feature).

After adding the custom image to `odh/base/jupyterhub/notebook-images` if the images do not show up in the JH Spawner UI then ensure the following:

- The imagestream has the label:   opendatahub.io/notebook-image: "true"
- The imagestream was deployed by ArgoCD in [this app](https://argocd-server-argocd.apps.moc-infra.massopen.cloud/applications/opf-jupyterhub).

### User unable to start server

JupyterHub occasionally gets into a corrupted state where it thinks that the notebook pod for a user is running when it actually isn't running. When the user tells JupyterHub to take the server down, they will either get stuck in an infinite loop or get an error saying that the pod isn't running. The fix is as follows:

1. Login as an Operate First admin user to the jupyterhub namespace
2. Check to see if the user's pod shows up in the list of running pods. If it does, delete it.
3. Delete the _jupyterhub_ pod. Do __NOT__ delete the _jupyterhub-db_ pod.
4. Ask the user to login again. They should be able to spin up their notebook server.

### JupyterHub unreachable despite pods being up

JupyterHub can occasionally end up in a state where the main jupyterhub pod is unable to connect to the database. The logs will usually look like the following.

```bash
---> Running application from script (/opt/app-root/builder/run) ...
+ trap 'kill -TERM $PID' TERM INT
+ PID=37
+ wait 37
+ start-jupyterhub.sh
[I 2020-12-01 19:15:55.235 JupyterHub app:1673] Using Authenticator: oauthenticator.openshift.OpenShiftOAuthenticator-0.9.0dev
[I 2020-12-01 19:15:55.235 JupyterHub app:1673] Using Spawner: builtins.OpenShiftSpawner
[D 2020-12-01 19:15:55.236 JupyterHub app:1625] Could not load pycurl: No module named 'pycurl'
    pycurl is recommended if you have a large number of users.
[D 2020-12-01 19:15:55.237 JupyterHub app:1071] Connecting to db: postgresql://jupyterhub:<password>@jupyterhub-db:5432/jupyterhub
/opt/app-root/lib/python3.6/site-packages/psycopg2/__init__.py:144: UserWarning: The psycopg2 wheel package will be renamed from release 2.8; in order to keep installing from binary please use "pip install psycopg2-binary" instead. For details see: <http://initd.org/psycopg/docs/install.html#binary-install-from-pypi> .
  """)
[D 2020-12-01 19:15:55.283 JupyterHub orm:685] database schema version found: 896818069c98
[I 2020-12-01 19:15:55.340 JupyterHub app:1201] Not using whitelist. Any authenticated user will be allowed.

------ nothing else after the above lines
```

The issue can be resolved by restarting both pods.
