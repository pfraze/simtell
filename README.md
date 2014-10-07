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
  pull.map(function(data) {
    // tests
    function normalize(str) { return str.toLowerCase().replace(/\-\s/g, '') }
    assert(normalize(data.in[0]) == normalize(data.out[0]))
    return data.out // pass along the inputs to the next function
  }),
  simtell.async(save),
  pull.map(function(data) {
    // tests
    var err = data.out[0], res = data.out[1]
    assert(!err)
    assert(!!res)
    assert(res.status === 200)
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
    simtell.wait(10*000), // do this
    addProductToCartSim.pipe, // or this
    checkoutSim.pipe, // or this
    clearCartSim.pipe, // or this
  ])),

  // choose a final action with a probability
  simtell.choose({
    99: simtell.nothing(), // 99/100 times, do nothing
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