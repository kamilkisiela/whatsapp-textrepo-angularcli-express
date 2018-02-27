# Step 15: Subscriptions

[//]: # (head-end)


## Server

In order to use WebSockets we don't need to install any package, it comes for free with Apollo Server 2.0.

Our GraphQL server will use WebSockets only for subscriptions, while using HTTP for everything else. That means that we will have to add subscriptions on a specific path.
We're using `connectionParams` for the authentication over WebSockets, that means that we won't be using the `Passport` framework at all. Instead we will use the `onConnect` hook to manually validate the parameters provided by the user to either validate the WebSocket connection or throw an error.
We will also return the user object we retrieved from the db, to let the resolvers know who is the current user.

[{]: <helper> (diffStep "5.1" files="index.ts" module="server")

#### Step 5.1: Subscriptions

##### Changed index.ts
```diff
@@ -7,9 +7,12 @@
 ┊ 7┊ 7┊import * as basicStrategy from 'passport-http';
 ┊ 8┊ 8┊import * as bcrypt from 'bcrypt-nodejs';
 ┊ 9┊ 9┊import { db, User } from "./db";
+┊  ┊10┊import { createServer } from "http";
 ┊10┊11┊
 ┊11┊12┊let users = db.users;
 ┊12┊13┊
+┊  ┊14┊console.log(users);
+┊  ┊15┊
 ┊13┊16┊function generateHash(password: string) {
 ┊14┊17┊  return bcrypt.hashSync(password, bcrypt.genSaltSync(8));
 ┊15┊18┊}
```
```diff
@@ -69,9 +72,30 @@
 ┊ 69┊ 72┊  schema,
 ┊ 70┊ 73┊  context(received: any) {
 ┊ 71┊ 74┊    return {
-┊ 72┊   ┊      user: received.req!['user'],
+┊   ┊ 75┊      user: received.connection ? received.connection.context.user : received.req!['user'],
 ┊ 73┊ 76┊    }
 ┊ 74┊ 77┊  },
+┊   ┊ 78┊  subscriptions: {
+┊   ┊ 79┊    onConnect: (connectionParams: any, webSocket: any) => {
+┊   ┊ 80┊      if (connectionParams.authToken) {
+┊   ┊ 81┊        // create a buffer and tell it the data coming in is base64
+┊   ┊ 82┊        const buf = new Buffer(connectionParams.authToken.split(' ')[1], 'base64');
+┊   ┊ 83┊        // read it back out as a string
+┊   ┊ 84┊        const [username, password]: string[] = buf.toString().split(':');
+┊   ┊ 85┊        if (username && password) {
+┊   ┊ 86┊          const user = users.find(user => user.username == username);
+┊   ┊ 87┊
+┊   ┊ 88┊          if (user && validPassword(password, user.password)) {
+┊   ┊ 89┊            // Set context for the WebSocket
+┊   ┊ 90┊            return {user};
+┊   ┊ 91┊          } else {
+┊   ┊ 92┊            throw new Error('Wrong credentials!');
+┊   ┊ 93┊          }
+┊   ┊ 94┊        }
+┊   ┊ 95┊      }
+┊   ┊ 96┊      throw new Error('Missing auth token!');
+┊   ┊ 97┊    }
+┊   ┊ 98┊  }
 ┊ 75┊ 99┊});
 ┊ 76┊100┊
 ┊ 77┊101┊apollo.applyMiddleware({
```
```diff
@@ -79,4 +103,11 @@
 ┊ 79┊103┊  path: '/graphql'
 ┊ 80┊104┊});
 ┊ 81┊105┊
-┊ 82┊   ┊app.listen(PORT);
+┊   ┊106┊// Wrap the Express server
+┊   ┊107┊const ws = createServer(app);
+┊   ┊108┊
+┊   ┊109┊apollo.installSubscriptionHandlers(ws);
+┊   ┊110┊
+┊   ┊111┊ws.listen(PORT, () => {
+┊   ┊112┊  console.log(`Apollo Server is now running on http://localhost:${PORT}`);
+┊   ┊113┊});
```

[}]: #

We will use the `PubSub` implementation from `apollo-server-express`, and publish the data using with Apollo Server.

The process of setting up a GraphQL subscriptions server consist of the following steps:

1. Declaring subscriptions in the GraphQL schema
2. Setup a PubSub instance that our server will publish new events to
3. Hook together `PubSub` event and GraphQL subscription.
4. Setting up `ApolloServer`

[{]: <helper> (diffStep "5.1" files="schema/typeDefs.ts" module="server")

#### Step 5.1: Subscriptions

##### Changed schema&#x2F;typeDefs.ts
```diff
@@ -5,6 +5,11 @@
 ┊ 5┊ 5┊    chat(chatId: ID!): Chat
 ┊ 6┊ 6┊  }
 ┊ 7┊ 7┊
