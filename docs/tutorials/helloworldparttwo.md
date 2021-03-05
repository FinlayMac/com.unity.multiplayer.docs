---
id: helloworldtwo
title: Building on "Hello world"
sidebar_label: Building on "Hello world"
---



1. Right click in the Hierarchy tab of the Main Unity Window
1. Select **Game Object**
1. Select **UI**
1. Create a Canvas object inside of our scene.
    ![Create a canvas Object](../../static/img/createcanvas.gif)
1. Click the Assets folder
1. Create a  new Folder and call it **Scripts**
1. Create an empty GameObject rename it **HelloWorldManager**
1. Create a script called `HelloWorldManager`
1. Add the `HelloWorldManager` script as a component.
    ![Create a Hellowowrldscriptt](../../static/img/Helloworldcreatescript.gif)
1. Open the `HelloWorldManager.cs` script 
1. Edit the `HellowWorldManager.cs` script to match  the following
:::tip 
You can copy the script from here and paste it into your file.
   1. Select the code sample 
   1. Click Copy in the top right corner
   1. Paste it into your code editor
:::
``` c sharp
using MLAPI;
using UnityEngine;

namespace HelloWorld
{
    public class HelloWorldManager : MonoBehaviour
    {
        static string s_NameToEdit = string.Empty;

        void OnGUI()
        {
            GUILayout.BeginArea(new Rect(10, 10, 300, 300));
            if (!NetworkManager.Singleton.IsClient && !NetworkManager.Singleton.IsServer)
            {
                StartButtons();
            }
            else
            {
                StatusLabels();

                SubmitNewPosition();
            }

            GUILayout.EndArea();
        }

        static void StartButtons()
        {
            if (GUILayout.Button("Host")) NetworkManager.Singleton.StartHost();
            if (GUILayout.Button("Client")) NetworkManager.Singleton.StartClient();
            if (GUILayout.Button("Server")) NetworkManager.Singleton.StartServer();
        }

        static void StatusLabels()
        {
            var mode = NetworkManager.Singleton.IsHost ? 
                "Host" : NetworkManager.Singleton.IsServer ? "Server" : "Client";

            GUILayout.Label("Transport: " + 
                NetworkManager.Singleton.NetworkConfig.NetworkTransport.GetType().Name);
            GUILayout.Label("Mode: " + mode);
        }

        static void SubmitNewPosition()
        {
            if (GUILayout.Button(NetworkManager.Singleton.IsServer ? "Move" : "Request Position Change"))
            {
                if (NetworkManager.Singleton.ConnectedClients.TryGetValue(NetworkManager.Singleton.LocalClientId,
                    out var networkedClient))
                {
                    var player = networkedClient.PlayerObject.GetComponent<HelloWorldPlayer>();
                    if (player)
                    {
                        player.Move();
                    }
                }
            }
        }

        static void ModifyName()
        {
            s_NameToEdit = GUILayout.TextField(s_NameToEdit, 25);
        }

        void SubmitName()
        {
            if (GUILayout.Button(NetworkManager.Singleton.IsServer ? "Change Name" : "Request Name Change"))
            {
                if (NetworkManager.Singleton.ConnectedClients.TryGetValue(NetworkManager.Singleton.LocalClientId,
                    out var networkedClient))
                {
                    var player = networkedClient.PlayerObject.GetComponent<HelloWorldPlayer>();
                    if (player)
                    {
                        if (NetworkManager.Singleton.IsServer)
                        {
                            player.NameTag.Value = s_NameToEdit;
                        }
                        else
                        {
                            player.SubmitName(s_NameToEdit);
                        }
                    }
                }
            }
        }
    }
}

```
Inside the `HellowWorldMAnager.cs` script, we  define two methods which mimic the editor buttons inside of NetworkManager during Play mode:

11. Add the following to your `HelloWorldManager.cs` script

<!---
```c sharp reference
https://github.com/Unity-Technologies/com.unity.multiplayer.samples.poc/tree/feature/hello-world/Assets/Scripts/Shared/HelloWorldManager.cs#L27-L42

```
-->
```c sharp
        static void StartButtons()
        {
            if (GUILayout.Button("Host")) NetworkManager.Singleton.StartHost();
            if (GUILayout.Button("Client")) NetworkManager.Singleton.StartClient();
            if (GUILayout.Button("Server")) NetworkManager.Singleton.StartServer();
        }

        static void StatusLabels()
        {
            var mode = NetworkManager.Singleton.IsHost ? 
                "Host" : NetworkManager.Singleton.IsServer ? "Server" : "Client";

            GUILayout.Label("Transport: " + 
                NetworkManager.Singleton.NetworkConfig.NetworkTransport.GetType().Name);
            GUILayout.Label("Mode: " + mode);
        }
```

