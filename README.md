# Step-by-step NEAR Create-Read-Update-Delete (CRUD) dApp development from scratch

## Usage
Setup​ pre-requisites​:
1. npm
2. Near-cli
3. Make sure you have Node.js 14.16 installed, then use it to install yarn: npm install --global yarn (or just npm i -g yarn)

You need near-cli installed globally. Here's how:

'npm install --global near-cli'
This will give you the near CLI tool. Ensure that it's installed with:

'near --version'
In this tutorial we will be building a standard Create-Read-Update-Delete (CRUD) Smart Contract on the blockchain.

## Smart Contract
For this example you will be writing the smart contract in AssemblyScript which is similar to TypeScript and complies to WebAssembly.
We'll use the near-sdk-as library to help us write our contract allowing us to interact with the blockchain.

### AssemblyScript
In your terminal, create a new directory for your smart contract and inside the newly created directory, initialize an AssemblyScript application:
```ts
mkdir todos-crud-contract && cd todos-crud-contract
yarn init -y
yarn add @assemblyscript/loader@latest assemblyscript@latest asbuild near-cli near-sdk-as
yarn asinit .
```
### near-sdk-as
```ts
Replace the asconfig.json file with:{
  "extends": "near-sdk-as/asconfig.json"
}
```
Then create an assembly/as_types.d.ts file with:
```ts
/// <reference types="near-sdk-as/assembly/as_types" />
```
### Data Storage
With NEAR we can conveniently store information on the blockchain by using one of the SDK-provided collections. These collections will take the place of a traditional database for us and can be thought of like database tables.

In our todo application we'll use a collection inside of our model code to persist data to the blockchain.

In particular, our todo application will want to lookup a todo by its id and iterate through our todos to get paginated results. The PersistentUnorderedMap is perfect for this. It gives us the ability to lookup by key with the get and getSome methods and allows us to iterate through all the values with the values method.

To properly separate concerns we are going to create a model.ts file to handle all of our data persistence. In that file we are going to create our PersistentUnorderedMap and a Todo model class.

```ts
// assembly/model.ts
import { PersistentUnorderedMap, math } from "near-sdk-as";

// Think of this PersistentUnorderedMap like a database table.
// We'll use this to persist and retrieve data.
export const todos = new PersistentUnorderedMap<u32, Todo>("todos");

// Think of this like a model class in something like mongoose or
// sequelize. It defines the shape or schema for our data. It will
// also contain static methods to read and write data from and to
// the todos PersistentUnorderedMap.
@nearBindgen
export class Todo {
  id: u32;
  task: string;
  done: bool;

  constructor(task: string) {
    this.id = math.hash32<string>(task);
    this.task = task;
    this.done = false;
  }
}
```

## C - Create
To start off we'll need to create new todos and store those todos on the blockchain. In web2 this would often mean creating an HTTP POST endpoint. In web3, however, we'll be creating a smart contract method.
### Model
In order to store our todo in the todos PersistentUnorderedMap we are going to add a static insert method to our Todo class. This method will be responsible for persisting a todo into the todos PersistentUnorderedMap.

```ts
// contract/assembly/model.ts
import { PersistentUnorderedMap, math } from "near-sdk-as";

export const todos = new PersistentUnorderedMap<u32, Todo>("todos");

@nearBindgen
export class Todo {
  id: u32;
  task: string;
  done: bool;

  constructor(task: string) {
    this.id = math.hash32<string>(task);
    this.task = task;
    this.done = false;
  }
// ADD CODE BELOW
  static insert(task: string): Todo {
    // create a new Todo
    const todo = new Todo(task);

    // add the todo to the PersistentUnorderedMap
    // where the key is the todo's id and the value
    // is the todo itself. Think of this like an
    // INSERT statement in SQL.
    todos.set(todo.id, todo);

    return todo;
  }
}
```

### Smart Contract Method
Smart contract methods act like endpoints that our web app will be able to call. These methods define the public interface for our smart contract. Here we define the create method which uses the Todo model to persist a new todo to the blockchain.
```ts
// contract/assembly/index.ts
import { Todo } from "./model";

// export the create method. This acts like an endpoint
// that we'll be able to call from our web app.
export function create(task: string): Todo {
  // use the Todo class to persist the todo data
  return Todo.insert(task);
}
```

### Deploy and Test
We can build our smart contract and deploy it to a development account.

