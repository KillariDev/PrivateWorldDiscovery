# Private world discovery
When the real world is being explored, the information about the new findings are only first known by the explorer themselves. This information is only shared to the other people if they also explore the same thing, or the people who have been there share this information with it.

A blockchain is a public ledger. A game built on blockchain is inheritely public, all the information shared is shared with everyone. Cryptography allows us to hide information on public ledger. Cryptography allows us to build a virtual world that exists on public ledger, that is consitent between the users, but is not known by ANYONE except for those who explore it or hear about it from explorers.

## The World
Let's construct our world as a 2d square grid of rooms. The grid contains rooms that have doors to various directions:
![image](https://hackmd.io/_uploads/S139iJHz1g.png)

The explorers in our world control a single character that can move between the rooms by using doors between them.

Initially the rooms are known by anyone; we don't know which rooms lead to which rooms or even if all the rooms are accessible. These rooms are not generated at the start to keep them hidden from anyone. There's no world builder on this world, no person that could have built the world as that would make them knowledgeable about the world. The world we live in, also does not have a single person who could have designed it for others to explore.

Let's imagine two explorers that have entered the world and explored their first rooms:
![image](https://hackmd.io/_uploads/B1T4MeBGkg.png)
Either of our explorers know what are in their own rooms, but they or nobody else knows about what is in the other rooms.

Now, the green explorer decides to enter the door and enter a new unexplored room. As the explorer enters the room, it's generated
![image](https://hackmd.io/_uploads/S1dLflHzJl.png)

### Generating a new room
To generate room, we randomize the four doors of the room. The doors can be closed or open. In the image above, the random generator has generated only one door up. We also know that there's a door towards the direction we came from (left).

To generate room we need to have an access to random oracle that gives us a random number. There's many options on what the oracle could be, but that is not important, let's just imagine we have an access to an oracle that provides us with a random number and we can trust it to be a number that was not known by anyone prior we get it. This number can be public for everyone.

Now, as this random number is public for everyone, we cannot use it directly to generate the room, but we need to modify it in a way that the random number we use to generate the room is only known by the initial explorer. This can be achieved simply by hashing it with users private key:

```
room_seed[x,y] = hash(publicRandom, privateKey)
```

This seed is then used to calculate on which directions this room has doors for. We also need to mark this room as being generated (green in the image). This means no explorer will generate this room again. The explorer commits the hash of the seed to chain, in order not to be able to change it:
```
room_hash[x,y] = hash(room_seed[x,y])
```

One interesting fault scenario here is that the explorer can manipulate the results here by refusing to calculate this if the result is not preferrable for them. To combat this attack vector, the explorer of the world need to be punished by some means (eg, by killing their character, or slashing their monetary stake). The fault here is provable, so it's easy to punish the explorer trustlessly.

### Generating a room next to an already generated room
Now it's the red explorers turn to move. The red explorer moves left next to the green explorer:
![image](https://hackmd.io/_uploads/rJmPNerGyx.png)
The red explorer generates the room the same way as before, but we now face a challenge; The room below is generated, which has an impact on the room being generated above, as if the room below has a door upwards, the room above also has to have door downwards. As a red explorer, we do not know what exists in the room below, we just know it has been generated, as that is public information.

The state of the doors about the room below is only known by the green explorer, so in order to generate the new room, we need to request this information from them. The red only needs and should get to know if there's a door between the rooms, nothing more. In order to exchange this information between the explorers, green needs to communicate red the variable $hasDoorUpwards[x,y]$ that tells if there's a door between the rooms, and a respective zero knowledge proof that proves from the public input $roomHash[x,y]$ that:
```circuit
roomHash[x,y] = hash(roomSeed[x,y]) and
hasDoorUpwards(roomSeed[x,y]) = has_door_upwards[x,y]
```
The green explorer shares this information onchain encrypted with red explorers public key (to prevent anyone else knowing the information). Sharing this information could happen outside chain in the happy case. However, if green explorer refuses to share the information the red explorer needs to able to slash green explorer on the bad behaviour.

An another attack vector here is to publish garbage information. If the red explorer cannot decrypt the information, they can generate a proof that when the message is decrypted, it does not result into information about the door and proof about it:
```circuit
decrypt(message) != (room_hash[x,y], proof)
```

The red explorer can then generate the room as before and set door between the rooms open using the green explorers information.

#### Two additional edge cases
There's also two other scenarios that can happen when an explorer moves to a room:
1) The room can already be fully generated, so we need to get the whole room generating seed. This is easy as the other explorer can just share `room_seed[x,y]` directly, this will also allow the new explorer the power to generate new proofs from this location to the other explorers.
2) More than one nearby rooms can be already generated, in this situation we need to request information from explorers that have visited the nearby rooms using the same protocol.

## Hidden movement
Our current world leaks information about the other explorers. Everyone know at all times where the other explorers are. This is not how real explorable world works.

We need to do a big overhaul on the protocol to be able to hide the explorers from each other, while at the same time have the exploring mechanics. We need a way for exploers to move privately in the world and then a way for players to share appropriate information about the world when needed. There's a blog post [zk-hunt](https://0xparc.org/blog/zk-hunt) by [Flynn Calcutt](https://twitter.com/FlynnCalcutt) that explains nicely on how you can make a game with private movement, so we do not go in detail on how that can be accomplished. However, we'll explain more in detail on how to accomplish the hidden world. 

Flynn discusses a hidden world idea in their blogpost, but the proposed world requires a world creator, a trusted party that creates the world and shares it with the other users. We want to have a world that is not known by anyone, and only known by explorers and those they like to share this information with.

As the explorers are now moving in the world in a hidden manner, when a new room is being explored, we have no information if this room exists already, or which explorer might know some information abut it. In order to acquire this information, we need a way to query the other explorers on if they have been in that room or nearby it. The other explorers then need to be able to reply this query and prove that their information about it is correct, while at the same not leak any additional information.

An interesting challenge arises from this, the explorer need to query information about their room, without telling the other explorers on what the room is in question. The other exploers also need to reply to this, without leaking anything extra.

We have to come up with completely different way to tackle this situation and we cannot really use any other logic discussed previously.

Problem description:
> - We have an user A that wants to know information about one specific room
> - We have user group B with information about the rooms
> - The user A wants to query the group B about the information about one room, without sharing any information on what the room is in question
> - The user group B only want to share information about the one particular unknown room, and nothing about the others

### Private Set Intersection
Our problem description fits a known problem: [Private Set Intersection (PSI)](https://en.wikipedia.org/wiki/Private_set_intersection).
PSI allows two users to compare two sets of data with each other and share the outcome only between either of the explorers, eg. explorer 1 could query information about room A: $set_1 = [A]$. The explorer 2 knows information about rooms: $set_2 = [A, C,D,E]$. The explorers can conduct a private set intersection to be able to make explorer 1 know about the room A, without telling the explorer 2 which room they queried for.

There's multiple protocols than can achieve PSI, I believe the simplest such protocol is [Diffie-Hellman based PSI](https://blog.openmined.org/private-set-intersection-with-diffie-hellman/). Diffie-Hellman based PSI can be found implemented in [ZheroTag](https://github.com/kilyig/ZheroTag/tree/main) game. The game by [kilyig](https://github.com/kilyig) implements cryptographical Fog of War (FoW) with DH-PSI protocol. Our problem is very similar to the Fog of War problem, but not exactly the same, as in our world the world itself is under the Fog of War, not only the players.

To start implementing the FoG of war to the world
The sets users need to compare are not only cordinates, but square+door combinations: $X$, $Y$, $doorUp$, $doorRight$, $doorDown$, $doorLeft$. Nearby rooms also share information between them, as if there's a room on right with a door on left, then the room on left must have door to the right. This means that you can uniquily represent a pathway with only variables: $X$, $Y$, $doorUp$, $doorRight$, no need to have variables $doorDown$, $doorLeft$ as these are stored in the nearby room already.

## Applying PSI
When one explorer makes a hidden move in a way that other explorers do not know where they are moving, the explorer need to ask information about the room from other explorers, without revealing to the other explorers on which room they are and what information they are asking.

This can be accomplished by the moving explorer by creating a PSI query with the room they are in, and the other explorers need to reply to this with the information of all the rooms they know about. The PSI allows us to achieve this in a private manner, however, as the moving explorer is individually communicating with each explorer, the moving explorer will know which of the explorers have been in that location before, and who haven't. The explorer also knows if anyone has been in the room before as well. In the current protocol this is a mandatory information, as if nobody has been in the room or its neighbours, the explorer generates a completely new room for it.

### hiding information on who has been in the location
We can improve the mechanism by aggregating all the queries together, so that the explorer will no longer know where the data came from. When the moving explorer is querying for the information, all the other explorers need to communicate this information in a way, that its not revealed on who does the communication. This can be made using relay services, a explorer communicates their information to a relay service, and the relay service will then communicate this information to the querying explorer. The relay can be whoever, who can keep the secret on who communicated with them, but even this is not mandatory, as you could use a The Onion Router or similar protocol to communicate with relay. The PSI protocols also require back and fourth communication, so this communicating via relay needs to be done multiple times, which is not optimal.

After the explorer gets all the information from all the explorers, the user will perform PSI protocol with them. The explorer will get the set of pathways that have been visited and if the doors to them are open or closed. The explorer will also know how many explorers have seen those doors, so our protocol is still leaking information.

### Hiding information on number of explorers that have been in the location
It's an interesting question that if it's possible to make it so that when explorers share information it would not actually be known if they have been in a location, but the room would be completely generated from the inputs of the other explorers. It would not matter if they have been in the room or not.

The challenge here is that some people can know what the room should look like, while other explorers do not know. So somehow we need to figure out which of the explorers know the result of the room and who do not, while at the same not actually know if any one the explorers know this information.

The problem statement for this protocol is as follows:
> There's multiple people with real information about a variable
> There's multiple people with no information about a variable
> -> How to combine this information in a way that I can get the value of the variable if it exists, and garbage otherwise?

I believe there's a protocol that can accomplish this, but I wasn't able to come up with one for now. If there's a no solution for this problem, one can design a game around this mechanic. For example, storywise, you could explain that you can see the footprints of other explorers on the ground. You cannot still see whose footprints they are, but you can see how many explorers have been there, this can be a tracing game mechanic, you can see how many explorers have been there and that might tell that other explorers are near. You can also use this as a way to trace other explorers, as you can see which rooms have been generated and which have not been, so by following the path of generated rooms, you eventually end up in a square with other explorer in them. Everytime you move to a square not explored by other explorers, you know are the first one to find this place.

### Communication protocol

### which PSI protocol to use
- the PSI protcol needs to work in a such way that explorers cannot cheat, or if they can cheat, that is detectable and punishable
- This can be achieved by using most of the PSI protcols, and then applying a ZK proof that it was computed correctly.

When one explorer moves, the other explorers need to communicate:
1) 
explorer state:
```
current_position = (0,1)
move_history = [(0,0), (0,1), (0,2), (0,1)]
squares_visited_with_seeds = [(0,0, 0x..), (0,1, 0x...), (0,2, 0x...)]
doors_open = [
    (0,0, [0,0,1,1])
    (0,1, [0,1,0,0])
    (0,2, [1,0,0,1])
]
```


## explanding the world
The world does not need to be limited to be just rooms 


- x,y grid, with doors to 1-4 directions with tile info
- discoverable world
- root hash
- private world discovery
- locality-dependant discovery
- onchain world discovery
- witness encryption
![image](https://hackmd.io/_uploads/r1qOU0NzJx.png)




https://www.youtube.com/watch?v=cWrSpTMpx4E&t=6027s
https://0xparc.org/blog/zk-hunt


- private set intersection

# challenges & notes
- all the explorers need to communicate with each other regularly, making the protocol $o(n^2)$ at best
- The room history lists of the explorer continue to grow all the time as the game progresses. One way to reduce this is make the explorers forget the information over time, making the dungeon to also change when nobody remembers about it.
- explorers can refuse to share information and there needs to be some kind of method to punish explorers for behaving badly, and a way to recover from these situations. One such way is that the explorer refusing to cooperate will just get killed and they lose the character, after this they are booted of the game.
- Can we make it so that you don't know who has been in the location before?
- one world builder that knows the world
- write about meeting other players in the world if the players enter the same location (zhero tag)


