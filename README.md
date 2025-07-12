// SPDX-License-Identifier: MIT
pragma solidity 0.8.30;

contract Voting {
    // This is a voting contract that helps users(*voters) to vote by individuality.

    event Voted(address indexed voterAddress, address indexed candidateAddress);
    event VoterRegistered(uint indexed id, string name, address indexed userAddress);
    event VotingEventCreated(uint id, string name, uint startTime, uint endTime);
    event CandidateRegistered(uint id, string name, address indexed userAddress);

    address winner;
    address admin;

    uint voteCount;
    uint eventCount;
    uint candidateCount;

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
        Voter voter;
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
    mapping (uint id => Candidate) public candidates;
    mapping (uint id => Voter) public voters;
    mapping (uint id => Vote) public voteCast;

    mapping (uint eventId => Voter) public eventVoters;
    mapping (uint eventId => Candidate) public eventCandidates;

    mapping (uint eventId => uint[]) public allVoterIdByEvent;
    mapping (uint eventId => uint[]) public allCandidateIdByEvent;

    mapping (uint eventId => Candidate[]) public allCandidateByEvent;
    mapping (uint eventId => Voter[]) public allVoterByEvent;
    mapping(uint eventId => Vote[]) public allVotesByEvent;

    mapping(address candidateAddress => Vote[]) public allVotesByCandidate;  // for the vote to be individualized.
    mapping (address candidateAddress => Voter[]) public allVotersByCandidate;

    mapping (uint votes => address addr) public votesPerCandidate;

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

    function createVotingEvent(string memory name, uint startTime, uint endTime) public  {
        uint eventId = eventCount++;

        if(events[eventId].status == EventStatus.pending) {
            revert("Event already created!");
        }

        VotingEvent storage votingEvent  = events[eventId];
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

        allVotingEventId.push(eventId);
        events[eventId] = votingEvent;

        emit VotingEventCreated(eventId, name, startTime, endTime);
    }

    function approveEvent(uint eventId) public onlyAdmin {
        if(events[eventId].status != EventStatus.pending) {
            revert("Event is already approved!");
        }

        if(events[eventId].status != EventStatus.approved){
            events[eventId].status = EventStatus.approved;
        }
    }

    function registerCandidate(address addr, string memory name, uint eventId) public onlyOwner(events[eventId].owner) {
        uint candidateId = candidateCount++;
        
        if(events[eventId].status != EventStatus.approved) {
            revert("Event is not yet approved!");
        }

        if(candidates[candidateId].candidateAddress != address(0)) {
            revert("Candidate already registered!");
        }

        if(events[eventId].ended) {
            revert("Event is ended!");
        }

        Candidate storage candidate = candidates[candidateId];
        candidate.name = name;
        candidate.candidateAddress = addr;
        candidate.id = candidateId;

        allCandidateId.push(candidateId);
        allCandidateIdByEvent[eventId].push(candidateId);

        events[eventId].totalRegisteredCandidates = allCandidateIdByEvent[eventId].length;

        eventCandidates[eventId] = candidate;

        emit CandidateRegistered(candidateId, name, addr);
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
        uint voteId = voteCount++;

        Vote storage vote = voteCast[voteId];

        vote.candidateAddress = candidateAddr;
        vote.id = voteId;
        vote.voter = eventVoters[eventId];
        vote.voted = true;

        // events[eventId]
        allVoteId.push(voteId);
        allVotesByCandidate[candidateAddr].push(vote);
        allVotesByEvent[eventId].push(vote);
        allVoterByEvent[eventId].push(vote.voter);
        // allValidVoterByCandidate[e]

        emit Voted(msg.sender, candidateAddr);

        if(block.timestamp >= events[eventId].endTime) {
            events[eventId].ended = true;
        }
        voteCount++;
    }

    function getTotalCandidatesByEvent(uint eventId) public view returns(uint) {
        return allCandidateIdByEvent[eventId].length;
    }

    function getTotalVoterByEvent(uint eventId) public  view returns(uint) {
        return allVoterByEvent[eventId].length;
    }

    function getTotalVotesByEvent(uint eventId) public view returns(uint) {
        return allVotesByEvent[eventId].length;
    }

    function getAllVotersByEvent(uint eventId) public view returns(Voter[] memory) {
        return allVoterByEvent[eventId];
    }

    function getAllVotesByEvent(uint eventId) public view returns(Vote[] memory) {
        return allVotesByEvent[eventId];
    }

    function getAllCandidatesByEvent(uint eventId) public view returns(Candidate[] memory) {
        return allCandidateByEvent[eventId];
    }

    function getAllVotesByCandidate(address addr) public view returns(Vote[] memory) {
        return allVotesByCandidate[addr];
    }

    function getAllVoterByCandidate(address addr) public view returns(Voter[] memory) {
        return allVotersByCandidate[addr];
    }

    function getTotalVotesByCandidates(address addr) public view returns(uint) {
        return allVotesByCandidate[addr].length;
    }

    function getTotalVoterByCandidates(address addr) public view returns(uint) {
        return allVotersByCandidate[addr].length;
    }

    function getAllCandidate() public view returns(uint) {
        return allCandidateId.length;
    }

    function getAllVotes() public view returns(uint) {
        return allVoteId.length;
    }

    function getAllVoter() public view returns(uint) {
        return allVoterId.length;
    }

    function getTotalVoters() public view returns(uint) {
        return allVoterId.length;
    }

    function getAllCandidates() public view returns(uint) {
        return candidateCount;
    }

    function getEventWinner(uint eventId) public onlyOwner(events[eventId].owner) returns(address, uint) {
        uint eventTotalCandidate = getTotalCandidatesByEvent(eventId);

        uint[] memory data;

        for(uint i = 0; i < eventTotalCandidate; i++) {
            address addr = candidates[allCandidateId[i]].candidateAddress;
            data[i] = getTotalVotesByCandidates(addr);
            votesPerCandidate[data[i]] = addr;
        }

        sort(data);

        uint maxVote = findMax(data, 0);
        return (votesPerCandidate[maxVote], maxVote);
    }

    function sort(uint[] memory data) public pure {
        for (uint i = 0; i < data.length; i++) {
            for (uint j = i + 1; j < data.length; j++) {
                if (data[i] > data[j]) {
                    uint temp = data[i];
                    data[i] = data[j];
                    data[j] = temp;
                }
            }
        }
    }

    function findMax(uint[] memory data, uint negativeInfinity) public pure returns (uint) {
        uint i = data.length;
        while (i > 0) {
            i--;
            if (data[i] > negativeInfinity) {
                negativeInfinity = data[i];
            }
        }
        return negativeInfinity;
    }
}
