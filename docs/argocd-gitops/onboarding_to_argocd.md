# ArgoCD onboarding procedure

The ArgoCD instance is deployed on [MOC Infra][1] cluster and can be used to deploy the application manifests located in this repo.

Each team is given an [ArgoCD Project][2] that has an allow-list of clusters and namespaces to which they can deploy. They also have an allow-list of resources they can deploy within those namespaces. These restrictions exist to prevent teams from being able to use ArgoCD to deploy cluster scoped resources, or resources onto other team's namespaces. You can browse the projects [here][3] to get an idea of the current set of permissions. Note that for most projects, majority of these permissions are inherited from the [global project][4].

Teams are given edit access (for ui/cli console) to their ArgoCD Projects using OpenShift RBAC and OpenShift Groups.

The following steps should be completed to fully onboard and enable a team to use ArgoCD to deploy `Applications` to their ArgoCD `Projects`.

> Note: ArgoCD Projects should not to be confused with OpenShift Projects.

## Pre-requisites
Team requesting ArgoCD access must have been onboarded to the cluster. See [here][5].

Please fork/clone the [operate-first/apps][6] repo.

## OpenShift Group

To add multi-tenancy support, we require the team to have an OpenShift group on the cluster on which our ArgoCD instance resides. This OpenShift group should include all the people belonging to the team that will need write-level access to applications belonging to the team's ArgoCD Project (explained later). The team being onboarded to ArgoCD should have had a group already created during cluster onboarding, as described [here][5].

> Note: We use teams/project names interchangeably throughout this doc as all ArgoCD projects (for end-users) are named after their team names picked during the cluster onboarding process.

## Create project directories for this repository
Pick an environment/cluster this team will be deploying to and create a folder [here][8] under that environment/cluster accordingly, we'll refer to this folder name as `<team-name>` and will re-use this value later.

Teams should be directed [here][9] for instructions on where to create their ArgoCD Applications. All Applications should be pointed to the team's [ArgoCD Project][3] (i.e. the `project` field in the ArgoCD [Application][7] manifest), see below for instructions on adding an ArgoCD Project.

## Create the ArgoCD Project
The team will need a dedicated [ArgoCD Project][2] for their [ArgoCD Applications][7]. The ArgoCD project should be added [here][3].

A typical project will look like the following:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: <team-name>
  labels:
    project-template: global
