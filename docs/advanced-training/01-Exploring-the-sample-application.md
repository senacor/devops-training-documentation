# BAY - DevOps : 01 Exploring the sample application

The best way to learn new concepts is to get your hands dirty and try
things out. We've prepared a small application that we are going to move
to the cloud throughout this training. We assume that you've done all
the prerequisites and are ready to go.

Log in to the AWS account that you want to use for the training. On the following pages, you will see a lot of bash commands which you need to execute in a terminal on your local machine.

So let's get the code for the sample application which is used throughout the traingin:

```bash
git clone https://github.com/senacor/devops-training-application.git
```

Let's make ourselves a bit familiar with the code. In the folder `app` you will find a small voting application written in Node.js. Open the file `api.js`, where the main logic of the application is:

**_devops-training-application/app/api.js:_**

```javascript
function register(app, client) {
  app.post('/vote', (request, response) => castVote(client, request, response));
  app.get('/vote', (request, response) => getVotes(client, request, response));
}

/**
 * Increments the counter for the casted votes and returns the current vote count.
 */
function castVote(redisClient, request, response) {
  redisClient.incr(request.body.vote, function(err, reply) {
    if (err) {
      console.log(err);
    } else {
      getVotes(redisClient, request, response);
    }
  });
}

/**
 * Retrieves the current vote count from redis and returns it.
 */
function getVotes(redisClient, request, response) {
  redisClient.mget([process.env.VOTE_VALUE_1, process.env.VOTE_VALUE_2], function(err, reply) {
    if (err) {
      console.log(err);
    } else {
      var result = {};
      result[process.env.VOTE_VALUE_1] = reply[0];
      result[process.env.VOTE_VALUE_2] = reply[1];
      response.send(result);
    }
  });
}
...
```

If you've seen a Node.js application before, this is nothing spectacular. The
function registers two API methods in lines 2 and 3:

- GET /vote - making an HTTP request to this resource will return the current voting count in JSON format
- POST /vote -
  making an HTTP POST request on the resource /vote will call the method
  `castVote`. It increments the vote counter of the option that is sent in the request body and returns the new voting score.

As mentioned before, nothing spectacular here. You might have noticed that a
redisClient is called multiple times to perform some operations.
<a href="https://redis.io/" class="external-link">Redis</a> is an
in-memory data structure store, which can be used as a database or a
cache. In our case, we use it as a database that stores our voting
scores. It stores two key-value pairs: The key is the voting option
(`DOGS` or `CATS` in our case) and the value is the current voting score. Each time a
vote is cast for one of the two options, the value is incremented by 1.

Let's run the application to see what it looks like. From your terminal window, navigate into the `/app` folder, install the dependencies and start the application (don't worry about any errors in the terminal):

```bash
cd ~/devops-training/devops-training-application/app
npm install
npm start
```

You might notice that you're getting a lot of error messages in your terminal window and that your application doesn't do anything, when you click the buttons. That's fine, we'll deal with that soon.

Open `localhost:8080` in your browser and you should see something like this:

![AppPreview](../assets/images/AppPreview2.png)

## Running tests

As you probably know, you should write tests for your source code. Since this is a common best practice, we already prepared some tests for you. Take a look at the file `app/spec/test_spec.js`, where our unit tests are defined:

**_devops-training-application/app/spec/test_spec.js:_**

```javascript
const redis = require('redis-mock');

var base_url = 'http://localhost:8080';

describe('test application', function() {
  /**
   * Initialize by starting the service and filling some data
   */
  beforeAll(function() {
    // test setup
  });

  it('GET / returns status code 200', function(done) {
    // ...
  });

  it('GET /vote returns vote count', function(done) {
    // ...
  });

  it('POST /vote increments vote count', function(done) {
    // ...
  });
});
```

Unit tests test a single functionality of the software, which covers just a single or very few connected classes or functions. It does not connect to interfaces or other services - these are mocked. That's why you see that we import a library called `redis-mock` in the first line of the file. It does as the name says, it mocks our Redis database and behaves like it, without the need of having a real database. You can go ahead and run the test cases to see if everything is working correctly. Execute the following command (you can stop the application that we started earlier by hitting `ctrl+c`):

```bash
cd ~/devops-training/devops-training-application/app
npm test
```

The lines at the bottom of the output should tell you that all tests passed successfully:

```bash
3 specs, 0 failures
Finished in 0.092 seconds
Randomized with seed 01736 (jasmine --random=true --seed=01736)
```

Great, our application code seems to be working, so let's move on!
