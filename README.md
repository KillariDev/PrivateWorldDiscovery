# Private world discovery
When exploring the real world, new discoveries are initially known only to the person who finds them. These findings are shared with others only if they independently discover the same thing or if the original explorer chooses to reveal the information.

In contrast, a blockchain operates as a public ledger, where all information is inherently transparent and accessible to everyone. While this transparency is advantageous for trust and security, it conflicts with the concept of a private, explorable world. However, cryptography allows us to address this challenge by hiding information even on a public ledger. With cryptographic techniques, we can create a virtual world that remains consistent for all users but is only revealed to those who explore it or receive shared knowledge from other explorers.

## Designing the World
Imagine a 2D grid of rooms forming the foundation of our world. Each room may have doors connecting it to adjacent rooms in the four cardinal directions:
![image](https://hackmd.io/_uploads/S139iJHz1g.png)

Explorers navigate this world by moving a character through these rooms, entering new areas through doors. Initially, the world is completely unknown — no one knows which rooms are connected or even if all rooms are accessible. Importantly, the rooms are not pre-generated; instead, they come into existence only when an explorer discovers them. This ensures that no individual can design or pre-know the structure of the world, paralleling the way the real world lacks a single "builder" who knows everything about it.

## The Process of Exploration

Let’s consider two explorers who have just entered the world:
![image](https://hackmd.io/_uploads/B1T4MeBGkg.png)
Either of our explorers know what are in their own rooms, but they or nobody else knows about what is in the other rooms.

Each explorer knows only the details of the room they occupy. Beyond their current location, neither they nor anyone else knows what lies in the adjacent rooms.

Now, suppose the green explorer ventures into an adjacent unexplored room. As they enter, the new room is generated dynamically:
![image](https://hackmd.io/_uploads/S1dLflHzJl.png)

### Room Generation Mechanics
When a new room is generated, its features — such as which doors are open or closed—are determined randomly. For instance, in the image above, the random process created a room with one open door leading upwards, while the door to the left corresponds to the explorer's entry point.

To generate room we need to have an access to random oracle that gives us a random number. There's many options on how this oracle could be implemented, but that is not important, let's just imagine we have an access to an oracle that provides us with a random number and we can trust the number to be a number that was not known by anyone prior we get it. This number should be public for everyone.

1) Room Seed Generation
The explorer hashes the public random number with their private key to produce a room seed which is only known by the explorer:
```
room_seed[x,y] = hash(publicRandom, privateKey)
```

2) Room Hash Commitment
The room seed is used to calculate the room's layout (e.g., door positions). To prevent tampering, the explorer commits the hash of this seed to the blockchain along with the public random number that was used:
```
room_hash[x,y] = hash(room_seed[x,y])
room_random_number[x,y] = publicRandom
```

This cryptographic process ensures that the explorer cannot retroactively alter the room's properties. If an explorer attempts to manipulate results — such as by discarding undesirable outcomes — this misbehavior can be detected and punished (e.g., by killing their character or slashing their stake). Since the fault is provable, penalties can be enforced in a trustless manner.

### Generating a Room Next to an Already Generated Room

#### Problem: Room Dependencies
When the red explorer moves left next to the green explorer, they must generate a new room. However, this new room depends on the state of the already-generated room below. For example, if the lower room has an upward door, the new room must have a downward door.

![image](https://hackmd.io/_uploads/rJmPNerGyx.png)

While the red explorer knows the room below exists (public information), they lack details about its door configuration. This information is private to the green explorer.

#### Solution: Sharing Information with Zero-Knowledge Proofs
The red explorer requests the specific information about whether a door exists between the rooms. The green explorer communicates the variable $hasDoorUpwards[x, y]$, which indicates if there's a door upwards in the room below, along with a zero-knowledge proof (ZKP) that ensures the information is valid. The ZKP must prove that:

```circuit
roomHash[x,y] = hash(roomSeed[x,y]) and
hasDoorUpwards(roomSeed[x,y]) = has_door_upwards[x,y]
```

To preserve privacy, the green explorer encrypts the information using the red explorer's public key and shares it publicly on chain. While off-chain sharing suffices in the ideal case, an on-chain mechanism is necessary to handle disputes.

#### Handling Misbehavior
1) **Refusal to Share**: If the green explorer refuses to share the required information, the red explorer can slash the green explorer for bad behavior using the on-chain protocol.

2) **Providing Invalid Information**: If the red explorer cannot decrypt the shared data or if the decrypted message does not include valid information about the room's doors, they can provide a proof of failure:
```circuit
decrypt(message) != (room_hash[x,y], proof)
```

The red explorer can then generate the room as before and set door between the rooms open using the green explorers information.

