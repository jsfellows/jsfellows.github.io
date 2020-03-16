---
layout: post
title:  "Ensuring that opened connections/transactions to database were closed using automated tests in NodeJS"
author: viniciusls
categories: [ javascript, nodejs, tests, automation, vanillajs ]
image: assets/images/3.png
featured: true
hidden: false
---

One of the biggest issues that my team was having in one of our projects using JavaScript and NodeJS when I joined in was at the same time the simplest one: As we don't use any database framework, we have to open and close all connections to DB2 manually. In other words, for every query that we wanna run we need to call `openConnection()` at the beginning to get a DB2 connection object and `closeSync()` to close this connection at the end. If, for any reason, someone forget to call `closeSync` at the end of the method it'd remain opened and at this point you can imagine how easy it'll be to get a **memory leak**, which is a serious problem, or other issues like deadlocks and so, specially in a large scale application (even in a smaller one, in fact). Same is valid for transactions as you need to call `beginTransaction()` and `commitTransaction()` or `rollbackTransaction()` if you wanna create an explicit transaction (which is an useful feature in a bunch of cases).

As my project uses `Pool` class from `ibm_db` (DB2 module for NodeJS), we create the `openConnection()` method to abstract the open connection function from pool. The implementation is:

```javascript
// db2.js
const { Pool } = require('ibm_db');

class Db2 {
  constructor() {
    this.pool = new Pool();
  }

  openConnection(callback) {
    return this.pool.open('<my_database_connection_string>', callback);
  }
```

Of course the obvious exit would be to just educate everyone to always close its resources, valid for both connections and transactions, but even if all your fellows agreed to pay atention on that you'll never be 100% safe if you don't implement an automated way to check that and this is the solution that we designed in our project to both uncover existing gaps and avoid them to reappear on future.