Then add a few scripts to your package.json:
```ts
"scripts": {
  "build:release": "asb",
  "deploy": "near dev-deploy build/release/todos-crud-contract.wasm",
  "dev": "yarn build:release && yarn deploy",
  "test": "asp",
}
```
The build step will compile the AssemblyScript code we wrote above to WebAssembly. Then the deploy step will send and store the WebAssembly file to the blockchain.
```ts
yarn build:release
yarn deploy
```
Copy your Contract id(for ex:dev-1651447733571-12325797478508).Export it so you do not have to copy and paste it while calling contract methods:
```ts
export CONTRACT=YOUR-CONTRACT-ID 
```
And finally we can test our deployed smart contract:
```ts
near call $CONTRACT create '{"task":"Drink water"}' --accountId YOUR_ACCOUNT_ID.testnet
```

## R - Read by id
Now that we've created a todo, let's retrieve the todo using a getById method. In web2 this functionality might be accomplished with an express endpoint like this:
```ts
app.get('/todos/:id', async(req, res) => {
  // Find a todo by its id. Maybe using a SQL query like:
  // SELECT * FROM todos WHERE id=?
  const todo = await Todo.findById(req.params.id);
  res.send(todo);
});
```
### Model
In order to get our todos we'll add a static findById method that will get a todo from the todos PersistentUnorderedMap using the getSome method.
```ts
In order to get our todos we'll add a static findById method that will get a todo from the todos PersistentUnorderedMap using the getSome method.// contract/assembly/model.ts
import { PersistentUnorderedMap, math } from "near-sdk-as";

export const todos = new PersistentUnorderedMap<u32, Todo>("todos");

@nearBindgen
export class Todo {
  id: u32;
  task: string;
  done: bool;

  constructor(task: string) {
    this.id = math.hash32<string>(task);
    this.task = task;
    this.done = false;
  }

  static insert(task: string): Todo {...}



  // ADD THE CODE BELOW
  static findById(id: u32): Todo {
    // Lookup a todo in the PersistentUnorderedMap by its id.
    // This is like a SELECT * FROM todos WHERE id=?
    return todos.getSome(id);
  }
}
```

### Smart Contract Method
Now that we have a model method that will find a todo by id, we can continue to define our smart contracts public interface by defining and exporting a getById method.
```ts
// contract/assembly/index.ts
import { Todo } from "./model";

export function create(task: string): Todo {...}

export function getById(id: u32): Todo {
  return Todo.findById(id);
}
```

### Deploy and Test
Now that the getById method is finished we can build our smart contract and deploy it to a development account.
```ts
yarn dev
```
And finally we can test our deployed smart contract.
Replace SOME_ID_HERE with the id that was logged when we used the create method previously or call the create method again and copy the id returned. (for ex: id: 2342859876)

```ts
near view $CONTRACT getById '{"id":SOME_ID_HERE}' --accountId YOUR_ACCOUNT_ID.testnet
```

## R - Read List
Next we'll want to get a paged list of results back from our smart contract. We don't want to return all todos (there may be too many). Instead, we want to return a subset of todos. To do this we'll use the offset (how many to skip) and limit (how many to get) patterns.

In web2 this may be accomplished with an express endpoint like this:
```ts
app.get('/todos', async(req, res) => {
  // SELECT * FROM todos LIMIT ? OFFSET ?
  const todos = Todo.find(req.query.offset, req.query.limit);
  res.send(todos);
})
```
### Model
```ts
// contract/assembly/model.ts
import { PersistentUnorderedMap, math } from "near-sdk-as";

export const todos = new PersistentUnorderedMap<u32, Todo>("todos");

@nearBindgen
export class Todo {
  id: u32;
  task: string;
  done: bool;

  constructor(task: string) {
    this.id = math.hash32<string>(task);
    this.task = task;
    this.done = false;
  }

  static insert(task: string): Todo {...}

  static findById(id: u32): Todo {...}



  // ADD THE CODE BELOW
  static find(offset: u32, limit: u32): Todo[] {
    // the PersistentUnorderedMap values method will
    // takes two parameters: start and end. we'll start
    // at the offset (skipping all todos before the offset)
    // and collect all todos until we reach the offset + limit
    // todo. For example, if offset is 10 and limit is 3 then
    // this would return the 10th, 11th, and 12th todo.
    return todos.values(offset, offset + limit);
  }
}```
### Smart Contract Method

