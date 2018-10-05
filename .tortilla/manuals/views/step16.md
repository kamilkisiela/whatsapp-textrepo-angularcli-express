# Step 16: TypeORM with PostgreSQL

[//]: # (head-end)


## Server

First of all you will have to install PostgreSQL on your operating system. Since there so many options (different Linux distributions, MacOS X, Windows...) I will assume that you already know how to install a software in your OS and take that part for granted.

Then you will have to install a couple of packages:

    npm install pg reflect-metadata typeorm
    npm install --save-dev @types/pg

We aren't going to use plain SQL, instead we will use an Object-relational mapping framework (ORM) called `TypeORM`.
`TypeORM` takes advantage of Typescript classes and type declarations in order to infer the db structure.

We will need to enable support for experimental decorators, emit type metadata for decorators and disable strict property initialization:

[{]: <helper> (diffStep "6.1" files="tsconfig.json" module="server")

#### Step 6.1: TypeORM with PostgreSQL

##### Changed tsconfig.json
```diff
@@ -22,11 +22,11 @@
 ┊22┊22┊    // "isolatedModules": true,               /* Transpile each file as a separate module (similar to 'ts.transpileModule'). */
 ┊23┊23┊
 ┊24┊24┊    /* Strict Type-Checking Options */
-┊25┊  ┊    "strict": true,                            /* Enable all strict type-checking options. */
+┊  ┊25┊    "strict": true,                           /* Enable all strict type-checking options. */
 ┊26┊26┊    // "noImplicitAny": true,                 /* Raise error on expressions and declarations with an implied 'any' type. */
 ┊27┊27┊    // "strictNullChecks": true,              /* Enable strict null checks. */
 ┊28┊28┊    // See https://github.com/DefinitelyTyped/DefinitelyTyped/issues/21359
-┊29┊  ┊    "strictFunctionTypes": false              /* Enable strict checking of function types. */
+┊  ┊29┊    "strictFunctionTypes": false,             /* Enable strict checking of function types. */
 ┊30┊30┊    // "noImplicitThis": true,                /* Raise error on 'this' expressions with an implied 'any' type. */
 ┊31┊31┊    // "alwaysStrict": true,                  /* Parse in strict mode and emit "use strict" for each source file. */
 ┊32┊32┊
```
```diff
@@ -53,7 +53,8 @@
 ┊53┊53┊    // "inlineSources": true,                 /* Emit the source alongside the sourcemaps within a single file; requires '--inlineSourceMap' or '--sourceMap' to be set. */
 ┊54┊54┊
 ┊55┊55┊    /* Experimental Options */
-┊56┊  ┊    // "experimentalDecorators": true,        /* Enables experimental support for ES7 decorators. */
-┊57┊  ┊    // "emitDecoratorMetadata": true,         /* Enables experimental support for emitting type metadata for decorators. */
+┊  ┊56┊    "experimentalDecorators": true,           /* Enables experimental support for ES7 decorators. */
+┊  ┊57┊    "emitDecoratorMetadata": true,            /* Enables experimental support for emitting type metadata for decorators. */
+┊  ┊58┊    "strictPropertyInitialization": false
 ┊58┊59┊  }
 ┊59┊60┊}🚫↵
```

[}]: #

The next step is to create Entities. An Entity is a class that maps to a database table. You can create a entity by defining a new class and mark it with @Entity():

[{]: <helper> (diffStep "6.1" files="entity" module="server")

#### Step 6.1: TypeORM with PostgreSQL

##### Added entity&#x2F;Chat.ts
```diff
@@ -0,0 +1,79 @@
+┊  ┊ 1┊import { Entity, Column, PrimaryGeneratedColumn, OneToMany, JoinTable, ManyToMany, ManyToOne } from "typeorm";
+┊  ┊ 2┊import { Message } from "./Message";
+┊  ┊ 3┊import { User } from "./User";
+┊  ┊ 4┊import { Recipient } from "./Recipient";
+┊  ┊ 5┊
+┊  ┊ 6┊interface ChatConstructor {
+┊  ┊ 7┊  name?: string;
+┊  ┊ 8┊  picture?: string;
+┊  ┊ 9┊  allTimeMembers?: User[];
+┊  ┊10┊  listingMembers?: User[];
+┊  ┊11┊  actualGroupMembers?: User[];
+┊  ┊12┊  admins?: User[];
+┊  ┊13┊  owner?: User;
+┊  ┊14┊  messages?: Message[];
+┊  ┊15┊}
+┊  ┊16┊
+┊  ┊17┊@Entity()
+┊  ┊18┊export class Chat {
+┊  ┊19┊  @PrimaryGeneratedColumn()
+┊  ┊20┊  id: number;
+┊  ┊21┊
+┊  ┊22┊  @Column({nullable: true})
+┊  ┊23┊  name: string;
+┊  ┊24┊
+┊  ┊25┊  @Column({nullable: true})
+┊  ┊26┊  picture: string;
+┊  ┊27┊
+┊  ┊28┊  @ManyToMany(type => User, user => user.allTimeMemberChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊29┊  @JoinTable()
+┊  ┊30┊  allTimeMembers: User[];
+┊  ┊31┊
+┊  ┊32┊  @ManyToMany(type => User, user => user.listingMemberChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊33┊  @JoinTable()
+┊  ┊34┊  listingMembers: User[];
+┊  ┊35┊
+┊  ┊36┊  @ManyToMany(type => User, user => user.actualGroupMemberChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊37┊  @JoinTable()
+┊  ┊38┊  actualGroupMembers?: User[];
+┊  ┊39┊
+┊  ┊40┊  @ManyToMany(type => User, user => user.adminChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊41┊  @JoinTable()
+┊  ┊42┊  admins?: User[];
+┊  ┊43┊
+┊  ┊44┊  @ManyToOne(type => User, user => user.ownerChats, {cascade: ["insert", "update"], eager: false})
+┊  ┊45┊  owner?: User | null;
+┊  ┊46┊
+┊  ┊47┊  @OneToMany(type => Message, message => message.chat, {cascade: ["insert", "update"], eager: true})
+┊  ┊48┊  messages: Message[];
+┊  ┊49┊
+┊  ┊50┊  @OneToMany(type => Recipient, recipient => recipient.chat)
+┊  ┊51┊  recipients: Recipient[];
+┊  ┊52┊
+┊  ┊53┊  constructor({name, picture, allTimeMembers, listingMembers, actualGroupMembers, admins, owner, messages}: ChatConstructor = {}) {
+┊  ┊54┊    if (name) {
+┊  ┊55┊      this.name = name;
+┊  ┊56┊    }
+┊  ┊57┊    if (picture) {
+┊  ┊58┊      this.picture = picture;
+┊  ┊59┊    }
+┊  ┊60┊    if (allTimeMembers) {
+┊  ┊61┊      this.allTimeMembers = allTimeMembers;
+┊  ┊62┊    }
+┊  ┊63┊    if (listingMembers) {
+┊  ┊64┊      this.listingMembers = listingMembers;
+┊  ┊65┊    }
+┊  ┊66┊    if (actualGroupMembers) {
+┊  ┊67┊      this.actualGroupMembers = actualGroupMembers;
+┊  ┊68┊    }
+┊  ┊69┊    if (admins) {
+┊  ┊70┊      this.admins = admins;
+┊  ┊71┊    }
+┊  ┊72┊    if (owner) {
+┊  ┊73┊      this.owner = owner;
+┊  ┊74┊    }
+┊  ┊75┊    if (messages) {
+┊  ┊76┊      this.messages = messages;
+┊  ┊77┊    }
+┊  ┊78┊  }
+┊  ┊79┊}
```

##### Added entity&#x2F;Message.ts
```diff
@@ -0,0 +1,70 @@
+┊  ┊ 1┊import {
+┊  ┊ 2┊  Entity, Column, PrimaryGeneratedColumn, OneToMany, ManyToOne, ManyToMany, JoinTable, CreateDateColumn
+┊  ┊ 3┊} from "typeorm";
+┊  ┊ 4┊import { Chat } from "./Chat";
+┊  ┊ 5┊import { User } from "./User";
+┊  ┊ 6┊import { Recipient } from "./Recipient";
+┊  ┊ 7┊import { MessageType } from "../db";
+┊  ┊ 8┊
+┊  ┊ 9┊interface MessageConstructor {
+┊  ┊10┊  sender?: User;
+┊  ┊11┊  content?: string;
+┊  ┊12┊  createdAt?: Date,
+┊  ┊13┊  type?: MessageType;
+┊  ┊14┊  recipients?: Recipient[];
+┊  ┊15┊  holders?: User[];
+┊  ┊16┊  chat?: Chat;
+┊  ┊17┊}
+┊  ┊18┊
+┊  ┊19┊@Entity()
+┊  ┊20┊export class Message {
+┊  ┊21┊  @PrimaryGeneratedColumn()
+┊  ┊22┊  id: number;
+┊  ┊23┊
+┊  ┊24┊  @ManyToOne(type => User, user => user.senderMessages, {eager: true})
+┊  ┊25┊  sender: User;
+┊  ┊26┊
+┊  ┊27┊  @Column()
+┊  ┊28┊  content: string;
+┊  ┊29┊
+┊  ┊30┊  @CreateDateColumn({nullable: true})
+┊  ┊31┊  createdAt: Date;
+┊  ┊32┊
+┊  ┊33┊  @Column()
+┊  ┊34┊  type: number;
+┊  ┊35┊
+┊  ┊36┊  @OneToMany(type => Recipient, recipient => recipient.message, {cascade: ["insert", "update"], eager: true})
+┊  ┊37┊  recipients: Recipient[];
+┊  ┊38┊
+┊  ┊39┊  @ManyToMany(type => User, user => user.holderMessages, {cascade: ["insert", "update"], eager: true})
+┊  ┊40┊  @JoinTable()
+┊  ┊41┊  holders: User[];
+┊  ┊42┊
+┊  ┊43┊  @ManyToOne(type => Chat, chat => chat.messages)
+┊  ┊44┊  chat: Chat;
+┊  ┊45┊
+┊  ┊46┊  constructor({sender, content, createdAt, type, recipients, holders, chat}: MessageConstructor = {}) {
+┊  ┊47┊    if (sender) {
+┊  ┊48┊      this.sender = sender;
+┊  ┊49┊    }
+┊  ┊50┊    if (content) {
+┊  ┊51┊      this.content = content;
+┊  ┊52┊    }
+┊  ┊53┊    if (createdAt) {
+┊  ┊54┊      this.createdAt = createdAt;
+┊  ┊55┊    }
+┊  ┊56┊    if (type) {
+┊  ┊57┊      this.type = type;
+┊  ┊58┊    }
+┊  ┊59┊    if (recipients) {
+┊  ┊60┊      recipients.forEach(recipient => recipient.message = this);
+┊  ┊61┊      this.recipients = recipients;
+┊  ┊62┊    }
+┊  ┊63┊    if (holders) {
+┊  ┊64┊      this.holders = holders;
+┊  ┊65┊    }
+┊  ┊66┊    if (chat) {
+┊  ┊67┊      this.chat = chat;
+┊  ┊68┊    }
+┊  ┊69┊  }
+┊  ┊70┊}
```

##### Added entity&#x2F;Recipient.ts
```diff
@@ -0,0 +1,44 @@
+┊  ┊ 1┊import { Entity, ManyToOne, Column } from "typeorm";
+┊  ┊ 2┊import { Message } from "./Message";
+┊  ┊ 3┊import { User } from "./User";
+┊  ┊ 4┊import { Chat } from "./Chat";
+┊  ┊ 5┊
+┊  ┊ 6┊interface RecipientConstructor {
+┊  ┊ 7┊  user?: User;
+┊  ┊ 8┊  message?: Message;
+┊  ┊ 9┊  receivedAt?: Date;
+┊  ┊10┊  readAt?: Date;
+┊  ┊11┊}
+┊  ┊12┊
+┊  ┊13┊@Entity()
+┊  ┊14┊export class Recipient {
+┊  ┊15┊  @ManyToOne(type => User, user => user.recipients, { primary: true })
+┊  ┊16┊  user: User;
+┊  ┊17┊
+┊  ┊18┊  @ManyToOne(type => Message, message => message.recipients, { primary: true })
+┊  ┊19┊  message: Message;
+┊  ┊20┊
+┊  ┊21┊  @ManyToOne(type => Chat, chat => chat.recipients)
+┊  ┊22┊  chat: Chat;
+┊  ┊23┊
+┊  ┊24┊  @Column({nullable: true})
+┊  ┊25┊  receivedAt: Date;
+┊  ┊26┊
+┊  ┊27┊  @Column({nullable: true})
+┊  ┊28┊  readAt: Date;
+┊  ┊29┊
+┊  ┊30┊  constructor({user, message, receivedAt, readAt}: RecipientConstructor = {}) {
+┊  ┊31┊    if (user) {
+┊  ┊32┊      this.user = user;
+┊  ┊33┊    }
+┊  ┊34┊    if (message) {
+┊  ┊35┊      this.message = message;
+┊  ┊36┊    }
+┊  ┊37┊    if (receivedAt) {
+┊  ┊38┊      this.receivedAt = receivedAt;
+┊  ┊39┊    }
+┊  ┊40┊    if (readAt) {
+┊  ┊41┊      this.readAt = readAt;
+┊  ┊42┊    }
+┊  ┊43┊  }
+┊  ┊44┊}
```

##### Added entity&#x2F;User.ts
```diff
@@ -0,0 +1,75 @@
+┊  ┊ 1┊import { Entity, Column, PrimaryGeneratedColumn, ManyToMany, OneToMany } from "typeorm";
+┊  ┊ 2┊import { Chat } from "./Chat";
+┊  ┊ 3┊import { Message } from "./Message";
+┊  ┊ 4┊import { Recipient } from "./Recipient";
+┊  ┊ 5┊
+┊  ┊ 6┊interface UserConstructor {
+┊  ┊ 7┊  username?: string;
+┊  ┊ 8┊  password?: string;
+┊  ┊ 9┊  name?: string;
+┊  ┊10┊  picture?: string;
+┊  ┊11┊  phone?: string;
+┊  ┊12┊}
+┊  ┊13┊
+┊  ┊14┊@Entity()
+┊  ┊15┊export class User {
+┊  ┊16┊  @PrimaryGeneratedColumn()
+┊  ┊17┊  id: number;
+┊  ┊18┊
+┊  ┊19┊  @Column()
+┊  ┊20┊  username: string;
+┊  ┊21┊
+┊  ┊22┊  @Column()
+┊  ┊23┊  password: string;
+┊  ┊24┊
+┊  ┊25┊  @Column()
+┊  ┊26┊  name: string;
+┊  ┊27┊
+┊  ┊28┊  @Column({nullable: true})
+┊  ┊29┊  picture: string;
+┊  ┊30┊
+┊  ┊31┊  @Column({nullable: true})
+┊  ┊32┊  phone?: string;
+┊  ┊33┊
+┊  ┊34┊  @ManyToMany(type => Chat, chat => chat.allTimeMembers)
+┊  ┊35┊  allTimeMemberChats: Chat[];
+┊  ┊36┊
+┊  ┊37┊  @ManyToMany(type => Chat, chat => chat.listingMembers)
+┊  ┊38┊  listingMemberChats: Chat[];
+┊  ┊39┊
+┊  ┊40┊  @ManyToMany(type => Chat, chat => chat.actualGroupMembers)
+┊  ┊41┊  actualGroupMemberChats: Chat[];
+┊  ┊42┊
+┊  ┊43┊  @ManyToMany(type => Chat, chat => chat.admins)
+┊  ┊44┊  adminChats: Chat[];
+┊  ┊45┊
+┊  ┊46┊  @ManyToMany(type => Message, message => message.holders)
+┊  ┊47┊  holderMessages: Message[];
+┊  ┊48┊
+┊  ┊49┊  @OneToMany(type => Chat, chat => chat.owner)
+┊  ┊50┊  ownerChats: Chat[];
+┊  ┊51┊
+┊  ┊52┊  @OneToMany(type => Message, message => message.sender)
+┊  ┊53┊  senderMessages: Message[];
+┊  ┊54┊
+┊  ┊55┊  @OneToMany(type => Recipient, recipient => recipient.user)
+┊  ┊56┊  recipients: Recipient[];
+┊  ┊57┊
+┊  ┊58┊  constructor({username, password, name, picture, phone}: UserConstructor = {}) {
+┊  ┊59┊    if (username) {
+┊  ┊60┊      this.username = username;
+┊  ┊61┊    }
+┊  ┊62┊    if (password) {
+┊  ┊63┊      this.password = password;
+┊  ┊64┊    }
+┊  ┊65┊    if (name) {
+┊  ┊66┊      this.name = name;
+┊  ┊67┊    }
+┊  ┊68┊    if (picture) {
+┊  ┊69┊      this.picture = picture;
+┊  ┊70┊    }
+┊  ┊71┊    if (phone) {
+┊  ┊72┊      this.phone = phone;
+┊  ┊73┊    }
+┊  ┊74┊  }
+┊  ┊75┊}
```

[}]: #

Basic entities consist of columns and relations. Each entity MUST have a primary column.

Each entity must be registered in your connection options:

[{]: <helper> (diffStep "6.1" files="ormconfig.json" module="server")

#### Step 6.1: TypeORM with PostgreSQL

##### Added ormconfig.json
```diff
@@ -0,0 +1,24 @@
+┊  ┊ 1┊{
+┊  ┊ 2┊   "type": "postgres",
+┊  ┊ 3┊   "host": "localhost",
+┊  ┊ 4┊   "port": 5432,
+┊  ┊ 5┊   "username": "test",
+┊  ┊ 6┊   "password": "",
+┊  ┊ 7┊   "database": "test",
+┊  ┊ 8┊   "synchronize": true,
+┊  ┊ 9┊   "logging": false,
+┊  ┊10┊   "entities": [
+┊  ┊11┊      "entity/**/*.ts"
+┊  ┊12┊   ],
+┊  ┊13┊   "migrations": [
+┊  ┊14┊      "migration/**/*.ts"
+┊  ┊15┊   ],
+┊  ┊16┊   "subscribers": [
+┊  ┊17┊      "subscriber/**/*.ts"
+┊  ┊18┊   ],
+┊  ┊19┊   "cli": {
+┊  ┊20┊      "entitiesDir": "entity",
+┊  ┊21┊      "migrationsDir": "migration",
+┊  ┊22┊      "subscribersDir": "subscriber"
+┊  ┊23┊   }
+┊  ┊24┊}🚫↵
```

[}]: #

Since database table consist of columns your entities must consist of columns too. Each entity class property you marked with @Column will be mapped to a database table column.
Each entity must have at least one primary column. There are several types of primary columns, but in our case `@PrimaryGeneratedColumn()` creates a primary column which value will be automatically generated with an auto-increment value.
`@CreateDateColumn` is a special column that is automatically set to the entity's insertion date. You don't need set this column - it will be automatically set.
For the Recipient Entity we use a composite primary key that consists of two foreign keys.

The next thing to do is to create a connection with the database before firing up the web server:

[{]: <helper> (diffStep "6.1" files="index.ts" module="server")

#### Step 6.1: TypeORM with PostgreSQL

##### Changed index.ts
```diff
@@ -1,3 +1,5 @@
+┊ ┊1┊// For TypeORM
+┊ ┊2┊import "reflect-metadata";
 ┊1┊3┊import { schema } from "./schema";
 ┊2┊4┊import * as bodyParser from "body-parser";
 ┊3┊5┊import * as cors from 'cors';
```
```diff
@@ -6,12 +8,12 @@
 ┊ 6┊ 8┊import * as passport from "passport";
 ┊ 7┊ 9┊import * as basicStrategy from 'passport-http';
 ┊ 8┊10┊import * as bcrypt from 'bcrypt-nodejs';
-┊ 9┊  ┊import { db, User } from "./db";
 ┊10┊11┊import { createServer } from "http";
 ┊11┊12┊import { SubscriptionServer } from "subscriptions-transport-ws";
 ┊12┊13┊import { execute, subscribe } from "graphql";
-┊13┊  ┊
-┊14┊  ┊let users = db.users;
+┊  ┊14┊import { createConnection } from "typeorm";
+┊  ┊15┊import { User } from "./entity/User";
+┊  ┊16┊import { addSampleData } from "./db";
 ┊15┊17┊
 ┊16┊18┊function generateHash(password: string) {
 ┊17┊19┊  return bcrypt.hashSync(password, bcrypt.genSaltSync(8));
```
```diff
@@ -21,94 +23,97 @@
 ┊ 21┊ 23┊  return bcrypt.compareSync(password, localPassword);
 ┊ 22┊ 24┊}
 ┊ 23┊ 25┊
-┊ 24┊   ┊passport.use('basic-signin', new basicStrategy.BasicStrategy(
-┊ 25┊   ┊  function (username, password, done) {
-┊ 26┊   ┊    const user = users.find(user => user.username == username);
-┊ 27┊   ┊    if (user && validPassword(password, user.password)) {
-┊ 28┊   ┊      return done(null, user);
+┊   ┊ 26┊createConnection().then(async connection => {
+┊   ┊ 27┊  await addSampleData(connection);
+┊   ┊ 28┊
+┊   ┊ 29┊  passport.use('basic-signin', new basicStrategy.BasicStrategy(
+┊   ┊ 30┊    async function (username, password, done) {
+┊   ┊ 31┊      const user = await connection.getRepository(User).findOne({where: { username }});
+┊   ┊ 32┊      if (user && validPassword(password, user.password)) {
+┊   ┊ 33┊        return done(null, user);
+┊   ┊ 34┊      }
+┊   ┊ 35┊      return done(null, false);
 ┊ 29┊ 36┊    }
-┊ 30┊   ┊    return done(null, false);
-┊ 31┊   ┊  }
-┊ 32┊   ┊));
+┊   ┊ 37┊  ));
 ┊ 33┊ 38┊
-┊ 34┊   ┊passport.use('basic-signup', new basicStrategy.BasicStrategy({passReqToCallback: true},
-┊ 35┊   ┊  function (req: any, username: any, password: any, done: any) {
-┊ 36┊   ┊    const userExists = !!users.find(user => user.username === username);
-┊ 37┊   ┊    if (!userExists && password && req.body.name) {
-┊ 38┊   ┊      const user: User = {
-┊ 39┊   ┊        id: (users.length && users[users.length - 1].id + 1) || 1,
-┊ 40┊   ┊        username,
-┊ 41┊   ┊        password: generateHash(password),
-┊ 42┊   ┊        name: req.body.name,
-┊ 43┊   ┊      };
-┊ 44┊   ┊      users.push(user);
-┊ 45┊   ┊      return done(null, user);
+┊   ┊ 39┊  passport.use('basic-signup', new basicStrategy.BasicStrategy({passReqToCallback: true},
+┊   ┊ 40┊    async function (req: any, username: any, password: any, done: any) {
+┊   ┊ 41┊      const userExists = !!(await connection.getRepository(User).findOne({where: { username }}));
+┊   ┊ 42┊      if (!userExists && password && req.body.name) {
+┊   ┊ 43┊        const user = await connection.manager.save(new User({
+┊   ┊ 44┊          username,
+┊   ┊ 45┊          password: generateHash(password),
+┊   ┊ 46┊          name: req.body.name,
+┊   ┊ 47┊        }));
+┊   ┊ 48┊        return done(null, user);
+┊   ┊ 49┊      }
+┊   ┊ 50┊      return done(null, false);
 ┊ 46┊ 51┊    }
-┊ 47┊   ┊    return done(null, false);
-┊ 48┊   ┊  }
-┊ 49┊   ┊));
+┊   ┊ 52┊  ));
 ┊ 50┊ 53┊
-┊ 51┊   ┊const PORT = 3000;
+┊   ┊ 54┊  const PORT = 3000;
 ┊ 52┊ 55┊
-┊ 53┊   ┊const app = express();
+┊   ┊ 56┊  const app = express();
 ┊ 54┊ 57┊
-┊ 55┊   ┊app.use(cors());
-┊ 56┊   ┊app.use(bodyParser.json());
-┊ 57┊   ┊app.use(passport.initialize());
+┊   ┊ 58┊  app.use(cors());
+┊   ┊ 59┊  app.use(bodyParser.json());
+┊   ┊ 60┊  app.use(passport.initialize());
 ┊ 58┊ 61┊
-┊ 59┊   ┊app.post('/signup',
-┊ 60┊   ┊  passport.authenticate('basic-signup', {session: false}),
-┊ 61┊   ┊  function (req, res) {
-┊ 62┊   ┊    res.json(req.user);
-┊ 63┊   ┊  });
+┊   ┊ 62┊  app.post('/signup',
+┊   ┊ 63┊    passport.authenticate('basic-signup', {session: false}),
+┊   ┊ 64┊    function (req, res) {
+┊   ┊ 65┊      res.json(req.user);
+┊   ┊ 66┊    });
 ┊ 64┊ 67┊
-┊ 65┊   ┊app.use(passport.authenticate('basic-signin', {session: false}));
+┊   ┊ 68┊  app.use(passport.authenticate('basic-signin', {session: false}));
 ┊ 66┊ 69┊
-┊ 67┊   ┊app.post('/signin', function (req, res) {
-┊ 68┊   ┊  res.json(req.user);
-┊ 69┊   ┊});
+┊   ┊ 70┊  app.post('/signin', function (req, res) {
+┊   ┊ 71┊    res.json(req.user);
+┊   ┊ 72┊  });
 ┊ 70┊ 73┊
-┊ 71┊   ┊app.use('/graphql', graphqlExpress(req => ({
-┊ 72┊   ┊  schema: schema,
-┊ 73┊   ┊  context: {
-┊ 74┊   ┊    user: req!['user'],
-┊ 75┊   ┊  },
-┊ 76┊   ┊})));
+┊   ┊ 74┊  app.use('/graphql', graphqlExpress(req => ({
+┊   ┊ 75┊    schema: schema,
+┊   ┊ 76┊    context: {
+┊   ┊ 77┊      user: req!['user'],
+┊   ┊ 78┊      connection,
+┊   ┊ 79┊    },
+┊   ┊ 80┊  })));
 ┊ 77┊ 81┊
-┊ 78┊   ┊app.use('/graphiql', graphiqlExpress({
-┊ 79┊   ┊  endpointURL: '/graphql',
-┊ 80┊   ┊}));
+┊   ┊ 82┊  app.use('/graphiql', graphiqlExpress({
+┊   ┊ 83┊    endpointURL: '/graphql',
+┊   ┊ 84┊  }));
 ┊ 81┊ 85┊
 ┊ 82┊ 86┊// Wrap the Express server
-┊ 83┊   ┊const ws = createServer(app);
-┊ 84┊   ┊ws.listen(PORT, () => {
-┊ 85┊   ┊  console.log(`Apollo Server is now running on http://localhost:${PORT}`);
-┊ 86┊   ┊  // Set up the WebSocket for handling GraphQL subscriptions
-┊ 87┊   ┊  new SubscriptionServer({
-┊ 88┊   ┊    onConnect: (connectionParams: any, webSocket: any) => {
-┊ 89┊   ┊      if (connectionParams.authToken) {
-┊ 90┊   ┊        // create a buffer and tell it the data coming in is base64
-┊ 91┊   ┊        const buf = new Buffer(connectionParams.authToken.split(' ')[1], 'base64');
-┊ 92┊   ┊        // read it back out as a string
-┊ 93┊   ┊        const [username, password]: string[] = buf.toString().split(':');
-┊ 94┊   ┊        if (username && password) {
-┊ 95┊   ┊          const user = users.find(user => user.username == username);
+┊   ┊ 87┊  const ws = createServer(app);
+┊   ┊ 88┊  ws.listen(PORT, () => {
+┊   ┊ 89┊    console.log(`Apollo Server is now running on http://localhost:${PORT}`);
+┊   ┊ 90┊    // Set up the WebSocket for handling GraphQL subscriptions
+┊   ┊ 91┊    new SubscriptionServer({
+┊   ┊ 92┊      onConnect: async (connectionParams: any, webSocket: any) => {
+┊   ┊ 93┊        if (connectionParams.authToken) {
+┊   ┊ 94┊          // Create a buffer and tell it the data coming in is base64
+┊   ┊ 95┊          const buf = new Buffer(connectionParams.authToken.split(' ')[1], 'base64');
+┊   ┊ 96┊          // Read it back out as a string
+┊   ┊ 97┊          const [username, password]: string[] = buf.toString().split(':');
+┊   ┊ 98┊          if (username && password) {
+┊   ┊ 99┊            const user = await connection.getRepository(User).findOne({where: { username }});
 ┊ 96┊100┊
-┊ 97┊   ┊          if (user && validPassword(password, user.password)) {
-┊ 98┊   ┊            // Set context for the WebSocket
-┊ 99┊   ┊            return {user};
-┊100┊   ┊          } else {
-┊101┊   ┊            throw new Error('Wrong credentials!');
+┊   ┊101┊            if (user && validPassword(password, user.password)) {
+┊   ┊102┊              // Set context for the WebSocket
+┊   ┊103┊              return {user, connection};
+┊   ┊104┊            } else {
+┊   ┊105┊              throw new Error('Wrong credentials!');
+┊   ┊106┊            }
 ┊102┊107┊          }
 ┊103┊108┊        }
-┊104┊   ┊      }
-┊105┊   ┊      throw new Error('Missing auth token!');
-┊106┊   ┊    },
-┊107┊   ┊    execute,
-┊108┊   ┊    subscribe,
-┊109┊   ┊    schema
-┊110┊   ┊  }, {
-┊111┊   ┊    server: ws,
-┊112┊   ┊    path: '/subscriptions',
+┊   ┊109┊        throw new Error('Missing auth token!');
+┊   ┊110┊      },
+┊   ┊111┊      execute,
+┊   ┊112┊      subscribe,
+┊   ┊113┊      schema
+┊   ┊114┊    }, {
+┊   ┊115┊      server: ws,
+┊   ┊116┊      path: '/subscriptions',
+┊   ┊117┊    });
 ┊113┊118┊  });
 ┊114┊119┊});
```

[}]: #

We will also remove our fake db and replace it with some real data:

[{]: <helper> (diffStep "6.1" files="db.ts" module="server")

#### Step 6.1: TypeORM with PostgreSQL

##### Changed db.ts
```diff
@@ -1,4 +1,11 @@
+┊  ┊ 1┊// For TypeORM
+┊  ┊ 2┊import "reflect-metadata";
+┊  ┊ 3┊import { Chat } from "./entity/Chat";
+┊  ┊ 4┊import { Recipient } from "./entity/Recipient";
 ┊ 1┊ 5┊import * as moment from 'moment';
+┊  ┊ 6┊import { Message } from "./entity/Message";
+┊  ┊ 7┊import { User } from "./entity/User";
+┊  ┊ 8┊import { Connection } from "typeorm";
 ┊ 2┊ 9┊
 ┊ 3┊10┊export enum MessageType {
 ┊ 4┊11┊  PICTURE,
```
```diff
@@ -6,433 +13,280 @@
 ┊  6┊ 13┊  LOCATION,
 ┊  7┊ 14┊}
 ┊  8┊ 15┊
-┊  9┊   ┊export interface User {
-┊ 10┊   ┊  id: number,
-┊ 11┊   ┊  username: string,
-┊ 12┊   ┊  password: string,
-┊ 13┊   ┊  name: string,
-┊ 14┊   ┊  picture?: string | null,
-┊ 15┊   ┊  phone?: string | null,
-┊ 16┊   ┊}
-┊ 17┊   ┊
-┊ 18┊   ┊export interface Chat {
-┊ 19┊   ┊  id: number,
-┊ 20┊   ┊  name?: string | null,
-┊ 21┊   ┊  picture?: string | null,
-┊ 22┊   ┊  // All members, current and past ones.
-┊ 23┊   ┊  allTimeMemberIds: number[],
-┊ 24┊   ┊  // Whoever gets the chat listed. For groups includes past members who still didn't delete the group.
-┊ 25┊   ┊  listingMemberIds: number[],
-┊ 26┊   ┊  // Actual members of the group (they are not the only ones who get the group listed). Null for chats.
-┊ 27┊   ┊  actualGroupMemberIds?: number[] | null,
-┊ 28┊   ┊  adminIds?: number[] | null,
-┊ 29┊   ┊  ownerId?: number | null,
-┊ 30┊   ┊  messages: Message[],
-┊ 31┊   ┊}
-┊ 32┊   ┊
-┊ 33┊   ┊export interface Message {
-┊ 34┊   ┊  id: number,
-┊ 35┊   ┊  chatId: number,
-┊ 36┊   ┊  senderId: number,
-┊ 37┊   ┊  content: string,
-┊ 38┊   ┊  createdAt: number,
-┊ 39┊   ┊  type: MessageType,
-┊ 40┊   ┊  recipients: Recipient[],
-┊ 41┊   ┊  holderIds: number[],
-┊ 42┊   ┊}
-┊ 43┊   ┊
-┊ 44┊   ┊export interface Recipient {
-┊ 45┊   ┊  userId: number,
-┊ 46┊   ┊  messageId: number,
-┊ 47┊   ┊  chatId: number,
-┊ 48┊   ┊  receivedAt: number | null,
-┊ 49┊   ┊  readAt: number | null,
-┊ 50┊   ┊}
-┊ 51┊   ┊
-┊ 52┊   ┊const users: User[] = [
-┊ 53┊   ┊  {
-┊ 54┊   ┊    id: 1,
+┊   ┊ 16┊export async function addSampleData(connection: Connection) {
+┊   ┊ 17┊  const user1 = new User({
 ┊ 55┊ 18┊    username: 'ethan',
 ┊ 56┊ 19┊    password: '$2a$08$NO9tkFLCoSqX1c5wk3s7z.JfxaVMKA.m7zUDdDwEquo4rvzimQeJm', // 111
 ┊ 57┊ 20┊    name: 'Ethan Gonzalez',
 ┊ 58┊ 21┊    picture: 'https://randomuser.me/api/portraits/thumb/men/1.jpg',
 ┊ 59┊ 22┊    phone: '+391234567890',
-┊ 60┊   ┊  },
-┊ 61┊   ┊  {
-┊ 62┊   ┊    id: 2,
+┊   ┊ 23┊  });
+┊   ┊ 24┊  await connection.manager.save(user1);
+┊   ┊ 25┊
+┊   ┊ 26┊  const user2 = new User({
 ┊ 63┊ 27┊    username: 'bryan',
 ┊ 64┊ 28┊    password: '$2a$08$xE4FuCi/ifxjL2S8CzKAmuKLwv18ktksSN.F3XYEnpmcKtpbpeZgO', // 222
 ┊ 65┊ 29┊    name: 'Bryan Wallace',
 ┊ 66┊ 30┊    picture: 'https://randomuser.me/api/portraits/thumb/men/2.jpg',
 ┊ 67┊ 31┊    phone: '+391234567891',
-┊ 68┊   ┊  },
-┊ 69┊   ┊  {
-┊ 70┊   ┊    id: 3,
+┊   ┊ 32┊  });
+┊   ┊ 33┊  await connection.manager.save(user2);
+┊   ┊ 34┊
+┊   ┊ 35┊  const user3 = new User({
 ┊ 71┊ 36┊    username: 'avery',
 ┊ 72┊ 37┊    password: '$2a$08$UHgH7J8G6z1mGQn2qx2kdeWv0jvgHItyAsL9hpEUI3KJmhVW5Q1d.', // 333
 ┊ 73┊ 38┊    name: 'Avery Stewart',
 ┊ 74┊ 39┊    picture: 'https://randomuser.me/api/portraits/thumb/women/1.jpg',
 ┊ 75┊ 40┊    phone: '+391234567892',
-┊ 76┊   ┊  },
-┊ 77┊   ┊  {
-┊ 78┊   ┊    id: 4,
+┊   ┊ 41┊  });
+┊   ┊ 42┊  await connection.manager.save(user3);
+┊   ┊ 43┊
+┊   ┊ 44┊  const user4 = new User({
 ┊ 79┊ 45┊    username: 'katie',
 ┊ 80┊ 46┊    password: '$2a$08$wR1k5Q3T9FC7fUgB7Gdb9Os/GV7dGBBf4PLlWT7HERMFhmFDt47xi', // 444
 ┊ 81┊ 47┊    name: 'Katie Peterson',
 ┊ 82┊ 48┊    picture: 'https://randomuser.me/api/portraits/thumb/women/2.jpg',
 ┊ 83┊ 49┊    phone: '+391234567893',
-┊ 84┊   ┊  },
-┊ 85┊   ┊  {
-┊ 86┊   ┊    id: 5,
+┊   ┊ 50┊  });
+┊   ┊ 51┊  await connection.manager.save(user4);
+┊   ┊ 52┊
+┊   ┊ 53┊  const user5 = new User({
 ┊ 87┊ 54┊    username: 'ray',
 ┊ 88┊ 55┊    password: '$2a$08$6.mbXqsDX82ZZ7q5d8Osb..JrGSsNp4R3IKj7mxgF6YGT0OmMw242', // 555
 ┊ 89┊ 56┊    name: 'Ray Edwards',
 ┊ 90┊ 57┊    picture: 'https://randomuser.me/api/portraits/thumb/men/3.jpg',
 ┊ 91┊ 58┊    phone: '+391234567894',
-┊ 92┊   ┊  },
-┊ 93┊   ┊  {
-┊ 94┊   ┊    id: 6,
+┊   ┊ 59┊  });
+┊   ┊ 60┊  await connection.manager.save(user5);
+┊   ┊ 61┊
+┊   ┊ 62┊  const user6 = new User({
 ┊ 95┊ 63┊    username: 'niko',
 ┊ 96┊ 64┊    password: '$2a$08$fL5lZR.Rwf9FWWe8XwwlceiPBBim8n9aFtaem.INQhiKT4.Ux3Uq.', // 666
 ┊ 97┊ 65┊    name: 'Niccolò Belli',
 ┊ 98┊ 66┊    picture: 'https://randomuser.me/api/portraits/thumb/men/4.jpg',
 ┊ 99┊ 67┊    phone: '+391234567895',
-┊100┊   ┊  },
-┊101┊   ┊  {
-┊102┊   ┊    id: 7,
+┊   ┊ 68┊  });
+┊   ┊ 69┊  await connection.manager.save(user6);
+┊   ┊ 70┊
+┊   ┊ 71┊  const user7 = new User({
 ┊103┊ 72┊    username: 'mario',
 ┊104┊ 73┊    password: '$2a$08$nDHDmWcVxDnH5DDT3HMMC.psqcnu6wBiOgkmJUy9IH..qxa3R6YrO', // 777
 ┊105┊ 74┊    name: 'Mario Rossi',
 ┊106┊ 75┊    picture: 'https://randomuser.me/api/portraits/thumb/men/5.jpg',
 ┊107┊ 76┊    phone: '+391234567896',
-┊108┊   ┊  },
-┊109┊   ┊];
+┊   ┊ 77┊  });
+┊   ┊ 78┊  await connection.manager.save(user7);
+┊   ┊ 79┊
+┊   ┊ 80┊
 ┊110┊ 81┊
-┊111┊   ┊const chats: Chat[] = [
-┊112┊   ┊  {
-┊113┊   ┊    id: 1,
-┊114┊   ┊    name: null,
-┊115┊   ┊    picture: null,
-┊116┊   ┊    allTimeMemberIds: [1, 3],
-┊117┊   ┊    listingMemberIds: [1, 3],
-┊118┊   ┊    adminIds: null,
-┊119┊   ┊    ownerId: null,
+┊   ┊ 82┊
+┊   ┊ 83┊  await connection.manager.save(new Chat({
+┊   ┊ 84┊    allTimeMembers: [user1, user3],
+┊   ┊ 85┊    listingMembers: [user1, user3],
 ┊120┊ 86┊    messages: [
-┊121┊   ┊      {
-┊122┊   ┊        id: 1,
-┊123┊   ┊        chatId: 1,
-┊124┊   ┊        senderId: 1,
+┊   ┊ 87┊      new Message({
+┊   ┊ 88┊        sender: user1,
 ┊125┊ 89┊        content: 'You on your way?',
-┊126┊   ┊        createdAt: moment().subtract(1, 'hours').unix(),
+┊   ┊ 90┊        createdAt: moment().subtract(1, 'hours').toDate(),
 ┊127┊ 91┊        type: MessageType.TEXT,
+┊   ┊ 92┊        holders: [user1, user3],
 ┊128┊ 93┊        recipients: [
-┊129┊   ┊          {
-┊130┊   ┊            userId: 3,
-┊131┊   ┊            messageId: 1,
-┊132┊   ┊            chatId: 1,
-┊133┊   ┊            receivedAt: null,
-┊134┊   ┊            readAt: null,
-┊135┊   ┊          },
+┊   ┊ 94┊          new Recipient({
+┊   ┊ 95┊            user: user3,
+┊   ┊ 96┊          }),
 ┊136┊ 97┊        ],
-┊137┊   ┊        holderIds: [1, 3],
-┊138┊   ┊      },
-┊139┊   ┊      {
-┊140┊   ┊        id: 2,
-┊141┊   ┊        chatId: 1,
-┊142┊   ┊        senderId: 3,
+┊   ┊ 98┊      }),
+┊   ┊ 99┊      new Message({
+┊   ┊100┊        sender: user3,
 ┊143┊101┊        content: 'Yep!',
-┊144┊   ┊        createdAt: moment().subtract(1, 'hours').add(5, 'minutes').unix(),
+┊   ┊102┊        createdAt: moment().subtract(1, 'hours').add(5, 'minutes').toDate(),
 ┊145┊103┊        type: MessageType.TEXT,
+┊   ┊104┊        holders: [user1, user3],
 ┊146┊105┊        recipients: [
-┊147┊   ┊          {
-┊148┊   ┊            userId: 1,
-┊149┊   ┊            messageId: 2,
-┊150┊   ┊            chatId: 1,
-┊151┊   ┊            receivedAt: null,
-┊152┊   ┊            readAt: null,
-┊153┊   ┊          },
+┊   ┊106┊          new Recipient({
+┊   ┊107┊            user: user1,
+┊   ┊108┊          }),
 ┊154┊109┊        ],
-┊155┊   ┊        holderIds: [3, 1],
-┊156┊   ┊      },
+┊   ┊110┊      }),
 ┊157┊111┊    ],
-┊158┊   ┊  },
-┊159┊   ┊  {
-┊160┊   ┊    id: 2,
-┊161┊   ┊    name: null,
-┊162┊   ┊    picture: null,
-┊163┊   ┊    allTimeMemberIds: [1, 4],
-┊164┊   ┊    listingMemberIds: [1, 4],
-┊165┊   ┊    adminIds: null,
-┊166┊   ┊    ownerId: null,
+┊   ┊112┊  }));
+┊   ┊113┊
+┊   ┊114┊  await connection.manager.save(new Chat({
+┊   ┊115┊    allTimeMembers: [user1, user4],
+┊   ┊116┊    listingMembers: [user1, user4],
 ┊167┊117┊    messages: [
-┊168┊   ┊      {
-┊169┊   ┊        id: 1,
-┊170┊   ┊        chatId: 2,
-┊171┊   ┊        senderId: 1,
+┊   ┊118┊      new Message({
+┊   ┊119┊        sender: user1,
 ┊172┊120┊        content: 'Hey, it\'s me',
-┊173┊   ┊        createdAt: moment().subtract(2, 'hours').unix(),
+┊   ┊121┊        createdAt: moment().subtract(2, 'hours').toDate(),
 ┊174┊122┊        type: MessageType.TEXT,
+┊   ┊123┊        holders: [user1, user4],
 ┊175┊124┊        recipients: [
-┊176┊   ┊          {
-┊177┊   ┊            userId: 4,
-┊178┊   ┊            messageId: 1,
-┊179┊   ┊            chatId: 2,
-┊180┊   ┊            receivedAt: null,
-┊181┊   ┊            readAt: null,
-┊182┊   ┊          },
+┊   ┊125┊          new Recipient({
+┊   ┊126┊            user: user4,
+┊   ┊127┊          }),
 ┊183┊128┊        ],
-┊184┊   ┊        holderIds: [1, 4],
-┊185┊   ┊      },
+┊   ┊129┊      }),
 ┊186┊130┊    ],
-┊187┊   ┊  },
-┊188┊   ┊  {
-┊189┊   ┊    id: 3,
-┊190┊   ┊    name: null,
-┊191┊   ┊    picture: null,
-┊192┊   ┊    allTimeMemberIds: [1, 5],
-┊193┊   ┊    listingMemberIds: [1, 5],
-┊194┊   ┊    adminIds: null,
-┊195┊   ┊    ownerId: null,
+┊   ┊131┊  }));
+┊   ┊132┊
+┊   ┊133┊  await connection.manager.save(new Chat({
+┊   ┊134┊    allTimeMembers: [user1, user5],
+┊   ┊135┊    listingMembers: [user1, user5],
 ┊196┊136┊    messages: [
-┊197┊   ┊      {
-┊198┊   ┊        id: 1,
-┊199┊   ┊        chatId: 3,
-┊200┊   ┊        senderId: 1,
+┊   ┊137┊      new Message({
+┊   ┊138┊        sender: user1,
 ┊201┊139┊        content: 'I should buy a boat',
-┊202┊   ┊        createdAt: moment().subtract(1, 'days').unix(),
+┊   ┊140┊        createdAt: moment().subtract(1, 'days').toDate(),
 ┊203┊141┊        type: MessageType.TEXT,
+┊   ┊142┊        holders: [user1, user5],
 ┊204┊143┊        recipients: [
-┊205┊   ┊          {
-┊206┊   ┊            userId: 5,
-┊207┊   ┊            messageId: 1,
-┊208┊   ┊            chatId: 3,
-┊209┊   ┊            receivedAt: null,
-┊210┊   ┊            readAt: null,
-┊211┊   ┊          },
+┊   ┊144┊          new Recipient({
+┊   ┊145┊            user: user5,
+┊   ┊146┊          }),
 ┊212┊147┊        ],
-┊213┊   ┊        holderIds: [1, 5],
-┊214┊   ┊      },
-┊215┊   ┊      {
-┊216┊   ┊        id: 2,
-┊217┊   ┊        chatId: 3,
-┊218┊   ┊        senderId: 1,
+┊   ┊148┊      }),
+┊   ┊149┊      new Message({
+┊   ┊150┊        sender: user1,
 ┊219┊151┊        content: 'You still there?',
-┊220┊   ┊        createdAt: moment().subtract(1, 'days').add(16, 'hours').unix(),
+┊   ┊152┊        createdAt: moment().subtract(1, 'days').add(16, 'hours').toDate(),
 ┊221┊153┊        type: MessageType.TEXT,
+┊   ┊154┊        holders: [user1, user5],
 ┊222┊155┊        recipients: [
-┊223┊   ┊          {
-┊224┊   ┊            userId: 5,
-┊225┊   ┊            messageId: 2,
-┊226┊   ┊            chatId: 3,
-┊227┊   ┊            receivedAt: null,
-┊228┊   ┊            readAt: null,
-┊229┊   ┊          },
+┊   ┊156┊          new Recipient({
+┊   ┊157┊            user: user5,
+┊   ┊158┊          }),
 ┊230┊159┊        ],
-┊231┊   ┊        holderIds: [1, 5],
-┊232┊   ┊      },
+┊   ┊160┊      }),
 ┊233┊161┊    ],
-┊234┊   ┊  },
-┊235┊   ┊  {
-┊236┊   ┊    id: 4,
-┊237┊   ┊    name: null,
-┊238┊   ┊    picture: null,
-┊239┊   ┊    allTimeMemberIds: [3, 4],
-┊240┊   ┊    listingMemberIds: [3, 4],
-┊241┊   ┊    adminIds: null,
-┊242┊   ┊    ownerId: null,
+┊   ┊162┊  }));
+┊   ┊163┊
+┊   ┊164┊  await connection.manager.save(new Chat({
+┊   ┊165┊    allTimeMembers: [user3, user4],
+┊   ┊166┊    listingMembers: [user3, user4],
 ┊243┊167┊    messages: [
-┊244┊   ┊      {
-┊245┊   ┊        id: 1,
-┊246┊   ┊        chatId: 4,
-┊247┊   ┊        senderId: 3,
+┊   ┊168┊      new Message({
+┊   ┊169┊        sender: user3,
 ┊248┊170┊        content: 'Look at my mukluks!',
-┊249┊   ┊        createdAt: moment().subtract(4, 'days').unix(),
+┊   ┊171┊        createdAt: moment().subtract(4, 'days').toDate(),
 ┊250┊172┊        type: MessageType.TEXT,
+┊   ┊173┊        holders: [user3, user4],
 ┊251┊174┊        recipients: [
-┊252┊   ┊          {
-┊253┊   ┊            userId: 4,
-┊254┊   ┊            messageId: 1,
-┊255┊   ┊            chatId: 4,
-┊256┊   ┊            receivedAt: null,
-┊257┊   ┊            readAt: null,
-┊258┊   ┊          },
+┊   ┊175┊          new Recipient({
+┊   ┊176┊            user: user4,
+┊   ┊177┊          }),
 ┊259┊178┊        ],
-┊260┊   ┊        holderIds: [3, 4],
-┊261┊   ┊      },
+┊   ┊179┊      }),
 ┊262┊180┊    ],
-┊263┊   ┊  },
-┊264┊   ┊  {
-┊265┊   ┊    id: 5,
-┊266┊   ┊    name: null,
-┊267┊   ┊    picture: null,
-┊268┊   ┊    allTimeMemberIds: [2, 5],
-┊269┊   ┊    listingMemberIds: [2, 5],
-┊270┊   ┊    adminIds: null,
-┊271┊   ┊    ownerId: null,
+┊   ┊181┊  }));
+┊   ┊182┊
+┊   ┊183┊  await connection.manager.save(new Chat({
+┊   ┊184┊    allTimeMembers: [user2, user5],
+┊   ┊185┊    listingMembers: [user2, user5],
 ┊272┊186┊    messages: [
-┊273┊   ┊      {
-┊274┊   ┊        id: 1,
-┊275┊   ┊        chatId: 5,
-┊276┊   ┊        senderId: 2,
+┊   ┊187┊      new Message({
+┊   ┊188┊        sender: user2,
 ┊277┊189┊        content: 'This is wicked good ice cream.',
-┊278┊   ┊        createdAt: moment().subtract(2, 'weeks').unix(),
+┊   ┊190┊        createdAt: moment().subtract(2, 'weeks').toDate(),
 ┊279┊191┊        type: MessageType.TEXT,
+┊   ┊192┊        holders: [user2, user5],
 ┊280┊193┊        recipients: [
-┊281┊   ┊          {
-┊282┊   ┊            userId: 5,
-┊283┊   ┊            messageId: 1,
-┊284┊   ┊            chatId: 5,
-┊285┊   ┊            receivedAt: null,
-┊286┊   ┊            readAt: null,
-┊287┊   ┊          },
+┊   ┊194┊          new Recipient({
+┊   ┊195┊            user: user5,
+┊   ┊196┊          }),
 ┊288┊197┊        ],
-┊289┊   ┊        holderIds: [2, 5],
-┊290┊   ┊      },
-┊291┊   ┊      {
-┊292┊   ┊        id: 2,
-┊293┊   ┊        chatId: 6,
-┊294┊   ┊        senderId: 5,
+┊   ┊198┊      }),
+┊   ┊199┊      new Message({
+┊   ┊200┊        sender: user5,
 ┊295┊201┊        content: 'Love it!',
-┊296┊   ┊        createdAt: moment().subtract(2, 'weeks').add(10, 'minutes').unix(),
+┊   ┊202┊        createdAt: moment().subtract(2, 'weeks').add(10, 'minutes').toDate(),
 ┊297┊203┊        type: MessageType.TEXT,
+┊   ┊204┊        holders: [user2, user5],
 ┊298┊205┊        recipients: [
-┊299┊   ┊          {
-┊300┊   ┊            userId: 2,
-┊301┊   ┊            messageId: 2,
-┊302┊   ┊            chatId: 5,
-┊303┊   ┊            receivedAt: null,
-┊304┊   ┊            readAt: null,
-┊305┊   ┊          },
+┊   ┊206┊          new Recipient({
+┊   ┊207┊            user: user2,
+┊   ┊208┊          }),
 ┊306┊209┊        ],
-┊307┊   ┊        holderIds: [5, 2],
-┊308┊   ┊      },
+┊   ┊210┊      }),
 ┊309┊211┊    ],
-┊310┊   ┊  },
-┊311┊   ┊  {
-┊312┊   ┊    id: 6,
-┊313┊   ┊    name: null,
-┊314┊   ┊    picture: null,
-┊315┊   ┊    allTimeMemberIds: [1, 6],
-┊316┊   ┊    listingMemberIds: [1],
-┊317┊   ┊    adminIds: null,
-┊318┊   ┊    ownerId: null,
-┊319┊   ┊    messages: [],
-┊320┊   ┊  },
-┊321┊   ┊  {
-┊322┊   ┊    id: 7,
-┊323┊   ┊    name: null,
-┊324┊   ┊    picture: null,
-┊325┊   ┊    allTimeMemberIds: [2, 1],
-┊326┊   ┊    listingMemberIds: [2],
-┊327┊   ┊    adminIds: null,
-┊328┊   ┊    ownerId: null,
-┊329┊   ┊    messages: [],
-┊330┊   ┊  },
-┊331┊   ┊  {
-┊332┊   ┊    id: 8,
-┊333┊   ┊    name: 'A user 0 group',
+┊   ┊212┊  }));
+┊   ┊213┊
+┊   ┊214┊  await connection.manager.save(new Chat({
+┊   ┊215┊    allTimeMembers: [user1, user6],
+┊   ┊216┊    listingMembers: [user1],
+┊   ┊217┊  }));
+┊   ┊218┊
+┊   ┊219┊  await connection.manager.save(new Chat({
+┊   ┊220┊    allTimeMembers: [user2, user1],
+┊   ┊221┊    listingMembers: [user2],
+┊   ┊222┊  }));
+┊   ┊223┊
+┊   ┊224┊  await connection.manager.save(new Chat({
+┊   ┊225┊    name: 'Ethan\'s group',
 ┊334┊226┊    picture: 'https://randomuser.me/api/portraits/thumb/lego/1.jpg',
-┊335┊   ┊    allTimeMemberIds: [1, 3, 4, 6],
-┊336┊   ┊    listingMemberIds: [1, 3, 4, 6],
-┊337┊   ┊    actualGroupMemberIds: [1, 4, 6],
-┊338┊   ┊    adminIds: [1, 6],
-┊339┊   ┊    ownerId: 1,
+┊   ┊227┊    allTimeMembers: [user1, user3, user4, user6],
+┊   ┊228┊    listingMembers: [user1, user3, user4, user6],
+┊   ┊229┊    actualGroupMembers: [user1, user4, user6],
+┊   ┊230┊    admins: [user1, user6],
+┊   ┊231┊    owner: user1,
 ┊340┊232┊    messages: [
-┊341┊   ┊      {
-┊342┊   ┊        id: 1,
-┊343┊   ┊        chatId: 8,
-┊344┊   ┊        senderId: 1,
+┊   ┊233┊      new Message({
+┊   ┊234┊        sender: user1,
 ┊345┊235┊        content: 'I made a group',
-┊346┊   ┊        createdAt: moment().subtract(2, 'weeks').unix(),
+┊   ┊236┊        createdAt: moment().subtract(2, 'weeks').toDate(),
 ┊347┊237┊        type: MessageType.TEXT,
+┊   ┊238┊        holders: [user1, user3, user4, user6],
 ┊348┊239┊        recipients: [
-┊349┊   ┊          {
-┊350┊   ┊            userId: 3,
-┊351┊   ┊            messageId: 1,
-┊352┊   ┊            chatId: 8,
-┊353┊   ┊            receivedAt: null,
-┊354┊   ┊            readAt: null,
-┊355┊   ┊          },
-┊356┊   ┊          {
-┊357┊   ┊            userId: 4,
-┊358┊   ┊            messageId: 1,
-┊359┊   ┊            chatId: 8,
-┊360┊   ┊            receivedAt: moment().subtract(2, 'weeks').add(1, 'minutes').unix(),
-┊361┊   ┊            readAt: moment().subtract(2, 'weeks').add(5, 'minutes').unix(),
-┊362┊   ┊          },
-┊363┊   ┊          {
-┊364┊   ┊            userId: 6,
-┊365┊   ┊            messageId: 1,
-┊366┊   ┊            chatId: 8,
-┊367┊   ┊            receivedAt: null,
-┊368┊   ┊            readAt: null,
-┊369┊   ┊          },
+┊   ┊240┊          new Recipient({
+┊   ┊241┊            user: user3,
+┊   ┊242┊          }),
+┊   ┊243┊          new Recipient({
+┊   ┊244┊            user: user4,
+┊   ┊245┊          }),
+┊   ┊246┊          new Recipient({
+┊   ┊247┊            user: user6,
+┊   ┊248┊          }),
 ┊370┊249┊        ],
-┊371┊   ┊        holderIds: [1, 3, 4, 6],
-┊372┊   ┊      },
-┊373┊   ┊      {
-┊374┊   ┊        id: 2,
-┊375┊   ┊        chatId: 8,
-┊376┊   ┊        senderId: 1,
-┊377┊   ┊        content: 'Ops, user 3 was not supposed to be here',
-┊378┊   ┊        createdAt: moment().subtract(2, 'weeks').add(2, 'minutes').unix(),
+┊   ┊250┊      }),
+┊   ┊251┊      new Message({
+┊   ┊252┊        sender: user1,
+┊   ┊253┊        content: 'Ops, Avery was not supposed to be here',
+┊   ┊254┊        createdAt: moment().subtract(2, 'weeks').add(2, 'minutes').toDate(),
 ┊379┊255┊        type: MessageType.TEXT,
+┊   ┊256┊        holders: [user1, user4, user6],
 ┊380┊257┊        recipients: [
-┊381┊   ┊          {
-┊382┊   ┊            userId: 4,
-┊383┊   ┊            messageId: 2,
-┊384┊   ┊            chatId: 8,
-┊385┊   ┊            receivedAt: moment().subtract(2, 'weeks').add(3, 'minutes').unix(),
-┊386┊   ┊            readAt: moment().subtract(2, 'weeks').add(5, 'minutes').unix(),
-┊387┊   ┊          },
-┊388┊   ┊          {
-┊389┊   ┊            userId: 6,
-┊390┊   ┊            messageId: 2,
-┊391┊   ┊            chatId: 8,
-┊392┊   ┊            receivedAt: null,
-┊393┊   ┊            readAt: null,
-┊394┊   ┊          },
+┊   ┊258┊          new Recipient({
+┊   ┊259┊            user: user4,
+┊   ┊260┊          }),
+┊   ┊261┊          new Recipient({
+┊   ┊262┊            user: user6,
+┊   ┊263┊          }),
 ┊395┊264┊        ],
-┊396┊   ┊        holderIds: [1, 4, 6],
-┊397┊   ┊      },
-┊398┊   ┊      {
-┊399┊   ┊        id: 3,
-┊400┊   ┊        chatId: 8,
-┊401┊   ┊        senderId: 4,
+┊   ┊265┊      }),
+┊   ┊266┊      new Message({
+┊   ┊267┊        sender: user4,
 ┊402┊268┊        content: 'Awesome!',
-┊403┊   ┊        createdAt: moment().subtract(2, 'weeks').add(10, 'minutes').unix(),
+┊   ┊269┊        createdAt: moment().subtract(2, 'weeks').add(10, 'minutes').toDate(),
 ┊404┊270┊        type: MessageType.TEXT,
+┊   ┊271┊        holders: [user1, user4, user6],
 ┊405┊272┊        recipients: [
-┊406┊   ┊          {
-┊407┊   ┊            userId: 1,
-┊408┊   ┊            messageId: 3,
-┊409┊   ┊            chatId: 8,
-┊410┊   ┊            receivedAt: null,
-┊411┊   ┊            readAt: null,
-┊412┊   ┊          },
-┊413┊   ┊          {
-┊414┊   ┊            userId: 6,
-┊415┊   ┊            messageId: 3,
-┊416┊   ┊            chatId: 8,
-┊417┊   ┊            receivedAt: null,
-┊418┊   ┊            readAt: null,
-┊419┊   ┊          },
+┊   ┊273┊          new Recipient({
+┊   ┊274┊            user: user1,
+┊   ┊275┊          }),
+┊   ┊276┊          new Recipient({
+┊   ┊277┊            user: user6,
+┊   ┊278┊          }),
 ┊420┊279┊        ],
-┊421┊   ┊        holderIds: [1, 4, 6],
-┊422┊   ┊      },
+┊   ┊280┊      }),
 ┊423┊281┊    ],
-┊424┊   ┊  },
-┊425┊   ┊  {
-┊426┊   ┊    id: 9,
-┊427┊   ┊    name: 'A user 5 group',
-┊428┊   ┊    picture: null,
-┊429┊   ┊    allTimeMemberIds: [6, 3],
-┊430┊   ┊    listingMemberIds: [6, 3],
-┊431┊   ┊    actualGroupMemberIds: [6, 3],
-┊432┊   ┊    adminIds: [6],
-┊433┊   ┊    ownerId: 6,
-┊434┊   ┊    messages: [],
-┊435┊   ┊  },
-┊436┊   ┊];
+┊   ┊282┊  }));
 ┊437┊283┊
-┊438┊   ┊export const db = {users, chats};
+┊   ┊284┊  await connection.manager.save(new Chat({
+┊   ┊285┊    name: 'Ray\'s group',
+┊   ┊286┊    allTimeMembers: [user3, user6],
+┊   ┊287┊    listingMembers: [user3, user6],
+┊   ┊288┊    actualGroupMembers: [user3, user6],
+┊   ┊289┊    admins: [user6],
+┊   ┊290┊    owner: user6,
+┊   ┊291┊  }));
+┊   ┊292┊}
```

[}]: #

It's time to deal with resolvers:

[{]: <helper> (diffStep "6.1" files="schema/resolvers.ts" module="server")

#### Step 6.1: TypeORM with PostgreSQL

##### Changed schema&#x2F;resolvers.ts
```diff
@@ -1,4 +1,4 @@
-┊1┊ ┊import { Chat, db, Message, MessageType, Recipient, User } from "../db";
+┊ ┊1┊import { MessageType } from "../db";
 ┊2┊2┊import { IResolvers } from "graphql-tools/dist/Interfaces";
 ┊3┊3┊import {
 ┊4┊4┊  AddChatMutationArgs, AddGroupMutationArgs, AddMessageMutationArgs, ChatQueryArgs, MessageAddedSubscriptionArgs,
```
```diff
@@ -6,87 +6,123 @@
 ┊  6┊  6┊} from "../types";
 ┊  7┊  7┊import * as moment from "moment";
 ┊  8┊  8┊import { PubSub, withFilter } from "graphql-subscriptions";
-┊  9┊   ┊
-┊ 10┊   ┊let users = db.users;
-┊ 11┊   ┊let chats = db.chats;
+┊   ┊  9┊import { User } from "../entity/User";
+┊   ┊ 10┊import { Chat } from "../entity/Chat";
+┊   ┊ 11┊import { Message } from "../entity/Message";
+┊   ┊ 12┊import { Recipient } from "../entity/Recipient";
+┊   ┊ 13┊import { Connection } from "typeorm";
 ┊ 12┊ 14┊
 ┊ 13┊ 15┊export const pubsub = new PubSub();
 ┊ 14┊ 16┊
 ┊ 15┊ 17┊export const resolvers: IResolvers = {
 ┊ 16┊ 18┊  Query: {
 ┊ 17┊ 19┊    // Show all users for the moment.
-┊ 18┊   ┊    users: (obj: any, args: any, {user: currentUser}: {user: User}): User[] => users.filter(user => user.id !== currentUser.id),
-┊ 19┊   ┊    chats: (obj: any, args: any, {user: currentUser}: {user: User}): Chat[] => chats.filter(chat => chat.listingMemberIds.includes(currentUser.id)),
-┊ 20┊   ┊    chat: (obj: any, {chatId}: ChatQueryArgs): Chat | null => chats.find(chat => chat.id === Number(chatId)) || null,
+┊   ┊ 20┊    users: async (obj: any, args: any, {user: currentUser, connection}: { user: User, connection: Connection }): Promise<User[]> => {
+┊   ┊ 21┊      return await connection
+┊   ┊ 22┊        .createQueryBuilder(User, "user")
+┊   ┊ 23┊        .where('user.id != :id', {id: currentUser.id})
+┊   ┊ 24┊        .getMany();
+┊   ┊ 25┊    },
+┊   ┊ 26┊    chats: async (obj: any, args: any, {user: currentUser, connection}: { user: User, connection: Connection }): Promise<any[]> => {
+┊   ┊ 27┊      return await connection
+┊   ┊ 28┊        .createQueryBuilder(Chat, "chat")
+┊   ┊ 29┊        .leftJoin('chat.listingMembers', 'listingMembers')
+┊   ┊ 30┊        .where('listingMembers.id = :id', {id: currentUser.id})
+┊   ┊ 31┊        .getMany();
+┊   ┊ 32┊    },
+┊   ┊ 33┊    chat: async (obj: any, {chatId}: ChatQueryArgs, {connection}: { user: User, connection: Connection }): Promise<any> => {
+┊   ┊ 34┊      return await connection
+┊   ┊ 35┊        .createQueryBuilder(Chat, "chat")
+┊   ┊ 36┊        .whereInIds(chatId)
+┊   ┊ 37┊        .getOne();
+┊   ┊ 38┊    },
 ┊ 21┊ 39┊  },
 ┊ 22┊ 40┊  Mutation: {
-┊ 23┊   ┊    addChat: (obj: any, {recipientId}: AddChatMutationArgs, {user: currentUser}: {user: User}): Chat => {
-┊ 24┊   ┊      if (!users.find(user => user.id === Number(recipientId))) {
+┊   ┊ 41┊    addChat: async (obj: any, {recipientId}: AddChatMutationArgs, {user: currentUser, connection}: { user: User, connection: Connection }): Promise<Chat | null> => {
+┊   ┊ 42┊      const recipient = await connection
+┊   ┊ 43┊        .createQueryBuilder(User, "user")
+┊   ┊ 44┊        .whereInIds(recipientId)
+┊   ┊ 45┊        .getOne();
+┊   ┊ 46┊
+┊   ┊ 47┊      if (!recipient) {
 ┊ 25┊ 48┊        throw new Error(`Recipient ${recipientId} doesn't exist.`);
 ┊ 26┊ 49┊      }
 ┊ 27┊ 50┊
-┊ 28┊   ┊      const chat = chats.find(chat => !chat.name && chat.allTimeMemberIds.includes(currentUser.id) && chat.allTimeMemberIds.includes(Number(recipientId)));
+┊   ┊ 51┊      let chat = await connection
+┊   ┊ 52┊        .createQueryBuilder(Chat, "chat")
+┊   ┊ 53┊        .where('chat.name IS NULL')
+┊   ┊ 54┊        .innerJoin('chat.allTimeMembers', 'allTimeMembers1', 'allTimeMembers1.id = :currentUserId', {currentUserId: currentUser.id})
+┊   ┊ 55┊        .innerJoin('chat.allTimeMembers', 'allTimeMembers2', 'allTimeMembers2.id = :recipientId', {recipientId})
+┊   ┊ 56┊        .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊ 57┊        .getOne();
+┊   ┊ 58┊
 ┊ 29┊ 59┊      if (chat) {
-┊ 30┊   ┊        // Chat already exists. Both users are already in the allTimeMemberIds array
-┊ 31┊   ┊        const chatId = chat.id;
-┊ 32┊   ┊        if (!chat.listingMemberIds.includes(currentUser.id)) {
+┊   ┊ 60┊        // Chat already exists. Both users are already in the userIds array
+┊   ┊ 61┊        const listingMembers = await connection
+┊   ┊ 62┊          .createQueryBuilder(User, "user")
+┊   ┊ 63┊          .innerJoin('user.listingMemberChats', 'listingMemberChats', 'listingMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊ 64┊          .getMany();
+┊   ┊ 65┊
+┊   ┊ 66┊        if (!listingMembers.find(user => user.id === currentUser.id)) {
 ┊ 33┊ 67┊          // The chat isn't listed for the current user. Add him to the memberIds
-┊ 34┊   ┊          chat.listingMemberIds.push(currentUser.id);
-┊ 35┊   ┊          chats.find(chat => chat.id === chatId)!.listingMemberIds.push(currentUser.id);
-┊ 36┊   ┊          return chat;
+┊   ┊ 68┊          chat.listingMembers.push(currentUser);
+┊   ┊ 69┊          chat = await connection.getRepository(Chat).save(chat);
+┊   ┊ 70┊
+┊   ┊ 71┊          return chat || null;
 ┊ 37┊ 72┊        } else {
 ┊ 38┊ 73┊          throw new Error(`Chat already exists.`);
 ┊ 39┊ 74┊        }
 ┊ 40┊ 75┊      } else {
 ┊ 41┊ 76┊        // Create the chat
-┊ 42┊   ┊        const id = (chats.length && chats[chats.length - 1].id + 1) || 1;
-┊ 43┊   ┊        const chat: Chat = {
-┊ 44┊   ┊          id,
-┊ 45┊   ┊          name: null,
-┊ 46┊   ┊          picture: null,
-┊ 47┊   ┊          adminIds: null,
-┊ 48┊   ┊          ownerId: null,
-┊ 49┊   ┊          allTimeMemberIds: [currentUser.id, Number(recipientId)],
+┊   ┊ 77┊        chat = await connection.getRepository(Chat).save(new Chat({
+┊   ┊ 78┊          allTimeMembers: [currentUser, recipient],
 ┊ 50┊ 79┊          // Chat will not be listed to the other user until the first message gets written
-┊ 51┊   ┊          listingMemberIds: [currentUser.id],
-┊ 52┊   ┊          actualGroupMemberIds: null,
-┊ 53┊   ┊          messages: [],
-┊ 54┊   ┊        };
-┊ 55┊   ┊        chats.push(chat);
+┊   ┊ 80┊          listingMembers: [currentUser],
+┊   ┊ 81┊        }));
 ┊ 56┊ 82┊
-┊ 57┊   ┊        return chat;
+┊   ┊ 83┊        return chat || null;
 ┊ 58┊ 84┊      }
 ┊ 59┊ 85┊    },
-┊ 60┊   ┊    addGroup: (obj: any, {recipientIds, groupName}: AddGroupMutationArgs, {user: currentUser}: {user: User}): Chat => {
-┊ 61┊   ┊      recipientIds.forEach(recipientId => {
-┊ 62┊   ┊        if (!users.find(user => user.id === Number(recipientId))) {
+┊   ┊ 86┊    addGroup: async (obj: any, {recipientIds, groupName}: AddGroupMutationArgs, {user: currentUser, connection}: { user: User, connection: Connection }): Promise<Chat | null> => {
+┊   ┊ 87┊      let recipients: User[] = [];
+┊   ┊ 88┊      for (let recipientId of recipientIds) {
+┊   ┊ 89┊        const recipient = await connection
+┊   ┊ 90┊          .createQueryBuilder(User, "user")
+┊   ┊ 91┊          .whereInIds(recipientId)
+┊   ┊ 92┊          .getOne();
+┊   ┊ 93┊        if (!recipient) {
 ┊ 63┊ 94┊          throw new Error(`Recipient ${recipientId} doesn't exist.`);
 ┊ 64┊ 95┊        }
-┊ 65┊   ┊      });
+┊   ┊ 96┊        recipients.push(recipient);
+┊   ┊ 97┊      }
 ┊ 66┊ 98┊
-┊ 67┊   ┊      const id = (chats.length && chats[chats.length - 1].id + 1) || 1;
-┊ 68┊   ┊      const chat: Chat = {
-┊ 69┊   ┊        id,
+┊   ┊ 99┊      const chat = await connection.getRepository(Chat).save(new Chat({
 ┊ 70┊100┊        name: groupName,
-┊ 71┊   ┊        picture: null,
-┊ 72┊   ┊        adminIds: [currentUser.id],
-┊ 73┊   ┊        ownerId: currentUser.id,
-┊ 74┊   ┊        allTimeMemberIds: [currentUser.id, ...recipientIds.map(id => Number(id))],
-┊ 75┊   ┊        listingMemberIds: [currentUser.id, ...recipientIds.map(id => Number(id))],
-┊ 76┊   ┊        actualGroupMemberIds: [currentUser.id, ...recipientIds.map(id => Number(id))],
-┊ 77┊   ┊        messages: [],
-┊ 78┊   ┊      };
-┊ 79┊   ┊      chats.push(chat);
+┊   ┊101┊        admins: [currentUser],
+┊   ┊102┊        owner: currentUser,
+┊   ┊103┊        allTimeMembers: [...recipients, currentUser],
+┊   ┊104┊        listingMembers: [...recipients, currentUser],
+┊   ┊105┊        actualGroupMembers: [...recipients, currentUser],
+┊   ┊106┊      }));
 ┊ 80┊107┊
 ┊ 81┊108┊      pubsub.publish('chatAdded', {
 ┊ 82┊109┊        creatorId: currentUser.id,
 ┊ 83┊110┊        chatAdded: chat,
 ┊ 84┊111┊      });
 ┊ 85┊112┊
-┊ 86┊   ┊      return chat;
+┊   ┊113┊      return chat || null;
 ┊ 87┊114┊    },
-┊ 88┊   ┊    removeChat: (obj: any, {chatId}: RemoveChatMutationArgs, {user: currentUser}: {user: User}): number => {
-┊ 89┊   ┊      const chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊115┊    removeChat: async (obj: any, {chatId}: RemoveChatMutationArgs, {user: currentUser, connection}: { user: User, connection: Connection }) => {
+┊   ┊116┊      const chat = await connection
+┊   ┊117┊        .createQueryBuilder(Chat, "chat")
+┊   ┊118┊        .whereInIds(Number(chatId))
+┊   ┊119┊        .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊120┊        .leftJoinAndSelect('chat.actualGroupMembers', 'actualGroupMembers')
+┊   ┊121┊        .leftJoinAndSelect('chat.admins', 'admins')
+┊   ┊122┊        .leftJoinAndSelect('chat.owner', 'owner')
+┊   ┊123┊        .leftJoinAndSelect('chat.messages', 'messages')
+┊   ┊124┊        .leftJoinAndSelect('messages.holders', 'holders')
+┊   ┊125┊        .getOne();
 ┊ 90┊126┊
 ┊ 91┊127┊      if (!chat) {
 ┊ 92┊128┊        throw new Error(`The chat ${chatId} doesn't exist.`);
```
```diff
@@ -94,186 +130,188 @@
 ┊ 94┊130┊
 ┊ 95┊131┊      if (!chat.name) {
 ┊ 96┊132┊        // Chat
-┊ 97┊   ┊        if (!chat.listingMemberIds.includes(currentUser.id)) {
-┊ 98┊   ┊          throw new Error(`The user is not a member of the chat ${chatId}.`);
+┊   ┊133┊        if (!chat.listingMembers.find(user => user.id === currentUser.id)) {
+┊   ┊134┊          throw new Error(`The user is not a listing member of the chat ${chatId}.`);
 ┊ 99┊135┊        }
 ┊100┊136┊
 ┊101┊137┊        // Instead of chaining map and filter we can loop once using reduce
-┊102┊   ┊        const messages = chat.messages.reduce<Message[]>((filtered, message) => {
-┊103┊   ┊          // Remove the current user from the message holders
-┊104┊   ┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
+┊   ┊138┊        chat.messages = await chat.messages.reduce<Promise<Message[]>>(async (filtered$, message) => {
+┊   ┊139┊          const filtered = await filtered$;
 ┊105┊140┊
-┊106┊   ┊          if (message.holderIds.length !== 0) {
+┊   ┊141┊          message.holders = message.holders.filter(user => user.id !== currentUser.id);
+┊   ┊142┊
+┊   ┊143┊          if (message.holders.length !== 0) {
+┊   ┊144┊            // Remove the current user from the message holders
+┊   ┊145┊            await connection.getRepository(Message).save(message);
 ┊107┊146┊            filtered.push(message);
-┊108┊   ┊          } // else discard the message
+┊   ┊147┊          } else {
+┊   ┊148┊            // Simply remove the message
+┊   ┊149┊            const recipients = await connection
+┊   ┊150┊              .createQueryBuilder(Recipient, "recipient")
+┊   ┊151┊              .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', {messageId: message.id})
+┊   ┊152┊              .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊153┊              .getMany();
+┊   ┊154┊            for (let recipient of recipients) {
+┊   ┊155┊              await connection.getRepository(Recipient).remove(recipient);
+┊   ┊156┊            }
+┊   ┊157┊            await connection.getRepository(Message).remove(message);
+┊   ┊158┊          }
 ┊109┊159┊
 ┊110┊160┊          return filtered;
-┊111┊   ┊        }, []);
+┊   ┊161┊        }, Promise.resolve([]));
 ┊112┊162┊
 ┊113┊163┊        // Remove the current user from who gets the chat listed. The chat will no longer appear in his list
-┊114┊   ┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser.id);
+┊   ┊164┊        chat.listingMembers = chat.listingMembers.filter(user => user.id !== currentUser.id);
 ┊115┊165┊
 ┊116┊166┊        // Check how many members are left
-┊117┊   ┊        if (listingMemberIds.length === 0) {
+┊   ┊167┊        if (chat.listingMembers.length === 0) {
 ┊118┊168┊          // Delete the chat
-┊119┊   ┊          chats = chats.filter(chat => chat.id !== Number(chatId));
+┊   ┊169┊          await connection.getRepository(Chat).remove(chat);
 ┊120┊170┊        } else {
 ┊121┊171┊          // Update the chat
-┊122┊   ┊          chats = chats.map(chat => {
-┊123┊   ┊            if (chat.id === Number(chatId)) {
-┊124┊   ┊              chat = {...chat, listingMemberIds, messages};
-┊125┊   ┊            }
-┊126┊   ┊            return chat;
-┊127┊   ┊          });
+┊   ┊172┊          await connection.getRepository(Chat).save(chat);
 ┊128┊173┊        }
-┊129┊   ┊        return Number(chatId);
+┊   ┊174┊        return chatId;
 ┊130┊175┊      } else {
 ┊131┊176┊        // Group
-┊132┊   ┊        if (chat.ownerId !== currentUser.id) {
-┊133┊   ┊          throw new Error(`Group ${chatId} is not owned by the user.`);
-┊134┊   ┊        }
 ┊135┊177┊
 ┊136┊178┊        // Instead of chaining map and filter we can loop once using reduce
-┊137┊   ┊        const messages = chat.messages.reduce<Message[]>((filtered, message) => {
-┊138┊   ┊          // Remove the current user from the message holders
-┊139┊   ┊          message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
+┊   ┊179┊        chat.messages = await chat.messages.reduce<Promise<Message[]>>(async (filtered$, message) => {
+┊   ┊180┊          const filtered = await filtered$;
+┊   ┊181┊
+┊   ┊182┊          message.holders = message.holders.filter(user => user.id !== currentUser.id);
 ┊140┊183┊
-┊141┊   ┊          if (message.holderIds.length !== 0) {
+┊   ┊184┊          if (message.holders.length !== 0) {
+┊   ┊185┊            // Remove the current user from the message holders
+┊   ┊186┊            await connection.getRepository(Message).save(message);
 ┊142┊187┊            filtered.push(message);
-┊143┊   ┊          } // else discard the message
+┊   ┊188┊          } else {
+┊   ┊189┊            // Simply remove the message
+┊   ┊190┊            const recipients = await connection
+┊   ┊191┊              .createQueryBuilder(Recipient, "recipient")
+┊   ┊192┊              .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', {messageId: message.id})
+┊   ┊193┊              .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊194┊              .getMany();
+┊   ┊195┊            for (let recipient of recipients) {
+┊   ┊196┊              await connection.getRepository(Recipient).remove(recipient);
+┊   ┊197┊            }
+┊   ┊198┊            await connection.getRepository(Message).remove(message);
+┊   ┊199┊          }
 ┊144┊200┊
 ┊145┊201┊          return filtered;
-┊146┊   ┊        }, []);
+┊   ┊202┊        }, Promise.resolve([]));
 ┊147┊203┊
 ┊148┊204┊        // Remove the current user from who gets the group listed. The group will no longer appear in his list
-┊149┊   ┊        const listingMemberIds = chat.listingMemberIds.filter(listingId => listingId !== currentUser.id);
+┊   ┊205┊        chat.listingMembers = chat.listingMembers.filter(user => user.id !== currentUser.id);
 ┊150┊206┊
 ┊151┊207┊        // Check how many members (including previous ones who can still access old messages) are left
-┊152┊   ┊        if (listingMemberIds.length === 0) {
+┊   ┊208┊        if (chat.listingMembers.length === 0) {
 ┊153┊209┊          // Remove the group
-┊154┊   ┊          chats = chats.filter(chat => chat.id !== Number(chatId));
+┊   ┊210┊          await connection.getRepository(Chat).remove(chat);
 ┊155┊211┊        } else {
 ┊156┊212┊          // Update the group
 ┊157┊213┊
 ┊158┊214┊          // Remove the current user from the chat members. He is no longer a member of the group
-┊159┊   ┊          const actualGroupMemberIds = chat.actualGroupMemberIds!.filter(memberId => memberId !== currentUser.id);
+┊   ┊215┊          chat.actualGroupMembers = chat.actualGroupMembers && chat.actualGroupMembers.filter(user => user.id !== currentUser.id);
 ┊160┊216┊          // Remove the current user from the chat admins
-┊161┊   ┊          const adminIds = chat.adminIds!.filter(memberId => memberId !== currentUser.id);
-┊162┊   ┊          // Set the owner id to be null. A null owner means the group is read-only
-┊163┊   ┊          let ownerId: number | null = null;
-┊164┊   ┊
-┊165┊   ┊          // Check if there is any admin left
-┊166┊   ┊          if (adminIds!.length) {
-┊167┊   ┊            // Pick an admin as the new owner. The group is no longer read-only
-┊168┊   ┊            ownerId = chat.adminIds![0];
-┊169┊   ┊          }
+┊   ┊217┊          chat.admins = chat.admins && chat.admins.filter(user => user.id !== currentUser.id);
+┊   ┊218┊          // If there are no more admins left the group goes read only
+┊   ┊219┊          chat.owner = chat.admins && chat.admins[0] || null; // A null owner means the group is read-only
 ┊170┊220┊
-┊171┊   ┊          chats = chats.map(chat => {
-┊172┊   ┊            if (chat.id === Number(chatId)) {
-┊173┊   ┊              chat = {...chat, messages, listingMemberIds, actualGroupMemberIds, adminIds, ownerId};
-┊174┊   ┊            }
-┊175┊   ┊            return chat;
-┊176┊   ┊          });
+┊   ┊221┊          await connection.getRepository(Chat).save(chat);
 ┊177┊222┊        }
-┊178┊   ┊        return Number(chatId);
+┊   ┊223┊        return chatId;
 ┊179┊224┊      }
 ┊180┊225┊    },
-┊181┊   ┊    addMessage: (obj: any, {chatId, content}: AddMessageMutationArgs, {user: currentUser}: {user: User}): Message => {
+┊   ┊226┊    addMessage: async (obj: any, {chatId, content}: AddMessageMutationArgs, {user: currentUser, connection}: { user: User, connection: Connection }): Promise<Message | null> => {
 ┊182┊227┊      if (content === null || content === '') {
 ┊183┊228┊        throw new Error(`Cannot add empty or null messages.`);
 ┊184┊229┊      }
 ┊185┊230┊
-┊186┊   ┊      let chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊231┊      let chat = await connection
+┊   ┊232┊        .createQueryBuilder(Chat, "chat")
+┊   ┊233┊        .whereInIds(chatId)
+┊   ┊234┊        .innerJoinAndSelect('chat.allTimeMembers', 'allTimeMembers')
+┊   ┊235┊        .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊236┊        .leftJoinAndSelect('chat.actualGroupMembers', 'actualGroupMembers')
+┊   ┊237┊        .getOne();
 ┊187┊238┊
 ┊188┊239┊      if (!chat) {
 ┊189┊240┊        throw new Error(`Cannot find chat ${chatId}.`);
 ┊190┊241┊      }
 ┊191┊242┊
-┊192┊   ┊      let holderIds = chat.listingMemberIds;
+┊   ┊243┊      let holders: User[];
 ┊193┊244┊
 ┊194┊245┊      if (!chat.name) {
 ┊195┊246┊        // Chat
-┊196┊   ┊        if (!chat.listingMemberIds.find(listingId => listingId === currentUser.id)) {
+┊   ┊247┊        if (!chat.listingMembers.map(user => user.id).includes(currentUser.id)) {
 ┊197┊248┊          throw new Error(`The chat ${chatId} must be listed for the current user before adding a message.`);
 ┊198┊249┊        }
 ┊199┊250┊
-┊200┊   ┊        const recipientId = chat.allTimeMemberIds.filter(userId => userId !== currentUser.id)[0];
+┊   ┊251┊        const recipientUser = chat.allTimeMembers.find(user => user.id !== currentUser.id);
 ┊201┊252┊
-┊202┊   ┊        if (!chat.listingMemberIds.find(listingId => listingId === recipientId)) {
-┊203┊   ┊          // Chat is not listed for the recipient. Add him to the listingMemberIds
-┊204┊   ┊          const listingMemberIds = chat.listingMemberIds.concat(recipientId);
+┊   ┊253┊        if (!recipientUser) {
+┊   ┊254┊          throw new Error(`Cannot find recipient user.`);
+┊   ┊255┊        }
 ┊205┊256┊
-┊206┊   ┊          chats = chats.map(chat => {
-┊207┊   ┊            if (chat.id === Number(chatId)) {
-┊208┊   ┊              chat = {...chat, listingMemberIds};
-┊209┊   ┊            }
-┊210┊   ┊            return chat;
-┊211┊   ┊          });
+┊   ┊257┊        if (!chat.listingMembers.find(user => user.id === recipientUser.id)) {
+┊   ┊258┊          // Chat is not listed for the recipient. Add him to the listingIds
+┊   ┊259┊          chat.listingMembers.push(recipientUser);
 ┊212┊260┊
-┊213┊   ┊          holderIds = listingMemberIds;
+┊   ┊261┊          await connection.getRepository(Chat).save(chat);
 ┊214┊262┊
 ┊215┊263┊          pubsub.publish('chatAdded', {
 ┊216┊264┊            creatorId: currentUser.id,
 ┊217┊265┊            chatAdded: chat,
 ┊218┊266┊          });
 ┊219┊267┊        }
+┊   ┊268┊
+┊   ┊269┊        holders = chat.listingMembers;
 ┊220┊270┊      } else {
 ┊221┊271┊        // Group
-┊222┊   ┊        if (!chat.actualGroupMemberIds!.find(memberId => memberId === currentUser.id)) {
+┊   ┊272┊        if (!chat.actualGroupMembers || !chat.actualGroupMembers.find(user => user.id === currentUser.id)) {
 ┊223┊273┊          throw new Error(`The user is not a member of the group ${chatId}. Cannot add message.`);
 ┊224┊274┊        }
 ┊225┊275┊
-┊226┊   ┊        holderIds = chat.actualGroupMemberIds!;
+┊   ┊276┊        holders = chat.actualGroupMembers;
 ┊227┊277┊      }
 ┊228┊278┊
-┊229┊   ┊      const id = (chat.messages.length && chat.messages[chat.messages.length - 1].id + 1) || 1;
-┊230┊   ┊
-┊231┊   ┊      let recipients: Recipient[] = [];
-┊232┊   ┊
-┊233┊   ┊      holderIds.forEach(holderId => {
-┊234┊   ┊        if (holderId !== currentUser.id) {
-┊235┊   ┊          recipients.push({
-┊236┊   ┊            userId: holderId,
-┊237┊   ┊            messageId: id,
-┊238┊   ┊            chatId: Number(chatId),
-┊239┊   ┊            receivedAt: null,
-┊240┊   ┊            readAt: null,
-┊241┊   ┊          });
-┊242┊   ┊        }
-┊243┊   ┊      });
-┊244┊   ┊
-┊245┊   ┊      const message: Message = {
-┊246┊   ┊        id,
-┊247┊   ┊        chatId: Number(chatId),
-┊248┊   ┊        senderId: currentUser.id,
+┊   ┊279┊      const message = await connection.getRepository(Message).save(new Message({
+┊   ┊280┊        chat: chat,
+┊   ┊281┊        sender: currentUser,
 ┊249┊282┊        content,
-┊250┊   ┊        createdAt: moment().unix(),
 ┊251┊283┊        type: MessageType.TEXT,
-┊252┊   ┊        recipients,
-┊253┊   ┊        holderIds,
-┊254┊   ┊      };
-┊255┊   ┊
-┊256┊   ┊      chats = chats.map(chat => {
-┊257┊   ┊        if (chat.id === Number(chatId)) {
-┊258┊   ┊          chat = {...chat, messages: chat.messages.concat(message)}
-┊259┊   ┊        }
-┊260┊   ┊        return chat;
-┊261┊   ┊      });
+┊   ┊284┊        holders,
+┊   ┊285┊        recipients: holders.reduce<Recipient[]>((filtered, user) => {
+┊   ┊286┊          if (user.id !== currentUser.id) {
+┊   ┊287┊            filtered.push(new Recipient({
+┊   ┊288┊              user,
+┊   ┊289┊            }));
+┊   ┊290┊          }
+┊   ┊291┊          return filtered;
+┊   ┊292┊        }, []),
+┊   ┊293┊      }));
 ┊262┊294┊
 ┊263┊295┊      pubsub.publish('messageAdded', {
 ┊264┊296┊        messageAdded: message,
 ┊265┊297┊      });
 ┊266┊298┊
-┊267┊   ┊      return message;
+┊   ┊299┊      return message || null;
 ┊268┊300┊    },
-┊269┊   ┊    removeMessages: (obj: any, {chatId, messageIds, all}: RemoveMessagesMutationArgs, {user: currentUser}: {user: User}): number[] => {
-┊270┊   ┊      const chat = chats.find(chat => chat.id === Number(chatId));
+┊   ┊301┊    removeMessages: async (obj: any, {chatId, messageIds, all}: RemoveMessagesMutationArgs, {user: currentUser, connection}: { user: User, connection: Connection }) => {
+┊   ┊302┊      const chat = await connection
+┊   ┊303┊        .createQueryBuilder(Chat, "chat")
+┊   ┊304┊        .whereInIds(chatId)
+┊   ┊305┊        .innerJoinAndSelect('chat.listingMembers', 'listingMembers')
+┊   ┊306┊        .innerJoinAndSelect('chat.messages', 'messages')
+┊   ┊307┊        .innerJoinAndSelect('messages.holders', 'holders')
+┊   ┊308┊        .getOne();
 ┊271┊309┊
 ┊272┊310┊      if (!chat) {
 ┊273┊311┊        throw new Error(`Cannot find chat ${chatId}.`);
 ┊274┊312┊      }
 ┊275┊313┊
-┊276┊   ┊      if (!chat.listingMemberIds.find(listingId => listingId === currentUser.id)) {
+┊   ┊314┊      if (!chat.listingMembers.find(user => user.id === currentUser.id)) {
 ┊277┊315┊        throw new Error(`The chat/group ${chatId} is not listed for the current user, so there is nothing to delete.`);
 ┊278┊316┊      }
 ┊279┊317┊
```
```diff
@@ -281,79 +319,166 @@
 ┊281┊319┊        throw new Error(`Cannot specify both 'all' and 'messageIds'.`);
 ┊282┊320┊      }
 ┊283┊321┊
+┊   ┊322┊      if (!all && !(messageIds && messageIds.length)) {
+┊   ┊323┊        throw new Error(`'all' and 'messageIds' cannot be both null`);
+┊   ┊324┊      }
+┊   ┊325┊
 ┊284┊326┊      let deletedIds: number[] = [];
-┊285┊   ┊      chats = chats.map(chat => {
-┊286┊   ┊        if (chat.id === Number(chatId)) {
-┊287┊   ┊          // Instead of chaining map and filter we can loop once using reduce
-┊288┊   ┊          const messages = chat.messages.reduce<Message[]>((filtered, message) => {
-┊289┊   ┊            if (all || messageIds!.includes(String(message.id))) {
-┊290┊   ┊              deletedIds.push(message.id);
-┊291┊   ┊              // Remove the current user from the message holders
-┊292┊   ┊              message.holderIds = message.holderIds.filter(holderId => holderId !== currentUser.id);
-┊293┊   ┊            }
+┊   ┊327┊      // Instead of chaining map and filter we can loop once using reduce
+┊   ┊328┊      chat.messages = await chat.messages.reduce<Promise<Message[]>>(async (filtered$, message) => {
+┊   ┊329┊        const filtered = await filtered$;
 ┊294┊330┊
-┊295┊   ┊            if (message.holderIds.length !== 0) {
-┊296┊   ┊              filtered.push(message);
-┊297┊   ┊            } // else discard the message
+┊   ┊331┊        if (all || messageIds!.includes(String(message.id))) {
+┊   ┊332┊          deletedIds.push(message.id);
+┊   ┊333┊          // Remove the current user from the message holders
+┊   ┊334┊          message.holders = message.holders.filter(user => user.id !== currentUser.id);
 ┊298┊335┊
-┊299┊   ┊            return filtered;
-┊300┊   ┊          }, []);
-┊301┊   ┊          chat = {...chat, messages};
 ┊302┊336┊        }
-┊303┊   ┊        return chat;
-┊304┊   ┊      });
+┊   ┊337┊
+┊   ┊338┊        if (message.holders.length !== 0) {
+┊   ┊339┊          // Remove the current user from the message holders
+┊   ┊340┊          await connection.getRepository(Message).save(message);
+┊   ┊341┊          filtered.push(message);
+┊   ┊342┊        } else {
+┊   ┊343┊          // Simply remove the message
+┊   ┊344┊          const recipients = await connection
+┊   ┊345┊            .createQueryBuilder(Recipient, "recipient")
+┊   ┊346┊            .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', {messageId: message.id})
+┊   ┊347┊            .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊348┊            .getMany();
+┊   ┊349┊          for (let recipient of recipients) {
+┊   ┊350┊            await connection.getRepository(Recipient).remove(recipient);
+┊   ┊351┊          }
+┊   ┊352┊          await connection.getRepository(Message).remove(message);
+┊   ┊353┊        }
+┊   ┊354┊
+┊   ┊355┊        return filtered;
+┊   ┊356┊      }, Promise.resolve([]));
+┊   ┊357┊
+┊   ┊358┊      await connection.getRepository(Chat).save(chat);
+┊   ┊359┊
 ┊305┊360┊      return deletedIds;
 ┊306┊361┊    },
 ┊307┊362┊  },
 ┊308┊363┊  Subscription: {
 ┊309┊364┊    messageAdded: {
 ┊310┊365┊      subscribe: withFilter(() => pubsub.asyncIterator('messageAdded'),
-┊311┊   ┊        ({messageAdded}: {messageAdded: Message & {chat: {id: number}}}, {chatId}: MessageAddedSubscriptionArgs, {user: currentUser}: { user: User }) => {
+┊   ┊366┊        ({messageAdded}: {messageAdded: Message}, {chatId}: MessageAddedSubscriptionArgs, {user: currentUser}: { user: User }) => {
 ┊312┊367┊          return (!chatId || messageAdded.chat.id === Number(chatId)) &&
-┊313┊   ┊            !!messageAdded.recipients.find((recipient: Recipient) => recipient.userId === currentUser.id);
+┊   ┊368┊            !!messageAdded.recipients.find((recipient: Recipient) => recipient.user.id === currentUser.id);
 ┊314┊369┊        }),
 ┊315┊370┊    },
 ┊316┊371┊    chatAdded: {
 ┊317┊372┊      subscribe: withFilter(() => pubsub.asyncIterator('chatAdded'),
-┊318┊   ┊        ({creatorId, chatAdded}: {creatorId: string, chatAdded: Chat}, variables: any, {user: currentUser}: { user: User }) => {
-┊319┊   ┊          return Number(creatorId) !== currentUser.id && !chatAdded.listingMemberIds.includes(currentUser.id);
+┊   ┊373┊        ({creatorId, chatAdded}: {creatorId: string, chatAdded: Chat}, variables, {user: currentUser}: { user: User }) => {
+┊   ┊374┊          return Number(creatorId) !== currentUser.id &&
+┊   ┊375┊            !!chatAdded.listingMembers.find((user: User) => user.id === currentUser.id);
 ┊320┊376┊        }),
 ┊321┊377┊    }
 ┊322┊378┊  },
 ┊323┊379┊  Chat: {
-┊324┊   ┊    name: (chat: Chat, args: any, {user: currentUser}: {user: User}): string => chat.name ? chat.name : users
-┊325┊   ┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.name,
-┊326┊   ┊    picture: (chat: Chat, args: any, {user: currentUser}: {user: User}) => chat.name ? chat.picture : users
-┊327┊   ┊      .find(user => user.id === chat.allTimeMemberIds.find(userId => userId !== currentUser.id))!.picture,
-┊328┊   ┊    allTimeMembers: (chat: Chat): User[] => users.filter(user => chat.allTimeMemberIds.includes(user.id)),
-┊329┊   ┊    listingMembers: (chat: Chat): User[] => users.filter(user => chat.listingMemberIds.includes(user.id)),
-┊330┊   ┊    actualGroupMembers: (chat: Chat): User[] => users.filter(user => chat.actualGroupMemberIds && chat.actualGroupMemberIds.includes(user.id)),
-┊331┊   ┊    admins: (chat: Chat): User[] => users.filter(user => chat.adminIds && chat.adminIds.includes(user.id)),
-┊332┊   ┊    owner: (chat: Chat): User | null => users.find(user => chat.ownerId === user.id) || null,
-┊333┊   ┊    messages: (chat: Chat, {amount = null}: {amount: number}, {user: currentUser}: {user: User}): Message[] => {
-┊334┊   ┊      const messages = chat.messages
-┊335┊   ┊      .filter(message => message.holderIds.includes(currentUser.id))
-┊336┊   ┊      .sort((a, b) => b.createdAt - a.createdAt) || <Message[]>[];
-┊337┊   ┊      return (amount ? messages.slice(0, amount) : messages).reverse();
+┊   ┊380┊    name: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<string | null> => {
+┊   ┊381┊      if (chat.name) {
+┊   ┊382┊        return chat.name;
+┊   ┊383┊      }
+┊   ┊384┊      const user = await connection
+┊   ┊385┊        .createQueryBuilder(User, "user")
+┊   ┊386┊        .where('user.id != :userId', {userId: currentUser.id})
+┊   ┊387┊        .innerJoin('user.allTimeMemberChats', 'allTimeMemberChats', 'allTimeMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊388┊        .getOne();
+┊   ┊389┊      return user && user.name || null;
+┊   ┊390┊    },
+┊   ┊391┊    picture: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<string | null> => {
+┊   ┊392┊      if (chat.name) {
+┊   ┊393┊        return chat.picture;
+┊   ┊394┊      }
+┊   ┊395┊      const user = await connection
+┊   ┊396┊        .createQueryBuilder(User, "user")
+┊   ┊397┊        .where('user.id != :userId', {userId: currentUser.id})
+┊   ┊398┊        .innerJoin('user.allTimeMemberChats', 'allTimeMemberChats', 'allTimeMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊399┊        .getOne();
+┊   ┊400┊      return user ? user.picture : null;
+┊   ┊401┊    },
+┊   ┊402┊    allTimeMembers: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<User[]> => {
+┊   ┊403┊      return await connection
+┊   ┊404┊        .createQueryBuilder(User, "user")
+┊   ┊405┊        .innerJoin('user.allTimeMemberChats', 'allTimeMemberChats', 'allTimeMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊406┊        .getMany();
+┊   ┊407┊    },
+┊   ┊408┊    listingMembers: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<User[]> => {
+┊   ┊409┊      return await connection
+┊   ┊410┊        .createQueryBuilder(User, "user")
+┊   ┊411┊        .innerJoin('user.listingMemberChats', 'listingMemberChats', 'listingMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊412┊        .getMany();
+┊   ┊413┊    },
+┊   ┊414┊    actualGroupMembers: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<User[]> => {
+┊   ┊415┊      return await connection
+┊   ┊416┊        .createQueryBuilder(User, "user")
+┊   ┊417┊        .innerJoin('user.actualGroupMemberChats', 'actualGroupMemberChats', 'actualGroupMemberChats.id = :chatId', {chatId: chat.id})
+┊   ┊418┊        .getMany();
+┊   ┊419┊    },
+┊   ┊420┊    admins: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<User[]> => {
+┊   ┊421┊      return await connection
+┊   ┊422┊        .createQueryBuilder(User, "user")
+┊   ┊423┊        .innerJoin('user.adminChats', 'adminChats', 'adminChats.id = :chatId', {chatId: chat.id})
+┊   ┊424┊        .getMany();
 ┊338┊425┊    },
-┊339┊   ┊    unreadMessages: (chat: Chat, args: any, {user: currentUser}: {user: User}): number => chat.messages
-┊340┊   ┊      .filter(message => message.holderIds.includes(currentUser.id) &&
-┊341┊   ┊        message.recipients.find(recipient => recipient.userId === currentUser.id && !recipient.readAt))
-┊342┊   ┊      .length,
-┊343┊   ┊    isGroup: (chat: Chat): boolean => !!chat.name,
+┊   ┊426┊    owner: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<User | null> => {
+┊   ┊427┊      return await connection
+┊   ┊428┊        .createQueryBuilder(User, "user")
+┊   ┊429┊        .innerJoin('user.ownerChats', 'ownerChats', 'ownerChats.id = :chatId', {chatId: chat.id})
+┊   ┊430┊        .getOne() || null;
+┊   ┊431┊    },
+┊   ┊432┊    messages: async (chat: Chat, {amount = null}: {amount: number}, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<Message[]> => {
+┊   ┊433┊      const query = connection
+┊   ┊434┊        .createQueryBuilder(Message, "message")
+┊   ┊435┊        .innerJoin('message.chat', 'chat', 'chat.id = :chatId', {chatId: chat.id})
+┊   ┊436┊        .innerJoin('message.holders', 'holders', 'holders.id = :userId', {userId: currentUser.id})
+┊   ┊437┊        .orderBy({"message.createdAt": "DESC"});
+┊   ┊438┊      return (amount ? await query.take(amount).getMany() : await query.getMany()).reverse();
+┊   ┊439┊    },
+┊   ┊440┊    unreadMessages: async (chat: Chat, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<number> => {
+┊   ┊441┊      return await connection
+┊   ┊442┊        .createQueryBuilder(Message, "message")
+┊   ┊443┊        .innerJoin('message.chat', 'chat', 'chat.id = :chatId', {chatId: chat.id})
+┊   ┊444┊        .innerJoin('message.recipients', 'recipients', 'recipients.user.id = :userId AND recipients.readAt IS NULL', {userId: currentUser.id})
+┊   ┊445┊        .getCount();
+┊   ┊446┊    },
+┊   ┊447┊    isGroup: (chat: Chat) => !!chat.name,
 ┊344┊448┊  },
 ┊345┊449┊  Message: {
-┊346┊   ┊    chat: (message: Message): Chat | null => chats.find(chat => message.chatId === chat.id) || null,
-┊347┊   ┊    sender: (message: Message): User | null => users.find(user => user.id === message.senderId) || null,
-┊348┊   ┊    holders: (message: Message): User[] => users.filter(user => message.holderIds.includes(user.id)),
-┊349┊   ┊    ownership: (message: Message, args: any, {user: currentUser}: {user: User}): boolean => message.senderId === currentUser.id,
-┊350┊   ┊  },
-┊351┊   ┊  Recipient: {
-┊352┊   ┊    user: (recipient: Recipient): User | null => users.find(user => recipient.userId === user.id) || null,
-┊353┊   ┊    message: (recipient: Recipient): Message | null => {
-┊354┊   ┊      const chat = chats.find(chat => recipient.chatId === chat.id);
-┊355┊   ┊      return chat ? chat.messages.find(message => recipient.messageId === message.id) || null : null;
+┊   ┊450┊    sender: async (message: Message, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<User | null> => {
+┊   ┊451┊      return (await connection
+┊   ┊452┊        .createQueryBuilder(User, "user")
+┊   ┊453┊        .innerJoin('user.senderMessages', 'senderMessages', 'senderMessages.id = :messageId', {messageId: message.id})
+┊   ┊454┊        .getOne()) || null;
+┊   ┊455┊    },
+┊   ┊456┊    ownership: async (message: Message, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<boolean> => {
+┊   ┊457┊      return !!(await connection
+┊   ┊458┊        .createQueryBuilder(User, "user")
+┊   ┊459┊        .whereInIds(currentUser.id)
+┊   ┊460┊        .innerJoin('user.senderMessages', 'senderMessages', 'senderMessages.id = :messageId', {messageId: message.id})
+┊   ┊461┊        .getCount());
+┊   ┊462┊    },
+┊   ┊463┊    recipients: async (message: Message, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<Recipient[]> => {
+┊   ┊464┊      return await connection
+┊   ┊465┊        .createQueryBuilder(Recipient, "recipient")
+┊   ┊466┊        .innerJoinAndSelect('recipient.message', 'message', 'message.id = :messageId', {messageId: message.id})
+┊   ┊467┊        .innerJoinAndSelect('recipient.user', 'user')
+┊   ┊468┊        .innerJoinAndSelect('recipient.chat', 'chat')
+┊   ┊469┊        .getMany();
+┊   ┊470┊    },
+┊   ┊471┊    holders: async (message: Message, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<User[]> => {
+┊   ┊472┊      return await connection
+┊   ┊473┊        .createQueryBuilder(User, "user")
+┊   ┊474┊        .innerJoin('user.holderMessages', 'holderMessages', 'holderMessages.id = :messageId', {messageId: message.id})
+┊   ┊475┊        .getMany();
+┊   ┊476┊    },
+┊   ┊477┊    chat: async (message: Message, args: any, {user: currentUser, connection}: {user: User, connection: Connection}): Promise<Chat | null> => {
+┊   ┊478┊      return (await connection
+┊   ┊479┊        .createQueryBuilder(Chat, "chat")
+┊   ┊480┊        .innerJoin('chat.messages', 'messages', 'messages.id = :messageId', {messageId: message.id})
+┊   ┊481┊        .getOne()) || null;
 ┊356┊482┊    },
-┊357┊   ┊    chat: (recipient: Recipient): Chat | null => chats.find(chat => recipient.chatId === chat.id) || null,
 ┊358┊483┊  },
 ┊359┊484┊};
```

[}]: #

`QueryBuilder` is one of the most powerful features of `TypeORM` - it allows you to build SQL queries using elegant and convenient syntax, execute them and get automatically transformed entities.

You can find more informations on `TypeORM` on http://typeorm.io

The best part is that you won't have to do anything on the client, everything will be completely transparent to it, even if migrated from NoSQL-like db structure to a relational one!
Of course, you could remove the custom normalization for the messages because now they have their own table and they are no longer embedded (so they have unique IDs), but we could leave it as well in order to be free to use any kind of backend.

[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step15.md) |
|:----------------------|

[}]: #
