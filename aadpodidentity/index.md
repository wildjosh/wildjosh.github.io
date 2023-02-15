# Azure AD Pod Identity


## Introduction

Hi! I'm Josh, a Cloud Engineer working at a consultancy in Manchester, UK. 

I've been meaning to write on this site for a while, but never really found the time or motivation to do so. With the new year now in full swing, I've been inspired by various people in the Azure community (Gregor Suttie, Gwyneth Peña-Siguenza to name a few) to finally make a start. Hopefully you'll learn something, but if nothing else, the posts on this site will be something I can refer back to in future to get a refresher on a topic.

This will be the first post in a small series, covering Azure Active Directory Pod Identity and its successor, Azure Active Directory Workload Identity. 

# Part 1 - Azure Active Directory Pod Identity:

This is a product I've only started using in anger recently, and I've come across a few problems that made me think it'd be a good candidate to begin this blog with. I'll write a little about the service, what problem it solves, why you'd use it, alternatives, and some limitations.

## What is Azure Active Directory Pod Identity, what problem does it solve? 

If you're running workloads in AKS, there's an extremely high chance it's not the only service in Azure that you're going to be using. How do you handle your applications authentication to the other Azure services (Azure SQL, Service Bus, SignalR for example)? If you're thinking along the lines of *"Create a Service Principal, create a secret, store the secret in Kubernetes secret"*, or **worse** still *"hardcode the credentials into your application"* 🤢 (never do this, please!), then please, keep on reading.

The solution that Pod Identity provides is to allow you to create User-Assigned Identities, and allow your application to securely use that identity to access other Azure resources.

## How does it work? 

Pod Identity supports two modes: Standard Mode, and Managed Mode. My experience lies solely within the Standard mode of operation, so I won't go into specifics around Managed Mode in this post, aside from saying the main difference is **only** NMI is deployed in Managed mode. The only times you'd really want to use this would be using short-lived, ephemeral applications where you don't want to wait for the MIC to assign the identity to the VMSS/VM (which can take 10-20 seconds for a single VM or 40-60 for a VMSS). You would need to then handle the identity assignment yourself.

In Standard mode, there are 2 components deployed into your cluster: 
- Managed Identity Controller
- Node Managed Identity DaemonSet

### Managed Identity Controller (MIC)

 *"An MIC is a Kubernetes controller that watches for changes to pods, AzureIdentity and AzureIdentityBinding through the Kubernetes API Server. When it detects a relevant change, the MIC adds or deletes AzureAssignedIdentity as needed. Specifically, when a pod is scheduled, the MIC assigns the managed identity on Azure to the underlying Virtual Machine Scale Set used by the node pool during the creation phase. When all pods using the identity are deleted, it removes the identity from the Virtual Machine Scale Set of the node pool, unless the same managed identity is used by other pods. The MIC takes similar actions when AzureIdentity or AzureIdentityBinding are created or deleted."* - [1]

Put simply, an MIC will add and remove the configured managed identities from the Virtual Machine Scale set of the node pool.

### Node Managed Identity (NMI)

*"NMI is a pod that runs as a DaemonSet on each node in the AKS cluster. NMI intercepts security token requests to the Azure Instance Metadata Service on each node, redirect them to itself and validates if the pod has access to the identity it's requesting a token for and fetch the token from the Azure AD tenant on behalf of the application."* [1]

NMI is really the glue behind the solution. It intercepts requests from your application to the Instance Metadata Service endpoint (IMDS), validates if the pod has access to the identity it wants to use, fetches the token, and returns it to your application. 

To intercept the request, NMI adds iptables rules that redirect all non-host traffic with the IMDS endpoint over port 80 as the destination to the NMI endpoint. The pod is then identified based on the IP address it sent the request from, and NMI queries Kubernetes via AzureAssignedIdentity for a matching Azure user-assigned identity. If it matches, it sends an ADAL (Azure Active Directory Authentication Library) to get the token for the identity, and returns it as a response to the pod. 

Put together, here is the flow:

- Pod is created and scheduled against a node in the pool 
- Application starts 
- MIC detects the pod and matching labels 
- MIC sends a request off to ARM to associate the requested identity to the underlying VMSS/VM 
- Application sends request to IMDS for a token 
- NMI intercepts this request, checks it is valid, retrieves the token and returns it to the application 
- Application uses token to access Azure resources

You can see also see the diagram below, borrowed from the Microsoft Learn site:

[![targets](/images/pod-identities.png)](https://learn.microsoft.com/en-us/azure/aks/media/operator-best-practices-identity/pod-identities.png)
## Limitations

**Maximum of 200 pod-managed identities are allowed per cluster** - Depending on the size of your workloads in AKS, this could be a showstopper. This is a fairly low amount of identities, especially if you run multiple tenants in your AKS cluster. You may also have multiple identites per application if you're using a microservice architecture.  

**Maximum of 200 pod-managed identity exceptions per cluster** - If you have applications that make use of the IMDS service, you won't want the requests they make to be intercepted by NMI. The way to work around this is to add an AzurePodIdentityException for the namespace the application runs in. Again, if you run a large amount of apps or multi-tenanted clusters, then you could easily hit this limit.

**Only available on Linux node pools** - A big one for anyone running Windows workloads on AKS - Pod Identity is not for you.

**Only supported on VMSS-backed clusters** - Your cluster has to be backed by a Virtual Machine Scale Set in order to be supported.

## Support

I'd like to note that the open-source Azure AD Pod Identity has been deprecated in favour of Azure AD Workload Identity. However, the AKS managed add-on is still supported, but to reiterate both Pod Identity and Workload Identity are still in preview, they're not GA services as of yet.

## Troubleshooting

In the time I've been using AAD Pod Identity, I've come across a number of issues. The largest issue I had was due to a fairly busy Service Bus triggered Azure Function app in AKS, using KEDA. Due to the large amount of messages in the Service Bus queue KEDA spun up multiple instances of the Function app, all of which needed to retrieve a token to access other Azure resources. It couldn't provide these tokens quickly enough and the function apps eventually crashed due to this. 

I used some of the commands below to investigate. They're fairly basic commands, but it's more knowing where to look that's important.

As with any troubleshooting scenario, your first port of call is most likely going to be looking at logs. I'm not going to go in detail into logging configurations, but in this case the fastest way to troubleshoot was using kubectl and viewing the logs directly:

#### Check your application pod logs
                kubectl logs *insert pod name* --namespace *insert namespace name* 
#### Find the node your application pod is running on
                kubectl get pod *insert pod name* --output wide --namespace *insert namespace name*
#### Find the NMI pod on the same node, it'll most likely be prefixed with "nmi-" (typically in the default namespace)
                kubectl get pods --output wide 
#### View the logs of the NMI pod (typically in the default namespace)
                kubectl logs *insert pod name* 


## Final Words

In this post we've looked at Azure AD Pod Identity, what it is, how it works, some limitations and how to troubleshoot it. You can find much more information in the documentation. If needed, I found the documentation in the GitHub repository much more useful than the documentation on the Microsoft Learn site: https://azure.github.io/azure-workload-identity/docs/

Hopefully this helps somebody, if not I'm sure I'll end up using it as a refresher at some point 😁 

In Part 2, I'll cover Azure AD Workload Identity and compare the differences between the two offerings.

Thanks for reading!

























[1] - https://learn.microsoft.com/en-us/azure/aks/use-azure-ad-pod-identity