+┊  ┊ 8┊  type Subscription {
+┊  ┊ 9┊    messageAdded(chatId: ID): Message
+┊  ┊10┊    chatAdded: Chat
+┊  ┊11┊  }
+┊  ┊12┊
 ┊ 8┊13┊  enum MessageType {
 ┊ 9┊14┊    LOCATION
 ┊10┊15┊    TEXT
```

[}]: #

We created two subscriptions: one to notify for new chats and one to notify for new messages.

[{]: <helper> (diffStep "5.1" files="schema/resolvers.ts" module="server")

#### Step 5.1: Subscriptions

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,7 +1,7 @@
-┊1┊ ┊import { IResolvers } from 'apollo-server-express';
+┊ ┊1┊import { PubSub, withFilter, IResolvers } from 'apollo-server-express';
 ┊2┊2┊import { Chat, db, Message, MessageType, Recipient, User } from "../db";
 ┊3┊3┊import {
-┊4┊ ┊  AddChatMutationArgs, AddGroupMutationArgs, AddMessageMutationArgs, ChatQueryArgs,
+┊ ┊4┊  AddChatMutationArgs, AddGroupMutationArgs, AddMessageMutationArgs, ChatQueryArgs, MessageAddedSubscriptionArgs,
 ┊5┊5┊  RemoveChatMutationArgs, RemoveMessagesMutationArgs
 ┊6┊6┊} from "../types";
 ┊7┊7┊import * as moment from "moment";
```
```diff
@@ -9,6 +9,8 @@
 ┊ 9┊ 9┊let users = db.users;
 ┊10┊10┊let chats = db.chats;
 ┊11┊11┊
+┊  ┊12┊export const pubsub = new PubSub();
+┊  ┊13┊
 ┊12┊14┊export const resolvers: IResolvers = {
 ┊13┊15┊  Query: {
 ┊14┊16┊    // Show all users for the moment.
```
```diff
@@ -50,6 +52,7 @@
 ┊50┊52┊          messages: [],
 ┊51┊53┊        };
 ┊52┊54┊        chats.push(chat);
+┊  ┊55┊
 ┊53┊56┊        return chat;
 ┊54┊57┊      }
 ┊55┊58┊    },
```
```diff
@@ -73,6 +76,12 @@
 ┊73┊76┊        messages: [],
 ┊74┊77┊      };
 ┊75┊78┊      chats.push(chat);
+┊  ┊79┊
+┊  ┊80┊      pubsub.publish('chatAdded', {
+┊  ┊81┊        creatorId: currentUser.id,
+┊  ┊82┊        chatAdded: chat,
+┊  ┊83┊      });
+┊  ┊84┊
 ┊76┊85┊      return chat;
 ┊77┊86┊    },
 ┊78┊87┊    removeChat: (obj: any, {chatId}: RemoveChatMutationArgs, {user: currentUser}: {user: User}): number => {
```
```diff
@@ -201,6 +210,11 @@
 ┊201┊210┊          });
 ┊202┊211┊
 ┊203┊212┊          holderIds = listingMemberIds;
+┊   ┊213┊
+┊   ┊214┊          pubsub.publish('chatAdded', {
+┊   ┊215┊            creatorId: currentUser.id,
+┊   ┊216┊            chatAdded: chat,
+┊   ┊217┊          });
 ┊204┊218┊        }
 ┊205┊219┊      } else {
 ┊206┊220┊        // Group
```
```diff
@@ -245,6 +259,10 @@
 ┊245┊259┊        return chat;
 ┊246┊260┊      });
 ┊247┊261┊
+┊   ┊262┊      pubsub.publish('messageAdded', {
+┊   ┊263┊        messageAdded: message,
+┊   ┊264┊      });
+┊   ┊265┊
 ┊248┊266┊      return message;
 ┊249┊267┊    },
 ┊250┊268┊    removeMessages: (obj: any, {chatId, messageIds, all}: RemoveMessagesMutationArgs, {user: currentUser}: {user: User}): number[] => {
```
```diff
@@ -286,6 +304,21 @@
 ┊286┊304┊      return deletedIds;
 ┊287┊305┊    },
 ┊288┊306┊  },
+┊   ┊307┊  Subscription: {
+┊   ┊308┊    messageAdded: {
+┊   ┊309┊      subscribe: withFilter(() => pubsub.asyncIterator('messageAdded'),
+┊   ┊310┊        ({messageAdded}: {messageAdded: Message & {chat: {id: number}}}, {chatId}: MessageAddedSubscriptionArgs, {user: currentUser}: { user: User }) => {
+┊   ┊311┊          return (!chatId || messageAdded.chat.id === Number(chatId)) &&
+┊   ┊312┊            !!messageAdded.recipients.find((recipient: Recipient) => recipient.userId === currentUser.id);
+┊   ┊313┊        }),
+┊   ┊314┊    },
+┊   ┊315┊    chatAdded: {
+┊   ┊316┊      subscribe: withFilter(() => pubsub.asyncIterator('chatAdded'),
+┊   ┊317┊        ({creatorId, chatAdded}: {creatorId: string, chatAdded: Chat}, variables: any, {user: currentUser}: { user: User }) => {
+┊   ┊318┊          return Number(creatorId) !== currentUser.id && !chatAdded.listingMemberIds.includes(currentUser.id);
+┊   ┊319┊        }),
+┊   ┊320┊    }
+┊   ┊321┊  },
 ┊289┊322┊  Chat: {
 ┊290┊323┊    name: (chat: Chat, args: any, {user: currentUser}: {user: User}): string => chat.name ? chat.name : users
 ┊291┊324┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.name,
```

[}]: #

