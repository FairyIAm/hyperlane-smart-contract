# hyperlane-smart-contract
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@hyperlane-xyz/core/contracts/interfaces/IMessageRecipient.sol";
import "@hyperlane-xyz/core/contracts/interfaces/IMailbox.sol";

contract HyperlaneBridge is IMessageRecipient {
    address public admin;
    IMailbox public mailbox;
    mapping(bytes32 => bool) public processedMessages;

    event TokensLocked(address indexed user, uint256 amount, address token, uint32 destinationChain, bytes32 messageId);
    event TokensUnlocked(address indexed user, uint256 amount, address token, bytes32 messageId);
    event CrossChainMessageReceived(uint32 origin, bytes32 sender, string message);

    constructor(address _mailbox) {
        admin = msg.sender;
        mailbox = IMailbox(_mailbox);
    }

    modifier onlyAdmin() {
        require(msg.sender == admin, "Only admin can perform this action");
        _;
    }

    function lockTokens(address token, uint256 amount, uint32 destinationChain) external {
        require(amount > 0, "Amount must be greater than 0");
        require(IERC20(token).transferFrom(msg.sender, address(this), amount), "Transfer failed");

        bytes32 messageId = keccak256(abi.encodePacked(msg.sender, amount, token, destinationChain, block.timestamp));
        emit TokensLocked(msg.sender, amount, token, destinationChain, messageId);
    }

    function unlockTokens(address user, address token, uint256 amount, bytes32 messageId) external onlyAdmin {
        require(!processedMessages[messageId], "Message already processed");
        require(IERC20(token).balanceOf(address(this)) >= amount, "Not enough liquidity");

        processedMessages[messageId] = true;
        require(IERC20(token).transfer(user, amount), "Transfer failed");
        emit TokensUnlocked(user, amount, token, messageId);
    }

    function receiveMessage(uint32 _origin, bytes32 _sender, bytes calldata _body) external override {
        require(msg.sender == address(mailbox), "Unauthorized sender");
        require(!processedMessages[keccak256(_body)], "Message already processed");

        string memory message = string(_body);
        processedMessages[keccak256(_body)] = true;
        emit CrossChainMessageReceived(_origin, _sender, message);
    }

    function sendCrossChainMessage(uint32 _destinationDomain, address _recipient, string calldata _message) external onlyAdmin {
        bytes memory body = bytes(_message);
        mailbox.dispatch(_destinationDomain, bytes32(uint256(uint160(_recipient))), body);
    }
}
gtyhkkkm
