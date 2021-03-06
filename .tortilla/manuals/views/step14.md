# Step 14: Authentication

[//]: # (head-end)


## Authentication on server

Authentication is an hot topic in the GraphQL world and there are some projects which aim at authenticating through GraphQL.
Since often you will be required to use a specific auth framework (because of a feature you need or because of an existing authorization infrastructure) I will show you how to use a classic REST API framework within your GraphQL application.
This approach is completely fine and in line with the official GraphQL best practices.
We will use `Passport` for the authentication and `BasicAuth` as the auth mechanism:

    yarn add bcrypt-nodejs passport passport-http
    yarn add -D @types/bcrypt-nodejs @types/passport @types/passport-http

`BasicAuth` basically involves to send username e password in an Authorization Header together with each request and is fully supported by any browser (meaning that we will be able to use `Graphiql` simply by proving username and password in the login window provided by the browser itself).
It's the most simple auth mechanism but it's completely fine for our needs. Later we could decide to use something more complicated like `JWT`, but it's outside of the scope of this tutorial.

[{]: <helper> (diffStep "4.1" files="index.ts" module="server")

#### Step 4.1: Authentication

##### Changed index.ts
```diff
@@ -3,6 +3,47 @@
 ┊ 3┊ 3┊import * as cors from 'cors';
 ┊ 4┊ 4┊import * as express from 'express';
 ┊ 5┊ 5┊import { ApolloServer } from "apollo-server-express";
+┊  ┊ 6┊import * as passport from "passport";
+┊  ┊ 7┊import * as basicStrategy from 'passport-http';
+┊  ┊ 8┊import * as bcrypt from 'bcrypt-nodejs';
+┊  ┊ 9┊import { db, User } from "./db";
+┊  ┊10┊
+┊  ┊11┊let users = db.users;
+┊  ┊12┊
+┊  ┊13┊function generateHash(password: string) {
+┊  ┊14┊  return bcrypt.hashSync(password, bcrypt.genSaltSync(8));
+┊  ┊15┊}
+┊  ┊16┊
+┊  ┊17┊function validPassword(password: string, localPassword: string) {
+┊  ┊18┊  return bcrypt.compareSync(password, localPassword);
+┊  ┊19┊}
+┊  ┊20┊
+┊  ┊21┊passport.use('basic-signin', new basicStrategy.BasicStrategy(
+┊  ┊22┊  function (username: string, password: string, done: any) {
+┊  ┊23┊    const user = users.find(user => user.username == username);
+┊  ┊24┊    if (user && validPassword(password, user.password)) {
+┊  ┊25┊      return done(null, user);
+┊  ┊26┊    }
+┊  ┊27┊    return done(null, false);
+┊  ┊28┊  }
+┊  ┊29┊));
+┊  ┊30┊
+┊  ┊31┊passport.use('basic-signup', new basicStrategy.BasicStrategy({passReqToCallback: true},
+┊  ┊32┊  function (req: any, username: any, password: any, done: any) {
+┊  ┊33┊    const userExists = !!users.find(user => user.username === username);
+┊  ┊34┊    if (!userExists && password && req.body.name) {
+┊  ┊35┊      const user: User = {
+┊  ┊36┊        id: (users.length && users[users.length - 1].id + 1) || 1,
+┊  ┊37┊        username,
+┊  ┊38┊        password: generateHash(password),
+┊  ┊39┊        name: req.body.name,
+┊  ┊40┊      };
+┊  ┊41┊      users.push(user);
+┊  ┊42┊      return done(null, user);
+┊  ┊43┊    }
+┊  ┊44┊    return done(null, false);
+┊  ┊45┊  }
+┊  ┊46┊));
 ┊ 6┊47┊
 ┊ 7┊48┊const PORT = 3000;
 ┊ 8┊49┊
```
```diff
@@ -10,9 +51,27 @@
 ┊10┊51┊
 ┊11┊52┊app.use(cors());
 ┊12┊53┊app.use(bodyParser.json());
+┊  ┊54┊app.use(passport.initialize());
+┊  ┊55┊
+┊  ┊56┊app.post('/signup',
+┊  ┊57┊  passport.authenticate('basic-signup', {session: false}),
+┊  ┊58┊  function (req: any, res) {
+┊  ┊59┊    res.json(req.user);
+┊  ┊60┊  });
+┊  ┊61┊
+┊  ┊62┊app.use(passport.authenticate('basic-signin', {session: false}));
+┊  ┊63┊
+┊  ┊64┊app.post('/signin', function (req: any, res) {
+┊  ┊65┊  res.json(req.user);
+┊  ┊66┊});
 ┊13┊67┊
 ┊14┊68┊const apollo = new ApolloServer({
-┊15┊  ┊  schema
+┊  ┊69┊  schema,
+┊  ┊70┊  context(received: any) {
+┊  ┊71┊    return {
+┊  ┊72┊      user: received.req!['user'],
+┊  ┊73┊    }
+┊  ┊74┊  },
 ┊16┊75┊});
 ┊17┊76┊
 ┊18┊77┊apollo.applyMiddleware({
```

[}]: #

We are going to store hashes instead of plain passwords, that's why we're using `bcrypt-nodejs`.
With `passport.use('basic-signin')` and `passport.use('basic-signup')` we define how the auth framework deals with our database (well, our JSON file for the moment).
`app.post('/signup')` is the endpoint for creating new accounts, so we left it out of the authentication middleware (`app.use(passport.authenticate('basic-signin')`).
What's of particular interest is that we're passing the user object to the GraphQL context.

[{]: <helper> (diffStep "4.1" files="schema/resolvers.ts" module="server")

#### Step 4.1: Authentication

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -8,29 +8,28 @@
 ┊ 8┊ 8┊
 ┊ 9┊ 9┊let users = db.users;
 ┊10┊10┊let chats = db.chats;
-┊11┊  ┊const currentUser = 1;
 ┊12┊11┊
 ┊13┊12┊export const resolvers: IResolvers = {
 ┊14┊13┊  Query: {
 ┊15┊14┊    // Show all users for the moment.
-┊16┊  ┊    users: (): User[] => users.filter(user => user.id !== currentUser),
-┊17┊  ┊    chats: (): Chat[] => chats.filter(chat => chat.listingMemberIds.includes(currentUser)),
+┊  ┊15┊    users: (obj: any, args: any, {user: currentUser}: {user: User}): User[] => users.filter(user => user.id !== currentUser.id),
+┊  ┊16┊    chats: (obj: any, args: any, {user: currentUser}: {user: User}): Chat[] => chats.filter(chat => chat.listingMemberIds.includes(currentUser.id)),
 ┊18┊17┊    chat: (obj: any, {chatId}: ChatQueryArgs): Chat | null => chats.find(chat => chat.id === Number(chatId)) || null,
 ┊19┊18┊  },
 ┊20┊19┊  Mutation: {
-┊21┊  ┊    addChat: (obj: any, {recipientId}: AddChatMutationArgs): Chat => {
+┊  ┊20┊    addChat: (obj: any, {recipientId}: AddChatMutationArgs, {user: currentUser}: {user: User}): Chat => {
 ┊22┊21┊      if (!users.find(user => user.id === Number(recipientId))) {
 ┊23┊22┊        throw new Error(`Recipient ${recipientId} doesn't exist.`);
 ┊24┊23┊      }
 ┊25┊24┊
-┊26┊  ┊      const chat = chats.find(chat => !chat.name && chat.allTimeMemberIds.includes(currentUser) && chat.allTimeMemberIds.includes(Number(recipientId)));
+┊  ┊25┊      const chat = chats.find(chat => !chat.name && chat.allTimeMemberIds.includes(currentUser.id) && chat.allTimeMemberIds.includes(Number(recipientId)));
 ┊27┊26┊      if (chat) {
 ┊28┊27┊        // Chat already exists. Both users are already in the allTimeMemberIds array
 ┊29┊28┊        const chatId = chat.id;
-┊30┊  ┊        if (!chat.listingMemberIds.includes(currentUser)) {
+┊  ┊29┊        if (!chat.listingMemberIds.includes(currentUser.id)) {
 ┊31┊30┊          // The chat isn't listed for the current user. Add him to the memberIds
-┊32┊  ┊          chat.listingMemberIds.push(currentUser);
-┊33┊  ┊          chats.find(chat => chat.id === chatId)!.listingMemberIds.push(currentUser);
+┊  ┊31┊          chat.listingMemberIds.push(currentUser.id);
+┊  ┊32┊          chats.find(chat => chat.id === chatId)!.listingMemberIds.push(currentUser.id);
 ┊34┊33┊          return chat;
 ┊35┊34┊        } else {
 ┊36┊35┊          throw new Error(`Chat already exists.`);
```
```diff
@@ -44,9 +43,9 @@
 ┊44┊43┊          picture: null,
 ┊45┊44┊          adminIds: null,
 ┊46┊45┊          ownerId: null,
-┊47┊  ┊          allTimeMemberIds: [currentUser, Number(recipientId)],
+┊  ┊46┊          allTimeMemberIds: [currentUser.id, Number(recipientId)],
 ┊48┊47┊          // Chat will not be listed to the other user until the first message gets written
-┊49┊  ┊          listingMemberIds: [currentUser],
+┊  ┊48┊          listingMemberIds: [currentUser.id],
 ┊50┊49┊          actualGroupMemberIds: null,
 ┊51┊50┊          messages: [],
 ┊52┊51┊        };
```
```diff
@@ -54,7 +53,7 @@
 ┊54┊53┊        return chat;
 ┊55┊54┊      }
 ┊56┊55┊    },
-┊57┊  ┊    addGroup: (obj: any, {recipientIds, groupName}: AddGroupMutationArgs): Chat => {
+┊  ┊56┊    addGroup: (obj: any, {recipientIds, groupName}: AddGroupMutationArgs, {user: currentUser}: {user: User}): Chat => {
 ┊58┊57┊      recipientIds.forEach(recipientId => {
 ┊59┊58┊        if (!users.find(user => user.id === Number(recipientId))) {
 ┊60┊59┊          throw new Error(`Recipient ${recipientId} doesn't exist.`);
```
```diff
@@ -66,17 +65,17 @@
 ┊66┊65┊        id,
 ┊67┊66┊        name: groupName,
 ┊68┊67┊        picture: null,
-┊69┊  ┊        adminIds: [currentUser],
-┊70┊  ┊        ownerId: currentUser,
-┊71┊  ┊        allTimeMemberIds: [currentUser, ...recipientIds.map(id => Number(id))],
-┊72┊  ┊        listingMemberIds: [currentUser, ...recipientIds.map(id => Number(id))],
-┊73┊  ┊        actualGroupMemberIds: [currentUser, ...recipientIds.map(id => Number(id))],
+┊  ┊68┊        adminIds: [currentUser.id],
+┊  ┊69┊        ownerId: currentUser.id,
+┊  ┊70┊        allTimeMemberIds: [currentUser.id, ...recipientIds.map(id => Number(id))],
+┊  ┊71┊        listingMemberIds: [currentUser.id, ...recipientIds.map(id => Number(id))],
+┊  ┊72┊        actualGroupMemberIds: [currentUser.id, ...recipientIds.map(id => Number(id))],
 ┊74┊73┊        messages: [],
 ┊75┊74┊      };
 ┊76┊75┊      chats.push(chat);
 ┊77┊76┊      return chat;
 ┊78┊77┊    },
-┊79┊  ┊    removeChat: (obj: any, {chatId}: RemoveChatMutationArgs): number => {
+┊  ┊78┊    removeChat: (obj: any, {chatId}: RemoveChatMutationArgs, {user: currentUser}: {user: User}): number => {
 ┊80┊79┊      const chat = chats.find(chat => chat.id === Number(chatId));
 ┊81┊80┊
 ┊82┊81┊      if (!chat) {
```
```diff
@@ -85,14 +84,14 @@
 ┊85┊84┊
 ┊86┊85┊      if (!chat.name) {
 ┊87┊86┊        // Chat
-┊88┊  ┊        if (!chat.listingMemberIds.includes(currentUser)) {
+┊  ┊87┊        if (!chat.listingMemberIds.includes(currentUser.id)) {
 ┊89┊88┊          throw new Error(`The user is not a member of the chat ${chatId}.`);
 ┊90┊89┊        }
 ┊91┊90┊
 ┊92┊91┊        // Instead of chaining map and filter we can loop once using reduce
 ┊93┊92┊        const messages = chat.messages.reduce<Message[]>((filtered, message) => {
 ┊94┊93┊          // Remove the current user from the message holders
-┊95┊  ┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser);
+┊  ┊94┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
 ┊96┊95┊
 ┊97┊96┊          if (message.holderIds.length !== 0) {
 ┊98┊97┊            filtered.push(message);
```
```diff
@@ -102,7 +101,7 @@
 ┊102┊101┊        }, []);
 ┊103┊102┊
 ┊104┊103┊        // Remove the current user from who gets the chat listed. The chat will no longer appear in his list
-┊105┊   ┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser);
+┊   ┊104┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser.id);
 ┊106┊105┊
 ┊107┊106┊        // Check how many members are left
 ┊108┊107┊        if (listingMemberIds.length === 0) {
```
```diff
@@ -120,14 +119,14 @@
 ┊120┊119┊        return Number(chatId);
 ┊121┊120┊      } else {
 ┊122┊121┊        // Group
-┊123┊   ┊        if (chat.ownerId !== currentUser) {
+┊   ┊122┊        if (chat.ownerId !== currentUser.id) {
 ┊124┊123┊          throw new Error(`Group ${chatId} is not owned by the user.`);
 ┊125┊124┊        }
 ┊126┊125┊
 ┊127┊126┊        // Instead of chaining map and filter we can loop once using reduce
 ┊128┊127┊        const messages = chat.messages.reduce<Message[]>((filtered, message) => {
 ┊129┊128┊          // Remove the current user from the message holders
-┊130┊   ┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser);
+┊   ┊129┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
 ┊131┊130┊
 ┊132┊131┊          if (message.holderIds.length !== 0) {
 ┊133┊132┊            filtered.push(message);
```
```diff
@@ -137,7 +136,7 @@
 ┊137┊136┊        }, []);
 ┊138┊137┊
 ┊139┊138┊        // Remove the current user from who gets the group listed. The group will no longer appear in his list
-┊140┊   ┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser);
+┊   ┊139┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser.id);
 ┊141┊140┊
 ┊142┊141┊        // Check how many members (including previous ones who can still access old messages) are left
 ┊143┊142┊        if (listingMemberIds.length === 0) {
```
```diff
@@ -147,9 +146,9 @@
 ┊147┊146┊          // Update the group
 ┊148┊147┊
 ┊149┊148┊          // Remove the current user from the chat members. He is no longer a member of the group
-┊150┊   ┊          const actualGroupMemberIds = chat.actualGroupMemberIds!.filter(memberId => memberId !== currentUser);
+┊   ┊149┊          const actualGroupMemberIds = chat.actualGroupMemberIds!.filter(memberId => memberId !== currentUser.id);
 ┊151┊150┊          // Remove the current user from the chat admins
-┊152┊   ┊          const adminIds = chat.adminIds!.filter(memberId => memberId !== currentUser);
+┊   ┊151┊          const adminIds = chat.adminIds!.filter(memberId => memberId !== currentUser.id);
 ┊153┊152┊          // Set the owner id to be null. A null owner means the group is read-only
 ┊154┊153┊          let ownerId: number | null = null;
 ┊155┊154┊
```
```diff
@@ -169,7 +168,7 @@
 ┊169┊168┊        return Number(chatId);
 ┊170┊169┊      }
 ┊171┊170┊    },
-┊172┊   ┊    addMessage: (obj: any, {chatId, content}: AddMessageMutationArgs): Message => {
+┊   ┊171┊    addMessage: (obj: any, {chatId, content}: AddMessageMutationArgs, {user: currentUser}: {user: User}): Message => {
 ┊173┊172┊      if (content === null || content === '') {
 ┊174┊173┊        throw new Error(`Cannot add empty or null messages.`);
 ┊175┊174┊      }
```
```diff
@@ -184,11 +183,11 @@
 ┊184┊183┊
 ┊185┊184┊      if (!chat.name) {
 ┊186┊185┊        // Chat
-┊187┊   ┊        if (!chat.listingMemberIds.find(listingId => listingId === currentUser)) {
+┊   ┊186┊        if (!chat.listingMemberIds.find(listingId => listingId === currentUser.id)) {
 ┊188┊187┊          throw new Error(`The chat ${chatId} must be listed for the current user before adding a message.`);
 ┊189┊188┊        }
 ┊190┊189┊
-┊191┊   ┊        const recipientId = chat.allTimeMemberIds.filter(userId => userId !== currentUser)[0];
+┊   ┊190┊        const recipientId = chat.allTimeMemberIds.filter(userId => userId !== currentUser.id)[0];
 ┊192┊191┊
 ┊193┊192┊        if (!chat.listingMemberIds.find(listingId => listingId === recipientId)) {
 ┊194┊193┊          // Chat is not listed for the recipient. Add him to the listingMemberIds
```
```diff
@@ -205,7 +204,7 @@
 ┊205┊204┊        }
 ┊206┊205┊      } else {
 ┊207┊206┊        // Group
-┊208┊   ┊        if (!chat.actualGroupMemberIds!.find(memberId => memberId === currentUser)) {
+┊   ┊207┊        if (!chat.actualGroupMemberIds!.find(memberId => memberId === currentUser.id)) {
 ┊209┊208┊          throw new Error(`The user is not a member of the group ${chatId}. Cannot add message.`);
 ┊210┊209┊        }
 ┊211┊210┊
```
```diff
@@ -217,7 +216,7 @@
 ┊217┊216┊      let recipients: Recipient[] = [];
 ┊218┊217┊
 ┊219┊218┊      holderIds.forEach(holderId => {
-┊220┊   ┊        if (holderId !== currentUser) {
+┊   ┊219┊        if (holderId !== currentUser.id) {
 ┊221┊220┊          recipients.push({
 ┊222┊221┊            userId: holderId,
 ┊223┊222┊            messageId: id,
```
```diff
@@ -231,7 +230,7 @@
 ┊231┊230┊      const message: Message = {
 ┊232┊231┊        id,
 ┊233┊232┊        chatId: Number(chatId),
-┊234┊   ┊        senderId: currentUser,
+┊   ┊233┊        senderId: currentUser.id,
 ┊235┊234┊        content,
 ┊236┊235┊        createdAt: moment().unix(),
 ┊237┊236┊        type: MessageType.TEXT,
```
```diff
@@ -248,14 +247,14 @@
 ┊248┊247┊
 ┊249┊248┊      return message;
 ┊250┊249┊    },
-┊251┊   ┊    removeMessages: (obj: any, {chatId, messageIds, all}: RemoveMessagesMutationArgs): number[] => {
+┊   ┊250┊    removeMessages: (obj: any, {chatId, messageIds, all}: RemoveMessagesMutationArgs, {user: currentUser}: {user: User}): number[] => {
 ┊252┊251┊      const chat = chats.find(chat => chat.id === Number(chatId));
 ┊253┊252┊
 ┊254┊253┊      if (!chat) {
 ┊255┊254┊        throw new Error(`Cannot find chat ${chatId}.`);
 ┊256┊255┊      }
 ┊257┊256┊
-┊258┊   ┊      if (!chat.listingMemberIds.find(listingId => listingId === currentUser)) {
+┊   ┊257┊      if (!chat.listingMemberIds.find(listingId => listingId === currentUser.id)) {
 ┊259┊258┊        throw new Error(`The chat/group ${chatId} is not listed for the current user, so there is nothing to delete.`);
 ┊260┊259┊      }
 ┊261┊260┊
```
```diff
@@ -271,7 +270,7 @@
 ┊271┊270┊            if (all || messageIds!.includes(String(message.id))) {
 ┊272┊271┊              deletedIds.push(message.id);
 ┊273┊272┊              // Remove the current user from the message holders
-┊274┊   ┊              message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser);
+┊   ┊273┊              message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
 ┊275┊274┊            }
 ┊276┊275┊
 ┊277┊276┊            if (message.holderIds.length !== 0) {
```
```diff
@@ -288,24 +287,24 @@
 ┊288┊287┊    },
 ┊289┊288┊  },
 ┊290┊289┊  Chat: {
-┊291┊   ┊    name: (chat: Chat): string => chat.name ? chat.name : users
-┊292┊   ┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser))!.name,
-┊293┊   ┊    picture: (chat: Chat) => chat.name ? chat.picture : users
-┊294┊   ┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser))!.picture,
+┊   ┊290┊    name: (chat: Chat, args: any, {user: currentUser}: {user: User}): string => chat.name ? chat.name : users
+┊   ┊291┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.name,
+┊   ┊292┊    picture: (chat: Chat, args: any, {user: currentUser}: {user: User}) => chat.name ? chat.picture : users
+┊   ┊293┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.picture,
 ┊295┊294┊    allTimeMembers: (chat: Chat): User[] => users.filter(user => chat.allTimeMemberIds.includes(user.id)),
 ┊296┊295┊    listingMembers: (chat: Chat): User[] => users.filter(user => chat.listingMemberIds.includes(user.id)),
 ┊297┊296┊    actualGroupMembers: (chat: Chat): User[] => users.filter(user => chat.actualGroupMemberIds && chat.actualGroupMemberIds.includes(user.id)),
 ┊298┊297┊    admins: (chat: Chat): User[] => users.filter(user => chat.adminIds && chat.adminIds.includes(user.id)),
 ┊299┊298┊    owner: (chat: Chat): User | null => users.find(user => chat.ownerId === user.id) || null,
-┊300┊   ┊    messages: (chat: Chat, {amount = 0}: {amount: number}): Message[] => {
+┊   ┊299┊    messages: (chat: Chat, {amount = 0}: {amount: number}, {user: currentUser}: {user: User}): Message[] => {
 ┊301┊300┊      const messages = chat.messages
-┊302┊   ┊      .filter(message => message.holderIds.includes(currentUser))
+┊   ┊301┊      .filter(message => message.holderIds.includes(currentUser.id))
 ┊303┊302┊      .sort((a, b) => b.createdAt - a.createdAt) || <Message[]>[];
 ┊304┊303┊      return (amount ? messages.slice(0, amount) : messages).reverse();
 ┊305┊304┊    },
-┊306┊   ┊    unreadMessages: (chat: Chat): number => chat.messages
-┊307┊   ┊      .filter(message => message.holderIds.includes(currentUser) &&
-┊308┊   ┊        message.recipients.find(recipient => recipient.userId === currentUser && !recipient.readAt))
+┊   ┊305┊    unreadMessages: (chat: Chat, args: any, {user: currentUser}: {user: User}): number => chat.messages
+┊   ┊306┊      .filter(message => message.holderIds.includes(currentUser.id) &&
+┊   ┊307┊        message.recipients.find(recipient => recipient.userId === currentUser.id && !recipient.readAt))
 ┊309┊308┊      .length,
 ┊310┊309┊    isGroup: (chat: Chat): boolean => !!chat.name,
 ┊311┊310┊  },
```
```diff
@@ -313,7 +312,7 @@
 ┊313┊312┊    chat: (message: Message): Chat | null => chats.find(chat => message.chatId === chat.id) || null,
 ┊314┊313┊    sender: (message: Message): User | null => users.find(user => user.id === message.senderId) || null,
 ┊315┊314┊    holders: (message: Message): User[] => users.filter(user => message.holderIds.includes(user.id)),
-┊316┊   ┊    ownership: (message: Message): boolean => message.senderId === currentUser,
+┊   ┊315┊    ownership: (message: Message, args: any, {user: currentUser}: {user: User}): boolean => message.senderId === currentUser.id,
 ┊317┊316┊  },
 ┊318┊317┊  Recipient: {
 ┊319┊318┊    user: (recipient: Recipient): User | null => users.find(user => recipient.userId === user.id) || null,
```

[}]: #

In the resolvers we're basically making use of the user object taken from the context.

## Authentication on client

Let's start installing `@angular/flex-layout`, because we will use it later:

    yarn add @angular/flex-layout


First of all we need to create an `HTTP Interceptor`, which will intercept every HTTP request and will add authentication headers.
For the moment we still don't have those headers, but we are going to store them in the `LocalStorage` later.
We are also creating an `AuthGuard`, which we will use to stop the user from reaching unauthorized Routes.

The `AuthGuard` will simply look for the presence of the `Authentication` Header, but will not guarantee that the header is authentic.
This is no problem, because client side AuthGuards are not safe by design and the real authentication will be done server side anyway.
AuthGuards are here just to redirect the user to the Login page.

The service we are going to create will contain some auth methods we are going to use across the app.

[{]: <helper> (diffStep "10.1" files="src/app/login/services" module="client")

#### Step 10.1: Authentication

##### Added src&#x2F;app&#x2F;login&#x2F;services&#x2F;auth.guard.ts
```diff
@@ -0,0 +1,18 @@
+┊  ┊ 1┊import {Injectable} from '@angular/core';
+┊  ┊ 2┊import {CanActivate, Router} from '@angular/router';
+┊  ┊ 3┊import {LoginService} from './login.service';
+┊  ┊ 4┊
+┊  ┊ 5┊@Injectable()
+┊  ┊ 6┊export class AuthGuard implements CanActivate {
+┊  ┊ 7┊  constructor(private router: Router,
+┊  ┊ 8┊              private loginService: LoginService) {}
+┊  ┊ 9┊
+┊  ┊10┊  canActivate() {
+┊  ┊11┊    if (this.loginService.getAuthHeader()) {
+┊  ┊12┊      return true;
+┊  ┊13┊    } else {
+┊  ┊14┊      this.router.navigate(['/login']);
+┊  ┊15┊      return false;
+┊  ┊16┊    }
+┊  ┊17┊  }
+┊  ┊18┊}
```

##### Added src&#x2F;app&#x2F;login&#x2F;services&#x2F;auth.interceptor.ts
```diff
@@ -0,0 +1,20 @@
+┊  ┊ 1┊import {Injectable} from '@angular/core';
+┊  ┊ 2┊import {HttpEvent, HttpHandler, HttpInterceptor, HttpRequest} from '@angular/common/http';
+┊  ┊ 3┊import {Observable} from 'rxjs';
+┊  ┊ 4┊import {LoginService} from './login.service';
+┊  ┊ 5┊
+┊  ┊ 6┊@Injectable()
+┊  ┊ 7┊export class AuthInterceptor implements HttpInterceptor {
+┊  ┊ 8┊  constructor(private loginService: LoginService) {}
+┊  ┊ 9┊  intercept(request: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
+┊  ┊10┊    const auth = this.loginService.getAuthHeader();
+┊  ┊11┊    if (auth) {
+┊  ┊12┊      request = request.clone({
+┊  ┊13┊        setHeaders: {
+┊  ┊14┊          Authorization: auth,
+┊  ┊15┊        }
+┊  ┊16┊      });
+┊  ┊17┊    }
+┊  ┊18┊    return next.handle(request);
+┊  ┊19┊  }
+┊  ┊20┊}
```

##### Added src&#x2F;app&#x2F;login&#x2F;services&#x2F;login.service.ts
```diff
@@ -0,0 +1,24 @@
+┊  ┊ 1┊import { Injectable } from '@angular/core';
+┊  ┊ 2┊import {User} from '../../../graphql';
+┊  ┊ 3┊
+┊  ┊ 4┊@Injectable()
+┊  ┊ 5┊export class LoginService {
+┊  ┊ 6┊
+┊  ┊ 7┊  constructor() { }
+┊  ┊ 8┊
+┊  ┊ 9┊  storeAuthHeader(auth: string) {
+┊  ┊10┊    localStorage.setItem('Authorization', auth);
+┊  ┊11┊  }
+┊  ┊12┊
+┊  ┊13┊  getAuthHeader(): string {
+┊  ┊14┊    return localStorage.getItem('Authorization');
+┊  ┊15┊  }
+┊  ┊16┊
+┊  ┊17┊  storeUser(user: User) {
+┊  ┊18┊    localStorage.setItem('user', JSON.stringify(user));
+┊  ┊19┊  }
+┊  ┊20┊
+┊  ┊21┊  getUser(): User {
+┊  ┊22┊    return JSON.parse(localStorage.getItem('user'));
+┊  ┊23┊  }
+┊  ┊24┊}
```

[}]: #

Now it's time to create a `SignIn`/`SignUp` component. Since we use Passport in the server we are going to make REST calls for the authentication, instead of using GraphQL.
Since we use `Basic Auth` we will simply combine the username and the password together to create the authentication header.
We will also store the response from the server, which will contain the user information like the ID, etc. which we are going to need later.

[{]: <helper> (diffStep "10.1" files="src/app/login/containers, src/app/login/login.module.ts" module="client")

#### Step 10.1: Authentication

##### Added src&#x2F;app&#x2F;login&#x2F;containers&#x2F;login.component.scss
```diff
@@ -0,0 +1,18 @@
+┊  ┊ 1┊:host {
+┊  ┊ 2┊  display: block;
+┊  ┊ 3┊}
+┊  ┊ 4┊
+┊  ┊ 5┊form:first-of-type {
+┊  ┊ 6┊  margin-top: 24px;
+┊  ┊ 7┊  margin-bottom: 48px;
+┊  ┊ 8┊}
+┊  ┊ 9┊
+┊  ┊10┊label {
+┊  ┊11┊  display: block;
+┊  ┊12┊  margin-top: 4px;
+┊  ┊13┊  margin-bottom: 4px;
+┊  ┊14┊}
+┊  ┊15┊
+┊  ┊16┊.error {
+┊  ┊17┊  color: red;
+┊  ┊18┊}
```

##### Added src&#x2F;app&#x2F;login&#x2F;containers&#x2F;login.component.ts
```diff
@@ -0,0 +1,133 @@
+┊   ┊  1┊import {Component} from '@angular/core';
+┊   ┊  2┊import {HttpClient} from '@angular/common/http';
+┊   ┊  3┊import {FormBuilder, Validators} from '@angular/forms';
+┊   ┊  4┊// import {matchOtherValidator} from '@moebius/ng-validators';
+┊   ┊  5┊import {Router} from '@angular/router';
+┊   ┊  6┊import {User} from '../../../graphql';
+┊   ┊  7┊import {LoginService} from '../services/login.service';
+┊   ┊  8┊
+┊   ┊  9┊@Component({
+┊   ┊ 10┊  selector: 'app-login',
+┊   ┊ 11┊  template: `
+┊   ┊ 12┊    <form (ngSubmit)="signIn()" [formGroup]="signInForm" novalidate>
+┊   ┊ 13┊      <fieldset fxLayout="column" fxLayoutGap="17px">
+┊   ┊ 14┊        <legend>Sign in</legend>
+┊   ┊ 15┊        <div>
+┊   ┊ 16┊          <label>Username</label>
+┊   ┊ 17┊          <input formControlName="username" autocomplete="username" type="text">
+┊   ┊ 18┊        </div>
+┊   ┊ 19┊        <div class="error" *ngIf="signInForm.get('username').hasError('required') && signInForm.get('username').touched">
+┊   ┊ 20┊          Username is required
+┊   ┊ 21┊        </div>
+┊   ┊ 22┊
+┊   ┊ 23┊        <div>
+┊   ┊ 24┊          <label>Password</label>
+┊   ┊ 25┊          <input formControlName="password" autocomplete="current-password" type="password">
+┊   ┊ 26┊        </div>
+┊   ┊ 27┊        <div class="error" *ngIf="signInForm.get('password').hasError('required') && signInForm.get('password').touched">
+┊   ┊ 28┊          Password is required
+┊   ┊ 29┊        </div>
+┊   ┊ 30┊
+┊   ┊ 31┊        <button type="submit" [disabled]="signInForm.invalid">Sign in</button>
+┊   ┊ 32┊      </fieldset>
+┊   ┊ 33┊    </form>
+┊   ┊ 34┊
+┊   ┊ 35┊    <form (ngSubmit)="signUp()" [formGroup]="signUpForm" novalidate>
+┊   ┊ 36┊      <fieldset fxLayout="column" fxLayoutGap="17px">
+┊   ┊ 37┊        <legend>Sign up</legend>
+┊   ┊ 38┊        <div>
+┊   ┊ 39┊          <label>Name</label>
+┊   ┊ 40┊          <input formControlName="name" type="text">
+┊   ┊ 41┊        </div>
+┊   ┊ 42┊
+┊   ┊ 43┊        <div>
+┊   ┊ 44┊          <label>Username</label>
+┊   ┊ 45┊          <input formControlName="username" autocomplete="username" type="text">
+┊   ┊ 46┊        </div>
+┊   ┊ 47┊        <div class="error" *ngIf="signUpForm.get('username').hasError('required') && signUpForm.get('username').touched">
+┊   ┊ 48┊          Username is required
+┊   ┊ 49┊        </div>
+┊   ┊ 50┊
+┊   ┊ 51┊        <div>
+┊   ┊ 52┊          <label>Password</label>
+┊   ┊ 53┊          <input formControlName="newPassword" autocomplete="new-password" type="password">
+┊   ┊ 54┊        </div>
+┊   ┊ 55┊        <div class="error" *ngIf="signUpForm.get('newPassword').hasError('required') && signUpForm.get('newPassword').touched">
+┊   ┊ 56┊          Password is required
+┊   ┊ 57┊        </div>
+┊   ┊ 58┊
+┊   ┊ 59┊        <div>
+┊   ┊ 60┊          <label>Password</label>
+┊   ┊ 61┊          <input formControlName="confirmPassword" type="password">
+┊   ┊ 62┊        </div>
+┊   ┊ 63┊        <div class="error" *ngIf="signUpForm.get('confirmPassword').hasError('required') && signUpForm.get('confirmPassword').touched">
+┊   ┊ 64┊          Passwords must match
+┊   ┊ 65┊        </div>
+┊   ┊ 66┊
+┊   ┊ 67┊        <button type="submit" [disabled]="signUpForm.invalid">Sign up</button>
+┊   ┊ 68┊      </fieldset>
+┊   ┊ 69┊    </form>
+┊   ┊ 70┊  `,
+┊   ┊ 71┊  styleUrls: ['./login.component.scss'],
+┊   ┊ 72┊})
+┊   ┊ 73┊export class LoginComponent {
+┊   ┊ 74┊  signInForm = this.fb.group({
+┊   ┊ 75┊    username: [null, [
+┊   ┊ 76┊      Validators.required,
+┊   ┊ 77┊    ]],
+┊   ┊ 78┊    password: [null, [
+┊   ┊ 79┊      Validators.required,
+┊   ┊ 80┊    ]],
+┊   ┊ 81┊  });
+┊   ┊ 82┊
+┊   ┊ 83┊  signUpForm = this.fb.group({
+┊   ┊ 84┊    name: [null, [
+┊   ┊ 85┊      Validators.required,
+┊   ┊ 86┊    ]],
+┊   ┊ 87┊    username: [null, [
+┊   ┊ 88┊      Validators.required,
+┊   ┊ 89┊    ]],
+┊   ┊ 90┊    newPassword: [null, [
+┊   ┊ 91┊      Validators.required,
+┊   ┊ 92┊    ]],
+┊   ┊ 93┊    confirmPassword: [null, [
+┊   ┊ 94┊      Validators.required,
+┊   ┊ 95┊      // matchOtherValidator('newPassword'),
+┊   ┊ 96┊    ]],
+┊   ┊ 97┊  });
+┊   ┊ 98┊
+┊   ┊ 99┊  constructor(private http: HttpClient,
+┊   ┊100┊              private fb: FormBuilder,
+┊   ┊101┊              private router: Router,
+┊   ┊102┊              private loginService: LoginService) {}
+┊   ┊103┊
+┊   ┊104┊  signIn() {
+┊   ┊105┊    const {username, password} = this.signInForm.value;
+┊   ┊106┊    const auth = `Basic ${btoa(`${username}:${password}`)}`;
+┊   ┊107┊    this.http.post('http://localhost:3000/signin', null, {
+┊   ┊108┊      headers: {
+┊   ┊109┊        Authorization: auth,
+┊   ┊110┊      }
+┊   ┊111┊    }).subscribe((user: User) => {
+┊   ┊112┊      this.loginService.storeAuthHeader(auth);
+┊   ┊113┊      this.loginService.storeUser(user);
+┊   ┊114┊      this.router.navigate(['/chats']);
+┊   ┊115┊    }, err => console.error(err));
+┊   ┊116┊  }
+┊   ┊117┊
+┊   ┊118┊  signUp() {
+┊   ┊119┊    const {username, newPassword: password, name} = this.signInForm.value;
+┊   ┊120┊    const auth = `Basic ${btoa(`${username}:${password}`)}`;
+┊   ┊121┊    this.http.post('http://localhost:3000/signup', {
+┊   ┊122┊      name,
+┊   ┊123┊    }, {
+┊   ┊124┊      headers: {
+┊   ┊125┊        Authorization: auth,
+┊   ┊126┊      }
+┊   ┊127┊    }).subscribe((user: User) => {
+┊   ┊128┊      this.loginService.storeAuthHeader(auth);
+┊   ┊129┊      this.loginService.storeUser(user);
+┊   ┊130┊      this.router.navigate(['/chats']);
+┊   ┊131┊    }, err => console.error(err));
+┊   ┊132┊  }
+┊   ┊133┊}
```

##### Added src&#x2F;app&#x2F;login&#x2F;login.module.ts
```diff
@@ -0,0 +1,52 @@
+┊  ┊ 1┊import {RouterModule, Routes} from '@angular/router';
+┊  ┊ 2┊import {NgModule} from '@angular/core';
+┊  ┊ 3┊import {TruncateModule} from 'ng2-truncate';
+┊  ┊ 4┊import {MatButtonModule, MatIconModule, MatListModule, MatMenuModule} from '@angular/material';
+┊  ┊ 5┊import {SharedModule} from '../shared/shared.module';
+┊  ┊ 6┊import {BrowserModule} from '@angular/platform-browser';
+┊  ┊ 7┊import {FormsModule, ReactiveFormsModule} from '@angular/forms';
+┊  ┊ 8┊import {BrowserAnimationsModule} from '@angular/platform-browser/animations';
+┊  ┊ 9┊import {LoginComponent} from './containers/login.component';
+┊  ┊10┊import {FlexLayoutModule} from '@angular/flex-layout';
+┊  ┊11┊import {AuthInterceptor} from './services/auth.interceptor';
+┊  ┊12┊import {AuthGuard} from './services/auth.guard';
+┊  ┊13┊import {LoginService} from './services/login.service';
+┊  ┊14┊
+┊  ┊15┊
+┊  ┊16┊const routes: Routes = [
+┊  ┊17┊  {path: 'login', component: LoginComponent},
+┊  ┊18┊];
+┊  ┊19┊
+┊  ┊20┊@NgModule({
+┊  ┊21┊  declarations: [
+┊  ┊22┊    LoginComponent,
+┊  ┊23┊  ],
+┊  ┊24┊  imports: [
+┊  ┊25┊    BrowserModule,
+┊  ┊26┊    // Material
+┊  ┊27┊    MatMenuModule,
+┊  ┊28┊    MatIconModule,
+┊  ┊29┊    MatButtonModule,
+┊  ┊30┊    MatListModule,
+┊  ┊31┊    // Animations
+┊  ┊32┊    BrowserAnimationsModule,
+┊  ┊33┊    // Flex layout
+┊  ┊34┊    FlexLayoutModule,
+┊  ┊35┊    // Routing
+┊  ┊36┊    RouterModule.forChild(routes),
+┊  ┊37┊    // Forms
+┊  ┊38┊    FormsModule,
+┊  ┊39┊    ReactiveFormsModule,
+┊  ┊40┊    // Truncate Pipe
+┊  ┊41┊    TruncateModule,
+┊  ┊42┊    // Feature modules
+┊  ┊43┊    SharedModule,
+┊  ┊44┊  ],
+┊  ┊45┊  providers: [
+┊  ┊46┊    LoginService,
+┊  ┊47┊    AuthInterceptor,
+┊  ┊48┊    AuthGuard,
+┊  ┊49┊  ],
+┊  ┊50┊})
+┊  ┊51┊export class LoginModule {
+┊  ┊52┊}
```

[}]: #

Now it's time use the Interceptor we just created:

[{]: <helper> (diffStep "10.1" files="src/app/app.module.ts" module="client")

#### Step 10.1: Authentication

##### Changed src&#x2F;app&#x2F;app.module.ts
```diff
@@ -2,12 +2,14 @@
 ┊ 2┊ 2┊import { NgModule } from '@angular/core';
 ┊ 3┊ 3┊
 ┊ 4┊ 4┊import { AppComponent } from './app.component';
-┊ 5┊  ┊import { HttpClientModule } from '@angular/common/http';
+┊  ┊ 5┊import { HttpClientModule, HTTP_INTERCEPTORS } from '@angular/common/http';
 ┊ 6┊ 6┊import { GraphQLModule } from './graphql.module';
 ┊ 7┊ 7┊import {ChatsListerModule} from './chats-lister/chats-lister.module';
 ┊ 8┊ 8┊import {RouterModule, Routes} from '@angular/router';
 ┊ 9┊ 9┊import {ChatViewerModule} from './chat-viewer/chat-viewer.module';
 ┊10┊10┊import {ChatsCreationModule} from './chats-creation/chats-creation.module';
+┊  ┊11┊import {LoginModule} from './login/login.module';
+┊  ┊12┊import {AuthInterceptor} from './login/services/auth.interceptor';
 ┊11┊13┊const routes: Routes = [];
 ┊12┊14┊
 ┊13┊15┊@NgModule({
```
```diff
@@ -24,8 +26,15 @@
 ┊24┊26┊    ChatsListerModule,
 ┊25┊27┊    ChatViewerModule,
 ┊26┊28┊    ChatsCreationModule,
+┊  ┊29┊    LoginModule,
+┊  ┊30┊  ],
+┊  ┊31┊  providers: [
+┊  ┊32┊    {
+┊  ┊33┊      provide: HTTP_INTERCEPTORS,
+┊  ┊34┊      useClass: AuthInterceptor,
+┊  ┊35┊      multi: true,
+┊  ┊36┊    },
 ┊27┊37┊  ],
-┊28┊  ┊  providers: [],
 ┊29┊38┊  bootstrap: [AppComponent]
 ┊30┊39┊})
 ┊31┊40┊export class AppModule {}
```

[}]: #

As well as the `AuthGuard`:

[{]: <helper> (diffStep "10.1" files="src/app/chat-viewer/chat-viewer.module.ts, src/app/chats-creation/chats-creation.module.ts, src/app/chats-lister/chats-lister.module.ts" module="client")

#### Step 10.1: Authentication

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;chat-viewer.module.ts
```diff
@@ -12,11 +12,12 @@
 ┊12┊12┊import {NewMessageComponent} from './components/new-message/new-message.component';
 ┊13┊13┊import {SharedModule} from '../shared/shared.module';
 ┊14┊14┊import {NgxSelectableListModule} from 'ngx-selectable-list';
+┊  ┊15┊import {AuthGuard} from '../login/services/auth.guard';
 ┊15┊16┊
 ┊16┊17┊const routes: Routes = [
 ┊17┊18┊  {
 ┊18┊19┊    path: 'chat', children: [
-┊19┊  ┊      {path: ':id', component: ChatComponent},
+┊  ┊20┊      {path: ':id', canActivate: [AuthGuard], component: ChatComponent},
 ┊20┊21┊    ],
 ┊21┊22┊  },
 ┊22┊23┊];
```

##### Changed src&#x2F;app&#x2F;chats-creation&#x2F;chats-creation.module.ts
```diff
@@ -16,10 +16,11 @@
 ┊16┊16┊import {NewGroupDetailsComponent} from './components/new-group-details/new-group-details.component';
 ┊17┊17┊import {SharedModule} from '../shared/shared.module';
 ┊18┊18┊import {NgxSelectableListModule} from 'ngx-selectable-list';
+┊  ┊19┊import {AuthGuard} from '../login/services/auth.guard';
 ┊19┊20┊
 ┊20┊21┊const routes: Routes = [
-┊21┊  ┊  {path: 'new-chat', component: NewChatComponent},
-┊22┊  ┊  {path: 'new-group', component: NewGroupComponent},
+┊  ┊22┊  {path: 'new-chat', canActivate: [AuthGuard], component: NewChatComponent},
+┊  ┊23┊  {path: 'new-group', canActivate: [AuthGuard], component: NewGroupComponent},
 ┊23┊24┊];
 ┊24┊25┊
 ┊25┊26┊@NgModule({
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;chats-lister.module.ts
```diff
@@ -12,10 +12,11 @@
 ┊12┊12┊import {TruncateModule} from 'ng2-truncate';
 ┊13┊13┊import {SharedModule} from '../shared/shared.module';
 ┊14┊14┊import {NgxSelectableListModule} from 'ngx-selectable-list';
+┊  ┊15┊import {AuthGuard} from '../login/services/auth.guard';
 ┊15┊16┊
 ┊16┊17┊const routes: Routes = [
 ┊17┊18┊  {path: '', redirectTo: 'chats', pathMatch: 'full'},
-┊18┊  ┊  {path: 'chats', component: ChatsComponent},
+┊  ┊19┊  {path: 'chats', canActivate: [AuthGuard], component: ChatsComponent},
 ┊19┊20┊];
 ┊20┊21┊
 ┊21┊22┊@NgModule({
```

[}]: #

Last but not the least we need to fix our main service in order to not use the hardcoded user anymore. Instead we will use our Login service to read the user info from the `LocalStorage`.

[{]: <helper> (diffStep "10.1" files="src/app/services/chats.service.ts" module="client")

#### Step 10.1: Authentication

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -24,6 +24,7 @@
 ┊24┊24┊} from '../../graphql';
 ┊25┊25┊import { DataProxy } from 'apollo-cache';
 ┊26┊26┊import { FetchResult } from 'apollo-link';
+┊  ┊27┊import {LoginService} from '../login/services/login.service';
 ┊27┊28┊
 ┊28┊29┊const currentUserId = '1';
 ┊29┊30┊const currentUserName = 'Ethan Gonzalez';
```
```diff
@@ -46,7 +47,8 @@
 ┊46┊47┊    private removeAllMessagesGQL: RemoveAllMessagesGQL,
 ┊47┊48┊    private getUsersGQL: GetUsersGQL,
 ┊48┊49┊    private addChatGQL: AddChatGQL,
-┊49┊  ┊    private addGroupGQL: AddGroupGQL
+┊  ┊50┊    private addGroupGQL: AddGroupGQL,
+┊  ┊51┊    private loginService: LoginService
 ┊50┊52┊  ) {
 ┊51┊53┊    this.getChatsWq = this.getChatsGQL.watch({
 ┊52┊54┊      amount: this.messagesAmount,
```
```diff
@@ -128,11 +130,11 @@
 ┊128┊130┊        addMessage: {
 ┊129┊131┊          id: ChatsService.getRandomId(),
 ┊130┊132┊          __typename: 'Message',
-┊131┊   ┊          senderId: currentUserId,
+┊   ┊133┊          senderId: this.loginService.getUser().id,
 ┊132┊134┊          sender: {
-┊133┊   ┊            id: currentUserId,
+┊   ┊135┊            id: this.loginService.getUser().id,
 ┊134┊136┊            __typename: 'User',
-┊135┊   ┊            name: currentUserName,
+┊   ┊137┊            name: this.loginService.getUser().name,
 ┊136┊138┊          },
 ┊137┊139┊          content,
 ┊138┊140┊          createdAt: moment().unix(),
```
```diff
@@ -315,7 +317,7 @@
 ┊315┊317┊  // Checks if the chat is listed for the current user and returns the id
 ┊316┊318┊  getChatId(recipientId: string) {
 ┊317┊319┊    const _chat = this.chats.find(chat => {
-┊318┊   ┊      return !chat.isGroup && !!chat.allTimeMembers.find(user => user.id === currentUserId) &&
+┊   ┊320┊      return !chat.isGroup && !!chat.allTimeMembers.find(user => user.id === this.loginService.getUser().id) &&
 ┊319┊321┊        !!chat.allTimeMembers.find(user => user.id === recipientId);
 ┊320┊322┊    });
 ┊321┊323┊    return _chat ? _chat.id : false;
```
```diff
@@ -335,7 +337,7 @@
 ┊335┊337┊            picture: users.find(user => user.id === recipientId).picture,
 ┊336┊338┊            allTimeMembers: [
 ┊337┊339┊              {
-┊338┊   ┊                id: currentUserId,
+┊   ┊340┊                id: this.loginService.getUser().id,
 ┊339┊341┊                __typename: 'User',
 ┊340┊342┊              },
 ┊341┊343┊              {
```
```diff
@@ -387,10 +389,10 @@
 ┊387┊389┊            __typename: 'Chat',
 ┊388┊390┊            name: groupName,
 ┊389┊391┊            picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
-┊390┊   ┊            userIds: [currentUserId, recipientIds],
+┊   ┊392┊            userIds: [this.loginService.getUser().id, recipientIds],
 ┊391┊393┊            allTimeMembers: [
 ┊392┊394┊              {
-┊393┊   ┊                id: currentUserId,
+┊   ┊395┊                id: this.loginService.getUser().id,
 ┊394┊396┊                __typename: 'User',
 ┊395┊397┊              },
 ┊396┊398┊              ...recipientIds.map(id => ({id, __typename: 'User'})),
```

[}]: #

We also need to fix our tests:

[{]: <helper> (diffStep "10.1" files="src/app/chat-viewer/containers/chat/chat.component.spec.ts, src/app/chats-lister/containers/chats/chats.component.spec.ts, src/app/services/chats.service.spec.ts" module="client")

#### Step 10.1: Authentication

##### Changed src&#x2F;app&#x2F;chat-viewer&#x2F;containers&#x2F;chat&#x2F;chat.component.spec.ts
```diff
@@ -29,6 +29,7 @@
 ┊29┊29┊import { NewMessageComponent } from '../../components/new-message/new-message.component';
 ┊30┊30┊import { MessagesListComponent } from '../../components/messages-list/messages-list.component';
 ┊31┊31┊import { MessageItemComponent } from '../../components/message-item/message-item.component';
+┊  ┊32┊import { LoginService } from '../../../login/services/login.service';
 ┊32┊33┊
 ┊33┊34┊describe('ChatComponent', () => {
 ┊34┊35┊  let component: ChatComponent;
```
```diff
@@ -134,6 +135,7 @@
 ┊134┊135┊            queryParams: of({}),
 ┊135┊136┊          },
 ┊136┊137┊        },
+┊   ┊138┊        LoginService,
 ┊137┊139┊      ],
 ┊138┊140┊      schemas: [NO_ERRORS_SCHEMA],
 ┊139┊141┊    }).compileComponents();
```

##### Changed src&#x2F;app&#x2F;chats-lister&#x2F;containers&#x2F;chats&#x2F;chats.component.spec.ts
```diff
@@ -24,6 +24,7 @@
 ┊24┊24┊import { ChatsListComponent } from '../../components/chats-list/chats-list.component';
 ┊25┊25┊import { ChatItemComponent } from '../../components/chat-item/chat-item.component';
 ┊26┊26┊import { ChatsService } from '../../../services/chats.service';
+┊  ┊27┊import { LoginService } from '../../../login/services/login.service';
 ┊27┊28┊
 ┊28┊29┊describe('ChatsComponent', () => {
 ┊29┊30┊  let component: ChatsComponent;
```
```diff
@@ -345,6 +346,7 @@
 ┊345┊346┊            return new InMemoryCache({ dataIdFromObject });
 ┊346┊347┊          },
 ┊347┊348┊        },
+┊   ┊349┊        LoginService,
 ┊348┊350┊      ],
 ┊349┊351┊      schemas: [NO_ERRORS_SCHEMA],
 ┊350┊352┊    }).compileComponents();
```

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.spec.ts
```diff
@@ -11,6 +11,7 @@
 ┊11┊11┊import { GetChats } from '../../graphql';
 ┊12┊12┊import { dataIdFromObject } from '../graphql.module';
 ┊13┊13┊import { ChatsService } from './chats.service';
+┊  ┊14┊import {LoginService} from '../login/services/login.service';
 ┊14┊15┊
 ┊15┊16┊describe('ChatsService', () => {
 ┊16┊17┊  let controller: ApolloTestingController;
```
```diff
@@ -312,6 +313,7 @@
 ┊312┊313┊      imports: [ApolloTestingModule],
 ┊313┊314┊      providers: [
 ┊314┊315┊        ChatsService,
+┊   ┊316┊        LoginService,
 ┊315┊317┊        {
 ┊316┊318┊          provide: APOLLO_TESTING_CACHE,
 ┊317┊319┊          useFactory() {
```

[}]: #


### GraphQL Server behind a firewall

There's still one thing remaining. Our GraphQL Code Generator setup is broken now. That's because every made request to the server has to have proper rights and currently the codegen's request introspection query is not authenticated. In short, we need to attach an `Authorization` header.

We're going to implement the following scenario:

1. User runs `yarn generator`
1. He is asked for a _username_ and a _password_
1. Our script generates an _Authorization_ header
1. GraphQL Code Generator get the GraphQL Schema of our server and outputs the types

To achieve the goal, we need to change the way we interact with the codegen.
Right now we use GraphQL Code Generator's CLI but the tool allows to use it programatically. So one issue solved.

What about getting user's name and password part? We will use in-terminal prompt so user can type those. There's a package for that:

    yarn add -D prompt

Next, create a `src/codegen.ts` file and write our script there:

[{]: <helper> (diffStep "10.1" files="src/codegen.ts" module="client")

#### Step 10.1: Authentication

##### Added src&#x2F;codegen.ts
```diff
@@ -0,0 +1,46 @@
+┊  ┊ 1┊import * as prompt from 'prompt';
+┊  ┊ 2┊import { generate } from 'graphql-code-generator';
+┊  ┊ 3┊
+┊  ┊ 4┊interface Credentials {
+┊  ┊ 5┊  username: string;
+┊  ┊ 6┊  password: string;
+┊  ┊ 7┊}
+┊  ┊ 8┊
+┊  ┊ 9┊async function getCredentials(): Promise<Credentials> {
+┊  ┊10┊  return new Promise<Credentials>(resolve => {
+┊  ┊11┊    prompt.start();
+┊  ┊12┊    prompt.get(['username', 'password'], (_err, result) => {
+┊  ┊13┊      resolve({
+┊  ┊14┊        username: result.username,
+┊  ┊15┊        password: result.password,
+┊  ┊16┊      });
+┊  ┊17┊    });
+┊  ┊18┊  });
+┊  ┊19┊}
+┊  ┊20┊
+┊  ┊21┊function generateHeaders({ username, password }: Credentials): string[] {
+┊  ┊22┊  const Authorization = `Basic ${Buffer.from(
+┊  ┊23┊    `${username}:${password}`,
+┊  ┊24┊  ).toString('base64')}`;
+┊  ┊25┊
+┊  ┊26┊  return [`Authorization: ${Authorization}`];
+┊  ┊27┊}
+┊  ┊28┊
+┊  ┊29┊async function main() {
+┊  ┊30┊  const credentials = await getCredentials();
+┊  ┊31┊  const headers = generateHeaders(credentials);
+┊  ┊32┊
+┊  ┊33┊  await generate(
+┊  ┊34┊    {
+┊  ┊35┊      schema: 'http://localhost:3000/graphql',
+┊  ┊36┊      template: 'graphql-codegen-apollo-angular-template',
+┊  ┊37┊      out: './src/graphql.ts',
+┊  ┊38┊      args: ['./src/graphql/**/*.ts'],
+┊  ┊39┊      header: headers,
+┊  ┊40┊      overwrite: true,
+┊  ┊41┊    },
+┊  ┊42┊    true,
+┊  ┊43┊  );
+┊  ┊44┊}
+┊  ┊45┊
+┊  ┊46┊main();
```

[}]: #

Let's take a closer look what we did there:

- We asks user for a _username_ and a _password_ with `prompt`
- Based on that we generate a header
- We put the header in `generate` function
- there's a `true` value as the second argument, it tells codegen to write the output to a file

With all of that, whenever we run `yarn generator`, we access the GraphQL server as an authenticated user.


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step13.md) | [Next Step >](step15.md) |
|:--------------------------------|--------------------------------:|

[}]: #
