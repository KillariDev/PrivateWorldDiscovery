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

## Generating a new room
To generate room, we randomize the four doors of the room. The doors can be closed or open. In the image above, the random generator has generated only one door up. We also know that there's a door towards the direction we came from (left).

To generate room we need to have an access to random oracle that gives us a random number. There's many options on what the oracle could be, but that is not important, let's just imagine we have an access to an oracle that provides us with a random number and we can trust it to be a number that was not known by anyone prior we get it. This number can be public for everyone.

Now, as this random number is public for everyone, we cannot use it directly to generate the room, but we need to modify it in a way that the random number we use to generate the room is only known by the initial explorer. This can be achieved simply by hashing it with users private key:

```
room_x_y_seed = hash(publicRandom, privateKey)
```

This seed is then used to calculate on which directions this room has doors for. We also need to mark this room as being generated (green in the image). This means no player will generate this room again. The player commits the hash of the seed to chain, in order not to be able to change it:
```
room_hash_x_y = hash(room_x_y_seed)
```

One interesting fault scenario here is that the explorer can manipulate the results here by refusing to calculate this if the result is not preferrable for them. To combat this attack vector, the explorer of the world need to be punished by some means (eg, by killing their character, or slashing their monetary stake). The fault here is provable, so it's easy to punish the explorer trustlessly.

## Generating a room next to an already generated room
Now it's the red explorers turn to move. The red player moves left next to the green player:
![image](https://hackmd.io/_uploads/rJmPNerGyx.png)
The red player generates the room the same way as before, but we now face a challenge; The room below is generated, which has an impact on the room generated above, as if the room below has a door upwards, the room above also has to have door upwards. As a red player, we do not know what exists in the room below, we just know it has been generated.

The information about the below room is only known by the green player, so in order to generate the new room, we need to request information from green player. The red player only needs to know if there's a door between the rooms. In order to provide this information in trustless way to the red player, the green needs to tell the player the value `has_door_upwards_x_y` that tells if there's a door between the rooms, and a respective zero knowledge proof that proves from `room_hash_x_y` that:
```circuit
room_hash_x_y = hash(room_x_y_seed) and
hasDoorUpwards(room_x_y_seed) = has_door_upwards_x_y
```
The green player shares this information onchain encrypted with red players public key (to prevent anyone else knowing the information). Sharing this information could happen outside chain in the happy case. However, if green player refuses to share the information the red player needs to able to slash green player on the bad behaviour.

The red player can then generate the room as before and set door between the rooms open using the green players information.

### Defening against malicious player

The green player can act maliciously here: They can refuse this information, or provide garbage infomation. To counter this, the game needs to have a mechanic that if this information is withdrawn, the malicious player is punished. Its easy to prove on chain if the player is withdrawing information, so the malicious player can be punished accordingly. 

An another attack is to publish garbage information. If the red player cannot decrypt the information, they can generate a proof that when the message is decrypted, it does not result into information about the door and proof about it:
```circuit
decrypt(message) != (room_hash_x_y, proof)
```
And again the misbehaving player can be punished provably.

## Two additional edge cases
There's also two other scenarios that can happen when a player moves to a room:
1) The room can already be fully generated, so we need to get the whole hash. This is easy as the other player can just share `room_x_y_seed` directly, this will also allow the new player the power to generate new proofs from this location to the other players
2) More than one nearby rooms can be already generated, in this situation we need to request information from players that have visited the nearby rooms using the same protocol.

## Hidden movement
The previously explained game works as we know which rooms have already been discovered, and who do we need to ask for information. If we want to hide the players location while the explore the world, we cannot do any of this.

However, there's a way to accomplish this. When player explores a new room they need to ask ALL the players if they have been in that square or nearby squares. The other players need to be able to prove the claims, and also do all the communication without revealing information on that they have been in the room to anyone, as this would leak information.

We have to come up with completely different way to tackle this situation and we cannot really use any other logic discussed previously.