We will publish a message to the `messageAdded` subscription every time that a user sends a message, then we will filter them according to the current user (we don't want to send someone else's messages).
The `chatAdded` subscription is similar: we will publish each time that a group gets created, but not when chats get created. This is because when a user creates a chat the chat doesn't appear to the other peer until he writes the first message. That's why we also publish when new messages get added (we first look if the other peer already gets the chat listed).

## Client

In order to use WebSockets we will need to install a couple of dependencies:

    $ npm install apollo-link-ws apollo-utilities subscriptions-transport-ws

First let's create the queries for the GraphQL Subscriptions:

[{]: <helper> (diffStep "11.1" files="src/graphql" module="client")

#### Step 11.1: Subscriptions

##### Added src&#x2F;graphql&#x2F;chatAdded.subscription.ts
```diff
@@ -0,0 +1,17 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊import {fragments} from './fragment';
+┊  ┊ 3┊
+┊  ┊ 4┊// We use the gql tag to parse our query string into a query document
+┊  ┊ 5┊export const chatAddedSubscription = gql`
+┊  ┊ 6┊  subscription chatAdded {
+┊  ┊ 7┊    chatAdded {
+┊  ┊ 8┊      ...ChatWithoutMessages
+┊  ┊ 9┊      messages {
+┊  ┊10┊        ...Message
+┊  ┊11┊      }
+┊  ┊12┊    }
+┊  ┊13┊  }
+┊  ┊14┊
+┊  ┊15┊  ${fragments['chatWithoutMessages']}
+┊  ┊16┊  ${fragments['message']}
+┊  ┊17┊`;
```

##### Added src&#x2F;graphql&#x2F;messageAdded.subscription.ts
```diff
@@ -0,0 +1,16 @@
+┊  ┊ 1┊import gql from 'graphql-tag';
+┊  ┊ 2┊import {fragments} from './fragment';
+┊  ┊ 3┊
+┊  ┊ 4┊// We use the gql tag to parse our query string into a query document
+┊  ┊ 5┊export const messageAddedSubscription = gql`
+┊  ┊ 6┊  subscription messageAdded($chatId: ID) {
+┊  ┊ 7┊    messageAdded(chatId: $chatId) {
+┊  ┊ 8┊      ...Message
+┊  ┊ 9┊      chat {
+┊  ┊10┊        id,
+┊  ┊11┊      },
+┊  ┊12┊    }
+┊  ┊13┊  }
+┊  ┊14┊
+┊  ┊15┊  ${fragments['message']}
+┊  ┊16┊`;
```

[}]: #

Then we need to run `graphql-code-generator` to generate the types:

    $ npm run generator

Now we can update the chats service to update the getChats query every time that we receive a new chat from the subscription.
With GraphQL subscriptions your client will be alerted on push from the server and you should choose the pattern that fits your application the most:

- Use it as a notification and run any logic you want when it fires, for example alerting the user or refetching data
- Use the data sent along with the notification and merge it directly into the store (existing queries are automatically notified)

