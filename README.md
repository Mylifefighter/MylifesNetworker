# Mylife's Networking Repository
Some networking code allowing for users to build a unity multiplayer game. Provides message based networking, and provides some support for synchronizing client and server game objects transforms.

The type of server set up is a spoke wheel (that is, clients are connected to only the main server "host"). 

## How to get started

I will assume you can download the repository and add it somewhere in your Unity assets folder. 

There are 4 files (currently). These include the Networker.cs file, which houses the main class for performing all networking, ObjWatcher.cs, which is a monobehaviour that will attempt to sync the parent's transform across the network (and provides basic server authority and client authority). Similar to the objWatcher, the prefab manager class will coordinate the synchronization of the networked objects with the networker object.