Talking about **automated tests** and things related to **automation**, these will be the subject of some of my next posts so stay tuned here on JS Fellows and follow me on [Twitter](https://twitter.com/iViinii) :)

To achive this solution, I used the same tools I use to implement **unit tests**: [Mocha](https://mochajs.org/) as runtime library, [Chai](https://www.chaijs.com/) as assertion library and [Sinon](https://sinonjs.org/) as mock/spy/stub library. Let's talk about the responsibility of each of these libraries and then how to implement the solution:

#### Mocha

[Mocha](https://mochajs.org/) is a feature-rich JavaScript test framework running on Node.js and it's responsible by the execution of the tests, generating flexible and accurate reports (can generate both HTML and JSON reports besides logging on console and can be integrated with additional libraries to make it even more useful) and allowing us to customize how tests will be executed in an easy way.

#### Chai

[Chai](https://www.chaijs.com/) is a multi-interface library for assertions on automated tests. It allows us to choose between BDD or TDD methodology, supporting syntaxes like **Should**, **Expect** and **Assert**, making tests even more readable and easy to maintain.

#### Sinon

[Sinon](https://sinonjs.org/) is probably the most used library for the implementation of this solution. Basically we need Sinon in order to replace the original methods that we want to check if was called and to spy how they were called in some cases. But why do we need to replace them? Well, for example, to check if the `closeSync()` method inside DB2 module was called we don't actually wanna to call the `openConnection()` method and open a real database connection and close it, we just wanna to check if `closeSync()` was called once for each `openConnection()` that was called on our code. In other words, we just need to check how many times both methods were called and compare them to make it simple and don't wanna real actions against database so we don't make our tests dependent on database behavior and don't pollute it.

#### How to implement it?

In this step I'll summarize all that I said on the last steps but without showing a complex code as was needed in my project (due to some abstractions and other kind of specific implementations that we have and probably you don't). The rational line that you need to achieve this solution is basically: Everything that was opened should be closed. At this point you probably already realized that you'll have to implement a test for each function that you have that interact with the database and open/close transactions and connections, but hey, it's the same logic you use to implement unit test: no code without tests, everything should be covered and once it's covered, it should be safer to work with and change as much as you need without being scared of the side-effects. Once you implement it, you'll add it to your deployment pipeline to be executed everytime you try to deploy a new code change, this way you'll have a regression suite.

So considering you already installed all the three libraries that I mentioned above, first step is to override the methods used to **open** and **close** your database connections on the module that you use. In my case it'll be DB2 module with `openConnection()` and `closeSync()` methods. I have to mention that `closeSync()` is a method inside the connection object return by `openConnection()`, so it's kinda easier to check it.

```javascript
const { Db2 } = require('./db2');

const chai = require('chai');
const lodash = require('lodash');
const sinon = require('sinon');
const sinonChai = require('sinon-chai');

chai.should(); // Enable `should` assertions on Chai
chai.use(sinonChai); // Tell Chai to support Sinon integration

class DatabaseTestSuite {
  constructor(dao) {
    this.db2 = new Db2(); // New Db2 class
    this.dao = dao; // The instance of the class that I want to test, in this case a DAO class
  }

  async setupStubs(withErrors = false) {
    this.originalDb2OpenConnection = this.db2.openConnection; // Saving the original openConnection implementation to revert the change later

    Db2.prototype.openConnection = await this.openConnection(withErrors); // Overriding the original openConnection method with the one that we defined below here
  }

  async restoreStubs() {
    Db2.prototype.openConnection = this.originalDb2OpenConnection; // Rolling back the change we override on openConnection method
  }

  async execute(callback, ...data) {
    this.resources = []; // The list of the opened resources
    this.transactions = 0; // The total of existing opened transactions

    await this.testCloseResources(callback, ...data); // Call to test if all opened connection were closed
    await this.testCloseTransactions(callback, ...data); // Call to test if all opened transactions were commit or rolled back
  }

  async testCloseResources(callback, ...data) {
    await this.setupStubs(false); // Call to setup the override of openConnection method without force throwing errors (it'll be used later on rollback transaction method)

    try {
      const params = this.cloneParams(...data); // Create a clone of the params passed by the test implementation to avoid mutating the original ones that will be used on next tests too
      await callback(...params); // Call the function that we want to test and is passed by parameter to this function with the specified (and cloned) parameters
    } finally {
      await this.restoreStubs(); // Restore the overrided methods after the execution
    }

    for (let resource of this.resources) { // Here's the trick
      resource.closeSync.should.have.been.called; // Check if for each resource added to the resources array by openConnection method each time this was called had their closeSync() method called. If any of the openConnection connection object hasn't its related closeSync() method called, it'll throw an error showing that a opened connection was not closed (which is what will happen when your code is executed in production)
    }
  }

  async testCloseTransactions(callback, ...data) {
    // Successful connection/execution
    await this.setupStubs(false); // Call to setup the override of openConnection method without force throwing errors (it'll be used later on rollback transaction method)

    try {
      const params = this.cloneParams(...data); // Create a clone of the params passed by the test implementation to avoid mutating the original ones that will be used on next tests too
      await callback(...params); // Call the function that we want to test and is passed by parameter to this function with the specified (and cloned) parameters
    } finally {
      await this.restoreStubs(); // Restore the overrided methods after the execution
    }

    this.chai.expect(this.transactions).to.be.below(1); // Here's the trick: We expected that each opened transaction (which increments by 1 the transactions count) is closed (which decrements by 1 the transactions count). If any remains, it'll throw an error showing that a opened connection was not closed (which is what will happen when your code is executed in production)
    // At this point, we have to consider that one opened transaction can have multiple commits/rollbacks, so this test is not 100% safe but it helps a lot :)

    // Fail connection/execution
    await this.setupStubs(true); // Call to setup the override of openConnection method forcing throwing errors to test rollbackTransaction calls, so every opened transaction has to be closed in case of errors

    try {
      const params = this.cloneParams(...data); // Create a clone of the params passed by the test implementation to avoid mutating the original ones that will be used on next tests too
      await callback(...params); // Call the function that we want to test and is passed by parameter to this function with the specified (and cloned) parameters
    } catch (e) { /* We do not need to do anything with this error, just be sure that rollbackTransaction is called */
    } finally {
      await this.restoreStubs(); // Restore the overrided methods after the execution
    }

    this.chai.expect(this.transactions).to.be.below(1); // Here's the trick: We expected that each opened transaction (which increments by 1 the transactions count) is closed (which decrements by 1 the transactions count). If any remains, it'll throw an error showing that a opened connection was not closed (which is what will happen when your code is executed in production)
    // At this point, we have to consider that one opened transaction can have multiple commits/rollbacks, so this test is not 100% safe but it helps a lot :)
  }

  openConnection(withErrors = false) {
    const databaseTestSuite = this;

    // In this block I basically override all database functions that the methods that I want to test uses to avoid interacting directly with database and to add some desired behaviors. It's almost the same that I said that Sinon does for us, but as I was to customize the implementation of each methods, it's easier to do this way. Sinon would be good if, for example, I wanna override only the return of the method. In some cases on this example I did it manually just to avoid mixing uses (manual and Sinon in the same object)
    this.conn = {
      prepareSync: function() {
        let fn = {
          executeSync: function() {
            let fn2 = {
              fetchAllSync: function() {
                return [['1']];
              },
              fetchSync: function() {
                return [['1']];
              },
              closeSync: databaseTestSuite.sinon.spy(); // IMPORTANT: Add a spy to count if this method was called once on the for on line 98
            };

            databaseTestSuite.resources.push(fn2); // IMPORTANT: Add this function to the resources list to be iterated on the for on line 98

            return fn2;
          },
          closeSync: databaseTestSuite.sinon.spy(); // IMPORTANT: Add a spy to count if this method was called once on the for on line 98
        };

        databaseTestSuite.resources.push(fn); // IMPORTANT: Add this function to the resources list to be iterated on the for on line 98

        return fn;
      },
      prepare: function(sql, callback) {
        let fn = {
          closeSync: databaseTestSuite.sinon.spy(); // IMPORTANT: Add a spy to count if this method was called once on the for on line 98
        };

        databaseTestSuite.resources.push(fn); // IMPORTANT: Add this function to the resources list to be iterated on the for on line 98

        callback(null, fn);

        return fn;
      },
      querySync: function() {
        return [['1']];
      },
      query: function(query, params, callback) {
        callback(null, [{ VALUE: 1 }])
      },
      closeSync: databaseTestSuite.sinon.spy(), // IMPORTANT: Add a spy to count if this method was called once on the for on line 98
      beginTransaction: function(callback) {
        databaseTestSuite.transactions++; // IMPORTANT: Increment the transaction total that will be validated on transactions test

        callback(null);
      },
      commitTransaction: function(callback) {
        if (withErrors) {
          throw new Error('Cannot commit to database');
        }

        databaseTestSuite.transactions--; // IMPORTANT: Decrement the transaction total that will be validated on transactions test

        callback(null);
      },
      rollbackTransaction: function(callback) {
        databaseTestSuite.transactions--; // IMPORTANT: Decrement the transaction total that will be validated on transactions test

        callback(null);
      }
    };

    this.db2.openConnection = async (callback, autoclose = false) => {

      if (!autoclose) { // There's some functions that autoopen and autoclose the function on my code, so if that's the case, it shouldn't be added to resources list
        databaseTestSuite.resources.push(this.conn);
      }

      await callback(null, await this.conn); // Return the generate conn to the callback function
    };

    return this.db2.openConnection; // Return the new function that we want to use instead of the original openConnection function
  }

  cloneParams(...params) {
    return lodash.cloneDeep(params);
  }
}

module.exports = { DatabaseTestSuite };

```

And to execute it, we just to need to create a test file to Mocha like this:

```javascript
// Classes
const { UserDao } = require('./dao/user.dao');
const { DatabaseTestSuite } = require('./database.test.suite');

describe('Tests for UserDao', () => {
  let userDao;
  let databaseTestSuite;

  before(() => {
    userDao = new UserDao();

    databaseTestSuite = new DatabaseTestSuite();
  });

  describe('the getByName function', () => {
    it('should open, execute and close databases function', async () => {
      // Initial setup
      const data = 'Vinicius Silva';

      await databaseTestSuite.execute(userDao.getByName.bind(userDao), data); // Call the execute method from our database.test.suite.js passing the method that we want to test (WITHOU EXECUTING IT - NO () ON METHOD NAME - AND BINDED AS WE'RE USING CLASSES AND NEED THE CONTEXT DEFINED) and the parameters (you can pass as much as necessary).
    });
  });
});

```

And that's it. If `getByName(name)` method inside **userDao** with this parameter calls `openConnection()` once it should call `closeSync()` from the same connection as well. If `getByName(name)` calls `beginTransaction()` once it should call `commitTransaction()` or `rollbackTransaction()` once as well. If it calls any of these methods more than once it should call the respective closing method the same many times as it was opened.

I know that this code can be kinda complex (and it was complex to think as well) and the real implementation is private, so I had to resume it and make it as clean as possible just to education purposes. The point I want to advise you in this case is: focus on the concept explained, not on the implementation showed; it was only an example. But feel free to contact me in case you need further explanation :)

#### Conclusion

So, that's it! I hope I could help you to automated the verification of opened and closed connections and transactions in order to avoid some issues like **memory leaks** or **resources overload**. Feel free to ask me anything about this subject or any other subject that you wanna see here in this space using the comments section below :) See ya!