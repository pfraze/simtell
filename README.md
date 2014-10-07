# SimTell: pull-streams testing framework


```js
var simtell = require('simtell')
var pull = require('pull-stream')

// synchronous function to test
function toCamelCase(str) {
  // ...
  return newStr
}

// asynchronous function to test
function save(v, cb) {
  // ...
  cb(null, result)
}

// test pipeline
var camelCaseSim = simtell(
  simtell.sync(toCamelCase),
  pull.map(function(data, cb) {
    var normalize = function(str) { return str.toLowerCase().replace(/\-\s/g, '') }
    assert(normalize(data.in[0]) == normalize(data.out[0]))
    cb(null, data.out) // pass along the inputs to the next function
  }),
  simtell.async(save),
  pull.map(function(data, cb) {
    var err = data.out[0], res = data.out[1]
    assert(!err)
    assert(!!res)
    assert(res.status === 200)
    cb() // no need to pass anything along
  })
)

// run for the following values
camelCaseSim('foo')
camelCaseSim('foo-bar')
camelCaseSim('fooBar')
camelCaseSim('foo bar')
camelCaseSim('FOOBAR')

// run 1000 times with random strings
for (var i=0; i < 1000; i++) {
  var str = ''
  for (var j; j < i; j++)
    str += String.fromCharCode(Math.round(Math.random() * 255)) // example alg
  camelCaseSim(str)
}

// function to call after all sims are finished
camelCaseSim.after(function() {
  console.log('All tests pass')
})
```

Advanced:

```js
// create a user-behavior meta-sim
var userSessionSim = simtell(
  // embed the login sim
  loginSim.pipe,

  // take 100 random actions
  simtell.loop(100, simtell.choose([
    simtell.wait(10*000),
    addProductToCartSim.pipe,
    checkoutSim.pipe,
    clearCartSim.pipe,
  ])),

  // choose a final action with a probability
  simtell.choose({
    70: simtell.nothing(), // 70/100 times, do nothing
    29: simtell.wait(10*1000), // 29/100 times, wait 10s
    1: simtell.wait(maxSessionLength + 1000) // 1/100 times, wait for the session to expire
  }),
  logoutSim.pipe
)

// run the sim with no inputs
userSessionSim()

// create a perpetual testing environment
var env = simtell.env(userSessionSim, {
  maxActive: 30, // only 30 active sims at a time
  preferActive: 20, // if >20, free resources by killing sims that are idle (in wait())
  interval: 10*1000, // wait 10 seconds after a sim finishes to start a new one
})
env.on('error', console.error)
env.start() // leave me running
```