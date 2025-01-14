# The threat and trust model for confidential container

---

## Motivation

Confidential Computing Consortium (CCC) defines the threat model for confidential
computing[1: 5 Threat Model]. Confidential container is a specific use case of confidential
computing for the cloud native scenario. Therefore, the threat model of confidential container
may refer to it, with the details for confidential container.

Unfortunately, CCv0 doesn't have a clear definition for the threat model of confidential
container. If the confidential container is going to develop in a healthy manner, it is necessary
to define and clarify the threat model as soon as possible in order to provide guidance and
constraint on the components and architecture in the security principle of confidential container.
Otherwise, it is difficult to follow an unified security principle among the components composing
the confidential container. In addition, it's easy to fall into the constant discussions and
clarifications.

---

## The forms of confidential container

There is a number of ways to accomplish a confidential container, such as VM-based "vm-cc",
MicroVM-based "microvm-cc", and Intel SGX Enclave based "enclave-cc".

"vm-cc" uses a virtual machine with hardware TEE capability to run a K8s node. "microvm-cc" uses
a MicroVM with hardware TEE capability to run a Pod. "enclave-cc" uses a SGX Enclave to run a
LibOS.

The commonality of these forms is to run a container inside hardware TEE. The significant
differences among them are as following:

| | vm-cc | microvm-cc/kata-cc\*1 | enclave-cc |
| :-: | :-: | :-: | :-: |
| TCB size | Large | Medium | Small |
| Location of trust boundary\*2 | between K8s API server and kubelet | agent API\*3 | ECALL and OCALL |
| Threat vectors | More | Medium | Less |
| Deployment complexity | High | Low | Medium |
| Implementation difficulty | Low | High | Medium |

Note:
1. This doc may use kata-cc equivalent to microvm-cc in order to refer to a kata-specific component.
2. Here refers to the trust boundary of control plane.
3. A component used in kata and running in POD.

The scope of this doc is limited to vm-cc and microvm-cc/kata-cc.

---

## Execution stages

The life cycle of confidential container can be roughly divided into 3 execution stages:

- Pre-container stage: occurs before pulling container image. This stage contains the build and
initialization of TEEs. The involved components include the ones that directly associated with the
protections that TEEs provide (e.g, tdx seam-loader, tdx module, psp/sev firmware and so on),
Pre-OS (e.g, guest firmware, guest kernel, initrd and so on) and specific components (e.g,
kata-agent, attestation-agent and so on).

- Container launch stage: refers to the procedure of image pulling, unpacking and launching.

- Workload operation stage: refers to the procedure since the workload in the container starts to
run in TEEs.

These three execution stages are sufficient to describe the execution of any forms of confidential
container. Therefore, the following discussions about the threat model of confidential container
will focus on them.

---

## Goals and not-goals

### Goals

The commonality of goals between confidential container and confidential computing are to reduce
the ability for the platform owner, operator and attacker to access data and code inside TEEs
sufficiently such that this path is not an economically or logically viable attack during
execution[1: 5.1 Goal].

Around the three execution stages of confidential container, the scope of the threat model of
confidential container includes:

- Pre-container stage
  - Measure the TEE where confidential containers run, and store the measurements to a secure location.

- Container launch stage
  - Measure and attest the TEE where confidential containers run to prove the deployment and initialization of pre-container are valid and expected;
  - Pull the encrypted and signed container image in TEE;
  - Provision the runtime configurations through a secure and attested control plane with the protections of confidentiality and integrity.

- Workload operation stage
  - Provide the protections of confidentiality and integrity for the storage and transport of data derived from TEE outside the TEE instance;
  - Keep the dynamic measurement of TEE where confidential containers run, and store the measurements to a secure location;
  - Continuously launch the runtime attestations for TEE where confidential containers run according to the policy to ensure the TEEs are not comprised or out-of-date;
  - Migration of confidential containers between TEE instances in a secure manner.

### Not-goals

The following itesm are considered out-of-scope for the threat model of confidential container: 

- The assurance of protections for application itself.

- The "Availability" in CIA Triad. A specific solution can provide the protection against availability attacks.

- Items related to any software TEEs.

---

## Threat vectors

The threat vectors[1: 5.2.1 In-Scope] considered to be in-scope for Confidential Computing are
defined. The following threat vectors are specific to confidential container:

- Software supply chain attacks: include the attacks on components directly associated with the protections that specific TEEs provide, as well as pre-container components running inside TEEs. These components belong to the TCB of confidential container. Any attacks that may compromise TCB can fundamentally break the security of confidential container.

- Container image attacks: include the attacks on the container image. A container image includes the tenant's sensitive workload and data, so any attacks that may compromise the container image may cause the leakage of the tenant's sensitive information.