## Private Set Intersection
This can be accomplished with Private Set Intersection (PSI) protocols. PSI allows two users to compare two sets to each other and share the outcome only between either of the players, eg. player 1 could have information set $[A,B,C,D]$ and the player 2 $[C,D,E]$, and 

## Pathway data structure
The sets users need to compare are not only cordinates, but square+door combinations: $X$, $Y$, $doorUp$, $doorRight$, $doorDown$, $doorLeft$. Nearby rooms also share information between them, as if there's a room on right with a door on left, then the room on left must have door to the right. This means that you can uniquily represent a pathway with only variables: $X$, $Y$, $doorUp$, $doorRight$, no need to have variables $doorDown$, $doorLeft$ as these are stored in the nearby room already.

## Applying PSI
When one player makes a hidden move in a way that other players do not know where they are moving, the player need to ask information about the room from other players, without revealing to the other players on which room they are and what information they are asking.

This can be accomplished by the moving player by creating a PSI query with the room they are in, and the other players need to reply to this with the information of all the rooms they know about. The PSI allows us to achieve this in a private manner, however, as the moving player is individually communicating with each player, the moving player will know which of the players have been in that location before, and who haven't. The player also knows if anyone has been in the room before as well. In the current protocol this is a mandatory information, as if nobody has been in the room or its neighbours, the player generates a completely new room for it.

### hiding information on who has been in the location
We can improve the mechanism by aggregating all the queries together, so that the player will no longer know where the data came from. When the moving player is querying for the information, all the other players need to communicate this information in a way, that its not revealed on who does the communication. This can be made using relay services, a player communicates their information to a relay service, and the relay service will then communicate this information to the querying player. The relay can be whoever, who can keep the secret on who communicated with them, but even this is not mandatory, as you could use a The Onion Router or similar protocol to communicate with relay. The PSI protocols also require back and fourth communication, so this communicating via relay needs to be done multiple times, which is not optimal.

After the player gets all the information from all the players, the user will perform PSI protocol with them. The player will get the set of pathways that have been visited and if the doors to them are open or closed. The player will also know how many players have seen those doors, so our protocol is still leaking information.

### Hiding information on number of players that have been in the location
It's an interesting question that if it's possible to make it so that when players share information it would not actually be known if they have been in a location, but the room would be completely generated from the inputs of the other players. It would not matter if they have been in the room or not.

The challenge here is that some people can know what the room should look like, while other players do not know. So somehow we need to figure out which of the players know the result of the room and who do not, while at the same not actually know if any one the players know this information.

The problem statement for this protocol is as follows:
> There's multiple people with real information about a variable
> There's multiple people with no information about a variable
> -> How to combine this information in a way that I can get the value of the variable if it exists, and garbage otherwise?

I believe there's a protocol that can accomplish this, but I wasn't able to come up with one for now. If there's a no solution for this problem, one can design a game around this mechanic. For example, storywise, you could explain that you can see the footprints of other players on the ground. You cannot still see whose footprints they are, but you can see how many players have been there, this can be a tracing game mechanic, you can see how many players have been there and that might tell that other players are near. You can also use this as a way to trace other players, as you can see which rooms have been generated and which have not been, so by following the path of generated rooms, you eventually end up in a square with other player in them. Everytime you move to a square not explored by other players, you know are the first one to find this place.

### Communication protocol

### which PSI protocol to use
- the PSI protcol needs to work in a such way that players cannot cheat, or if they can cheat, that is detectable and punishable
- This can be achieved by using most of the PSI protcols, and then applying a ZK proof that it was computed correctly.

When one player moves, the other players need to communicate:
1) 
player state:
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
- all the players need to communicate with each other regularly, making the protocol $o(n^2)$ at best
- The room history lists of the player continue to grow all the time as the game progresses. One way to reduce this is make the players forget the information over time, making the dungeon to also change when nobody remembers about it.
- Players can refuse to share information and there needs to be some kind of method to punish players for behaving badly, and a way to recover from these situations. One such way is that the player refusing to cooperate will just get killed and they lose the character, after this they are booted of the game.
- Can we make it so that you don't know who has been in the location before?
- one world builder that knows the world


