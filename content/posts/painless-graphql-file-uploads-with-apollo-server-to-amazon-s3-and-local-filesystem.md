---
title: "Painless GraphQL File Uploads With Apollo Server to Amazon S3 and Local Filesystem"
date: 2020-08-03T03:05:09-03:00
draft: false
tags:
 - GraphQL
 - Apollo
 - Node
 - AWS
 - S3
description: "This tutorial-like article will demonstrate how to handle file uploads on Apollo Server and stream them to Amazon S3 or, optionally (but not preferably), to your server's filesystem."
---

This tutorial-like article will demonstrate how to handle file uploads on Apollo Server and stream them to Amazon S3 or, optionally (but not preferably), to your server's filesystem.

Before we go on, I assume you have basic familiarity with S3 and already have read on [this subject on the Apollo docs](https://www.apollographql.com/docs/apollo-server/data/file-uploads/).

**NOTE**: for the sake of simplicity, I have kept things to a very minimum (most of the time). You are encouraged to extract from this article what is most relevant to your project and adapt it the way you see fit.

## A walk through the file structure

```
├── .env
├── tmp/
|
├── bin/
│   │
│   └── upload-avatar.sh
|
└── src/
    │
    ├── config.js
    ├── uploaders.js
    ├── typedefs.js
    ├── resolvers.js
    ├── server.js
    ├── s3.js
    ├── index.js
    |
    └── lib/
        │
        └── gql-uploaders.js
```

- **.env** - the dotenv file where we'll be keeping our Amazon credentials and other useful environment variables.
- **src/lib/gql-uploaders** - our abstractions for the uploader functionality;
- **src/config.js** - loads the .env file and exports its variables in an application friendly format.
- **src/server.js** - where we'll configure our GraphQL server.
- **src/resolvers.js** - GraphQL resolvers.
- **src/typedefs.js** - GraphQL type definitions.
- **src/index.js** - the application entry point.
- **src/uploaders.js** - instances of our uploader abstractions.
- **src/s3.js** - exports our configured AWS.S3 instance.
- **bin/upload-avatar.sh** - a shell utility to allow us to manually test file uploads.
- **tmp/** - a temporary directory to store uploaded files.

## Installing the dependencies

Assuming you already have a package.json in place and have already laid out the file structure, you should now install the following dependencies (I'll use yarn for this, but you sure can do this with the npm command as well):

```
yarn add apollo-server graphql dotenv uuid aws-sdk
```

We'll be using `apollo-server` and `graphql` to power our graphql server, `dotenv` to load are environment variables, `aws-sdk` to handle uploads to Amazon S3 cloud and the `uuid` module to generate random file names.

## Getting to know how Apollo Server handle uploads

We'll start by coding our graphql type definitions.

```javascript
// src/typedefs.js -- revision 1

const { gql } = require('apollo-server');

module.exports = gql`
  type File {
    uri: String!
    filename: String!
    mimetype: String!
    encoding: String!
  }

  type Query {
    uploads: [File]
  }

  type Mutation {
    uploadAvatar(file: Upload!): File
  }
`;

```

As an example, we'll be uploading a user avatar pic. That's what our mutation `uploadAvatar` will be doing. It will return a `File` type, which is essentially just an object with the `uri` for the stored file and some less useful properties. We actually won't be working with the query `uploads` in this tutorial, but GraphQL demands we have a non-empty root Query type, and that is why we have it there. Just ignore it, please.

Our `uploadAvatar` mutation has just one parameter (`file`) of type `Upload`. Our resolver will receive a promise that resolves to an object containing the following properties:

- `filename` - a string representing the name of the uploaded file, such as `my-pic-at-the-beach-20200120.jpg`;
- `mimetype` - a string representing the MIME type of the uploaded file, such as `image/jpeg`;
- `encoding` - a string representing the file encoding, such as `7bit`;
- `createReadStream` - a function that initiates a binary read stream (In previous Apollo implementations, we were given a `stream` object instead of the function to create it).

If you've never worked with Node streams before, you can check out [Node's stream API](https://nodejs.org/api/stream.html). But don't be intimidated, as you'll soon see, will make plain simple usage of it.

```javascript
// src/resolvers.js -- first revision

module.exports = {
  Mutation: {
    uploadAvatar: async (_, { file }) => {
      const { createReadStream, filename, mimetype, encoding } = await file;

      return {
        filename,
        mimetype,
        encoding,
        uri: 'http://about:blank',
      };
    },
  },
};

```

So, in this first take we are simply returning the file attributes (with a placeholder for the uri value). We'll come back to it soon to effectively upload the file.

Now, let's set up our server:

```javascript
// src/server.js -- final revision

const { ApolloServer } = require('apollo-server');
const typeDefs = require('./typedefs');
const resolvers = require('./resolvers');

module.exports = new ApolloServer({
  typeDefs,
  resolvers,
});

```

And put it to work:

```javascript
// src/index.js -- final revision

const server = require('./server');

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`);
});

```

Alright. Now it's time to have a taste of it. We'll issue a file upload to our server and see it in action. Since we'll need to test the file upload more than one time, we'll create a shell script to send the request for us (You'll probably need to allow it to execute: `chmod +x ./bin/upload-avatar.sh`).

```shell
#!/bin/sh

# bin/upload-avatar.sh -- final revision

curl $1 \
  -F operations='{ "query": "mutation ($file: Upload!) { uploadAvatar(file: $file) { uri filename mimetype encoding } }", "variables": { "file": null } }' \
  -F map='{ "0": ["variables.file"] }' \
  -F 0=@$2

```

If this script seems a little cryptic to you (It sure seemed to me), don't worry. Explaining the details of it is beyond the scope of this tutorial, but I intend to write an article about making a javascript upload client soon. Meantime, if want you can find more information about the inner workings of it [here](https://github.com/jaydenseric/graphql-multipart-request-spec).

The script receives the server URI as the first argument and the file path as second. I'll be uploading a very sexy picture of me (which you won't have the pleasure to see) named sexy-me.jpg to my local server running on port 4000 (don't forget to start your server: `node src/index.js`):

```
./bin/upload-avatar.sh localhost:4000 ~/Pictures/sexy-me.jpg
```

And here is the formatted JSON response:

```json
{
  "data": {
    "uploadAvatar": {
      "uri": "http://about:blank",
      "filename": "sexy-me.jpg",
      "mimetype": "image/jpeg",
      "encoding": "7bit"
    }
  }
}

```

TIP: you can use the 'jq' utility to format the JSON response. Install jq and pipe the response to it like `./bin/upload-avatar.sh localhost:4000 ~/Pictures/sexy-me.jpg | jq`.

## Uploading files to Amazon S3

Looking good. Now, let's configure our S3 instance.

```shell
# .env -- final revision

AWS_ACCESS_KEY_ID=
AWS_ACCESS_KEY_SECRET=
AWS_S3_REGION=us-east-2
AWS_S3_BUCKET=acme-evil-labs

```

It's up to you to provide values for these variables, of course.

Our config module will look like this:

```javascript
// src/config.js -- final revision

require('dotenv').config()

module.exports = {
  s3: {
    credentials: {
      accessKeyId: process.env.AWS_ACCESS_KEY_ID,
      secretAccessKey: process.env.AWS_ACCESS_KEY_SECRET,
    },
    region: process.env.AWS_S3_REGION,
    params: {
      ACL: 'public-read',
      Bucket: process.env.AWS_S3_BUCKET,
    },
  },
  app: {
    storageDir: 'tmp',
  },
};

```

Let's configure our S3 instance:

```javascript
// src/s3.js -- final revision

const AWS = require('aws-sdk');
const config = require('./config');

module.exports = new AWS.S3(config.s3);

```

Now it's time to revisit our resolver and actually make the upload to S3:

```javascript
// src/resolvers.js -- second revision

const { extname } = require('path');
const { v4: uuid } = require('uuid'); // (A)
const s3 = require('./s3'); // (B)

module.exports = {
  Mutation: {
    uploadAvatar: async (_, { file }) => {
      const { createReadStream, filename, mimetype, encoding } = await file;

      const { Location } = await s3.upload({ // (C)
        Body: createReadStream(),               
        Key: `${uuid()}${extname(filename)}`,  
        ContentType: mimetype                   
      }).promise();                             

      return {
        filename,
        mimetype,
        encoding,
        uri: Location, // (D)
      }; 
    },
  },
};

```

Here is what is happening:

- **(A)**: we import the UUID/V4 function (as uuid) to generate our random UUIDs.
- **(B)**: we import our configured S3 instance.
- **(C)**: we call the `upload` function passing to it a readable stream object (created by calling `createReadStream`) as the `Body` parameter; the random UUID string suffixed with the filename as the `Key` parameter; and the mimetype as the `ContentType` parameter. `upload` is an asynchronous function that expects a callback, but we may return a promise from it by calling the `promise` method on it (in JavaScript, functions are objects too). When the promise is resolved, we destructure the resolved object to extract the `Location` property (`Location` is the URI from where we can download the uploaded file).
- **(D)**: we set `uri` to `Location`.

You can find it more info about the `upload` function [here](https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/S3.html#upload-property).

We can now call our shell script again `./bin/upload-avatar.sh localhost:4000 ~/Pictures/sexy-me.jpg` to see the result:

```json
{
  "data": {
    "uploadAvatar": {
      "uri": "https://acme-evil-labs.s3.us-east-2.amazonaws.com/c3127c4c-e4f9-4e79-b3d1-08e2cbb7ad5d.jpg",
      "filename": "sexy-me.jpg",
      "mimetype": "image/jpeg",
      "encoding": "7bit"
    }
  }
}

```

Notice that the URI now points to the Amazon cloud. We can save that URI in our database and have it served to our front-end application. Additionally, we can copy and paste the URI (not the one of this example, though) in the browser and see the file we just uploaded (if our S3 access policy configuration allows it).

That sure gets the work done, but if we want to reuse that functionality in other resolvers and present our colleagues with a nice and easy-to-use feature, we must abstract that functionality. To do so, we'll be creating two uploaders with the same interface: one of them will upload files to Amazon S3 (`S3Uploader`) and the other will save the files in the local hard drive (`FilesystemUploader`). There are few use cases for uploading files directly to the server drive nowadays, but it might be handy at some point during development. Then we'll see that we can can exchange one implementation for another seamlessly.

## Building abstractions

Let's start with the `S3Uploader` class:

```javascript
// src/lib/gql-uploaders.js -- first revision

const path = require('path');
const { v4: uuid } = require('uuid');

function uuidFilenameTransform(filename = '') { // (A)
  const fileExtension = path.extname(filename);

  return `${uuid()}${fileExtension}`;
}

class S3Uploader {
  constructor(s3, config) {
    const {
      baseKey = '',
      uploadParams = {},                           
      concurrencyOptions = {},
      filenameTransform = uuidFilenameTransform, // (A)
    } = config;

    this._s3 = s3;
    this._baseKey = baseKey.replace('/$', ''); // (B)
    this._filenameTransform = filenameTransform; 
    this._uploadParams = uploadParams;
    this._concurrencyOptions = concurrencyOptions;
  }

  async upload(stream, { filename, mimetype }) {
    const transformedFilename = this._filenameTransform(filename); // (A)

    const { Location } = await this._s3
      .upload(
        {
          ...this._uploadParams, // (C)
          Body: stream,
          Key: `${this._baseKey}/${transformedFilename}`,
          ContentType: mimetype,
        },
        this._concurrencyOptions
      )
      .promise();

    return Location; // (D)
  }
}

module.exports = { S3Uploader, uuidFilenameTransform };

```

- `S3Uploader` constructor receives a S3 instance and the following parameters:
  - `baseKey` - this is the key prefix for every file uploaded. Note that if there is a trailing '/', it will be erased **(B)**;
  - `uploadParams` - the default `params` object passed to S3 upload function. These parameters will me mixed with the more specific one on the upload method **(C)**.
  - `concurrencyOptions` - these are concurrency options accepted by the underlying S3 `upload` function;
  - `filenameTransform` - a customizable transform function for the filename. It defaults to a function that concatenates a random uuid and the file extension **(A)**.
- We return the URI of the file when the promise resolves **(D)**.

Before we see it in action, let's create a configured instance of it:

```javascript
// src/uploaders.js --- first revision

const s3 = require('./s3');
const { S3Uploader } = require('./lib/gql-uploaders');

const avatarUploader = new S3Uploader(s3, {
  baseKey: 'users/avatars',
  uploadParams: {
    CacheControl: 'max-age:31536000',
    ContentDisposition: 'inline',
  },
  filenameTransform: filename => filename,
});

module.exports = { avatarUploader };

```

Okay, here we have it. A few upload parameters (`CacheControl` and `ContentDispotision`) were added just to exercise the possibilities. These  will be used every time we call the `upload` method on the `avatarUploader` object. We defined a `filenameTransform` function that just takes the filename and returns it untouched, and set `baseKey` to `'users/avatars'`, so the files uploaded with `avatarUplaoder` will be stored on S3 with a key similar to `users/avatars/sexy-me.jpg`.

Now, the beauty of it: let's see how clean and concise our resolver gets:

```javascript
// src/resolvers.js -- final revision

const { avatarUploader } = require('./uploaders');

module.exports = {
  Mutation: {
    uploadAvatar: async (_, { file }) => {
      const { createReadStream, filename, mimetype, encoding } = await file;

      const uri = await avatarUploader.upload(createReadStream(), {
        filename,
        mimetype,
      });

      return {
        filename,
        mimetype,
        encoding,
        uri,
      };
    },
  },
};

```

And that is that for the resolver, just that. Now we'll implement our `FilesystemUploader` and we'll realize we won't even need to touch the resolver code when we switch implementations.

```javascript
// src/lib/gql-uploaders.js -- final revision (partial file)

const fs = require('fs');
const path = require('path');
const { v4: uuid } = require('uuid');

// `uuidFilenameTransform` function definition....

// `S3Uploader` class definition...
  
class FilesystemUploader {
  constructor(config = {}) {
    const {
      dir = '',
      filenameTransform = uuidFilenameTransform
    } = config;

    this._dir = path.normalize(dir);
    this._filenameTransform = filenameTransform;
  }

  upload(stream, { filename }) {
    const transformedFilename = this._filenameTransform(filename);

    const fileLocation = path.resolve(this._dir, transformedFilename);
    const writeStream = stream.pipe(fs.createWriteStream(fileLocation));

    return new Promise((resolve, reject) => {
      writeStream.on('finish', resolve);
      writeStream.on('error', reject);
    }).then(() => `file://${fileLocation}`);
  }
}

module.exports = {
  S3Uploader,
  FilesystemUploader,
  uuidFilenameTransform
};

```

- The constructor takes the filesystem path to the destination directory, `dir`.
- `filenameTransform` parameter is similar to the one of `S3Uploader`.
- The `upload` method creates a write stream to record the file on the `dir` directory. It then pipes the read stream to the write stream. `upload` returns a a promise that listens to the write stream events and resolves to the file URI on the drive if the write operation is successful.

Let's turn ourselves back to the src/uploaders.js file and switch the implementations. We'll be simply replacing the reference of the exported name with our new implementation, but you can do more sophisticated stuff like implementing a *Strategy* pattern if you need to switch between them conditionally.

```javascript
// src/uploaders.js -- final revision

const s3 = require('./s3');
const config = require('./config');
const {
  S3Uploader,
  FilesystemUploader,
} = require('./lib/gql-uploaders');

const s3AvatarUploader = new S3Uploader(s3, { // (A)
  baseKey: 'users/avatars',
  uploadParams: {
    CacheControl: 'max-age:31536000',
    ContentDisposition: 'inline',
  },
});

const fsAvatarUploader = new FilesystemUploader({ // (A)
  dir: config.app.storageDir, // (B)
  filenameTransform: filename => `${Date.now()}_${filename}`, // (C)
});

module.exports = { avatarUploader: fsAvatarUploader }; // (A)

```

- **(A)**: now we have two implementations, `s3AvatarUploader` and `fsAvatarUploader`. This time we'll export the `fsAvatarUploader` as `avatarUploader`.
- **(B)**: I'm referencing the tmp directory that I created on the project root folder.
- **(C)**: We customize `filenameTransform` again, just to show it in action one more time. This implementation will prepend the filenames with the current timestamp. Note that I also omitted this parameter on `s3AvatarUploader`, reseting it to its default algorithm (random UUID filenames);

So, enough talking! Let's see what we got!

I ran `./bin/upload-avatar.sh http://localhost:4000 ~/Pictures/sexy-me.jpg` again and got:

```json
{
  "data": {
    "uploadAvatar": {
      "uri": "file:///home/fhpriamo/blogpost-graphql-uploads/tmp/1586733824916_sexy-me.jpg",
      "filename": "sexy-me.jpg",
      "mimetype": "image/jpeg",
      "encoding": "7bit"
    }
  }
}

```

Nice! And we didn't even have to rewrite the resolver!

## Git repository

You can check out the full code [here](https://github.com/fhpriamo/blogpost-graphql-uploads). Clone it, modify it, play with it, extend it... it's you call.

**NOTE**: If you cloned the repo and want to run it, don't forget to write yourself a .env file (You can refer to .env.example if you need a template);

## Related articles:

- https://blog.apollographql.com/%EF%B8%8F-graphql-file-uploads-with-react-hooks-typescript-amazon-s3-tutorial-ef39d21066a2
