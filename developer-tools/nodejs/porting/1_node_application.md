# Setup our sample Node.js application

## Application details

* API HTTP Rest based on Node.js / [Sails.js](sailsjs.org)) and [MongoDB](https://www.mongodb.com/)
* A couple of prerequisites are needed to run the application locally
  * [Node.js](https://nodejs.org/en/)
  * [Mongo](https://docs.mongodb.org/manual/installation/)
* Provides CRUD (Create / Read / Update / Delete HTTP verbs) on a “Message” model

HTTP verb | URI | Action
----------| --- | ------
GET | /message | list all messages
GET | /message/ID | get message with ID
POST | /message | create a new message
PUT | /message/ID | modify message with ID
DELETE | /message/ID | delete message with ID

## Setup

* Install [Node.js](https://nodejs.org/en/download/) and [MongoDB](https://docs.mongodb.org/manual/installation/).
  * As of Sept 20, 2019 this was tested with node v10.16.3, npm v6.9.0, and MongoDB community v4.2.
* Install Sails.js (it's to Node.js what RoR is to Ruby): ```sudo npm install sails -g```
  * Tested with v1.2.3, Sept 20, 2019
* Create the  application:  ```sails new messageApp && cd messageApp```
  * When prompted to "Choose a template" select option _2. Empty_
* Link application to local MongoDB
  * `npm install sails-mongo --save`
* There are two SailsJS configuration files we need to modify to get our application to work. Change the lines shown below in the two files listed (both files have many other lines - you only need to edit the settings shown here):

```javascript
config/model.js:

module.exports.models = {
  migrate: 'alter',
  id: { type: 'string', columnName: '_id' },
};
```

```javascript
config/datastores.js:

module.exports.datastores = {
  default: {
     adapter: 'sails-mongo',
     url: process.env.MONGO_URL || 'mongodb://localhost/messageApp'
  }
};
```

You can also edit the package.json file so that we start up in the development environment:
```json
package.json:
  "start": "NODE_ENV=development node app.js",
```

* Generate the API scafold:
  * `sails generate api message`
* Run the application: 
  * `sails lift`
* The API is available locally on port 1337 (default Sails.js port)

## Test the application in command line

``` bash
# Get current list of messages
$ curl http://localhost:1337/message
[]

# Create new messages
$ curl -XPOST http://localhost:1337/message?text=hello
$ curl -XPOST http://localhost:1337/message?text=hola

# Get list of messages
$ curl http://localhost:1337/message
[
  {
    "text": "hello",
    "createdAt": "2015-11-08T13:15:15.363Z",
    "updatedAt": "2015-11-08T13:15:15.363Z",
    "id": "5638b363c5cd0825511690bd"
  },
  {
    "text": "hola",
    "createdAt": "2015-11-08T13:15:45.774Z",
    "updatedAt": "2015-11-08T13:15:45.774Z",
    "id": "5638b381c5cd0825511690be"
  }
]

# Modify a message
# Note that you will have to substitute your own "id" value from the previous list of messages in place of the value `5638b363c5cd0825511690bd` shown in the example.
$ curl -XPUT http://localhost:1337/message/5638b363c5cd0825511690bd?text=hey

# Delete a message
# Again, substitute one of your own "id" values
$ curl -XDELETE http://localhost:1337/message/5638b381c5cd0825511690be

# Get list of messages
$ curl http://localhost:1337/message


[
  {
    "text": "hey",
    "createdAt": "2015-11-08T13:15:15.363Z",
    "updatedAt": "2015-11-08T13:19:40.179Z",
    "id": "5638b363c5cd0825511690bd"
  }
]
```

Congratulations! You have a functioning Node.js HTTP API! Now [move on to containerizing it](./2_application_image.md).
