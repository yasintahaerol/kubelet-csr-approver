# kubelet-csr-approver

Kubelet CSR approver is a Kubernetes controller whose sole purpose is to auto-approve [`kubelet-serving`
Certificate Signing Request (CSR)](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#kubelet-serving-certs). 

Unlike other existing projects (such as `kubelet-rubber-stamp`), it implements a series a checks preventing an attacker from forging Certificates.

## attacker model -- what could go wrong ?

Shall our CSR auto-approver not be implemented correctly, it might permit an attacker to get forged CSRs to be 
approved and later on signed by the K8s certificate controller.

Indeed, while there are some verifications done by the final certificate signer controller (for the details, [here](https://github.com/kubernetes/kubernetes/blob/v1.22.2/pkg/controller/certificates/signer/signer.go#L253-L258) and [here](https://github.com/kubernetes/kubernetes/blob/v1.22.2/pkg/apis/certificates/helpers.go#L62-L88)), nothing prevents a CSR impersonating a `DNSName` or an `IPAddress` from getting signed.

Put more concretely, if a CSR requesting the DNS name `auth.company.com` or `control-plane.k8s.local` (or both!) was to be approved, it would get signed and the attacker would have a very valid certificate to make use of. (and depending on the `ca.key` used on your cluster, this could have a measurable impact)

## which verifications do we put in place ?

Taking inspiration from [Kubernetes built-in CSR approver](https://github.com/kubernetes/kubernetes/blob/v1.22.2/pkg/controller/certificates/approver/sarapprove.go), we check the following criteria:

- the `CSR.Spec.SignerName` name must be `"kubernetes.io/kubelet-serving"`
- the `CSR.Spec.Username` must be prefixed with `system:node:` (i.e. we only want to treat CSRs originating from the nodes themselves)
- the CSR `CommonName` must be equal to the `CSR.Spec.Username`
- the CSR DNS SubjectAlternativeNames (SAN) contains at most one entry
- at least one SAN IP address or SAN DNS Name must be specified
- the CSR SAN DNS Name (if specified) must comply with a provider-specific regex.
- the CSR SAN DNS Name (if specified) must be prefixed with the node hostname (where the hostname corresponds to `CSR.Spec.Username` trimmed of the `system:node:` prefix)
- the CSR SAN IP Addresses must all be part of the set of IP addresses resolved from the SAN DNS Name
- !! the CSR SAN DNS Name (if specified) must resolve to an IP address that falls within the set of provider-specified IP ranges.
- !! the CSR SAN IP Address(es) must fall within a set of provider-specified IP ranges

!! = not yet implemented

With those verifications in place, it makes it quite hard for an attacker to get a forged hostname to be signed, it would indeed require:

- to impersonate a user on the Kubernetes API server with a `Username` that prefixes the SAN DNS Name request. \
concretely, if the attacker wants to forge a CSR for the `auth.company.ch` domain, he would need to create a CSR with the username `system:node:a` (remember, we only check the that the DNS name is prefixed by the node name) \
it might then be possible to create a CSR from a node `a` (not a smart name for a node, I agree), or a node `auth`, already more plausible
- to modify the provider-specific regex of the `kubelet-csr-approver` (requires API access or direct access to the node where the controller is running). \
however with API access, the attacker could as well also directly approve the CSR, and with full node access, the attacker could retrieve the controller's ServiceAccount and approve the CSR as well.

## Is this CSR approver safe ?

Provided that the provider-specific regex is strict, that the IP ranges    set is correctly specified, that the DNS system is not compromised, this automatic CSR approver would make it quite hard for an attacker to start forging CSRs.

### Could this CSR approver be safer ?

For sure, this simply requires modifying the `ProviderChecks(csr , x509csr))` function to implement additional checks (such as validating the node identity in an external inventory)