```ts
// contract/assembly/index.ts
import { Todo } from "./model";

export function create(task: string): Todo {...}

export function getById(id: u32): Todo {...}

export function get(offset: u32, limit: u32 = 10): Todo[] {
  return Todo.find(offset, limit);
}
```
### Deploy and Test
Now that the get method is finished we can build our smart contract and deploy it to a development account.
```ts
yarn dev
```
And finally we can test our deployed smart contract:
```ts
near view $CONTRACT get '{"offset":0}' --accountId YOUR_ACCOUNT_ID.testnet
```

## U - Update
Now that we've created a todo, let's update it using an update method.
### Model
In order to update our todos we'll add a static findByIdAndUpdate method:
```ts
// contract/assembly/model.ts
import { PersistentUnorderedMap, math } from "near-sdk-as";

export const todos = new PersistentUnorderedMap<u32, Todo>("todos");
// PartialTodo class
@nearBindgen
export class PartialTodo {
 task: string;
 done: bool;
}

@nearBindgen
export class Todo {
  id: u32;
  task: string;
  done: bool;

  constructor(task: string) {
    this.id = math.hash32<string>(task);
    this.task = task;
    this.done = false;
  }

  static insert(task: string): Todo {...}

  static findById(id: u32): Todo {...}

  static find(offset: u32, limit: u32): Todo[] {...}

  // ADD CODE BELOW. DO NOT FORGET TO ADD THE CLASS PartialTodo ABOVE
  static findByIdAndUpdate(id: u32, partial: PartialTodo): Todo {
    // find a todo by its id
    const todo = this.findById(id);

    // update the todo in-memory
    todo.task = partial.task;
    todo.done = partial.done;

    // persist the updated todo
    todos.set(id, todo);

    return todo;
```
### Smart Contract Method
Now that we have a model method, we can continue to define our smart contract's public interface by defining an update function.
```ts
// contract/assembly/index.ts
import { Todo, PartialTodo } from "./model";

export function create(task: string): Todo {...}

export function getById(id: u32): Todo {...}

export function get(offset: u32, limit: u32 = 10): Todo[] {...}

export function update(id: u32, updates: PartialTodo): Todo {
  return Todo.findByIdAndUpdate(id, updates);
}
```
### Deploy and Test
Now that the update method is finished we can build our smart contract and deploy it to a development account.
```ts
yarn dev
```
And finally we can test our deployed smart contract.Replace SOME_ID_HERE with the id that was logged when we used the create method previously or call the create method again and copy the id returned. 

```ts
near call $CONTRACT update '{"id":SOME_ID_HERE, "updates":{"done":true, "task":"Drink nothing"} }' --accountId YOUR_ACCOUNT_ID.testnet
```
## D - Delete
Last but not least, let's delete a todo using a del method.
### Model
In order to delete our todos we'll add a static findByIdAndDelete method:
```ts
// contract/assembly/model.ts
import { PersistentUnorderedMap, math } from "near-sdk-as";

export const todos = new PersistentUnorderedMap<u32, Todo>("todos");

@nearBindgen
export class Todo {
  id: u32;
  task: string;
  done: bool;

  constructor(task: string) {
    this.id = math.hash32<string>(task);
    this.task = task;
    this.done = false;
  }

  static insert(task: string): Todo {...}

  static findById(id: u32): Todo {...}

  static find(offset: u32, limit: u32): Todo[] {...}

  static findByIdAndUpdate(id: u32, partial: PartialTodo): Todo {...}

  static findByIdAndDelete(id: u32): void {
    todos.delete(id);
  }
}
```
### Smart Contract Method
Now that we have a model method, we can continue to define our smart contract's public interface by defining a del function.
```ts
// contract/assembly/index.ts
import { Todo } from "./model";

export function create(task: string): Todo {...}

export function getById(id: u32): Todo {...}

export function get(offset: u32, limit: u32 = 10): Todo[] {...}

export function update(id: u32, updates: PartialTodo): Todo {...}

export function del(id: u32): void {
  Todo.findByIdAndDelete(id);
}
```
### Deploy and Test
We can build our smart contract and deploy it to a development account.
```ts
yarn dev
```
And finally, we can test our deployed smart contract:
```ts
near call $CONTRACT del '{"id":SOME_ID_HERE }' --accountId YOUR_ACCOUNT_ID.testnet
```