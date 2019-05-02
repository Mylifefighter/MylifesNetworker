# Mylife's Networking Repository
Some networking code allowing for users to build a unity multiplayer game. Provides message based networking, and provides some support for synchronizing client and server game objects transforms.

The type of server set up is a spoke wheel (that is, clients are connected to only the main server "host"). 

## How to get started

I will assume you can download the repository and add it somewhere in your Unity assets folder. 

There are 4 files (currently). These include the Networker.cs file, which houses the main class for performing all networking, ObjWatcher.cs, which is a monobehaviour that will attempt to sync the parent's transform across the network (and provides basic server authority and client authority). Similar to the objWatcher, the prefab manager class will coordinate the synchronization of the networked objects with the networker object. The fourth script is NETMSG.cs, which houses all possible message types that can be sent across the network. 

These files are somewhat convoluted in their current state. But how to use them is shown below:

### The manager
Every game requires at least one manager script to ensure some form of cohesion. For building a MP game you should have one script dedicated to the setting up a network, handling messages and handling client/host disconnects.

This, preferablly, should be an object that is not destroyable.

#### Setup: Channels (1/2)
The manager class needs to setup a few things before the server can be started. The first is registering channels for the servers and clients to talk on. There are quite a few available but the following are what I would personally recommend:
  Channel 0: Reliable (This is automatically assigned, and used for a lot of internal packets)
  Channel 1: UnreliableSequenced
  Channel 2: StateUpdate
  Channel 3: Unreliable
  Channel 4: At All Costs Delivery
  
These channels cover everything that should be required by a standard multiplayer game. All messages to and from players have to be sent along one of these channels, and each channel has their own drawbacks.

These are registed by the following:
```c#
int newChannelID = Networker.AddChannel(UnityEngine.Networking.QosType.UnreliableSequenced);
```
*Note that this has to be done **before** the network is started*
#### Setup: Users and User Messages (2/2)
First set the maximum number of players allowed for the network.
```c#
Networker.setMaxUsers(10);
```

Next you will want to observe events invoked from the Networker. The most common subscribed events are
* `Networker.OnClientDisconnect(bool, ConnectionObj)` - Called on the server when a client has disconnected from the server. 
* `Networker.OnServerDisconnected(bool, ConnectionObj)` - Called on the client when the server has disconnected from the client.
* `Networker.OnUserEvent(int ID, int subID, ArbObj)` - Called when a message comes in flagged as a `NetID.User` message
* `Networker.OnMsgEvent(MsgObj)` - Called when a message comes in flagged as `NetID.Msg`.
* `Networker.OnOtherEvent` - Called when a message comes in flagged as a different `NetID` than Other standard messages
* `Networker.OnClientConnect(bool, ConnectionObj)` - Called on server when a client is joining. 

Register to an event by doing `Networker.OnUserEvent += HandleNetworkMessages`.

#### Connecting/Starting
First, ensure that the ports are set by setting `Networker.port = int` and if developing for unity webgl `Networker.web_port = int`. Ensure these two are not the same. 

If connecting as a cilent, ensure that the field `Networker.ConnIP = string` is set.

To host a server, call `Networker.StartHost()`.

To start a client, call `Networker.StartClient()`.

#### Maintaining Server Health
So the networker is implemented using a singleton, that means that it doesn't have access to monobehaviour functions. To keep
the system going strong, you are required to manually "pump" all messages from the networker. This can be done simply by ensuring
that you have the following in your manager class:
```c#
void Update(){
  if(Networker.isActive()){
    Networker.Update();
  }
}
```
#### Messages, sending to and fro
Sending the server a message is easy. Just call
`Networker.SendServer(NETMSG, channelID);` - Sends the NETMSG across the network channel channelID

Note you can only call this on the client! The networker will complain if you try to send yourself a message!

