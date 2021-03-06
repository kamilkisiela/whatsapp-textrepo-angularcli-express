# Step 10: Updating the store

[//]: # (head-end)


## Client

Did you notice that after creating a new message you'll have to refresh the page in order to see it?

How to fix that?

Apollo performs two important core tasks which we covered in one of previous steps: executing queries and mutations, and caching the results.

Thanks to Apollo's store design, it's possible for the results of a query or mutation to update your UI in all the right places. In many cases it's possible for that to happen automatically, whereas in others you need to help the client out a little in doing so.

There are couple of APIs to update the cache after mutation:

- `refetchQueries` - is the most straightforward way of updating the cache. You request queries to be executed once again.
- `update` - super flexible and allows you to make changes to the store based on mutation. Works well with Optimistic UI which we will cover later.
- `updateQueries` - It works similar to the previous option but the API may be deprecated in the future and we recommend to stay with `update`.

If you thought about re-querying the server you would be wrong! The best solution is to use the response provided by the server to update our Apollo local cache.

Of course that we're going to use the `update` API!

### Updating the store

As you can see, the `update` option is a function that has two arguments:

- `DataProxy` which we named `store`
- mutation's result

The `DataProxy` seems very interesting! It's a middleman between you and the cache.

#### How to update fragments

In pretty much every case, that middleman would allow to write and read data. Take for example the JavaScript `Map` object.

It has a `get` and `set` methods and both works based on a `key`.

In Apollo, we do have that key, it's something that was covered in the 5th step, remember? An object of type `Message` with an `id` would be stored in the cache as `Message:<id>`.

Let's call that small object a fragment, for simplicity.

What are our _get_ and _set_ methods in Apollo's DataProxy? Simple, `readFragment` and `writeFragment`.

I think it's time to work on a simple example. Scenario: every message can be liked and we keep the sum of likes under the `likes` field.

How to give a like?

```typescript
const store: DataProxy = ...;
const messageId = 256;

const fragment = gql`
  fragment M on Message {
    likes
  }
`;
const fragmentId = `Message:${messageId}`;

const message = store.readFragment({
  fragment,
  id: fragmentId
});

store.writeFragment({
  fragment,
  id: fragmentId,
  data: {
    likes: message.likes + 1
  }
});
```

What just happened?!

At first, it might seem weird to talk to the store like that but you will see that it makes a lot of sense!

 - `messageId` - defined the actual id of our message
 - `fragment` - it's a document that tells Apollo what fields we want to read and write
 - `fragmentId` - an unique key under which the fragment is stored (we covered that in 5th step)
 - `readFragment` - reads the fragment in the store and returns the result in exact same shape as requested
 - `writeFragment` - accepts same properties as `readFragment` but also the `data` property

### How to update queries

Similar as in case of fragments, we update queries with `readQuery` and `writeQuery`.

We won't cover the same thing again but let's just look at the API based on our WhatsApp clone example:

[{]: <helper> (diffStep "6.1" module="client")

#### Step 6.1: Update the store

##### Changed src&#x2F;app&#x2F;services&#x2F;chats.service.ts
```diff
@@ -1,6 +1,13 @@
 ┊ 1┊ 1┊import {map} from 'rxjs/operators';
 ┊ 2┊ 2┊import {Injectable} from '@angular/core';
-┊ 3┊  ┊import {GetChatsGQL, GetChatGQL, AddMessageGQL} from '../../graphql';
+┊  ┊ 3┊import {
+┊  ┊ 4┊  GetChatsGQL,
+┊  ┊ 5┊  GetChatGQL,
+┊  ┊ 6┊  AddMessageGQL,
+┊  ┊ 7┊  AddMessage,
+┊  ┊ 8┊  GetChats,
+┊  ┊ 9┊  GetChat
+┊  ┊10┊} from '../../graphql';
 ┊ 4┊11┊
 ┊ 5┊12┊@Injectable()
 ┊ 6┊13┊export class ChatsService {
```
```diff
@@ -39,6 +46,50 @@
 ┊39┊46┊    return this.addMessageGQL.mutate({
 ┊40┊47┊      chatId,
 ┊41┊48┊      content,
+┊  ┊49┊    }, {
+┊  ┊50┊      update: (store, { data: { addMessage } }: {data: AddMessage.Mutation}) => {
+┊  ┊51┊        // Update the messages cache
+┊  ┊52┊        {
+┊  ┊53┊          // Read the data from our cache for this query.
+┊  ┊54┊          const {chat} = store.readQuery<GetChat.Query, GetChat.Variables>({
+┊  ┊55┊            query: this.getChatGQL.document,
+┊  ┊56┊            variables: {
+┊  ┊57┊              chatId,
+┊  ┊58┊            }
+┊  ┊59┊          });
+┊  ┊60┊          // Add our message from the mutation to the end.
+┊  ┊61┊          chat.messages.push(addMessage);
+┊  ┊62┊          // Write our data back to the cache.
+┊  ┊63┊          store.writeQuery({
+┊  ┊64┊            query: this.getChatGQL.document,
+┊  ┊65┊            data: {
+┊  ┊66┊              chat
+┊  ┊67┊            }
+┊  ┊68┊          });
+┊  ┊69┊        }
+┊  ┊70┊        // Update last message cache
+┊  ┊71┊        {
+┊  ┊72┊          // Read the data from our cache for this query.
+┊  ┊73┊          const {chats} = store.readQuery<GetChats.Query, GetChats.Variables>({
+┊  ┊74┊            query: this.getChatsGQL.document,
+┊  ┊75┊            variables: {
+┊  ┊76┊              amount: this.messagesAmount,
+┊  ┊77┊            },
+┊  ┊78┊          });
+┊  ┊79┊          // Add our comment from the mutation to the end.
+┊  ┊80┊          chats.find(chat => chat.id === chatId).messages.push(addMessage);
+┊  ┊81┊          // Write our data back to the cache.
+┊  ┊82┊          store.writeQuery<GetChats.Query, GetChats.Variables>({
+┊  ┊83┊            query: this.getChatsGQL.document,
+┊  ┊84┊            variables: {
+┊  ┊85┊              amount: this.messagesAmount,
+┊  ┊86┊            },
+┊  ┊87┊            data: {
+┊  ┊88┊              chats,
+┊  ┊89┊            },
+┊  ┊90┊          });
+┊  ┊91┊        }
+┊  ┊92┊      }
 ┊42┊93┊    });
 ┊43┊94┊  }
 ┊44┊95┊}
```

[}]: #

As you can see we used the `document` property of a generated GQL service. It contains the actual query that is used in every `watch` or `fetch` calls of the GQL service. It's a part of the open API.

### Keep on mind

It's very important to know few things:

- update function have to run synchronously
- the query should always match the query you want to update (even order of fields or variables matters)
- by updating a query you also update every normalized data it has (we called them fragments in few secions above)

When Apollo runs the `update` function? In two cases, rigth after the mutation happens but only if it has an optimistic response defined, and also once we get the response from the server.

> Optimistic Resposne is something we will cover in next steps but so you know, Apollo allows to optimistically update the store with a temporary data that is being replaced right after we get the response from the server.

But why the update function has to run synchornously? It's by design and not so interesting. We won't cover it in-depth here but in short, when Apollo runs the function it switches the store with the new, empty one, to record every change that's made by the function to later merge it with the original store. If you still have questions, please let us know so we can cover that thing in-depth as a blog post or even a separate chapter in that tutorial.

### Summary

Now you won't need to reload the page in order to see the new message. What's even more interesting is that the message you wrote would also be shown as the last message in the chats list, just hit the back button in the top-left corner to find out!

This is because we updated our store for both the `GetChat` and the `GetChats` query.


[//]: # (foot-start)

[{]: <helper> (navStep)

| [< Previous Step](step9.md) | [Next Step >](step11.md) |
|:--------------------------------|--------------------------------:|

[}]: #
