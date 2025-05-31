## Coinbase EOP-7702Proxy
[Contest Details](https://cantina.xyz/competitions/b0a948cd-c861-4807-b36e-d680d82598bf)

### [Info-01] Missing `tokenReceived` Callback in EIP7702Proxy Prevents ERC-223 Token Receipt Post-Upgrade

#### Summary
ERC-223 token transfers to an EOA upgraded to a smart contract via EIP-7702 using the `EIP7702Proxy` will always revert because the proxy does not implement the required `tokenReceived` callback function, breaking compatibility with the ERC-223 standard and limiting token-handling capabilities post-upgrade.

#### Finding Description
The ERC-223 standard requires that any recipient running code—whether a contract or an EOA with code—implements the `tokenReceived` function to accept token transfers. This callback prevents accidental or incorrect deposits by ensuring the recipient can handle the tokens, as noted in discussions like this [Ethereum Magicians](https://ethereum-magicians.org/t/eip-7702-set-eoa-account-code/19923/368) post. Without this function, an ERC-223 transfer reverts when the token contract attempts to call it. The ERC-223 standard, as outlined in EIP-223, defines two transfer functions to enhance token transfer safety:

- `transfer(address _to, uint _value)`: A backwards-compatible version with ERC-20, which transfers tokens and invokes `tokenReceived(address, uint256, bytes`) on `_to` if it’s a contract, reverting if the callback isn’t implemented.
- `transfer(address _to, uint _value, bytes calldata _data)`: The primary function for ERC-223 transfers, which also invokes tokenReceived on `_to` if it’s a contract, reverting if unimplemented, but skips the call if _to is an EOA (no code).
Per the standard, a contract is identified by checking if _to has code (e.g., via extcodesize), and `tokenReceived` must be implemented by any contract recipient to accept tokens, preventing accidental deposits to incompatible contracts. If _to is an EOA (no code), the transfer proceeds without invoking `tokenReceived`.

The proxy implements `onERC721Received` and `onERC1155Received` for ERC-721 and ERC-1155 compatibility but lacks `tokenReceived` for ERC-223. Post-upgrade, an EOA becomes a contract with code, so ERC-223 transfer calls will invoke `tokenReceived`. Since neither the proxy nor a typical implementation provides this function, the call fails, reverting the transfer.

This breaks the ERC-223 security guarantee that tokens are only sent to capable recipients. When an ERC-223 token contract executes a transfer:

The call to `tokenReceived` fails because neither the proxy nor a typical implementation includes it, causing the transfer to revert. This isn’t an exploitable vulnerability but a compatibility gap that hinders the proxy’s utility for ERC-223 token interactions post-EIP-7702 upgrade.

#### Impact Explanation
The absence of `tokenReceived` doesn’t undermine the proxy’s core security (initialization protection) or cause asset loss, but it prevents upgraded EOAs from receiving ERC-223 tokens via either transfer function when _to has code. This could disrupt wallet providers using EIP7702Proxy to upgrade EOAs into smart contract wallets, as users might expect support for ERC-223—a standard designed for safer transfers. Post-Pectra, as EIP-7702 adoption grows, this gap could hinder interoperability with ERC-223 dApps, reducing the proxy’s utility. It’s not “High” impact since it’s a compatibility issue, not a security flaw, but it’s notable given the proxy’s goal of enabling versatile wallets.

#### Likelihood Explanation
Wehn an EOA upgrades via EIP7702Proxy, it gains code, triggering the `tokenReceived` check in ERC-223 transfers. Without this callback, both` transfer(address, uint)` and `transfer(address, uint, bytes) `will revert when targeting the proxy. Given ERC-223’s growing use for secure token handling and the proxy’s intended role in smart contract wallets, this issue will likely manifest frequently post-upgrade, especially as wallet providers adopt EIP-7702.

#### Proof of Concept
Create a new PoC.t.sol file in the test folder and paste this
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.23;

import {Test} from "forge-std/Test.sol";
import {CoinbaseSmartWallet} from "../lib/smart-wallet/src/CoinbaseSmartWallet.sol";
import {EIP7702Proxy} from "../src/EIP7702Proxy.sol";
import {NonceTracker} from "../src/NonceTracker.sol";
import {DefaultReceiver} from "../src/DefaultReceiver.sol";
import {CoinbaseSmartWalletValidator} from "../src/validators/CoinbaseSmartWalletValidator.sol";

// ERC-223 Token for testing
contract ERC223Token {
    mapping(address => uint256) public balanceOf;

    constructor() {
        balanceOf[msg.sender] = 1000e18; // Mint initial supply
    }

    function transfer(address _to, uint _value, bytes calldata _data) public returns (bool) {
        require(balanceOf[msg.sender] >= _value, "Insufficient balance");
        if (_to.code.length > 0) {
            (bool success, ) = _to.call(
                abi.encodeWithSignature("tokenReceived(address,uint256,bytes)", msg.sender, _value, _data)
            );
            require(success, "Recipient rejected transfer");
        }
        balanceOf[msg.sender] -= _value;
        balanceOf[_to] += _value;
        return true;
    }

    function transfer(address _to, uint _value) external returns (bool) {
        require(balanceOf[msg.sender] >= _value, "Insufficient balance");
        if (_to.code.length > 0) {
            (bool success, ) = _to.call(
                abi.encodeWithSignature("tokenReceived(address,uint256)", msg.sender, _value)
            );
            require(success, "Recipient rejected transfer");
        }
        balanceOf[msg.sender] -= _value;
        balanceOf[_to] += _value;
        return true;
    }
}

contract CoinbaseSmartWalletValidatorTest is Test {
    uint256 constant _EOA_PRIVATE_KEY = 0xA11CE;
    address payable _eoa;
    uint256 constant _NEW_OWNER_PRIVATE_KEY = 0xB0B;
    address payable _newOwner;

    EIP7702Proxy _proxy;
    CoinbaseSmartWallet _implementation;
    NonceTracker _nonceTracker;
    DefaultReceiver _receiver;
    CoinbaseSmartWalletValidator _validator;
    ERC223Token _token;

    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    bytes32 _IMPLEMENTATION_SET_TYPEHASH = keccak256(
        "EIP7702ProxyImplementationSet(uint256 chainId,address proxy,uint256 nonce,address currentImplementation,address newImplementation,bytes callData,address validator)"
    );

    function setUp() public {
        _eoa = payable(vm.addr(_EOA_PRIVATE_KEY));
        _newOwner = payable(vm.addr(_NEW_OWNER_PRIVATE_KEY));

        _implementation = new CoinbaseSmartWallet();
        _nonceTracker = new NonceTracker();
        _receiver = new DefaultReceiver();
        _validator = new CoinbaseSmartWalletValidator(_implementation);
        _proxy = new EIP7702Proxy(address(_nonceTracker), address(_receiver));
        _token = new ERC223Token();

        vm.etch(_eoa, address(_proxy).code);
    }

    // Existing tests omitted for brevity

    function test_reverts_ERC223Transfer_afterUpgrade() public {
        // Initialize proxy with an owner
        bytes memory initArgs = _createInitArgs(_newOwner);
        bytes memory signature =
            _signSetImplementationData(_EOA_PRIVATE_KEY, initArgs, address(_implementation), address(_validator));
        EIP7702Proxy(_eoa).setImplementation(address(_implementation), initArgs, address(_validator), signature, true);

        // Verify proxy has code (simulating EIP-7702 upgrade)
        assertGt(_eoa.code.length, 0, "EOA should have code after upgrade");

        // Attempt ERC-223 transfer
        vm.prank(address(this)); // Token deployer has initial supply
        vm.expectRevert("Recipient rejected transfer");
        _token.transfer(_eoa, 100e18, "");

        // Attempt ERC-223 transfer with data
        vm.prank(address(this));
        vm.expectRevert("Recipient rejected transfer");
        _token.transfer(_eoa, 100e18, hex"1234");
    }

    // Helper functions from original suite
    function _createInitArgs(address owner) internal pure returns (bytes memory) {
        bytes[] memory owners = new bytes[](1);
        owners[0] = abi.encode(owner);
        bytes memory ownerArgs = abi.encode(owners);
        return abi.encodePacked(CoinbaseSmartWallet.initialize.selector, ownerArgs);
    }

    function _signSetImplementationData(
        uint256 signerPk,
        bytes memory initArgs,
        address implementation,
        address validator
    ) internal view returns (bytes memory) {
        bytes32 initHash = keccak256(
            abi.encode(
                _IMPLEMENTATION_SET_TYPEHASH,
                0,
                _proxy,
                _nonceTracker.nonces(_eoa),
                _getERC1967Implementation(address(_eoa)),
                address(implementation),
                keccak256(initArgs),
                address(validator)
            )
        );
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(signerPk, initHash);
        return abi.encodePacked(r, s, v);
    }

    function _getERC1967Implementation(address proxy) internal view returns (address) {
        return address(uint160(uint256(vm.load(proxy, _IMPLEMENTATION_SLOT))));
    }
}
```

```bash
Run the test
forge test --mt test_reverts_ERC223Transfer_afterUpgrade
```

#### Recommendation
Add `tokenReceived` to `DefaultReceiver` (since it handles token callbacks) to ensure ERC-223 support