#### Edge Cases
1) **Fully Generated Rooms**: If the room is already generated, the red explorer requests the full $roomSeed[x, y]$. This enables them to validate the room and generate proofs for other explorers if needed.
2) **Multiple Nearby Rooms**: When multiple neighboring rooms are already generated, the red explorer must query the respective explorers using the same protocol to gather all required information.

## Hidden movement
Currently, explorers' locations are fully visible, which is unrealistic. A new protocol is required to enable private movement while maintaining the ability to share world information appropriately.



We need a significant overhaul of the protocol to enable private movement for explorers while preserving the mechanics of world exploration. Explorers must be able to navigate the world without revealing their positions and share relevant information about the world only when necessary.

A blog post, [zk-hunt](https://0xparc.org/blog/zk-hunt) by [Flynn Calcutt](https://twitter.com/FlynnCalcutt), provides an excellent explanation of private movement in games. While we won’t delve into those details, we will focus on implementing a hidden world.

Flynn’s concept involves a trusted **world creator** who designs the world and shares it with players. However, we aim for a decentralized approach where the world is unknown to everyone except the explorers and those they choose to share information with.

When explorers move through the hidden world, a challenge arises when encountering new rooms. Explorers need to determine:

1) Whether a room already exists.
2) Which other explorers, if any, possess information about that room or its surroundings.

To achieve this:

1) **Query Mechanism**: Explorers must query others about a room's status without revealing the room in question.
2) **Selective Responses**: Responding explorers must provide accurate information about the queried room without disclosing details about other rooms.
This protocol must ensure privacy for both the querying and responding explorers, requiring a fundamentally different approach from existing methods.

Problem description:
> - We have an user A that wants to know information about one specific room
> - We have user group B with information about the rooms
> - The user A wants to query the group B about the information about one room, without sharing any information on what the room is in question
> - The user group B only want to share information about the one particular unknown room, and nothing about the others

### Private Set Intersection
The challenge described aligns with a well-known problem: [Private Set Intersection (PSI)](https://en.wikipedia.org/wiki/Private_set_intersection).
PSI enables two parties to compare their data sets and determine their intersection while revealing nothing beyond the result.

For instance:
1) Explorer 1 wants information about Room A, represented as $𝑠𝑒𝑡_1=[𝐴]$
2) Explorer 2 knows about multiple rooms, represented as $𝑠𝑒𝑡_2=[𝐴,𝐶,𝐷,𝐸]$
3) Using PSI, Explorer 1 can learn that Room A is in Explorer 2's set, without disclosing to Explorer 2 which room was queried.

There's multiple protocols than can achieve PSI, I believe the simplest such protocol is [Diffie-Hellman based PSI](https://blog.openmined.org/private-set-intersection-with-diffie-hellman/). Diffie-Hellman based PSI can be found implemented in [ZheroTag](https://github.com/kilyig/ZheroTag/tree/main) game by [kilyig](https://github.com/kilyig). The game implements cryptographical Fog of War (FoW) with DH-PSI protocol. Kilyig also impements zero knowledge circuits for he DH-PSI protocol that are needed to ensure that the participants are cannot lie. DH-PSI works without them as well, but then the users need to be assumed to be honest.

Our problem is very similar to the Fog of War problem, but not exactly the same, as in our world the world itself is under the Fog of War, not only the players. 

### Using the PSI protocol
In order to start performing Private Set Intersection, we need to find out what are the sets we need to be comparing. As a first thought you might think that we can compare the players current room cordinates to other player cordinates, but this is not correct. We need to compare the smallest information unit that we have, a pathway between the rooms. A room consists of 4 doors that are either open or closed. We also cannot first query for the doors, and then ask if they are open or closed, so we need to query all the possible combinations of the doors in the room at the same time. To query the completely state of a room ($x=5$, $y=5$), we need to query all the combinations:

```
1) x = 5, y = 5, doorUp = open
2) x = 5, y = 5, doorUp = closed
3) x = 5, y = 5, doorLeft = open
4) x = 5, y = 5, doorLeft = closed
5) x = 5, y = 5, doorDown = open
6) x = 5, y = 5, doorDown = closed
7) x = 5, y = 5, doorRight = open
8) x = 5, y = 5, doorRight = closed
```

The other explorers need to send us information about all the doors of all the rooms they know about, which is 4 information points per room.

We can perform some optimizations on this data amount. Nearby rooms have some information about the other nearby rooms, as if there's a room on right with a door on left, then the room on left must have door to the right. This means that you can uniquily represent a room with only variables: $X$, $Y$, $doorUp$, $doorRight$, no need to have variables $doorDown$, $doorLeft$ as these are stored in the nearby room already. This means we can modify our query to be:
```
1-4) as before
5) x = 5, y = 6, doorUp = open
6) x = 5, y = 6, doorUp = closed
7) x = 6, y = 5, doorLeft = open
8) x = 6, y = 5, doorLeft = closed
```

