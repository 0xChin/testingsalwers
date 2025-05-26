
# Governance Proposal Validator


## Overview


This document specifies the `ProposalValidator` contract, designed to enable permissionless proposals in the Optimism
governance system. The contract allows proposal submissions based on predefined rules and automated checks, removing the
need for manual gate keeping.


## Design


The `ProposalValidator` manages the proposal lifecycle through three main functions:

- `submitProposal`: Records new proposals (it doesn’t actually)
- `approveProposal`: Handles proposal approvals
- `moveToVote`: Transitions approved proposals to voting phase

The contract also integrates with EAS (Ethereum Attestation Service) to verify authorized proposers for specific proposal
types. For detailed flows of each proposal, see [design docs](https://github.com/ethereum-optimism/design-docs/pull/260).


## Roles


The contract has a single `owner` role (Optimism Foundation) with permissions to:

- Set minimum voting power threshold for delegate approvals
- Configure voting cycle parameters
- Set maximum token distribution limits for proposals

## Interface


### Public Functions


`submitProposal`


Submits a proposal for approval and voting. Based on the `ProposalType` provided this will require different validation
checks and actions.

- MUST only be called for `ProtocolOrGovernorUpgrade`, `MaintenanceUpgradeProposals`, or `CouncilMemberElections` types
- MUST be called by an approved address
- MUST check if the proposal is a duplicate
- MUST use the correct proposal type configurator from the `_proposalTypesData`
- MUST provide a valid attestation UID
- MUST NOT transfer any tokens or change any allowances
- MUST emit `ProposalSubmitted` event
- MUST store proposal data

For `GovernanceFund` and `CouncilBudget` types:

- The user MUST use the `submitFundingProposal` that uses specific `calldata` pre-defined by the owner

Note: `MaintenanceUpgradeProposals` type can move straight to voting if all submission checks pass, unlike the rest of the
proposals where they need to collect a number of approvals by top delegates in order to move to vote. This call should be
atomic.


```plain text
function submitProposal(
    address[] memory _targets,
    uint256[] memory _values,
    bytes[] memory _calldatas,
    string memory _description,
    ProposalType _proposalType,
    bytes32 _attestationUid
) external returns (bytes32 proposalHash_);
```


`submitFundingProposal`


Submits a `GovernanceFund` or `CouncilBudget` proposal type that transfers OP tokens for approval and voting.

- MUST only be called for `GovernanceFund` or `CouncilBudget` proposal type
- CAN be called by anyone
- MUST check if the proposal is a duplicate
- MUST use the correct proposal type configurator from the `_proposalTypesData`
- MUST use the `Predeploys.GOVERNANCE_TOKEN` and `TRANSFER_SIGNATURE` to create the `calldata`
- MUST NOT request to transfer more than `distributionThreshold` tokens
- MUST emit `ProposalSubmitted` event
- MUST store proposal data

```plain text
function submitFundingProposal(
    address _to,
    uint256 _amount,
    string memory _description,
    ProposalType _proposalType,
) external returns (bytes32 proposalHash_);
```


`approveProposal`


Approves a proposal before being moved for voting, used by the top delegates.

- MUST check if proposal hash corresponds to a valid proposal
- MUST check if caller is has enough voting power to call the function and approve a proposal
- MUST check if caller has already approved the same proposal
- MUST store the approval vote
- MUST emit `ProposalApproved` when successfully called

```plain text
function approveProposal(bytes32 _proposalHash) external
```


`moveToVote`


Checks if the provided proposal is ready to move for voting. Based on the Proposal Type different checks are being
validated. If all checks pass then `OptimismGovernor.propose` is being called to forward the proposal for voting. This
function can be called by anyone.


For `ProtocolOrGovernorUpgrade`:

- MUST check if provided data produces a valid `proposalHash`
- Proposal MUST have gathered X amount of approvals by top delegates
- MUST check if proposal has already moved for voting
- MUST emit `ProposalMovedToVote` event

For `MaintenanceUpgradeProposals`:

- This type does not require any checks and is being forwarded to the Governor contracts, this should happen atomically.
- MUST emit `ProposalMovedToVote` event

For `CouncilMemberElections`, `GovernanceFund` and `CouncilBudget`:

- MUST check if provided data produce the same `proposalHash`
- Proposal MUST have gathered X amount of approvals by top delegates
- Proposal MUST be moved to vote during a valid voting cycle
- MUST check if proposal has already moved for voting
- MUST check if the total amount of tokens that can possible be distributed during this voting cycle does not go over the
`VotingCycleData.votingCycleDistributionLimit`
- MUST emit `ProposalMovedToVote` event

```plain text
function moveToVote(
    address[] memory _targets,
    uint256[] memory _values,
    bytes[] memory _calldata,
    string memory description
)
    external
    returns (uint256 governorProposalId_)
```


`canSignOff`


Returns true if a delegate address has enough voting power to approve a proposal.

- Can be called by anyone
- MUST return TRUE if the delegates' voting power is above the `minimumVotingPower`

```plain text
function canSignOff(address _delegate) public view returns (bool canSignOff_)
```


`setMinimumVotingPower`


Sets the minimum voting power a delegate must have in order to be eligible to approve a proposal.

- MUST only be called by the owner of the contract
- MUST change the existing minimum voting power to the new
- MUST emit `MinimumVotingPowerSet` event

```plain text
function setMinimumVotingPower(uint256 _minimumVotingPower) external
```


`setVotingCycleData`


Sets the start and the duration of a voting cycle.

- MUST only be called by the owner of the contract
- MUST NOT change an existing voting cycle
- MUST emit `VotingCycleSet` event

```plain text
function setVotingCycleData(
    uint256 _cycleNumber,
    uint256 _startBlock,
    uint256 _duration,
    uint256 _distributionLimit
) external
```


`setDistributionThreshold`


Sets the maximum distribution threshold a proposal can request.

- MUST only be called by the owner of the contract
- MUST change the previous threshold to the new one
- MUST emit `DistributionThresholdSet` event

```plain text
function setDistributionThreshold(uint256 _threshold) external
```


`setProposalTypeApprovalThreshold`


Sets the number of approvals a specific proposal type should have before being able to move for voting.

- MUST only be called by the owner of the contract
- MUST change the previous value to the new one
- MUST emit `ProposalTypeApprovalThresholdSet` event

```plain text
function setProposalTypeApprovalThreshold(
    uint8 _proposalTypeId,
    uint256 _approvalThreshold
) external
```


### Properties


`ATTESTATION_SCHEMA_UID`


The EAS' schema UID that is used to verify attestation for approved addresses that can submit proposals for specific
`ProposalTypes`.


```plain text
/// Schema { approvedProposer: address, proposalType: uint8 }
bytes32 public immutable ATTESTATION_SCHEMA_UID;
```

