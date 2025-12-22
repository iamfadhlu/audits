# SOMNIA
[Contest Details](https://dashboard.hackenproof.com/user/programs/somnia-audit-contest)


## [L-01] WebSocket handshake uses a fixed client key

### Summary
The Somnia WebSocket client implementation violates RFC 6455 by using a hardcoded Sec-WebSocket-Key value for all handshake requests instead of generating fresh random keys per connection. websocket_serialisation.cc:69 This creates a client fingerprinting vulnerability that can lead to traffic blocking and peer-to-peer network isolation. This vulnerability has Medium severity due to potential network connectivity issues and consensus disruption.

### Finding Description
The vulnerability exists in the SerialiseClientHandshakeRequest() function which hardcodes the RFC 6455 example value "dGhlIHNhbXBsZSBub25jZQ==" for every WebSocket handshake. websocket_serialisation.cc:69 This violates the WebSocket specification requirement that clients must send a freshly generated, unpredictable 16-byte value encoded in Base64.

The WebSocket client is instantiated with is_server{false} indicating client mode, websocket_client.h:36 and calls the handshake serialization during connection establishment. websocket_client.cc:21-22 However, the serialization function always emits the same static key value regardless of the connection instance.

The server-side handshake response logic correctly processes the received key by concatenating it with the WebSocket magic GUID and computing the SHA-1 hash for the Sec-WebSocket-Accept response. websocket_serialisation.cc:407-416 While this server logic is standards-compliant, the client's use of a constant key creates the vulnerability.

### Impact Explanation
Using a fixed handshake key makes all Somnia WebSocket clients easily identifiable through network traffic analysis. Middleboxes, proxies, or security appliances that enforce RFC compliance may reject connections with non-random keys, leading to connection failures. Additionally, anti-abuse systems can fingerprint and throttle or block Somnia clients based on the predictable handshake pattern, causing selective network isolation that degrades transaction and block propagation.

### Likelihood Explanation
The likelihood is medium as it depends on network infrastructure that actively monitors or filters WebSocket handshakes. While many systems may be lenient with RFC compliance, enterprise networks, cloud load balancers, and security appliances increasingly implement strict protocol validation. The vulnerability becomes more probable in production deployments where nodes must traverse multiple network boundaries.

### Recommendation
Consider modifying SerialiseClientHandshakeRequest() to generate a fresh 16-byte random value for each connection. One approach would be to replace the hardcoded key with cryptographically random data:

```cpp
void WebsocketSerialisation::SerialiseClientHandshakeRequest(
    const std::string& ip_address, std::uint16_t port, const std::string& url_path,
    FunctionView<void(ByteSpanConst)> send_output) {
  static thread_local std::string message;
  message.clear();
  message += "GET " + url_path + " HTTP/1.1\r\n";
  message += "Host: " + ip_address + ":" + std::to_string(port) + "\r\n";
  message += "Upgrade: websocket\r\n";
  message += "Connection: Upgrade\r\n";
  
  // Generate fresh 16-byte random key per connection
  std::array<std::uint8_t, 16> nonce;
  GenerateSecureRandomBytes(nonce.data(), nonce.size());
  std::string key_b64;
  EncodeBase64Data(nonce, key_b64);
  message += "Sec-WebSocket-Key: " + key_b64 + "\r\n";
  
  message += "Sec-WebSocket-Version: 13\r\n";
  message += "\r\n";
  send_output(StringViewToByteSpan(message));
  // ... rest of function
}
```
This change maintains compatibility with the existing server handshake logic while ensuring RFC compliance and preventing client fingerprinting.

## [L-02] WebSocket client frames are never masked

### Summary
The Somnia WebSocket client implementation violates RFC 6455 by never masking frames sent to servers. websocket_serialisation.cc:49 The masking is globally disabled via kUseMask = false, and the client handshake uses a fixed Sec-WebSocket-Key instead of generating fresh nonces. This vulnerability has Medium severity as it can cause interoperability failures with RFC-compliant servers and middleboxes, potentially leading to peer-to-peer network disruptions.

### Finding Description
The vulnerability exists in the WebSocket serialization implementation where masking is controlled by a global constant kUseMask = false that disables masking for all frame types. websocket_serialisation.cc:49 The SerialiseFrame() function only applies masking when this flag is true, but since it's hardcoded to false, client frames are never masked. websocket_serialisation.cc:284-294

The WebSocket client is instantiated with is_server{false} in the client wrapper, websocket_client.h:36 but this parameter is not used to control masking behavior. The masking logic exists in the serialization code but is gated behind the disabled flag. websocket_serialisation.cc:326-331

Additionally, the client handshake uses a fixed Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ== (the RFC example value) instead of generating a fresh 16-byte nonce per connection. websocket_serialisation.cc:69 The close frame implementation also bypasses the normal serializer and uses a hardcoded buffer with an all-zero masking key. websocket_serialisation.cc:100

### Impact Explanation
RFC 6455 mandates that all client-to-server frames must be masked to prevent cache poisoning attacks through intermediary proxies. Standards-compliant servers, proxies, and load balancers may reject or drop unmasked client frames, leading to silent message loss or complete connection failures. In Somnia's peer-to-peer network context, this can prevent nodes from exchanging transactions and blocks across certain network configurations, potentially causing mempool inconsistencies and consensus divergence.

### Likelihood Explanation
The likelihood is medium as it depends on the network infrastructure between Somnia nodes. While some servers and middleboxes may be lenient with RFC compliance, many enterprise-grade proxies and security appliances strictly enforce WebSocket standards. The vulnerability becomes more likely in production deployments where nodes communicate through corporate networks or cloud load balancers that implement RFC-compliant filtering.

### Recommendation
Consider implementing role-based masking where clients mask frames but servers do not. One approach would be to modify the SerialiseFrame() function to use the is_server parameter:

```cpp
void WebsocketSerialisation::SerialiseFrame(int frame_type, ByteSpanConst message,
                                            std::vector<std::uint8_t>& output) {
  // Use role-based masking: clients must mask, servers must not
  bool use_mask = !is_server;
  
  // Generate fresh masking key per frame for clients
  std::uint8_t masking_key[4];
  if (use_mask) {
    // Generate cryptographically random masking key
    GenerateRandomBytes(masking_key, 4);
  }
  
  // Apply masking logic using use_mask instead of kUseMask
  // ... rest of frame serialization with dynamic masking
}
```

Additionally, modify SerialiseClientHandshakeRequest() to generate a fresh Base64-encoded 16-byte nonce for each connection instead of using the fixed RFC example value. Route control frames like close/ping/pong through the normal serializer to ensure consistent masking behavior.

### Notes
The WebSocket implementation correctly handles receiving masked frames from peers, websocket_serialisation.cc:218-227 so the issue is specifically with outbound frame masking. The masking infrastructure is already present in the codebase but disabled, making the fix relatively straightforward without requiring architectural changes to the transport layer.

## [L-03] ECDSA verify is malleability-tolerant without low-S enforcement

### Summary
The Somnia ECDSA verification implementation accepts both canonical (low-S) and non-canonical (high-S) signatures for the same message and public key, creating potential transaction ID ambiguity. While the core ECDSA functions are malleability-tolerant, the transaction processing system implements EIP-2 protection at the protocol level. This vulnerability has Low severity due to existing protocol-level mitigations that prevent exploitation in transaction contexts.

### Finding Description
The vulnerability exists in the core ECDSA verification function VerifySignature() which only validates that signature recovery produces the expected public key without enforcing canonical signature forms. ecdsa.cc:196-199 The function relies solely on public key recovery equality rather than signature canonicality.

The RecoverPublicKeyFromSignature() function accepts any compact ECDSA signature with recovery ID in the range 0-3 and performs no low-S enforcement or normalization. ecdsa.cc:128-159 This allows both (r, s) and (r, n-s) signature forms to verify successfully for the same message and public key, as both recover to the same public key.

The signature building and parsing utilities BuildSignatureFromParts() and BreakSignatureIntoParts() also handle signature components without any canonicalization or range validation beyond basic recovery ID bounds. ecdsa.cc:202-225

### Impact Explanation
The malleability tolerance means that multiple bytewise-distinct signatures can authenticate the same logical statement. If any system components rely on signature bytes as unique identifiers or for hash-based indexing, this could create ambiguity where the same authentic transaction appears with different identifiers. However, the practical impact is limited by protocol-level protections in the transaction validation system.

### Likelihood Explanation
The likelihood is low because Somnia's transaction processing implements EIP-2 malleability protection through ValidateSignatureValues() which rejects high-S signatures before they reach the ECDSA verification functions. transaction_encoding.cc:705-720 This validation occurs in both legacy and typed transaction processing paths, effectively preventing malleability exploitation in transaction contexts. transaction_encoding.cc:375-377

### Recommendation
Consider implementing low-S enforcement directly in the ECDSA verification functions to ensure canonical signatures across all use cases. One approach would be to modify VerifySignature() to normalize signatures and reject high-S values:

```cpp
bool VerifySignature(const PublicKey& public_key, const Hash& message_hash,
                     const Signature& signature) {
  // Parse signature and check if s is in canonical (low) form
  uint256 r, s;
  bool v;
  BreakSignatureIntoParts(signature, v, r, s);
  
  // Reject high-S signatures (EIP-2 compliance)
  static const uint256 secp256k1halfN = 
      uint256::FromHexString("0x7FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF5D576E7357A4501DDFE92F46681B20A0");
  if (s > secp256k1halfN) {
    return false;
  }
  
  return RecoverPublicKeyFromSignature(message_hash, signature) == Optional{public_key};
}
```
This ensures that all ECDSA verification enforces canonical signatures regardless of the calling context, providing defense-in-depth beyond the existing transaction-level protections.

### Notes
The transaction batching system also uses ECDSA signatures for non-Merkle transactions and benefits from the same protocol-level validation. mempool_transaction_validator.h:205-212 The core issue only affects contexts where the ECDSA functions might be used outside of the transaction validation pipeline, making it primarily a hardening concern rather than an active vulnerability.