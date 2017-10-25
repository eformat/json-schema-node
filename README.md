### NodeJS jsonschema example

```
npm i express spdy jsonschema --save

openssl genrsa -des3 -passout pass:xxxx -out server.pass.key 2048
openssl rsa -passin pass:xxxx -in server.pass.key -out server.key
rm -f server.pass.key
openssl req -new -key server.key -out server.csr -subj "/C=XX/L=Default City/O=Default Company Ltd"
openssl x509 -req -sha256 -days 365 -in server.csr -signkey server.key -out server.crt

cat <<EOF > server.js
const port = 8080
const express = require('express');
const app = express();
const validate = require('jsonschema').validate;
const bodyParser = require('body-parser');
const fs = require('fs')
const spdy = require('spdy')
app.use(bodyParser.json());
app.use(bodyParser.urlencoded({
    extended: true
  }));

// Create a json scehma 
var streetSchema = {
    id: '/Street',    
    type: 'object',
    properties: {
        number: {
            type: 'number',
            required: true
        },
        name: {
            type: 'string',
            required: true
        },
        type: {
            type: 'string',
            required: true,
            enum: ['Street', 'Avenue', 'Boulevard']
        }
    }
}
 
// This route validates req.body against the StreetSchema 
app.post('/street/', function(req, res) {    
    console.log(req.body);
    var result = validate(req.body, streetSchema);
    if (result.valid === true) {
        res
        .status(200)
        .json({message: 'ok'});
    } else {
        res
        .status(200)
        .json({message: 'fails validation'});
    }
    return res;
});

//app.listen(port, function(){
//    console.log('app is running')
//});

const options = {
    key: fs.readFileSync(__dirname + '/server.key'),
    cert:  fs.readFileSync(__dirname + '/server.crt'),
    protocols: [ 'h2', 'http/1.1' ] // 'spdy/3.1', ..., 'http/1.1', 'h2' ],
}
spdy
  .createServer(options, app)
  .listen(port, (error) => {
    if (error) {
      console.error(error)
      return process.exit(1)
    } else {
      console.log('Listening on port: ' + port + '.')
    }
})
EOF

npm init -f

oc new-project json-schema-node
oc new-build --binary --name=json-schema-node -l app=json-schema-node -i nodejs:6
oc start-build json-schema-node --from-dir=. --follow
oc new-app json-schema-node
oc expose svc json-schema-node
oc patch route/json-schema-node -p '{"spec":{"tls":{"termination":"passthrough"}}}'

HOST=`oc get routes/json-schema-node --template='{{.spec.host}}'`
curl -k -X POST --header "Content-Type: application/json" -d '{ "number": "12", "name": "Foo", "type": "Street"}' https://${HOST}/street
curl -k -X POST --header "Content-Type: application/json" -d '{ "number": 12, "name": "Foo", "type": "Street"}' https://${HOST}/street
```
