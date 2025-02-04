# Getting started
To just play around with this project, clone in and open it in your favority editor. 
Below the details of the project setup are explained.

## Requirements
Peerbit can only now easily be consumed with Typescript projects that are ESM compatible. You also need to turn on 

```sh
"experimentalDecorators" : true
```
in the `compilerOptions` in tsconfig.json

Check this [tsconfig.json](./tsconfig.json) for reference

## Installation
Following dependencies are needed (as of now)

```sh
"dependencies": {
	"@dao-xyz/peerbit": "^0.0.103",
	"@libp2p/websockets": "^5.0.3",
	"@chainsafe/libp2p-noise": "^10.2.0",
	"@chainsafe/libp2p-gossipsub": "^5.0.0"
}
```

See this project [package.json](./package.json) for reference

## Starting the client 
```typescript 
import { Peerbit } from "@dao-xyz/peerbit";
import { createLibp2p } from 'libp2p'
import { noise } from '@chainsafe/libp2p-noise'
import { gossipsub } from '@chainsafe/libp2p-gossipsub'

const libp2pInstance = await createLibp2p({
			transports: [webSockets()],
			connectionEncryption: [noise()], // Make connections encrypted
			pubsub: gossipsub()  // required in this version of Peerbit, but will not in the future
}); // readmore about configurations at https://github.com/libp2p/js-libp2p

const client = await Peerbit.create(libp2pInstance)
```
This will start a client where all data is stored in RAM, if you want to create a client where data persists after stop (so you can restart), you can provide a directory path to the options: 
```typescript
const client = await Peerbit.create(libp2pInstance, { directory: "./some-folder" })
```

## Your first database
Defining a database is done by extending ```Program```

```typescript

import { Program } from "@dao-xyz/peerbit-program";
import { variant } from "@dao-xyz/borsh"; // Serialization library


// The line below will make sure that every time the database manifest
// gets seriaized, "document-store" will prefix the serialized bytes (in UTF-8 encoding) so that peers
// who open the database (who recieve the database manifest in serialized bytes) can decode into this particular class. 
@variant("document-store") 
class MyDatabase extends Program {
    async setup() {
        // this will be invoked on startup
	console.log("Setting up!")
    }
}

```

This database can't do anything as of now. It just supports to be opened and closed.

Creating a new database from the definition
```typescript
const db = await client.open(new MyDatabase())
const address = db.address.toString(); // Prints the address
```

Opening an adatabase from an address
```typescript 
const db = await client.open(address)
```


Now let's add support to store some documents


```typescript
import { variant, field } from "@dao-xyz/borsh"; // Serialization library
import { nanoid } from 'nanoid';


// Abstract document definition we can create many kinds of document types from
export class BaseDocument { }


// For serialization, we version our documents with a single byte, so in the future, if we want to upgrade our document definition
// we can do @variant(1) which allows us to store and manage both Documents of type variant 0 and variant 1 in our Document database
// (This is not needed but recommended)
@variant(0)
export class TextDocument extends BaseDocument {

	// We instruct the serializer that this is a string field and shall be serialized when we save documents
	@field({ type: 'string' })
	id: string 


	// We instruct the serializer that this is a string field and shall be serialized when we save documents
	@field({ type: 'string' }) 
	text: string

	constructor(text: string) {
		super()

		// Documents need unique ids
		this.id = nanoid()
		this.text = text;
	}
}
```
and update the Database program definition

```typescript
import { field, variant } from "@dao-xyz/borsh";
import { Program } from "@dao-xyz/peerbit-program";
import { Documents, DocumentIndex } from "@dao-xyz/peerbit-document";

@variant("my-database")
export class MyDatabase extends Program {

	// Tell the serializer that this field is of type Documents, so when other peers open this database they know exactly how to intepret this field
	@field({ type: Documents })
	documents: Documents<BaseDocument>

	constructor() {
		super()
		this.documents = new Documents({ index: new DocumentIndex({ indexBy: 'id' }) }) // Construct the document store
	}

	async setup() {
		// this will be invoked on startup
		await this.documents.setup({ type: BaseDocument }) // Call the setup function on this.documents 
	}
}

```

See [this](./src/index.ts) for a completed code

As you might have seen the @field, @variant decorators are used on multiple places. They indicate how the client should serialize/deserialize the databases and content you create. Peerbit uses a serialization format by default called Borsh, it provides similiar functionality to Protobuf but is simpler to understand and the implementation in Typescript requires less code than a Protobuf equivalent (and is faster). There are many reasons for not using JSON as a serialization format. Mainly, JSON serialization does not enforce determinism when serializing, different peers might get different serialized bytes if they would use `JSON.stringify(...)` as a result of this, the hashes and therefore the content addresses of the things you would create with Peerbit would be ambigious. To circumvent this one could use deterministic JSON parsers, but then you would pay a great performance cost for that. Furthermore, serialization formats like Borsh and Protobuf comes with built-in typechecking which mean you have to write a lot less validation and testing code for you applications.   

In the future there might be support for more kinds of serialization formats, like protobuf and JSON (if you want to use it for other reasons), but for now you would have to stick with Borsh. 

For more info about how you can serialize things, [see this](https://github.com/dao-xyz/borsh-ts). 


## Adding the first documents
```typescript 
const db = await client.open(new MyDatabase())

for (let i = 0; i < 100; i++) {
	await db.documents.put(new TextDocument("text" + i))
}

let foundResults: Results<any> | undefined = undefined;
await db.documents.index.query(new DocumentQueryRequest({ queries: [] }), (results, from) => {

	foundResults = results
}, { local: true, remote: false }) // Only search locally

console.log(foundResults.result.length) // 100
console.log("First document:", (foundResults.results[0].value as TextDocument).text)
```

See the [tests](./src/index.test.ts) for working examples