With subscribeToMore, you can easily do the latter. We will manipulate the store to add the newly created chat.

We will do to do the same for the newMessage subscription, but this time we will have to update two different queries in the store: getChats and getChat.

[{]: <helper> (diffStep "11.1" files="src/app/services/chats.service.ts" module="client")

#### Step 11.1: Subscriptions

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -3,7 +3,10 @@
 ┊ 3┊ 3┊import {Apollo, QueryRef} from 'apollo-angular';
 ┊ 4┊ 4┊import {Injectable} from '@angular/core';
 ┊ 5┊ 5┊import {getChatsQuery} from '../../graphql/getChats.query';
-┊ 6┊  ┊import {AddChat, AddGroup, AddMessage, GetChat, GetChats, GetUsers, RemoveAllMessages, RemoveChat, RemoveMessages} from '../../types';
+┊  ┊ 6┊import {
+┊  ┊ 7┊  AddChat, AddGroup, AddMessage, GetChat, GetChats, GetUsers, MessageAdded, RemoveAllMessages, RemoveChat,
+┊  ┊ 8┊  RemoveMessages
+┊  ┊ 9┊} from '../../types';
 ┊ 7┊10┊import {getChatQuery} from '../../graphql/getChat.query';
 ┊ 8┊11┊import {addMessageMutation} from '../../graphql/addMessage.mutation';
 ┊ 9┊12┊import {removeChatMutation} from '../../graphql/removeChat.mutation';
```
```diff
@@ -17,6 +20,8 @@
 ┊17┊20┊import * as moment from 'moment';
 ┊18┊21┊import {FetchResult} from 'apollo-link';
 ┊19┊22┊import {LoginService} from '../login/services/login.service';
+┊  ┊23┊import {chatAddedSubscription} from '../../graphql/chatAdded.subscription';
+┊  ┊24┊import {messageAddedSubscription} from '../../graphql/messageAdded.subscription';
 ┊20┊25┊
 ┊21┊26┊@Injectable()
 ┊22┊27┊export class ChatsService {
```
```diff
@@ -35,6 +40,55 @@
 ┊35┊40┊        amount: this.messagesAmount,
 ┊36┊41┊      },
 ┊37┊42┊    });
+┊  ┊43┊
+┊  ┊44┊    this.getChatsWq.subscribeToMore({
+┊  ┊45┊      document: chatAddedSubscription,
+┊  ┊46┊      updateQuery: (prev: GetChats.Query, { subscriptionData }) => {
+┊  ┊47┊        if (!subscriptionData.data) {
+┊  ┊48┊          return prev;
+┊  ┊49┊        }
+┊  ┊50┊
+┊  ┊51┊        const newChat: GetChats.Chats = subscriptionData.data.chatAdded;
+┊  ┊52┊
+┊  ┊53┊        return Object.assign({}, prev, {
+┊  ┊54┊          chats: [...prev.chats, newChat]
+┊  ┊55┊        });
+┊  ┊56┊      }
+┊  ┊57┊    });
+┊  ┊58┊
+┊  ┊59┊    this.getChatsWq.subscribeToMore({
+┊  ┊60┊      document: messageAddedSubscription,
+┊  ┊61┊      updateQuery: (prev: GetChats.Query, { subscriptionData }) => {
+┊  ┊62┊        if (!subscriptionData.data) {
+┊  ┊63┊          return prev;
+┊  ┊64┊        }
+┊  ┊65┊
+┊  ┊66┊        const newMessage: MessageAdded.MessageAdded = subscriptionData.data.messageAdded;
+┊  ┊67┊
+┊  ┊68┊        // We need to update the cache for both Chat and Chats. The following updates the cache for Chat.
+┊  ┊69┊        try {
+┊  ┊70┊          // Read the data from our cache for this query.
+┊  ┊71┊          const {chat}: GetChat.Query = this.apollo.getClient().readQuery({
+┊  ┊72┊            query: getChatQuery, variables: {
+┊  ┊73┊              chatId: newMessage.chat.id,
+┊  ┊74┊            }
+┊  ┊75┊          });
+┊  ┊76┊
+┊  ┊77┊          // Add our message from the mutation to the end.
+┊  ┊78┊          chat.messages.push(newMessage);
+┊  ┊79┊          // Write our data back to the cache.
+┊  ┊80┊          this.apollo.getClient().writeQuery({ query: getChatQuery, data: {chat} });
+┊  ┊81┊        } catch {
+┊  ┊82┊          console.error('The chat we received an update for does not exist in the store');
+┊  ┊83┊        }
+┊  ┊84┊
+┊  ┊85┊        return Object.assign({}, prev, {
+┊  ┊86┊          chats: [...prev.chats.map(_chat =>
+┊  ┊87┊            _chat.id === newMessage.chat.id ? {..._chat, messages: [..._chat.messages, newMessage]} : _chat)]
+┊  ┊88┊        });
+┊  ┊89┊      }
+┊  ┊90┊    });
+┊  ┊91┊
 ┊38┊92┊    this.chats$ = this.getChatsWq.valueChanges.pipe(
 ┊39┊93┊      map((result: ApolloQueryResult<GetChats.Query>) => result.data.chats)
 ┊40┊94┊    );