`NetworkManager` implements the singleton pattern as it declares it's singleton named `Singleton`. This is defined when the `MonoBehaviour` is enabled. This component also contains very useful properties, such as `IsClient`, `IsServer`, and `IsLocalClient`. The first two dictate the connection state we have currently established. These will be used shortly.

We will call these methods inside of `OnGUI()`

<!---
```c sharp reference
https://github.com/Unity-Technologies/com.unity.multiplayer.samples.poc/tree/feature/hello-world/Assets/Scripts/Shared/HelloWorldManager.cs#L10-L25

```
-->

``` c sharp

 void OnGUI()
        {
            GUILayout.BeginArea(new Rect(10, 10, 300, 300));
            if (!NetworkManager.Singleton.IsClient && !NetworkManager.Singleton.IsServer)
            {
                StartButtons();
            }
            else
            {
                StatusLabels();

                SubmitNewPosition();
            }

            GUILayout.EndArea();
        }
```
:::note
You'll notice the introduction of a new method,  `SubmitNewPosition()`;which we will be using later in later. 
:::

13. Create a new script `HelloWorldPlayer`
1. Add the script `HelloWorldPlayer` to the Player Prefab
![Create a Hellowowrldplayer script](../../static/img/helloworldcreateplayerscript.gif)
1. Open the `HelloWorldPlayer.cs` script 
1. Edit the `HelloWorldPlayer.cs` script to match the following 
``` c sharp
using MLAPI;
using MLAPI.Messaging;
using MLAPI.NetworkVariable;
using UnityEngine;

namespace HelloWorld
{
    public class HelloWorldPlayer : NetworkBehaviour
    {
        public NetworkVariableString NameTag = new NetworkVariableString();
        
        public NetworkVariableVector3 Position = new NetworkVariableVector3(new NetworkVariableSettings
        {
            WritePermission = NetworkVariablePermission.ServerOnly,
            ReadPermission = NetworkVariablePermission.Everyone
        });
        
        public override void NetworkStart()
        {
            Move();
        }

        public void Move()
        {
            if (NetworkManager.Singleton.IsServer)
            {
                var randomPosition = GetRandomPositionOnPlane();
                transform.position = randomPosition;
                Position.Value = randomPosition;
            }
            else
            {
                SubmitPositionRequestServerRpc();
            }
        }
        
        [ServerRpc]
        void SubmitPositionRequestServerRpc(ServerRpcParams rpcParams = default)
        {
            Position.Value = GetRandomPositionOnPlane();
        }

        static Vector3 GetRandomPositionOnPlane()
        {
            return new Vector3(Random.Range(-3f, 3f), 1f, Random.Range(-3f, 3f));
        }

        void Update()
        {
            transform.position = Position.Value;
        }
        
        public void SubmitName(string nameTag)
        {
            SubmitNameServerRpc(nameTag);
        }
        
        [ServerRpc]
        void SubmitNameServerRpc(string nameTag, ServerRpcParams rpcParams = default)
        {
            NameTag.Value = nameTag;
        }
    }
}

```

This class will inherit from `NetworkBehaviour` instead of `MonoBehaviour` 
<!---
```c sharp reference
https://github.com/Unity-Technologies/com.unity.multiplayer.samples.poc/tree/feature/hello-world/Assets/Scripts/Shared/HelloWorldPlayer.cs#L8
`-->

``` csharp

    public class HelloWorldPlayer : NetworkBehaviour

```

Inside this class we will define a `NetworkVariable` to represent this player's networked position: 
<!---
```c sharp reference
https://github.com/Unity-Technologies/com.unity.multiplayer.samples.poc/tree/feature/hello-world/Assets/Scripts/Shared/HelloWorldPlayer.cs#L12-L16
```
-->
``` csharp

        public NetworkVariableVector3 Position = new NetworkVariableVector3(new NetworkVariableSettings
        {
            WritePermission = NetworkVariablePermission.ServerOnly,
            ReadPermission = NetworkVariablePermission.Everyone
        });
```

We introduce the concept of ownership on a `NetworkVariable` (aka read & write permissions). For the purposes of this demo, the server will be authoritative on the `NetworkVariable` representing position. All clients are able to read the value, however.

