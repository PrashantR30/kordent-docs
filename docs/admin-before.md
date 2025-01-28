# Before you start

Before you start working with k0rdent, it helps to understand a few basics.

## How K0rdent works

k0rdent has several important subsystems, notably:

* **KCM - k0rdent Cluster Manager** - KCM wraps and manages Kubernetes Cluster API, and lets you treat clusters as 
Kubernetes objects. Within a k0rdent management cluster, you'll have a `ClusterDeployment` object that 
represents a deployed cluster, with `Machine` objects, and so on. When you create a `ClusterDeployment`, 
k0rdent deploys the cluster. When you delete it, k0rdent deletes it, and so on.
* **KSM - k0rdent Service Manager** - KSM wraps and manages several interoperating open source projects like Helm and Sveltos, which let you treat services and applications as Kubernetes objects.

Together, KCM and KSM interoperate to manifest a complete, template-driven system for defining and managing complete Internal Development Platforms (IDPs) comprising suites of services, plus a cluster and its components as realized on a particular cloud or infrastructure substrate. 

[[[ DIAGRAM 1, SHOWING K0RDENT AND A CLUSTERDEPLOYMENT ]]]

* **ClusterAPI providers**: ClusterAPI uses `providers` to manage different clouds and infrastructures, including bare metal. k0rdent ships with providers for AWS, Azure, OpenStack and vSphere, and you can add additional providers in order to control other clouds or infrastructures that ClusterAPI supports.

[[[ DIAGRAM 2, WHICH TAKES DIAGRAM 1 AND ADDS IN THE DRIVERS ]]]

* **Templates**: When you create a cluster, that cluster is based on a template, which specifies all of the various information about
the cluster, such as where to find images, and so on. These templates get installed into k0rdent, but they don't do 
anything until you reference them in a `ClusterDeployment` that represents an actual cluster.

[[[ DIAGRAM 3, WHICH TAKES DIAGRAM 2 AND ADDS IN TEMPLATES AND CLUSTERDEPLOYMENTS ]]]

k0rdent can also manage these clusters, upgrading, scaling them, or installing software and services.

* **Services**: To add (or manage) services,
you also use templates. These `ServiceTemplate`s are like `ClusterTemplate`s, in that you install them into the cluster, but until
they're actually referenced, they don't do anything. When you reference a `ServiceTemplate` as part of a `ClusterDeployment`,
k0rdent knows to install that service into that cluster.

[[[ DIAGRAM 4, WHICH TAKES DIAGRAM 3 AND ADDS SERVICETEMPLATES AND SERVICES ]]]

These services can be actual services, such as Nginx or kyverno, or they can be user applications.

## How Credentials work

Of course you can't do any of this without permissions. As a human, you can log into, say, AWS, and tell it to create a new
instance on which you are going to install Kubernetes, but how does k0rdent get that permission? It gets it through the use of 
`Credential`s. 

When you create a `ClusterDeployment` or deploy an application, you include a reference to a `Credential` object that has been
installed in the k0rdent management cluster. Depending on whether the target infrastructure is AWS, Azure, or something else, that
`Credential` might reference an access key and secret, or it might reference a service provider, but all of that gets abstracted
out by the time you get to the `Credential`, which is what you'll actually reference.

[[[ DIAGRAM 5, WHICH SHOWS THE DIFFERENT STRUCTURES THAT LEAD UP TO THE CREDENTIAL ]]]

By abstracting everything out to create a standard `Credential` object, users never have to have access to actual credentials (lowercase "c").
This enables the administrator to keep those credentials private, and to rotate them as necessary without disturbing users or
their applications. The administrator simply updates the `Credential` object and everything continues to work.

You can find more information on creating these `Credential`s in [the Credentials chapter](admin-credentials.md).

## k0rdent vs GitOps

At its heart, k0rdent is a way to declaratively specify what should be happening in the infrastructure and
have that maintained. In other words, if you want to, say, scale up a cluster, you would give that cluster a new
definition that includes the additional nodes, and then k0rdent, seeing that reality doesn't match that definition, 
will make it happen.

In some ways that is very similar to GitOps, in which you commit definitions and tools such as Flux or ArgoCD 
ensure that reality matches the definition. We can say that k0rdent is GitOps-compatible, in the sense that you can (and should) consider storing k0rdent templates and YAML object definitions in Git repos, and can (and may want to) use GitOps tools like ArgoCD to modify and manage them upstream of k0rdent itself.

The main difference is that k0rdent's way of representing clusters and services is fully compliant with Kubernetes-native tools like ClusterAPI, Sveltos and Helm. So you can, in fact, port much of what you do with k0rdent templates and objects directly to other solution environments that leverage these standard tools.