spec:
  destinations:
    - namespace: '<team-namespace-prefix>-*'
      server: '<cluster-name>'
  sourceRepos:
    - '*'
  roles:
    - name: project-admin
      description: Read/Write access to this project only
      policies:
        - p, proj:<team-name>:project-admin, applications, get, <team-name>/*, allow
        - p, proj:<team-name>:project-admin, applications, create, <team-name>/*, allow
        - p, proj:<team-name>:project-admin, applications, update, <team-name>/*, allow
        - p, proj:<team-name>:project-admin, applications, delete, <team-name>/*, allow
        - p, proj:<team-name>:project-admin, applications, sync, <team-name>/*, allow
        - p, proj:<team-name>:project-admin, applications, override, <team-name>/*, allow
        - p, proj:<team-name>:project-admin, applications, action/*, <team-name>/*, allow
      groups:
        - <team-openshift-group>
        - operate-first
  clusterResourceWhitelist:
    - group: ''
      kind: Namespace
  namespaceResourceWhitelist:
  ....
```

Some notes:

* Ensure that the `spec.destinations` field contains a prefix for the team's namespaces. The team will be required to prefix their namespaces with this attribute if they want ArgoCD to be able to deploy to them, any other namespaces not following the prefix will need to be added manually under this field. See additional notes for more details.
* Ensure `operate-first` is added into the `roles.groups` for each role. This allows the operate-first team to help diagnose issues.
* `namespaceResourceWhitelist` generally contains the list of resources a project `admin` has access to. The general idea is that a team should be able to deploy via ArgoCD what they can deploy using `oc apply`. See other projects for a list of such resources.
* `<cluster-name>` should be a cluster found [here][cluster-list], use the `metadata.name` value.

Ensure that the ArgoCD project is included in the `kustomization.yaml` [here][11].

## Enable OpenShift auth to ArgoCD Console
By default all users should be able to see the [ArgoCD console][12]. To be able to make changes to applications belonging to the team's ArgoCD Project (via the cli or ui), the team will need to be able to log into the console with appropriate access. This is accomplished by adding the team's OpenShift group mentioned in the beginning under the dex config [here][13].

## Give team read access to non-app ArgoCD resources

Append this [list][14] and give the team the `standard-user` role. The group will correspond to the team's OCP group.
For example if onboarding the team `someteam` you would append the following to this file:

```yaml
g, someteam, role:standard-user
```

## Additional Notes:

### Namespace Prefix
An ArgoCD project allows us to ensure that a team cannot deploy applications onto another team's namespace via ArgoCD.

We accomplish this by allowing teams to add the namespaces that they will be deploying to in their ArgoCD Project CR under the `spec.destinations.` field. Example:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: example-project
  namespace: argocd
spec:
  destinations:
  - namespace: guestbook
    server: 'https://kubernetes.default.svc'
...
```

The project above can only deploy into the `guestbook` namespace in the host cluster.

It can get bit tedious having to always require a team to submit a pr updating the `spec.destinations` to include new namespaces their project applications can deploy onto, so we can use wildcards to include a set of namespaces that begin with a prefix. For example, the `thoth` team has the following prefix in their `spec.destinations`:

```yaml
spec:
  destinations:
    - namespace: 'thoth-*'
      server: 'https://kubernetes.default.svc'
```

Now, as long as all thoth team's namespaces have `metadata.name` beginning with a `thoth-` prefix, they can deploy into these namespaces using ArgoCD Applications that are part of the `thoth` project.

### Resources:

- To read more about ArgoCD Declarative setup [see here][15]
- To understand our authentication setup [see here][16]
- To read about ArgoCD Projects [see here][2]
- To learn about Kustomize bases & overlays [see here][17]

[1]: https://github.com/operate-first/apps/tree/master/argocd/overlays/moc-infra
[2]: https://argoproj.github.io/argo-cd/user-guide/projects/
[3]: https://github.com/operate-first/apps/tree/master/argocd/overlays/moc-infra/projects
[4]: https://github.com/operate-first/apps/blob/master/argocd/overlays/moc-infra/projects/global_project.yaml
[5]: https://github.com/operate-first/support/blob/main/docs/onboarding_to_cluster.md
[6]: https://github.com/operate-first/apps
[7]: https://argoproj.github.io/argo-cd/operator-manual/declarative-setup/#applications
[8]: https://github.com/operate-first/apps/tree/master/argocd/overlays/moc-infra/applications/envs
[9]: ./add_application.md
[10]: https://github.com/operate-first/apps/tree/master/argocd/overlays/moc-infra/secrets/clusters
[11]: https://github.com/operate-first/apps/blob/master/argocd/overlays/moc-infra/projects/kustomization.yaml
[12]: https://argocd-server-argocd.apps.moc-infra.massopen.cloud
[13]: https://github.com/operate-first/apps/blob/master/argocd/overlays/moc-infra/configs/argo_cm/dex.config#L11
[14]: https://github.com/operate-first/apps/blob/master/argocd/overlays/moc-infra/configs/argo_rbac_cm/policy.csv#L21
[15]: https://argoproj.github.io/argo-cd/operator-manual/declarative-setup/
[16]: https://argoproj.github.io/argo-cd/operator-manual/user-management/#dex
[17]: https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#bases-and-overlays
[18]: https://github.com/operate-first/argocd-apps
[cluster-list]: https://github.com/operate-first/apps/tree/master/acm/overlays/moc/infra/managedclusters
