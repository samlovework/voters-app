// SPDX-License-Identifier: MIT
pragma solidity 0.8.30;


// the smallest unit of data a computer can store is bits. Bits make up bytes.
// 8 bits - 1 byte
// 1111 1111, 0111 1101, 1001 0101

// in solidity, the max bytes a data can occupy is 32 bytes - 256 bits
// text data is a string e.g Paul, Food, Ball, Governor

// 1. string stores 32 bytes
// 2. uint stores 32 bytes - 256 bits
// 3. int stores 32 bytes - 256 bits
// 4. address stores 20 bytes: 0x - 3E D6 3b E7 b7 2D 29 d1 Dd 12 3F Da 28 67 6b 1A c1 3f 71 4a

// 1 byte = 3E - 1110 0001 
// paul - p[12] a[23] u[9f] l[e6]

// string constant name = "paul";
// bytes4 emai = "paul";


// 5. Boolean - 1 bytes - 1111 1111, 0000 0000
// 6. bytes32 - bytes16 - bytes8 - bytes4 - bytes1
// 7. Enum - Gender{male female}, Status{starting, pending, completed, done}, DeliveryStatus{created, started, ongoing, completed}
// 7. Struct - struct Person { string eyes, boolean male, }, struct Car {door, tyre, boot, engine}, struct Candidate{party, address, name}

// every computer stores data in storage and memory; storage is permanent while memory is temporary.
// every time we call a function, we run an operation, so it relies on memory. While the function can run 
// in memory, it doesn't mean any data that is to be stored in a function sits in memory, memory is just a location
// for data that will help the function to run the operation.
// if we keep storing multiple data to help a function run its operation, the memory will be full.

// data that stores in storage are kept in arrays and mappings
// arrays can store: addresses, uints, ints, bytes, string, and structs
// array is like a queue, that uses linear search to get a particular data
// address[] users;
// users[0] = 0x3ED63bE7b72D29d1Dd123FDa28676b1Ac13f714a;
// function gtUser(uint id) public view returns (address) {
    
//     for(uint counter = 0; counter < users.length; counter+= 5) {
//         if(users[counter] == id) {
//             return id;
//         }
//     }

//    return users[30];
// }
// array of struct - Person[] persons

// mappings is a database that store data better than an array of its fast lookup
// mappings uses indexing to store data

// mapping(uint => address) idToUser;
// idToUser[15] = sam
// function getUser(uint id) public view returns(address) {
//     return idToUser[id];
// }

// mappings can also link data together.
// mapping(string gate => mapping(string house => string room)) public gateToRoom; - this is a 2d mapping
// mapping (string school => mapping (string class => mapping (uint id => address))) public gateToClassToIdToUser;

// mapping(uint id = Person) idToPerson;


// 32 bytes - 256 bits

// string user = "Paul";
// uint a = 12;

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
