```
// All requests - add the headers:
x-covid-bot:12345678-1234-4123-8123-1234567890ab
```

First step is to add a new UUID to the db by going to the app.get() routing by using HEAD. Bypass the validating as express routes HEAD to app.get():
```
HEAD /register-covid-bot?newID=12345678-1234-4123-8123-1234567890ab
```
```js
// Declaring and initializing variables with a JSON object can set the value directly if the keys match
COVID_BOT_ID_REGEX = /^[a-f0-9]{8}-[a-f0-9]{4}-4[a-f0-9]{3}-[89aAbB][a-f0-9]{3}-[a-f0-9]{12}$/g;

let { newID } = {"newID":"12345678-1234-4123-8123-1234567890ab"};
console.log(newID);

if (newID.match(COVID_BOT_ID_REGEX)) {
    console.log("Matched");
}
```

Second step is to POST