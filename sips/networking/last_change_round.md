| Author     | Title                                             | Category | Status |
|------------|---------------------------------------------------|----------|--------|
| Niv Muroch | Handle non committee change round for all signers | Core     | Draft  |

**Summary**  
Allow nodes to save non committee (validators that are not belong to the node) change round messages in order to serve
other nodes when requesting last change round messages.

Rational & Design Goals  
When using subnets instead of topic per validator there is a need for supporting get last change round from any peer in
the subnet. in order to supports that, each node (peer) needs to handle incoming change round messages and save it per
signer and only the highest once. the reason for saving messages per signer is to keep the same behavior as it was with
validator topic.

----------

**Terms For Updating**

``
Terms when should update last change round message for non committee validator.
``  

- When local not exist. New message saves instantly.  
- New change round message should be with higher or equal height than the local message.
If higher, save instantly. if not, check if round is higher. if so, update.
- need to maintenance the last change round message for each of the signers in the specific validator committe.

