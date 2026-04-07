# Simple-DAO-with-Proposal-Execution
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v5.0.0/contracts/access/Ownable.sol";

contract SimpleDAO is Ownable {
    struct Proposal {
        string description;
        uint256 voteFor;
        uint256 voteAgainst;
        uint256 deadline;
        bool executed;
        address target;
        bytes data;
    }

    Proposal[] public proposals;
    mapping(address => uint256) public votingPower;
    mapping(uint256 => mapping(address => bool)) public voted;

    event ProposalCreated(uint256 id, string description);
    event Voted(uint256 id, address voter, bool support);
    event ProposalExecuted(uint256 id);

    constructor() Ownable(msg.sender) {}

    function createProposal(string memory desc, address target, bytes memory data, uint256 durationDays) external onlyOwner {
        proposals.push(Proposal({
            description: desc,
            voteFor: 0,
            voteAgainst: 0,
            deadline: block.timestamp + (durationDays * 1 days),
            executed: false,
            target: target,
            data: data
        }));
        emit ProposalCreated(proposals.length - 1, desc);
    }

    function vote(uint256 proposalId, bool support) external {
        require(proposalId < proposals.length, "Invalid ID");
        Proposal storage p = proposals[proposalId];
        require(block.timestamp < p.deadline, "Voting ended");
        require(!voted[proposalId][msg.sender], "Already voted");

        uint256 power = votingPower[msg.sender];
        require(power > 0, "No voting power");

        if (support) p.voteFor += power;
        else p.voteAgainst += power;

        voted[proposalId][msg.sender] = true;
        emit Voted(proposalId, msg.sender, support);
    }

    function executeProposal(uint256 proposalId) external {
        Proposal storage p = proposals[proposalId];
        require(!p.executed, "Already executed");
        require(block.timestamp > p.deadline, "Voting not ended");
        require(p.voteFor > p.voteAgainst, "Proposal rejected");

        (bool success,) = p.target.call(p.data);
        require(success, "Execution failed");

        p.executed = true;
        emit ProposalExecuted(proposalId);
    }

    function setVotingPower(address user, uint256 power) external onlyOwner {
        votingPower[user] = power;
    }
}