- Control plane attacks: include the attacks on K8s and container runtime control planes (such as CRI/shim/agent API). Traditionally, tenants trust these control planes controlled by the platform owner. If a specific confidential container based solution distrusts these control planes, the storage and transport of data over these control planes needs the protections for confidentiality and integrity. In addition, the Alternately, use another control plane to protect the data, e.g, a secure and attested control plane established with remote attestation procedure. In addition, the service APIs located at the trust boundary, e.g, agent API used by kata-cc, may cause trust boundary violation, without hardening or restriction.

- Storage of data attacks: include the attacks related to the storage of data derived by confidential container. These sensitive data belong to tenants, and the confidential container must provide the protections of confidentiality and integrity.

---

## Trust model

The main purpose of a trust model is to continuously measure trustworthiness of a set of entities
based on their behaviors, and it must include implicit or explicit validation of an entity's
identity or the characteristics necessary for a particular event or transaction to
occur[2: Defining Trust Modeling].

As for the confidential container, the TEE running the confidential container needs to provide a
reliable manner of measurement across the life cycle of a confidential container. In the following
attestation procedure, the attestation service fully verifies the validity of TEE measurements.
The real purpose is to persuade the tenant to believe that the workload in the confidential
container is running in a verified and genuine TEE. In summary: the trust model of confidential
container needs to address the trustworthiness of TEE.

The trust model of confidential container closely focuses on the establishment and extension
procedure of chain of trust, which covers the above three execution stages:

- Pre-container stage: create and extend the chain of trust, that is, how to create and initialize a TEE in a secure manner, including using measured boot and / or secure boot during pre-container bootstrap.

- Container launch stage: the chain of trust continues to extend to the container image, that is, how to ensure image pulling, unpacking and launching in a secure manner.

- Workload operation stage: the chain of trust continues to extend to the workload, that is, how to ensure the  workload, derived data and incoming data are protected and verified in a secure manner.

There are many ways for actors in the ecosystem who rely on the security guarantees of a TEE to
establish trust in the TEE. This is the so-called trust policy. One trust policy is to obtain
assurance of security statements about products through assessment by third-party evaluation
laboratories. Other trust policies include basing assurances on security statements from specific
vendors; community or other audit of open source components in hardware, firmware, and/or software;
and assessment or certification by industry or standards bodies[1: 5.1 Goal]. Theoretically, the
trust policy can be fine-grained to the component level, so there are many combinations of the
trust policies that a specific solution may adopt basing on component level. In fact, different
forms of confidential container and specific solutions adopt the trust policy based on the
granularity of execution stage, that is, all components involved in each stage will follow the
same trust policy.

The trust model also determines the location and direction of the trust boundary. Trust boundary
describes a location where program data or execution changes its level of "trust," or where two
principals with different capabilities exchange data or commands[3]. The trust boundary of
confidential container is also the trust boundary of TEE. Generally, the TEE side at the trust
boundary should be hardened to prevent the violation of the trust boundary.

In general, the trust model adopted by any confidential container based solutions can be binarized
into:
1) Partially trust the platform owner, operator and privileged components controlled by host
2) Completely distrust the platform owner, operator and privileged components controlled by host

where 1) means that some components intruded into TCB can be unconditionally trusted by the tenant.
For example, a vm-cc based solution is likely to unconditionally trust the guest firmware provided
by host. On the contrary, a microvm-cc based solution is likely to verify the measurement of a
lightweight guest firmware provided by host during attestation procedure. Another example is, in
some cases, a tenant would like to trust the secret provisioned to the container through K8s API
server, and thus the threat vector "control plane attack" is not deemed as a trust boundary
violation. 2) is the preferred approach providing the protection for the confidential container
with the highest level of trust. Note: if a tenant fully trusts the platform owner, operator and
privileged components controlled by host, the TEE boundary will no longer be the trust boundary of
the confidential container, so in this case there is no need to use confidential computing.

In practice, those (such as tenant) deploying workloads into TEE environments may have varying
levels of trust in the owner of the system (such as platform owner) that actually hosting the
workload. This trust may be based on factors such as the relationship with the owner or operator of
the host, the software and hardware it comprises, and the likelihood of physical, software, or
social engineering compromise[1: 5.1 Goal].

Therefore, the level of trust ultimately determines what kind of trust model to adopt. The trust
model needs to define their own trust policies for the execution stages. There are many approaches
to implement trust policies. The most important thing is to ensure the chain of trust is never
broken, and the validity of TEE in attestation procedure is proved sufficiently. In addition, TEE
needs to focus on preventing potential security threats at the trust boundary.

Nota that the trust may change over time. Therefore, the trust model needs to be flexible enough to
allow the tenant to select different trust policy used in an execution stage. 

---

## Reference

- [1] [A Technical Analysis of Confidential Computing v1.1](https://confidentialcomputing.io/wp-content/uploads/sites/85/2021/03/CCC-Tech-Analysis-Confidential-Computing-V1.pdf)
- [2] [Trust Modeling for Security Architecture Development](https://www.informit.com/articles/article.aspx?p=31546)
- [3] https://en.wikipedia.org/wiki/Trust_boundary
