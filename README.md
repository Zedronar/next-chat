# Table of Contents

- [Next Chat](#next-chat)
    + [Tech Stack](#tech-stack)
    + [Database Schema](#database-schema)
      - [User](#user)
      - [ChatRoom](#chatroom)
      - [ChatRoomMessage](#chatroommessage)
    + [Initialization](#initialization)
    + [Registration](#registration)
      - [Data storage](#data-storage)
      - [Data access](#data-access)
      - [User Data in Redis HashSet](#user-data-in-redis-hashset)
    + [Rooms](#rooms)
      - [Data storage](#data-storage-1)
      - [Data access](#data-access-1)
      - [Get all My Rooms](#get-all-my-rooms)
    + [Messages](#messages)
      - [Pub Sub](#pub-sub)
      - [Data storage](#data-storage-2)
      - [Data access](#data-access-2)
      - [Send Message](#send-message)
    + [Session handling](#session-handling)
      - [Data storage and access](#data-storage-and-access)
- [To run it locally](#to-run-it-locally)
    + [Have latest .netcore SDK](#have-latest-netcore-sdk)
    + [Have Redis running](#have-redis-running)
    + [Write in environment variable or Dockerfile actual connection to Redis](#write-in-environment-variable-or-dockerfile-actual-connection-to-redis)
    + [Run backend](#run-backend)
- [Features I would like to add or could have done better if I had more time](#features-i-would-like-to-add-or-could-have-done-better-if-i-had-more-time)

# Next Chat

### Tech Stack

* .Net Core 5   -> Latest netcore version.
* Redis         -> Used as in-memory DB for storing rooms, users, and messages. Also used for pub/sub. ([StackExchangeRedis](https://github.com/StackExchange/StackExchange.Redis))
* ReactJS       -> Model binding and UI logic.
* SignalR       -> Provides some benefits over using raw websockets (e.g. transport fallback for environments where WebSockets is not available).
* Swagger       -> For easy API endpoints visualization and documentation.
* Postman       -> External tool for rapid testing.

Redis [pub/sub](https://redis.io/topics/pubsub) feature combined with [SignalR](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/websockets?view=aspnetcore-5.0#signalr) is used for message communication between client and server.

### Database Schema

#### User

```csharp
public class User : BaseEntity
{
  public int Id { get; set; }
  public string Username { get; set; }
  public bool Online { get; set; } = false;
}
```

#### ChatRoom

```csharp
public class ChatRoom : BaseEntity
{
  public string Id { get; set; }
  public IEnumerable<string> Names { get; set; }
}
```

#### ChatRoomMessage

```csharp
public class ChatRoomMessage : BaseEntity
{
  public string From { get; set; }
  public int Date { get; set; }
  public string Message { get; set; }
  public string RoomId { get; set; }
}
```

### Initialization

At first, a **total_users** Redis key is checked (`EXISTS total_users`) - if it does not exist, the Redis database is filled with initial dummy data.

 (checks if the key exists)

The dummy data initialization is handled in multiple steps:

**Creation of dummy users:**
A new user id is created using `INCR total_users`. Then we set a user ID lookup key by user name:

**_E.g._** `SET username:nick user:1`.

And finally, the rest of the data is written to the hash set:

**_E.g._** `HSET user:1 username "nick" password "bcrypt_hashed_password"`.

Additionally, each user is added to the default "General" room. For handling rooms for each user, there is a set that holds the room ids. To add a user to a room:

**_E.g._** `SADD user:1:rooms "0"`.

**Populate private messages between users:**
At first, private rooms are created: if a private room needs to be established, for each user a room id: `room:1:2` is generated, where numbers correspond to the user ids in ascending order.

**_E.g._** Create a private room between 2 users: `SADD user:1:rooms 1:2` and `SADD user:2:rooms 1:2`.

Then we add messages to this room by writing to a sorted set:

**_E.g._** `ZADD room:1:2 1615480369 "{'from': 1, 'date': 1615480369, 'message': 'Hello', 'roomId': '1:2'}"`.

A stringified _JSON_ is used for keeping the message structure and simplify the implementation details for this app.

**Populate the "General" room with messages.** Messages are added to the sorted set with id of the "General" room: `room:0`

### Registration

Redis is used mainly as a database to keep the user/messages data and for sending messages (pub/sub) between connected clients.

#### Data storage

- The chat data is stored in various keys and various data types.
  - User data is stored in a hash set where each user entry contains the next values:
    - `username`: unique user name
    - `password`: hashed password

* User hash set is accessed by key `user:{userId}`. The data for it is stored with `HSET key field data`. User id is calculated by incrementing the `total_users`.

  - E.g `INCR total_users`

* Username is stored as a separate key (`username:{username}`) which returns the userId for quicker access.
  - E.g `SET username:Alex 4`

#### Data access

- **Get User** `HGETALL user:{id}`

  - E.g `HGETALL user:2`, where we get data for the user with id: 2.

- **Online users:** will return ids of users which are online
  - E.g `SMEMBERS online_users`

#### User Data in Redis HashSet

```csharp
var usernameKey = $"username:{username}";
var hashedPassword = BCrypt.Net.BCrypt.HashPassword(password);
var nextId = await redisDatabase.StringIncrementAsync("total_users");
var userKey = $"user:{nextId}";
await redisDatabase.StringSetAsync(usernameKey, userKey);
await redisDatabase.HashSetAsync(userKey, new HashEntry[] {
    new HashEntry("username", username),
    new HashEntry("password", hashedPassword)
});
```

### Rooms

#### Data storage

Each user has a set of rooms associated with them.

**Rooms** are sorted sets which contains messages where score is the timestamp for each message. Each room has a name associated with it.

- Rooms which user belongs too are stored at `user:{userId}:rooms` as a set of room ids.

  - E.g `SADD user:Alex:rooms 1`

- Set room name: `SET room:{roomId}:name {name}`
  - E.g `SET room:1:name General`

#### Data access

- **Get room name** `GET room:{roomId}:name`.

  - E. g `GET room:0:name`. This should return "General"

- **Get room ids of a user:** `SMEMBERS user:{id}:rooms`.
  - E. g `SMEMBERS user:2:rooms`. This will return IDs of rooms for user with ID: 2

#### Get all My Rooms

```csharp
var roomIds = await _database.SetMembersAsync($"user:{userId}:rooms");
var rooms = new List<ChatRoom>();
foreach (var roomIdRedisValue in roomIds)
{
    var roomId = roomIdRedisValue.ToString();
    var name = await _database.StringGetAsync($"room:{roomId}:name");
    if (name.IsNullOrEmpty)
    {
        var roomExists = await _database.KeyExistsAsync($"room:{roomId}");
        if (!roomExists)
        {
            continue;
        }

        var userIds = roomId.Split(':');
        if (userIds.Length != 2)
        {
            throw new Exception("You don't have access to this room");
        }

        rooms.Add(new ChatRoom()
        {
            Id = roomId,
            Names = new List<string>() {
                (await _database.HashGetAsync($"user:{userIds[0]}", "username")).ToString(),
                (await _database.HashGetAsync($"user:{userIds[1]}", "username")).ToString(),
            }
        });
    }
    else
    {
        rooms.Add(new ChatRoom()
        {
            Id = roomId,
            Names = new List<string>() {
                name.ToString()
            }
        });
    }
}
return rooms;
```

### Messages

#### Pub Sub

After initialization, a pub/sub subscription is created: `SUBSCRIBE MESSAGES`. At the same time, each server instance will run a listener on a message on this channel to receive real-time updates.

Again, for simplicity, each message is serialized to **_JSON_**, which is parsed and then handled in the same manner, as WebSocket messages.

Pub/sub allows connecting multiple servers written in different platforms without taking into consideration the implementation detail of each server.

#### Data storage

- Messages are stored at `room:{roomId}` key in a sorted set (as mentioned above). They are added with `ZADD room:{roomId} {timestamp} {message}` command. Message is serialized to an app-specific JSON string.
  - E.g `ZADD room:0 1617197047 { "From": "2", "Date": 1617197047, "Message": "Hello", "RoomId": "1:2" }`

#### Data access

- **Get list of messages** `ZREVRANGE room:{roomId} {offset_start} {offset_end}`.
  - E.g `ZREVRANGE room:1:2 0 50` will return 50 messages with 0 offsets for the private room between users with IDs 1 and 2.

#### Send Message

```csharp
public async Task SendMessage(UserDto user, ChatRoomMessage message)
{
  await _database.SetAddAsync("online_users", message.From);
  var roomKey = $"room:{message.RoomId}";
  await _database.SortedSetAddAsync(roomKey, JsonConvert.SerializeObject(message), (double)message.Date);
  await PublishMessage("message", message);
}
```

### Session handling

The chat server works as a basic _REST_ API which involves keeping the session and handling the user state in the chat rooms (besides the WebSocket/real-time part).

When a WebSocket/real-time server is instantiated, which listens for the next events:

**Connection**. A new user is connected. At this point, a user ID is captured and saved to the session (which is cached in Redis). Note, that session caching is language/library-specific and it's used here purely for persistence and maintaining the state between server reloads.

A global set with `online_users` key is used for keeping the online state for each user. So on a new connection, a user ID is written to that set:

**E.g.** `SADD online_users 1` (We add user with id 1 to the set **online_users**).

After that, a message is broadcasted to the clients to notify them that a new user is joined the chat.

**Disconnect**. It works similarly to the connection event, except we need to remove the user for **online_users** set and notify the clients: `SREM online_users 1` (makes user with id 1 offline).

**Message**. A user sends a message, and it needs to be broadcasted to the other clients. The pub/sub allows us also to broadcast this message to all server instances which are connected to this Redis:

`PUBLISH message "{'serverId': 4132, 'type':'message', 'data': {'from': 1, 'date': 1615480369, 'message': 'Hello', 'roomId': '1:2'}}"`

Note: There is additional data related to the type of the message and the server id being sent.
Server id is used to discard the messages by the server instance which sends them since it is connected to the same `MESSAGES` channel.

`type` field of the serialized JSON corresponds to the real-time method we use for real-time communication (connect/disconnect/message).

`data` is method-specific information. In the example above it's related to the new message.

#### Data storage and access

The session data is stored in Redis by utilizing the [**StackExchange.Redis**](https://github.com/StackExchange/StackExchange.Redis) client.

```csharp
services
    .AddDataProtection()
    .PersistKeysToStackExchangeRedis(redis, "DataProtectionKeys");

services.AddStackExchangeRedisCache(option =>
{
    option.Configuration = redisConnectionUrl;
    option.InstanceName = "RedisInstance";
});

services.AddSession(options =>
{
    options.IdleTimeout = TimeSpan.FromMinutes(30);
    options.Cookie.Name = "AppTest";
});
```

# To run it locally

### Have latest .netcore SDK

[Download here](https://dotnet.microsoft.com/download/dotnet/5.0)

### Have Redis running

Two alternatives:
* Have a [redis container](https://hub.docker.com/_/redis) running with default port (6379) exposed
* [Download redis server for windows here](https://medium.com/cook-php/how-to-run-redis-on-windows-97b829a42486)

### Write in environment variable or Dockerfile actual connection to Redis

```
   REDIS_ENDPOINT_URL = "Redis server URI:PORT_NUMBER"
   REDIS_PASSWORD = "Password to the server"
```

### Build frontend

```sh
cd WebApp/Chat/client
npm run build
```
Or for hot-reload:
```sh
yarn watch
```

### Run backend (Chatbot)

```sh
cd Chatbot
dotnet watch run
```

### Run backend (WebApp)

```sh
cd WebApp/Chat
dotnet watch run
```

### Run Offline-DirectLine

```sh
cd WebApp
directline -d 3000 -b "http://127.0.0.1:3978/api/messages"
```

# TODO (if I had more time)

- [ ] Dockerize + docker-compose
- [ ] Use a container orchestration technology (Kubernetes).
- [ ] Messages are being stored in memory. Ideally add persistent storage and use Redis only for caching.
- [ ] Host it in a cloud provider (Azure, AWS or GCP).
- [ ] Add a metrics server/provider (e.g. Datadog) and emit meaningful metrics.
- [ ] Setup CI/CD for the project.
