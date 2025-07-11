// SPDX-License-Identifier: MIT
pragma solidity 0.8.30;

contract Voting {
    // This is a voting contract that helps users(*voters) to vote by individuality.

    // what are the nouns in the question? the nouns form the data entity 
    // 1. Candidate
    // 2. Voters.
    // 3. Winner.
    // 4. admin.
    // 5. Vote.

    event Voted(address indexed voterAddress, address indexed candidateAddress);
    event VoterRegistered(uint indexed id, string name, address indexed userAddress);
    event VotingEventCreated(uint id, string name, uint startTime, uint endTime);
    event CandidateRegistered(uint id, string name, address indexed userAddress);

    address winner;
    address admin;

    uint constant votingPower = 1;
    uint voteCount;
    uint totalEventCount;

    struct Candidate {
        address candidateAddress;
        string name;
        uint id;
        uint eventid;
    }

    struct Voter {
        string name;
        address voterAddress;
        uint id;
        uint eventId;
        bool voted;
    }

    struct Vote {
        address candidateAddress;   // the address is only needed to vote
        uint id;
        address voter;
        uint voteWeight;
        bool voted;
    }

    struct VotingEvent {
        uint id;
        string name;
        address owner;
        address winner;
        bool started;
        bool ended;
        uint startTime;
        uint endTime;
        uint totalRegisteredCandidates;
        uint totalRegisteredVoters;
        uint totalVoteCast;
        EventStatus status;
    }

    enum EventStatus { pending, approved }
    
    mapping (uint id => VotingEvent ) public events;
    mapping (address addr => Candidate) public candidates;
    mapping (uint id => Voter) public voters;
    mapping (uint id => Vote) public voteCast;

    mapping (uint eventId => Voter) public eventVoters;
    mapping (uint eventId => Candidate) public eventCandidates;

    mapping (uint eventId => uint[]) public allVoterIdByEvent;
    mapping (uint eventId => uint[]) public allCandidateIdByEvent;

    mapping (uint eventId => address[]) public allVoterByEvent;
    mapping(address candidateAddress => Vote[]) public allVotesByCandidate;  // for the vote to be individualized.

    uint[] allCandidateId;
    uint[] allVoterId;
    uint[] allVoteId;
    uint[] allVotingEventId;

    constructor() {
        admin = msg.sender;
    }

    modifier onlyOwner (address owner) {
        if(owner == address(0)) {
            revert("Invalid owner address!");
        }
        if(msg.sender != owner) {
            revert("Only owner can register a candidate");
        }
        _;
    }

    modifier onlyAdmin () {
        if(msg.sender != admin) {
            revert("Only admin can register a candidate");
        }
        _;
    }

    function createVotingEvent(uint id, string memory name, uint startTime, uint endTime) public  {
        uint eventId = totalEventCount;

        if(events[eventId].status == EventStatus.pending) {
            revert("Event already created!");
        }

        // if() {
        //     revert("");
        // }

        VotingEvent storage votingEvent  = events[id];
        votingEvent.id = eventId;
        votingEvent.name = name;
        votingEvent.owner = msg.sender;
        votingEvent.winner = address(0);
        votingEvent.started = false;
        votingEvent.ended = false;
        votingEvent.startTime = startTime;
        votingEvent.endTime = endTime;
        votingEvent.totalRegisteredCandidates = 0;
        votingEvent.status = EventStatus.pending;

        allVotingEventId.push(id);
        events[eventId] = votingEvent;

        emit VotingEventCreated(id, name, startTime, endTime);
    }

    function approveEvent(uint eventId) public onlyAdmin {
        if(events[eventId].status != EventStatus.pending) {
            revert("Event is already approved!");
        }

        if(events[eventId].status != EventStatus.approved){
            events[eventId].status = EventStatus.approved;
        }
    }

    function registerCandidate(address addr, string memory name, uint id, uint eventId) public onlyOwner(events[eventId].owner) {
        if(events[eventId].status != EventStatus.approved) {
            revert("Event is not yet approved!");
        }

        if(candidates[addr].candidateAddress != address(0)) {
            revert("Candidate already registered!");
        }

        if(events[eventId].ended) {
            revert("Event is ended!");
        }

        Candidate storage candidate = candidates[addr];
        candidate.name = name;
        candidate.candidateAddress = addr;
        candidate.id = id;

        allCandidateId.push(id);
        allCandidateIdByEvent[eventId].push(id);
        events[eventId].totalRegisteredCandidates = allCandidateIdByEvent[eventId].length;

        eventCandidates[eventId] = candidate;

        emit CandidateRegistered(id, name, addr);
    }

    function startEvent(uint eventId) public onlyOwner(events[eventId].owner) {
        if(block.timestamp != events[eventId].startTime) {
            revert("Event time not reached!");
        }
        events[eventId].started = true;
    }

    function registerVoter(address addr, string memory name, uint id, uint eventId) public {
        if(voters[id].voterAddress != address(0)) {
            revert("Voter already registered!");
        }

        if(block.timestamp != events[eventId].startTime) {
            revert("Event time not reached!");
        }

        Voter storage voter = voters[id];
        voter.name = name;
        voter.voterAddress = addr;
        voter.id = id;

        allVoterId.push(id);
        allVoterIdByEvent[eventId].push(id);
        events[eventId].totalRegisteredVoters = allVoterIdByEvent[eventId].length;

        eventVoters[eventId] = voter;

        emit VoterRegistered(id, name, addr);
    }

    function castVote(address candidateAddr, uint eventId) public {
        if(block.timestamp >= events[eventId].endTime) {
            revert("Event is over!");
        }

        if(events[eventId].started = false) {
            revert("Event is not yet started!");
        }

        if(events[eventId].status != EventStatus.approved) {
            revert("Event is not yet approved!");
        }

        if(eventVoters[eventId].voted) {
            revert("Voter already casted a vote for this event");
        }

        // candidates[candidateAddr].eventid
        uint voteId = voteCount;

        Vote storage vote = voteCast[voteId];

        vote.candidateAddress = candidateAddr;
        vote.id = voteId;
        vote.voter = msg.sender;
        vote.voted = true;

        // events[eventId]
        allVotesByCandidate[candidateAddr].push(vote);
        allVoterByEvent[eventId].push(vote.voter);
        // allValidVoterByCandidate[e]

        emit Voted(msg.sender, candidateAddr);

        if(block.timestamp >= events[eventId].endTime) {
            events[eventId].ended = true;
        }
        voteCount++;
    }

    function getAllVotesByCandidate(address addr) public view returns(Vote[] memory) {
        return allVotesByCandidate[addr];
    }

    function getTotalVotesByCandidates(address addr) public view returns(uint) {
        return allVotesByCandidate[addr].length;
    }

    function getTotalCandidatesByEvent(uint eventId) public view returns(uint) {
        return allCandidateIdByEvent[eventId].length;
    }

    function getTotalVoterByEvent(uint eventId) public  view returns(uint) {
        return allVoterByEvent[eventId].length;
    }

    function getTotalVoterByEvent(uint ) public view returns() {

    }

    function getAllVotersByEvent(uint eventId) public view returns(address[] memory) {
        return allVoterByEvent[eventId];
    }

    function getEventWinner(uint eventId) public view onlyOwner(events[eventId].owner) returns(address) {
        
    }


    // function getAllRegistered(params) {
    //     code
    // }
}