To send a client a message, there are three options
* `Networker.SendAllClients(NETMSG[, channelID = 0]);` - Sends the NETMSG across the network to ALL clients.
* `Networker.SendClient(NETMSG, long UUID[, channelID = 0]);` - Sends the user with ID UUID the message in NETMSG.
* `Networker.SendAllClientsExcept(NETMSG, long exceptUUID[, channelID = 0]);` - Same as SendAllClients, except it doesn't send it to the 'except' target. Useful for sending updates from the except user to all other users.

Note you can only call this on the server! The networker will complain if you even think about doing this otherwise!

#### Objects?!
Yes there is an object watcher meant to kinda replicate how to move objects around the scene and keeping them in sync across a server. There is some basic network authority (although if you want, it is easy to spoof with your own messages, watch out!)

Every object that could be spawned across the network needs to be registered first with the prefab manager to have it assigned a network ID. 
This prefab manager is required for basically everything so it is best to ensure it is just attached to your manager object.

For an object to be registered it must have an ObjWatcher script attached. This comes with some bells an whistles that you can change, such as having the script send any movement or rotation across the server (note, this will not send ANYTHING else besides those two fields, that is on you to implement as there is no good way to do so). You can also state how fast or slow anything should send updates (which can ease up on bandwidth). 

The `Qos Stream = int` is the field that should be assigned on creation for the channel that you wish to send updates on. For this reason, I recommend sending on the StateUpdate channel if you have it.

The `Interp. Type = Enum {DECAYING, LINEAR, NONE}` is the field that determines how interpolation between network packets should be done for a player. 
* None means that object will only update to the position of the last packet sent. 
* Linear means that the object will attempt to interpolate between the last two positional packets sent to approximate the player's movement. Note this is only good short term.
* Decaying is like linear, except the time approximation will decay to 0, so it assumes that if it hasn't heard anything, it is best to try to stop moving (instead of phasing through the environment potentially).

Note the other fields are purely there for output, please do not abuse the fact that you can change these values.

A network object will not work when placed into the scene, it must be **created** by the ***server*** (clients must request an object to be created for them, a way to get around this issue is to spawn a fake and wait for the server to give you the real one).

To spawn a networked object across the net, use the following:
`GameObject go = Networker.NetworkSpawnObject(int networkedObjPrefabPos, Vector3 position, Quaternion rotation, long ownerID);`

This will spawn the networked Prefab object registered in position `networkedObjPrefabPos`, at position `position` with rotation `rotation`. It will grant ownership of this object to client `ownerID`. If this is -1, then the server only has control of it.

However, if the ownerID is not -1 (note all player UUID start at 0 and go up...) then that means that the client will be able to send positional and rotational updates to the server regarding the object (other clients will have the object jump back to the predicted locations if they try to move something they don't control).

The server still technically controls the objects, and as such the server and the client will attempt a dance where they will try to mesh their updates into the client and the server, effectively allowing the server "control" over the client's objects.


#### Helpful information/functions
Other helpful functions to keep the day going great.

* `bool = Networker.isActive()` - Returns if the *networker* is alive (aka, hasn't errored out or hasn't been killed externally). 
* `bool = Networker.isConnected()` - Returns if the client has successfully connected to the server and the server has acknowledged the request. Note: The client could be kicked from the server, but when this is true, the server has at least allowed the connection to be made.
* `long = Networker.getUserUUID()` - Returns the clients UUID, if the server calls this function it will return -1.
* `List<long> = Networker.getCurrentPlayers()` - Returns a list of the UUIDs of all currently connected players.
* `Networker.Shutdown()` - Yeah. Shuts down the server/client connection. If called on the client, it disconnects from server (Calling the event OnServerDisconnected). If called from server, it disconnects all clients (and still calls OnServerDisconnected).

Create your own userNetMessages to pass information around. 

#### Errors
There is one other event to subscribe to:
`OnErrorEvent(ErrorObj)` - Fired when an error occurs in packet handling.
## Road Map
Currently I am a full time college student, and so my time is a little strained. I am trying to clean up the code to make it somewhat presentable.
