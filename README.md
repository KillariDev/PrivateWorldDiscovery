# Private world discovery
Think of a world that exists in a 2d square grid. The grid contains rooms that have doors to various directions:
![image](https://hackmd.io/_uploads/S139iJHz1g.png)

The players control a single character that can move to nearby room via a door once per turn. The players take turns in this example

## Starting situation
All of the rooms start as non-generated rooms (we don't know if they have open doors to which directions) except each players starting room:
![image](https://hackmd.io/_uploads/B1T4MeBGkg.png)

The next turn the player moves to a square and the room gets generated: 
![image](https://hackmd.io/_uploads/S1dLflHzJl.png)

## Generating a new room

To generate room, we randomize which doors are open. In this situation the randomization picks only door open towards up direction, and we also know that there's a door towards the direction we came from (left). To generate room we need to have an access to random oracle. We can use `block hash` for this purpose, let's assume it's safe enough. Also we want to generate the room in a manner that not other players know what doors are generated. For this purpose we need some secret number that no other players have access. One such number is players private key. So we can get the hidden seed of the room as:
```
room_x_y_seed = sign(privatekey, x, y)
```
This seed is then used to calculate on which directions this room as doors for. This room also need to be marked as generated (green in the image). This means no players will generate this room again. The player commits the hash of the seed to chain, in order not to be able to change it:
```
room_hash_x_y = hash(room_x_y_seed)
```

## Generating a room next to an already generated room
Now it's the red players turn. The red player moves left next to the green player:
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

This can be accomplished with Private Set Intersection (PSI) protocols. PSI allows two users to compare two sets to each other and share the outcome only between either of the players. The sets users need to compare are not squares, but square door combinations: X, Y, door_up, door_right, door_down, door_left


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

