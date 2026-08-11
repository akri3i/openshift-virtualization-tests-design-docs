# Openshift-virtualization-tests Test plan

## **SR-IOV vGPU - Quality Engineering Plan**

### **Metadata & Tracking**

- **Enhancement(s):** N/A — no VEP linked for this STP; feature tracked via
  https://issues.redhat.com/browse/VIRTSTRAT-623
- **Feature Tracking:** https://issues.redhat.com/browse/VIRTSTRAT-623
- **Epic Tracking:** https://issues.redhat.com/browse/CNV-79151
  (related: https://issues.redhat.com/browse/CNV-74901)
- **Feature Maturity:**
  - DP: N/A
  - TP: v5.0.z
  - GA: v5.2.0
- **QE Owner(s):** Akriti Gupta (@akri3i)
- **Owning SIG:** sig-virt
- **Participating SIGs:** sig-virt

**Document Conventions:**

- **SR-IOV vGPU** — GPU virtualization that exposes NVIDIA GPU Virtual Functions (VFs)
  to VMs for shared GPU access. On RHCOS 10 kernels, this replaces mediated-device
  (mdev) based vGPU for Ampere and newer GPUs; pre-Ampere GPUs continue to use mdev
  vGPU on RHCOS 10
- **VF** — Virtual Function: a hardware-partitioned slice of a physical GPU that a
  VM can consume for GPU acceleration
- **RHCOS10** — Red Hat CoreOS 10 worker nodes (RHEL 10 kernel), required for
  SR-IOV-based vGPU support

### **Feature Overview**

With SR-IOV vGPU support on Ampere and newer NVIDIA GPUs, customers can share a
physical GPU across multiple VMs on RHCOS10 worker nodes. Cluster administrators can
configure GPU sharing on those nodes, and VM owners can request GPU acceleration for
their workloads without dedicating an entire GPU to a single VM. Multiple VMs can each
be assigned GPU acceleration from the same physical GPU at the same time, each with
its own vGPU. Going forward, the RHEL 10 kernel only supports SR-IOV-based vGPUs for Ampere
and newer GPUs, so this is the path customers need on RHCOS10 worker nodes with that
hardware; pre-Ampere GPUs (for example Tesla T4) continue to use mdev vGPU on RHCOS10.

This STP validates SR-IOV vGPU as Tech Preview in CNV 5.0.z.

---

### **I. Motivation and Requirements Review (QE Review Guidelines)**

This section documents the mandatory QE review process. The goal is to understand the
feature's value, technology, and testability before formal test planning.

#### **1. Requirement & User Story Review Checklist**

- [x] **Review Requirements**
  - *List the key D/S requirements reviewed:*
    - SR-IOV-based vGPUs are created and advertised for VMs on RHCOS10 worker nodes
    - VMs can request and consume an SR-IOV GPU VF
    - Multiple VMs can simultaneously use VFs from the same physical GPU with isolation
    - VMs that have been assigned an SR-IOV vGPU VF detect the GPU device in the guest

- [x] **Understand Value and Customer Use Cases**
  - *Describe the feature's value to customers:* Customers running AI/ML and other
    GPU-accelerated workloads need multi-tenant GPU sharing on RHCOS10. SR-IOV vGPU
    lets multiple VMs share one physical GPU with hardware-level VF isolation, without
    dedicating a full GPU to each VM.
  - *List the customer use cases identified:*
    - As a cluster administrator, I want to configure SR-IOV vGPU devices on RHCOS10
      GPU worker nodes so multiple VMs can share a physical GPU
    - As a VM owner, I want to request a vGPU for my VM so I can run GPU-accelerated
      workloads

- [x] **Testability**
  - *Note any requirements that are unclear or untestable:*
    - NVIDIA GPU Operator SR-IOV vGPU support targets 26.11 (Nov 2026)

- [x] **Acceptance Criteria**
  - *List the acceptance criteria:*
    - After SR-IOV vGPU setup on an RHCOS10 GPU worker node, the node advertises the
      expected vGPU / VF resources for VM scheduling
    - A VM requesting an SR-IOV vGPU reaches Running state and the guest detects
      GPU device
    - Two VMs sharing VFs from the same physical GPU are both Running concurrently,
      and each guest detects its assigned GPU
    - A VM with an SR-IOV vGPU can be paused and unpaused; after unpause, the VM is
      reachable again and the GPU remains visible
    - A VM with an SR-IOV vGPU can be restarted and the guest still detects the GPU
      after restart
    - A VM requesting an SR-IOV vGPU when no VF capacity remains stays
      Pending/unschedulable, and the scheduling event or message identifies exhausted
      SR-IOV vGPU capacity as the cause

- [x] **Non-Functional Requirements (NFRs)**
  - *List applicable NFRs and their targets:*
    - **Reliability:** GPU device must be visible inside the VM after successful
      SR-IOV vGPU assignment and after restart
    - **Documentation:** Documentation is required for supported SR-IOV vGPU
      configurations
    - **Compatibility:** Feature targets RHCOS10 nodes
  - *Note any NFRs not covered and why:*
    - **Performance:** NVIDIA owns GPU
      performance characteristics. OpenShift Virtualization does not run dedicated
      GPU performance benchmarking in this cycle.
    - **Monitoring:** N/A; no new CNV alerts/metrics claimed for this cycle.
    - **Observability:** No new logs, metrics, or events are introduced by this feature.
      Existing VM status (Running/Pending) and scheduling events are sufficient to
      diagnose successful vGPU assignment and VF-capacity-exhaustion cases; validated
      via the scheduling-feedback acceptance criterion and Usability Testing.
    - **UI:** No OpenShift Virtualization UI changes claimed for this feature.
      Customer value for UI testing is not identified
    - **Scalability:** No new scale requirements introduced; scalability is bounded by
      the maximum VFs supported by the physical GPU and the GPU Operator's scheduling
      limits, validated within standard functional boundaries.
    - **Security:** Hardware-level VF isolation is enforced by NVIDIA GPU firmware /
      SR-IOV; device exposure is provided by the NVIDIA GPU Operator. OpenShift
      Virtualization adds no new isolation mechanism here; QE validates VM consumption
      of advertised SR-IOV vGPU devices only.

#### **2. Known Limitations**

- **SR-IOV vGPU live migration is not supported**
  (tracked separately; listed as a non-requirement of the RHCOS10 epic)
  - *Sign-off:* Sudhakar Molli [Date]


#### **3. Technology and Design Review**

- [x] **Developer Handoff/QE Kickoff**
  - *Key takeaways and concerns:*
    - Ampere and newer GPUs need SR-IOV vGPU support on RHCOS10
    - SR-IOV vGPU needs NVIDIA GPU Operator version 26.11 (Nov 2026)

- [x] **Technology Challenges**
  - *List identified challenges:*
    - Requires NVIDIA SR-IOV-capable GPU hardware and RHCOS10 worker nodes
    - SR-IOV vGPU needs NVIDIA GPU Operator version 26.11
  - *Impact on testing approach:* Tests run only on RHCOS10 nodes with supported
    NVIDIA GPUs (Ampere and above) and a working GPU Operator SR-IOV vGPU configuration.

- [x] **API Extensions**
  - *List new or modified APIs:* No new user-facing APIs claimed for this STP.
  - *Testing impact:* N/A

- [x] **Test Environment Needs**
  - *See environment requirements in Section II.3 and testing tools in Section II.3.1*

- [x] **Topology Considerations**
  - *Describe topology requirements:* Bare-metal cluster with at least one RHCOS10
    worker node that has an NVIDIA SR-IOV-capable GPU, NVIDIA GPU Operator installed,
    and SR-IOV vGPU (VF) configuration applied
  - *Impact on test design:* Multi-VM sharing scenarios require enough VFs on that GPU for at least two concurrent VMs.

### **II. Software Test Plan (STP)**

This STP serves as the **overall roadmap for testing**, detailing the scope, approach,
resources, and schedule.

#### **1. Scope of Testing**

**Testing Goals**

- **[P0]** As a cluster admin, after SR-IOV vGPU is configured on an RHCOS10 GPU
  worker node, I can see the expected vGPU / VF resources advertised for scheduling,
  with capacity/allocatable reflecting the configured VF count
- **[P0]** As a VM operator, I can create a VM that requests an SR-IOV vGPU on an
  RHCOS10 worker node; the VM reaches Running and the guest detects exactly one GPU device
- **[P0]** As a VM operator, when no SR-IOV vGPU capacity remains, a VM that requests
  a vGPU stays Pending/unschedulable, and the scheduling event or message I see
  identifies exhausted SR-IOV vGPU capacity as the cause
- **[P1]** As a cluster admin, I can run two VMs concurrently, each with an SR-IOV
  vGPU VF from the same physical GPU; both reach Running and each guest detects its GPU
- **[P1]** As a VM operator, I can pause and unpause a Running VM with an SR-IOV vGPU;
  after unpause, connectivity recovers and the GPU remains visible in the guest
- **[P1]** As a VM operator, I can restart a VM with an SR-IOV vGPU; the VM returns to
  Running and the guest still detects the GPU after restart

**Out of Scope (Testing Scope Exclusions)**

- **VM continuity from RHCOS 9 to RHCOS 10 without VM spec reconfiguration**
  - *Rationale:* Explicit non-requirement of the feature epic; not in scope for
    this STP
  - *PM/Lead Agreement:* [Name/Date]

- **Cloud / non-bare-metal platforms**
  - *Rationale:* SR-IOV vGPU requires bare-metal NVIDIA GPU worker nodes; no dedicated
    cloud scenarios are planned
  - *PM/Lead Agreement:* [Name/Date]

**Test Limitations**

- **Testing blocked until NVIDIA GPU Operator SR-IOV vGPU support is available**
  (GA target 26.11 / Nov 2026). Functional scenarios cannot
  complete without that dependency.
  - *Sign-off:* Sudhakar Molli [Date]

- **Testing is limited to available NVIDIA SR-IOV-capable GPU hardware**
  - *Sign-off:* Sudhakar Molli [Date]

#### **2. Test Strategy**

**Functional**

- [x] **Functional Testing** — Validates that the feature works according to specified
  requirements and user stories
  - *Details:* Functional tests cover RHCOS10 node vGPU/VF advertisement, VM creation
    with SR-IOV vGPU, in-guest GPU visibility, pause/unpause, restart, concurrent
    multi-VM sharing of one physical GPU, and negative scheduling when capacity is
    exhausted.

- [x] **Automation Testing** — Confirms test automation plan is in place for CI and
  regression coverage
  - *Details:* All Section III scenarios will be automated in
    openshift-virtualization-tests, tracked in
    [CNV-74916](https://issues.redhat.com/browse/CNV-74916).

- [x] **Regression Testing** — Verifies that new changes do not break existing
  functionality
  - *Details:* Per NVIDIA (not CNV): on RHCOS10, mdev vGPU remains supported for
    pre-Ampere GPUs (for example Tesla T4). Ampere and newer GPUs do not support mdev on
    RHCOS10 — they use SR-IOV vGPU instead. Existing GPU passthrough and pre-Ampere mdev
    vGPU suites remain in regression on supported environments. New SR-IOV vGPU tests are
    additive for Ampere and newer GPUs on RHCOS10.

- [ ] **Self-Validation Testing** — Should any of the new tests be included in the
  self-validation test package?
  - *Details:* N/A. SR-IOV vGPU tests require specialized GPU hardware, RHCOS10 GPU
    nodes, and NVIDIA GPU Operator SR-IOV support not available in self-validation
    environments.

**Non-Functional**

- [ ] **Performance Testing**
  - *Details:* N/A for this STP. No CNV performance code changes; GPU performance is hardware-defined.

- [ ] **Scale Testing**
  - *Details:* N/A. Two-VM concurrent sharing is functional coverage only; no dedicated
    scale testing.

- [ ] **Security Testing**
  - *Details:* Out of scope for dedicated security scenarios. Isolation is enforced by
    NVIDIA GPU firmware / SR-IOV and device exposure is provided by the NVIDIA GPU
    Operator; OpenShift Virtualization adds no isolation mechanism. QE validates VM
    consumption of advertised SR-IOV vGPU devices only.

- [x] **Usability Testing** — Validates user experience for operational feedback
  - *Details:* Validate that the event or message shown when a VM cannot be scheduled
    identifies exhausted VF capacity as the specific cause (not a generic
    scheduling-failure message).

- [ ] **Monitoring**
  - *Details:* N/A. SR-IOV GPU performance monitoring is N/A; no new CNV metrics
    or alerts are required for this cycle.

**Integration & Compatibility**

- [x] **Compatibility Testing**
  - *Details:* Validate on OCP + OpenShift Virtualization 5.0.z,
     NVIDIA GPU Operator (SR-IOV vGPU-capable build).

- [ ] **Upgrade Testing**
  - *Details:* Upgrade path was evaluated. No dedicated upgrade scenario is required
    for this STP.

- [x] **Dependencies** — Blocked by deliverables from other components/products
  - *Details:* Blocked on NVIDIA GPU Operator SR-IOV vGPU support (target 26.11 /
    Nov 2026).

- [ ] **Cross Integrations**
  - *Details:* N/A for additional SIG-owned feature testing this cycle.

**Infrastructure**

- [ ] **Cloud Testing**
  - *Details:* N/A. Feature requires bare-metal nodes with NVIDIA SR-IOV-capable GPUs.
    No dedicated cloud scenarios.

#### **3. Test Environment**

- **Cluster Topology:** 3-control-plane / 3 worker nodes bare-metal with at least one RHCOS10
  worker node that has an NVIDIA SR-IOV-capable GPU

- **OCP & OpenShift Virtualization Version(s):** OCP 5.0.z with OpenShift Virtualization 5.0.z

- **CPU Virtualization:** VT-x (Intel) or AMD-V enabled

- **Compute Resources:** GPU nodes must have NVIDIA SR-IOV-capable GPU

- **Special Hardware:** NVIDIA SR-IOV-capable GPU (e.g., Ampere)

- **Storage:** ocs-storagecluster-ceph-rbd-virtualization

- **Network:** OVN-Kubernetes, IPv4

- **Required Operators:** NVIDIA GPU Operator with SR-IOV vGPU support
  (26.11) , OpenShift Virtualization

- **Platform:** Bare metal

- **Special Configurations:** SR-IOV GPU VFs configured and advertised
  for VM workloads prior to test execution

#### **3.1. Testing Tools & Frameworks**

- **Test Framework:** Standard

- **CI/CD:** N/A

- **Other Tools:** N/A

#### **4. Entry Criteria**

The following conditions must be met before testing can begin:

- [ ] Requirements and design documents are **approved and merged**
- [ ] Test environment can be **set up and configured** (see Section II.3 - Test Environment)
- [ ] RHCOS10 worker node with NVIDIA SR-IOV-capable GPU is available
- [ ] NVIDIA GPU Operator build with SR-IOV vGPU support  (GA 26.11)

#### **5. Risks**

**Timeline/Schedule**

- **Risk:** NVIDIA GPU Operator SR-IOV vGPU support targets Nov 2026 (26.11);
  testing may be delayed if the operator lands late
  - **Mitigation:** Start QE only after NVIDIA GPU Operator 26.11 GA with SR-IOV vGPU
    support is available.
  - *Estimated impact on schedule:* Testing start may slip until operator GA is usable
  - *Sign-off:* Sudhakar Molli [Date]

**Test Coverage**

- **Risk:** Coverage limited to Ampere GPU(s) — other SR-IOV-capable GPUs not validated
  - **Mitigation:** Ampere is representative of SR-IOV vGPU functionality; additional
    GPU models can be validated in future cycles when hardware becomes available
  - *Areas with reduced coverage:* SR-IOV-capable GPU models other than Ampere
  - *Sign-off:* Sudhakar Molli [Date]

**Test Environment**

- **None** — cluster availability risk is tracked once, under Resource Constraints.

**Untestable Aspects**

- **Risk:** Until NVIDIA GPU Operator SR-IOV vGPU support is available, end-to-end
  VM assignment cannot be validated
  - **Mitigation:** Gate execution on Entry Criteria (operator 26.11 GA); do not claim
    QE complete without P0 scenarios passing
  - *Reason untestable and mitigation approach:* Blocked on NVIDIA GPU Operator 26.11 GA;
    testing starts after Entry Criteria are met
  - *Sign-off:* Sudhakar Molli [Date]

**Resource Constraints**

- **Risk:** Limited availability of the required RHCOS10 Ampere GPU bare-metal cluster may
  delay SR-IOV vGPU testing
  - **Mitigation:** Coordinate with the team to schedule access; run P0 first when the
    cluster is available; keep automation ready so execution is not delayed once access
    is granted
  - *Missing resources or infrastructure:* Always-available RHCOS10 Ampere SR-IOV vGPU
    BM cluster
  - *Sign-off:* Sudhakar Molli [Date]

**Dependencies**

- **Risk:** Hard dependency on NVIDIA GPU Operator(26.11) SR-IOV vGPU support
  - **Mitigation:** Track operator 26.11 GA; keep Dev/PM informed if dependency slips
  - *Third-party services or blockers:* NVIDIA GPU Operator SR-IOV vGPU
  - *Sign-off:* Sudhakar Molli [Date]

---

### **III. Test Scenarios & Traceability**

- **[CNV-74901](https://issues.redhat.com/browse/CNV-74901)** — As a cluster administrator,
  I want SR-IOV vGPU / VF resources advertised on RHCOS10 GPU worker nodes after configuration
  - *Test Scenario:* [Tier 2] On an RHCOS10 GPU worker node with SR-IOV vGPU configured,
    verify the node advertises the expected vGPU/VF resources for scheduling
    (capacity/allocatable reflects configured VF count)
  - *Priority:* P0

- **[CNV-74901](https://issues.redhat.com/browse/CNV-74901)** — As a VM owner, I want to
  create a VM that consumes an SR-IOV vGPU VF so the guest can access GPU acceleration
  - *Test Scenario:* [Tier 2] Create a RHEL VM requesting an SR-IOV vGPU on the
    RHCOS10 GPU worker node; verify the VM reaches Running and the guest detects GPU device
  - *Priority:* P0

- **[CNV-74901](https://issues.redhat.com/browse/CNV-74901)** — As a VM operator, when no
  SR-IOV vGPU capacity remains, I want clear feedback that my VM cannot be scheduled
  - *Test Scenario:* [Tier 2] With all SR-IOV vGPU VFs consumed, create another VM
    requesting a vGPU; verify the VM remains Pending/unschedulable and the scheduling
    event or message shown to the user identifies exhausted SR-IOV vGPU capacity as
    the cause
  - *Priority:* P0

- **[CNV-74901](https://issues.redhat.com/browse/CNV-74901)** — As a cluster administrator,
  I want multiple VMs to share one physical GPU using SR-IOV vGPU VFs
  - *Test Scenario:* [Tier 2] Start two RHEL VMs concurrently, each requesting an
    SR-IOV vGPU from the same physical GPU; verify both are Running and each guest
    detects its GPU
  - *Priority:* P1

- **[CNV-74901](https://issues.redhat.com/browse/CNV-74901)** — As a VM operator, I want to
  pause and unpause a VM that has an SR-IOV vGPU without losing GPU access
  - *Test Scenario:* [Tier 2] With a Running RHEL VM that has an SR-IOV vGPU, pause
    and unpause the VM; verify connectivity recovers and the guest still detects the GPU
  - *Priority:* P1

- **[CNV-74901](https://issues.redhat.com/browse/CNV-74901)** — As a VM operator, I want to
  restart a VM that has an SR-IOV vGPU and still see the GPU afterward
  - *Test Scenario:* [Tier 2] Restart a RHEL VM that has an SR-IOV vGPU; verify the
    VM returns to Running and the guest detects the GPU after restart
  - *Priority:* P1

---

### **IV. Sign-off and Approval**

This Software Test Plan requires approval from the following stakeholders:

* **Reviewers:**
  - QE Member: dshchedr / @dshchedr
  - QE Member: vsibirsk / @vsibirsk
  - QE Architect: Ruth Netser / @rnetser
  - QE Member: SamAlber / @SamAlber
* **Approvers:**
  - QE Member: dshchedr / @dshchedr
  - QE Member: vsibirsk / @vsibirsk
  - QE Architect: Ruth Netser / @rnetser
  - Product Manager: Sudhakar Molli / @smolli-byte, Martin Tessun / @mtessun
  - Dev Lead: Luboslav Pivarc / @xpivarc
