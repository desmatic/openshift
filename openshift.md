I'd like to explore the following stack: the latest version of redhat openshift, NetApp Trident ONTAP storage array, HPE ProLiant Compute DL380 Gen12 144 cores 4TB memory with 3 single-wide NVIDIA L4 GPU, integrated with Luna Network S750 HSM.

I'd like to be able to run postgres databases that have gpu accelerated ML/AI capability, that is configured to run as highly available containerized solution.

I'd also like to implement a service mesh, so that two different application teams could work in parallel, but without access to each others environments. I'd like an rbac solution to separate the two application teams. I would like my HSM to ensure cryptographic isolation, so that teams could publish and subscribe to each others services, but would not have access to each others databases. 

I'd like to explore HashiCorp Vault using ansible for secret rotation on my databases? I don't want one application team to be able to view another application teams secrets.