HelloWorldPlayer overrides NetworkStart:
<!---
```c sharp reference
https://github.com/Unity-Technologies/com.unity.multiplayer.samples.poc/tree/feature/hello-world/Assets/Scripts/Shared/HelloWorldPlayer.cs#L18-L21
```
-->

``` csharp
        public override void NetworkStart()
        {
            Move();
        }
```
Any `MonoBehaviour` implementing `NetworkBehaviour` can override the MLAPI method `NetworkStart()`. This method is fired when message handlers are ready to be registered and the networking is setup. We override `NetworkStart` since a client and a server will run different logic here. 

:::note
This can be overriden on any `NetworkBehaviour`.
:::

On both client and server instances of this player, we call the `Move()` method, which will simply do the following:
<!---
```c sharp reference
https://github.com/Unity-Technologies/com.unity.multiplayer.samples.poc/tree/feature/hello-world/Assets/Scripts/Shared/HelloWorldPlayer.cs#L23-L40
```
-->

```c sharp
        public void Move()
        {
            if (NetworkManager.Singleton.IsServer)
            {
                var randomPosition = GetRandomPositionOnPlane();
                transform.position = randomPosition;
                Position.Value = randomPosition;
            }
            else
            {
                SubmitPositionRequestServerRpc();
            }
        }
```

If this player is a server-owned player, at `NetworkStart()` we can immediately move this player, as suggested in the following code.

<!---
```c sharp reference
https://github.com/Unity-Technologies/com.unity.multiplayer.samples.poc/tree/feature/hello-world/Assets/Scripts/Shared/HelloWorldPlayer.cs#L32-L34
```
-->

```c sharp
              {
                SubmitPositionRequestServerRpc();
            }
```
 If we are a client, we call a server RPC: (HelloWorldPlayer.cs lines 42-46).
<!---
```c sharp reference
https://github.com/Unity-Technologies/com.unity.multiplayer.samples.poc/tree/feature/hello-world/Assets/Scripts/Shared/HelloWorldPlayer.cs#L42-L46
```
-->

``` c sharp
        static Vector3 GetRandomPositionOnPlane()
        {
            return new Vector3(Random.Range(-3f, 3f), 1f, Random.Range(-3f, 3f));
        }
```

This server RPC simply sets the position `NetworkVariable` on the server's instance of this player by just picking a random point on the plane:
<!---
```c sharp reference
https://github.com/Unity-Technologies/com.unity.multiplayer.samples.poc/tree/feature/hello-world/Assets/Scripts/Shared/HelloWorldPlayer.cs#L48-L51
```
-->
``` c sharp
        void Update()
        {
            transform.position = Position.Value;
        }
```


The server instance of this player has just modified the Position NetworkVariable, meaning that if we are a client, we need to apply this position locally inside of our Update loop: 
<!---
```c sharp reference
https://github.com/Unity-Technologies/com.unity.multiplayer.samples.poc/tree/feature/hello-world/Assets/Scripts/Shared/HelloWorldPlayer.cs#L53-L56
```
-->
``` csharp
        public void SubmitName(string nameTag)
        {
            SubmitNameServerRpc(nameTag);
        }
```


We can now go back to `HelloWorldManager.cs` and define the contents of `SubmitNewPosition()`

<!---
```c sharp reference
https://github.com/Unity-Technologies/com.unity.multiplayer.samples.poc/tree/feature/hello-world/Assets/Scripts/Shared/HelloWorldMAnager.cs#L44-L58
```
-->
``` csharp
 static void SubmitNewPosition()
        {
            if (GUILayout.Button(NetworkManager.Singleton.IsServer ? "Move" : "Request Position Change"))
            {
                if (NetworkManager.Singleton.ConnectedClients.TryGetValue(NetworkManager.Singleton.LocalClientId,
                    out var networkedClient))
                {
                    var player = networkedClient.PlayerObject.GetComponent<HelloWorldPlayer>();
                    if (player)
                    {
                        player.Move();
                    }
                }
            }
        }
```

Whenever you press the GUI button (which is contextual depnding on if you are server or a client), you find your local player and simply call `Move()`.

You can now create a build which will demonstrate the concepts outlined above. 
:::tip
Make sure **SampleScene** is included in **BuildSettings**.
:::

One build instance can create a host. Another client can join the host's game. Both are able to press a GUI button to move. Server will move immediately and be replicated on client. Client can request a new position, which will instruct the server to modify that server instance's position NetworkVariable. That client will apply that NetworkVariable position inside of it's Update() method.

:::note Congrats   !!!!
Congratulations you have learnt the basics of a networked game 
:::