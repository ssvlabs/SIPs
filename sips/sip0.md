| Author      | Title          | Category | Status   |
|-------------|----------------|----------|----------|
| Alon Muroch | SIP life cycle | Core     | approved |

**Summary**  
Describes an SIPs life cycle from being submitted all the way to being accepted, standardized and merged into the spec (if applicable)

**Specification**  
First please use [SIP Template](../template_sip.md) for submitting new SIPs.  

**SIP life cycle**
- draft: the first step, set when an SIP is first pushed as a pull-request
- open-for-discussion: a step meant to indicate the SIP is in active dicussion. Changes are still possible. A [github discussion board](https://github.com/bloxapp/SIPs/discussions) will be opened (a link set in the SIP page)
- last-call: a final step before the SIP is marked as approved, unless major issues are found the SIP will be approved
- approved: the SIP was accepted and waiting execution (if applicable in case of spec changes, etc.)
- spec-merged: an optional step for SIP requiring changes to the spec
- rejected: the SIP was rejected and all discussions/ processes related to it are stopped

**Dependency SIP**  
An SIP requiring some other SIP as a dependency will not move to last-call before the dependency SIP is approved.
If dependency SIP is rejected the SIP will be rejected as well (unless changed to remove dependency)