```
```diff
@@ -110,6 +164,10 @@
 ┊110┊164┊        addMessage: {
 ┊111┊165┊          id: ChatsService.getRandomId(),
 ┊112┊166┊          __typename: 'Message',
+┊   ┊167┊          chat: {
+┊   ┊168┊            id: chatId,
+┊   ┊169┊            __typename: 'Chat',
+┊   ┊170┊          },
 ┊113┊171┊          senderId: this.loginService.getUser().id,
 ┊114┊172┊          sender: {
 ┊115┊173┊            id: this.loginService.getUser().id,
```

[}]: #

We can finally configure the WebSocket in the app module. Please notice that the WebSocket has its own authentication instead of using the HttpInterceptor, in fact we use `connectionParams` to send the authorization.
All queries will go through HTTP except the Subscriptions, which will use the WebSocket.

[{]: <helper> (diffStep "11.1" files="src/app/app.module.ts" module="client")

#### Step 11.1: Subscriptions



[}]: #

Finally, let's fix the tests:

[{]: <helper> (diffStep "11.1" files="src/app/chat-viewer/containers/chat/chat.component.spec.ts, src/app/chats-lister/containers/chats/chats.component.spec.ts, src/app/services/chats.service.spec.ts" module="client")

#### Step 11.1: Subscriptions

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.spec.ts
```diff
@@ -149,6 +149,8 @@
 ┊149┊149┊    fixture = TestBed.createComponent(ChatComponent);
 ┊150┊150┊    component = fixture.componentInstance;
 ┊151┊151┊    fixture.detectChanges();
+┊   ┊152┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'chatAdded', 'call to chatAdded api');
+┊   ┊153┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'messageAdded', 'call to messageAdded api');
 ┊152┊154┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'GetChats', 'call to getChats api');
 ┊153┊155┊    const req = httpMock.expectOne(httpReq => httpReq.body.operationName === 'GetChat', 'call to getChat api');
 ┊154┊156┊    req.flush({
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.spec.ts
```diff
@@ -364,7 +364,9 @@
 ┊364┊364┊    fixture = TestBed.createComponent(ChatsComponent);
 ┊365┊365┊    component = fixture.componentInstance;
 ┊366┊366┊    fixture.detectChanges();
-┊367┊   ┊    const req = httpMock.expectOne('http://localhost:3000/graphql', 'call to api');
+┊   ┊367┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'chatAdded', 'call to chatAdded api');
+┊   ┊368┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'messageAdded', 'call to messageAdded api');
+┊   ┊369┊    const req = httpMock.expectOne(httpReq => httpReq.body.operationName === 'GetChats', 'call to getChats api');
 ┊368┊370┊    req.flush({
 ┊369┊371┊      data: {
 ┊370┊372┊        chats
```

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.spec.ts
```diff
@@ -346,7 +346,9 @@
 ┊346┊346┊      }
 ┊347┊347┊    });
 ┊348┊348┊
-┊349┊   ┊    const req = httpMock.expectOne('http://localhost:3000/graphql', 'call to api');
+┊   ┊349┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'chatAdded', 'call to chatAdded api');
+┊   ┊350┊    httpMock.expectOne(httpReq => httpReq.body.operationName === 'messageAdded', 'call to messageAdded api');
+┊   ┊351┊    const req = httpMock.expectOne(httpReq => httpReq.body.operationName === 'GetChats', 'call to getChats api');
 ┊350┊352┊    expect(req.request.method).toBe('POST');
 ┊351┊353┊    req.flush({
 ┊352┊354┊      data: {
```

[}]: #


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step14.md) | [Next Step >](step16.md) |
|:--------------------------------|--------------------------------:|

[}]: #
