## Video Presentation
[Watch on YouTube](https://youtu.be/PrsQHDl_G8I)

# Real-time data technology on real-world use case

**Group 6**

**Members**: Zhe, Zhang; Zhiyuan, Lu

**Use Case**: Real-time chat application: Discord

**Task distribution**:

|            Zhiyuan, Lu            |         Zhe, Zhang         |
|-----------------------------------|----------------------------|
| Documentation setup               |  Presentation preparation  |
| Section 1. Rest&GraphQL use cases |  Section 1. pros and cons  |
| Section 2. Websocet workflow      |  Section 2. Differences and use case |
| Section 3. Justification          |  Section 3. Recommendation |

## Introduction:

For most famous real-time chat applications, such as Discord, WhatsApp, WeChat, we believe they utilize REST, GraphQL and WebSockets for sure. But unfortunately we cannot find single documentation that describes their approach, so we will have to express our thoughts through a fictional product that is based on Discord’s features, but in a simpler way. 


## Section 1: REST and GraphQL for Data Requests and Updates

In the following, we will discuss the common features that Discord have, and compare them by utilizing RESTful and GraphQL. 
Let's first take a look for a few examples in different scenarios:

> [!NOTE]  
> Since GraphQL use single `POST /graphql` endpoint, we will only include the `mutation` or `query`.

### Authentication
#### POST: Login, registration, etc.
##### Rest
endpoint: `POST /auth/login` ,

body: 
```json
{ "email": "user@example.com", "password": "userpassword" }
```
respond: 
```json
{ "token": "jwt-token" }
```

##### GraphQL
mutation: 
```graphql
mutation {
  login(email: "user@example.com", password: "userpassword") {
    token
  }
}
```

respond:
```json
{
  "data": {
    "login": {
      "token": "jwt-token"
    }
  }
}

```
### **Error Handling Example**
#### **REST API Error**
```json
{
  "error": "Invalid credentials",
  "status": 401
}

```
#### **GraphQL API Error**
```json
{
  "errors": [
    {
      "message": "User not found",
      "extensions": { "code": "USER_NOT_FOUND" }
    }
  ]
}
```
#### GET: Get user profile
##### Rest
endpoint: `GET /user/profile?id={uid}`

respond:
```json
{
  "id": "12345",
  "username": "user",
  "avatar": "url_of_image"
  ...ProfileData
}
```
##### GraphQl
query: 
```graphQL
query GetUserProfile($id: ID!) {
  user(id: $id) {
    id
    username
    avatar
  }
}
```
respond:
```json
{
  "data": {
    "user": {
      "id": "12345",
      "username": "user",
      "avatar": "profile_pic_url"
    }
  }
}
```
For this case, when logging in, there's no significant differences between graphql and rest api, because the required field and respond is fixed. But for the getting user profile part, there's a big difference. Imagine this case:
> In a mature product like discord, there are many props in user's profile, by default, when fetching user profile, REST api will return a huge object that contains all the relative data. If you are in a server that has 5000 members, it will take very long time and a lot space to cache everyone's data, even though you probably only need their name and avatar. With graphQL, you can only fetch the required field as needed.

Although we can modify REST api to return a smaller object, What if suddenly we want to display a new prop, like a badge? We can simply add a field into the graphql query, but we need to do much more changes in order to make REST api.

So in certain cases, graphql is **more flexible** than REST.

Let's take a look at another example:

### Messaging
#### POST: sending message
##### REST
endpoint: `POST /channels/{channel_id}/messages`
body:
```json
{
  "content": "Hey, check this out!"
}
```
respond:
```json
{
  "id": "00001",
  "content": "Hey, check this out!",
  "author": "user001",
  "timestamp": "2025-02-13T12:00:00Z",
  ...OtherProps
}
```
##### GraphQL
mutation:
```graphql
mutation {
  sendMessage(channelId: "000123", content: "Hey, check this out!",
  }) {
    id
    content
    author {
      username
    }
    timestamp
  }
}
```
respond:
```json
{
  "data": {
    "sendMessage": {
      "id": "000123",
      "content": "Hey, check this out!",
      "author": {
        "username": "user123"
      },
      "timestamp": "2025-02-13T12:00:00Z"
    }
  }
}
```
#### PATCH: modify a message
##### REST
endpoint: `PATCH /channels/{channel_id}/messages/{message_id}`,
body: `{newContent: "Hey there again!"}`
respond: 
```json
{
  "data": {
    "sendMessage": {
      "id": "000123",
      "content": "Hey there again!",
      "author": {
        "username": "user123"
      },
      "timestamp": "2025-02-13T12:30:00Z"
    }
  }
}
```
##### GraphQL
mutation:
```graphql
mutation {
  editMessage(messageId: "000123", content: "Hey there again!") {
    id
    content
    timestamp
  }
}
```
respond: 
```json
{
  "data": {
    "editMessage": {
      "id": "000123",
      "content": "Hey there again!",
      "author": {
        "username": "user123"
      },
      "timestamp": "2025-02-13T12:30:00Z"
    }
  }
}
```
#### DELETE: delete a message
##### REST
endpoint: `DELETE /channels/{channel_id}/messages/{message_id}`,
respond: `true`

##### GraphQl
mutation:
```graphql
mutation {
  deleteMessage(messageId: "000123") {
    success
    message
  }
}
```
respond:
```json
{
  "data": {
    "deleteMessage": {
      "success": true,
      "message": "Message deleted successfully"
    }
  }
}
```
In this case, the endpoint won't change frequently, and the data that returned is not very complicated, instead, it's a large amount of small objects. REST is actually better in this case, because messages requires caching, and operation like traditional CRUD operation is simpler with REST api.

Here's a table that specify either GraphQL or REST is better:

|       Case        |   Choice   |           Reason            |
|-------------------|------------|-----------------------------|
|  Log in/Register  |    REST    |  Simple response, but generally more secure than graphql  |
|  Get profile      |    GraphQL |  Flexible data query   |
|  Messaging        |    REST    |  No need for customizing caching rules, and REST works better with flat data like messages.   |
|  Complex/Nested Data(rules, etc.) | GraphQL | GraphQL can fetch deeply nested data in a single request | 
|  Third Party development (Discord Bot) | REST | Simple for integration and documentation |


## Section 2: WebSockets for Real-time Communication

1. **Event Subscription**: WebSocket allows users to receive real-time notifications when certain events occur, such as:
   - **New messages**: Notifying users when a new message arrives.
   - **Message updates**: Receiving updates when a message is edited or deleted.
   - **User mentions**: Sending alerts when a user is mentioned in a group chat.
   - **Channel updates**: Notifying users about changes in channels, roles, or settings.
   - **Presence updates**: Informing users when their friends come online or change their status.

2. Service status polling: The server will keep tracking the current service status such as latency when user connected to the video stream or voice channel.
3. Account status: There's an option that allows user to log out at all other devices, which can be done with websocket
4. Event bridge: Trigger third-party service upon specific event, such as discord bot can be programmed to react on certain events, this is done by websocket.

Note: Unlike simpler real-time communication apps, Discord does not use WebSocket for sending messages directly. Instead, WebSocket is primarily used for event notifications, status updates, and presence tracking.

### Differ from REST and GraphQL

WebSocket is a long-live connection between client and server, it will remain during the app's lifecycle. REST and GraphQL is stateless, undirectional, and it's more like a one-time communicate, which each request is standalone.

Websocket is using TCP protocol, which REST and GraphQL is using HTTP/HTTPS. REST and GraphQL can be used for polling, but still not perfect choice for high-frequency updates. Polling too frequent will increase the bandwidth usage and latency, especially when in the situation that hundres of thouthands users using it at the same time.


Consider the following two workflow, and see how they work differently:

> 1. User wants to log out all the devices that has his account.

**Websocket**: Server gets the command, and ask other clients to sign out themselves
<img width="1041" alt="Screenshot 2025-02-14 at 3 26 38 AM" src="https://github.com/user-attachments/assets/78966a35-1f8d-4d39-9ec7-cd1b6a468196" />

**REST**: Client keep asking server, once the state change on server, client will know at next communication, and proceed corresponding
<img width="1041" alt="Screenshot 2025-02-14 at 3 26 15 AM" src="https://github.com/user-attachments/assets/2a9a8762-2fd8-4dbf-b7b5-64ad8b68faa3" />


> 2. Discord publish a new version of the app, which tells user to update

**REST**: Instead of frequent polling, it might contact to server per day, or twice a day, etc.
<img width="793" alt="Screenshot 2025-02-14 at 3 32 21 AM" src="https://github.com/user-attachments/assets/b38a66f4-ba38-4286-a20f-1b05f46b20d2" />

**Websocket**: Establishing the connection all the time, but there might only be one message every month.
<img width="793" alt="Screenshot 2025-02-14 at 3 33 45 AM" src="https://github.com/user-attachments/assets/4f29d5c6-f29a-42f2-8d2e-fbbfb5404222" />


## Section 3: Technology Recommendation and Justification

### Justification

We are going for a hybrid approach, which utilizing REST, GraphQL, and WebSockers together, each one of them is responsible for different tasks. In this way, it balance the performance, flexibility and security.

#### 1. REST API vs. GraphQL
- REST API is more suitable for:
  - Authentication and user management, which REST is simpler and more secure, which does not rely on the complex function calling to finish the task.
  - Message operations such as sending, deleting, and modifying messages. The structure of messages and operations are relatively "stable", which GraphQL can only increase the complexity, but not actually enhancing the performance.
  - Third-party integrations, including bot interactions and external API calls, using REST is making the integration easier and more straight forward.

- GraphQL is beneficial for:
  - Fetching complex and nested data structures in a single request (e.g., retrieving user profile, friends, and roles).
  - Minimizing over-fetching and under-fetching by allowing clients to request only the necessary fields.
  - Efficient client-side customization without requiring backend modifications.

By combining both, REST API ensures **structured data management**, while GraphQL enhances **data query efficiency** in situations where flexibility is needed.

#### 2. WebSocket vs. REST API
- WebSocket is essential for real-time features:
  - Event-driven architecture allows instant updates.
  - Eliminates unnecessary API polling, reducing server load and improving responsiveness.
  - Maintains a persistent connection, ensuring live notifications for messages, mentions, and user status updates.

- REST API is still required for:
  - Sending messages (`POST /channels/{id}/messages`) to ensure data integrity and consistency.
  - Retrieving message history (`GET /channels/{id}/messages`) when users scroll up in a chat.
  - Handling secure authentication, which is stateless and requires each request to be validated.

Thus, WebSocket enables real-time communication, while REST API maintains structured data integrity.

### **Recommendation: Best Practices and Future Improvements**

Based on the findings from Section 1 and Section 2, the current architecture provides a solid foundation, but certain optimizations can improve efficiency and scalability.

#### **1. Enhancing WebSocket Stability**
- **Implementing heartbeat mechanisms** to detect inactive connections and prevent unnecessary disconnects.
- **Session resumption** to allow users to reconnect without needing to reload all data.
- **Rate limiting** on WebSocket messages to prevent spam and reduce excessive server load.

#### **2. Improving REST API Efficiency**
- **Caching frequently requested data** (e.g., user profiles, static server information) to reduce redundant API calls.
- **Implementing rate limiting** to prevent excessive requests from overloading the system.
- **Using WebSocket for real-time updates instead of API polling**, reducing unnecessary REST API calls.

#### **3. Optimizing GraphQL Performance**
- **Limiting nested queries depth** to prevent complex queries from overwhelming the database.
- **Using persistent queries** to minimize repeated GraphQL parsing and execution time.
- **Caching common GraphQL queries** to improve response time for frequently accessed data.

By applying these improvements, the system can **reduce latency, optimize bandwidth usage, and scale efficiently** while ensuring a **seamless real-time experience** for users.

---

## **Conclusion**
The combination of **REST API, GraphQL, and WebSocket** is the best approach for handling **real-time messaging, structured data retrieval, and efficient queries**.  
- **WebSocket** provides **low-latency, real-time updates**.
- **REST API** ensures **secure authentication and reliable message handling**.
- **GraphQL** enables **optimized, flexible data fetching**.

With continued **performance optimizations**, including **better connection management, caching, and query optimizations**, the system will be able to **handle large-scale real-time interactions while maintaining efficiency and reliability**.

## Demonstration

Our presentation slides is avaliable at: https://docs.google.com/presentation/d/1l2z5QtMLzUWkMGKm08vCXmWulXTKhdK3v-xI2DHiB9c/edit?usp=sharing

## Reference:

### Gen AI citation

Expand the following list in order to see AI's respond.

<details>
  <summary>1. How to Create foldable markdown content</summary>  
  
  > In Markdown, you can create foldable (collapsible) content using HTML \<details\> and \<summary\> tags. Markdown itself doesn’t support collapsible sections natively, but most platforms that render Markdown with HTML (like GitHub, GitLab, and some documentation tools) will support this approach.
  >    Example: 
  >    ```markdown
  >    <details>
  >      <summary>Click to expand</summary>
  >    
  >      This is the hidden content that appears when you expand the section.
  >    
  >      - You can include lists
  >      - **Bold** and *italic* text
  >      - Even `code snippets`
  >    
  >    </details>
  >    ```
</details>

<details>
  <summary>2. Does discord use any graphql or websocket techniques?</summary>

  > Discord utilizes WebSocket technology extensively in its products to facilitate real-time communication. The Discord Gateway, responsible for maintaining WebSocket connections, handles     
  > events such as message creation and user status updates, ensuring instantaneous interactions within the platform. 
  >  DISCORDENO
  >
  > Regarding GraphQL, Discord currently does not offer a public GraphQL API. Discussions among developers have highlighted the potential benefits of adopting GraphQL, such as reduced bandwidth 
  > usage and the ability to run multiple queries in a single request. However, integrating GraphQL into Discord's existing API would be a substantial undertaking. As of now, Discord continues 
  > to utilize its REST API for client-server communications. 
  > GITHUB.COM
> 
</details>

<details>
  <summary>3. Help me translate endpoint: `GET /user/profile?id={uid}` into graphQL</summary>
  
  >    ```graphQL
  >    query GetUserProfile($id: ID!) {
  >      userProfile(id: $id) {
  >        id
  >        name
  >        email
  >        avatarUrl
  >      }
  >    }
  >    ```
>      
</details>

<details>
  <summary>4. Is using websocket to monitor the service status(e.g. connection consistency) a potential approach?</summary>
  
  >  Yes, using WebSockets to monitor service status (such as connection consistency) is a potential approach, but it depends on the specific requirements and trade-offs.
  >
  >  Pros of Using WebSockets for Monitoring
  >  ✅ **Real-time Updates** – WebSockets provide instant feedback on connection status without polling.  
  >  ✅ **Persistent Connection** – WebSockets maintain a continuous connection, making it easy to detect drops.  
  >  ✅ **Lower Latency** – Compared to HTTP polling, WebSockets offer faster failure detection.  
  >  
  >  How It Can Work
  >  - **Heartbeat/Ping-Pong Messages** – The client and server periodically send "ping" messages, and the other side responds with a "pong." If no response is received within a timeout, the      >   connection is assumed to be lost.  
  >  - **Reconnect Logic** – If a WebSocket connection closes unexpectedly, the client can attempt to reconnect.  
  >  - **Status Event Streams** – The server can push status updates when issues arise (e.g., "server overload" messages).  
  >  
  >  Potential Challenges
  >  ⚠️ **WebSocket Overhead** – Maintaining many WebSocket connections can be resource-intensive.  
  >  ⚠️ **Network Instability** – Fluctuations in network conditions may trigger false positives for disconnection events.  
  >  ⚠️ **Load Balancing Issues** – WebSockets require sticky sessions (or a message broker) to maintain state across multiple servers.  
  >  
  >  Alternative Approaches
  >  - **Short Polling**: Periodically send HTTP requests to check the service status.  
  >  - **Long Polling**: Keep an HTTP request open until the server has a status update.  
  >  - **gRPC Streaming**: Uses a persistent HTTP/2 stream (like WebSockets but with better built-in error handling).  
  >  
  >  If real-time monitoring with low latency is the priority, **WebSockets are a solid choice**, but if scalability and reliability across multiple servers are more critical, alternatives like   >   **gRPC streaming** or managed monitoring solutions (e.g., **Prometheus + Webhooks**) might be better. 🚀

  
</details>

### Websites:
- https://discord.com/developers/docs/reference