And now the query repliers do not need to send data about the duplicate doors.

Now, when an explorer moves to a room, they query all the other explorers information about the room with PSI. This works, but still leaks some information that we would prefer to keep private; if a door match is found, they know that the explorer they are communicating with has been there, and if there's no match, they know they haven't been there before. The explorer also knows if they are first to discover the room before as well. In the current protocol this is a mandatory information, as if nobody has been in the room or its neighbours, the explorer generates a completely new room for it.

### Hiding information on who has been in the location
We can improve our PSI protocol by aggregating all the queries from all the explorers participating in a single query together, so that the explorer will no longer know where the data came from. We need to establish a communication protocol between the querier and the replier in a manner that the querier does not know on which explorer they are communicating with.

This can be achieved using a relay services, an explorer communicates their information to a relay service, and the relay service will then communicate this information to the querier. The relay can be whoever, who can keep the secret on who communicated with them, but even this is not mandatory, as you could use a TOR or similar protocol to communicate with a relay. Now the querier do not know the explorer they are chatting with, but still some information is leaded as the PSI protocol is done via each set of at the same time.

This could be further improved by having the relay to collect the responses at once and then sending them at once to the querier, or we could also use some other network and over time send all the replies in a random order. However, this is quite complicated and error prone, especially as the information might get lost on the way. The PSI protocols also require multiple back and forth communications. Even if all this were to work fine, we still leak some information, as the querier gets information on how many explorers have already seen some door.

### Hiding information on the count of explorers that have been in the location
It's an interesting question whether if it's possible to improve our protocol in a way that when explorers share information it would not actually be known if they have been in that location, but the room would be completely generated from the inputs of the other explorers. It would not matter if they have been in the room or not.

The challenge here is that some people can know what the room should look like, while other explorers do not know. So somehow we need to figure out which of the explorers know the result of the room and who do not, while at the same not actually know if any one the explorers know this information.

The problem statement for this protocol is as follows:
> - There's multiple people with real information about the variable
> - There's multiple people with no information about the variable
> - How to combine this information in a way that I can get the value of the variable if it exists, and garbage otherwise? And I do not know if I have gotten the right value, or just garbage.

I believe there's a protocol that can accomplish this, but I wasn't able to come up with one for now. If there's a no solution for this problem, one can design a game around this mechanic.

For example, storywise, you could explain that you can see the footprints of other explorers on the ground. You cannot still see whose footprints they are, but you can see how many explorers have been there, this can be a tracing game mechanic, you can see how many explorers have been there and that might tell that other explorers are near. You can also use this as a way to trace other explorers, as you can see which rooms have been generated and which have not been, so by following the path of generated rooms, you eventually end up in a square with other explorer in them. Everytime you move to a square not explored by other explorers, you know are the first one to find this place.

## Explanding the world and rules
- Currently the players can never see each other, but this is easy to change. When players communicate with each other, they can add their characters location to the communication as well. Location sharing works the same was as the doors.
- The world does not need to be limited to be just doors. We can have additional variables that can indicate what is in the rooms. However, PSI protocol is already quite inefficient and adding more variables will make the protocol even more demanding
- The world does not need to be limited to just rooms, we could have a hiearchical structure over the rooms in lower resolution. For example we could have biomes that consits of 4 rooms at once and we run a separate PSI protocol for the biome information. The biomes would then affect the generated rooms 
- The explorers door history list continue to grow all the time as the game progresses. One way to reduce this is make the explorers forget the information over time, making the dungeon to also change when nobody remembers about it.

## Challenges
- all the explorers need to communicate with each other regularly, its not possible to only communicate with blockchain. The PSI requires multiple rounds to communicate as well
- The amount of data communicated with users is also significant, PSI is not particularly efficient protocol.
- Making a world that would allow anyone to join at any time and explore as they like is not also really practical. All the explorers need to be online at the same time and be ready to communicate when other explorers make actions 
- The room history lists of the explorers continue to grow all the time as the game progresses. One way to reduce this is make the explorers forget the information over time, making the world also change when it's being forgotten

Thanks for [Ronan](https://x.com/wighawag) for this hackathon idea. The writing is also inspired by [Autonomous World Discovery Devcon Bogota talk](https://www.youtube.com/watch?v=cWrSpTMpx4E&t=6027s) by [Flynn Calcutt](https://twitter.com/FlynnCalcutt).