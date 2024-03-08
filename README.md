
## Edge devices communicating through a private local area network 

<br>

In this example, we will setup endpoint devices communicating to each other through a local area network using tcp. 

The devices are also accessible from the cloud so you can configure and develop your applications using a browser interface. 

![](assets/m2m-edge.svg)

## Edge Server 1 Setup

### 1. Create a project directory and install m2m.
```js
$ npm install m2m
```
### 2. Save the code below as device.js in your project directory.
```js
const m2m = require('m2m')

// simulated voltage data source
function voltageSource(){
  return 50 + Math.floor(Math.random() * 10)
}

let m2mServer = new m2m.Server(100) // using serverId of 100
let edge = new m2m.Edge()

m2m.connect(app)

function app(){
    /***
     * m2m device - internet bound communication
     */
    m2mServer.publish('m2m-voltage', (ws) => {
      let vs = voltageSource()
      ws.send({id:ws.id, topic:ws.topic, type:'voltage', value:vs})
    })
    
    /***
     * edge tcp server - private LAN communication
     */
    let port = 8125		// port must be open from your endpoint
    let host = '192.168.0.113' 	// use the actual ip of your endpoint
    
    edge.createServer(port, host, (server) => {
      console.log('edge server started :', host, port)
      
      server.publish('edge-voltage', (tcp) => { // using default 5 secs or 5000 ms polling interval
         let vs = voltageSource()
         tcp.send({port:port, topic:tcp.topic, type:'voltage', value:vs})
      })

      server.dataSource('data-source-1', (tcp) => {
         if(tcp.payload){
            tcp.send({port:port, topic:tcp.topic, payload:tcp.payload})
         }
      })
    })
}

```
### 3. Start your application.
```js
$ node device.js
```

## Edge Server 2 Setup

### 1. Create a project directory and install m2m.
```js
$ npm install m2m
```
### 2. Save the code below as device.js in your project directory.
```js
const m2m = require('m2m')

// simulated temperature sensor data source
function tempSource(){
  return 20 + Math.floor(Math.random() * 4)
}

let m2mServer = new m2m.Server(200)
let edge = new m2m.Edge()

async function main(){

    await m2m.connect()

    /***
    * m2m device - internet bound communication
     */
    m2mServer.publish('m2m-temperature', (ws) => {
      let ts = tempSource()
      ws.send({id:ws.id, topic:ws.topic, type:'temperature', value:ts})
    })
    
    /***
     * edge tcp server - private LAN communication
     */
    let port = 8125		// port must be open from your endpoint
    let host = '192.168.0.142' 	// use the actual ip of your endpoint 
    
    let server = edge.createServer(port, host)
    console.log('tcp server started :', host, port)
      
    server.publish('edge-temperature', (tcp) => {
       let ts = tempSource()
       tcp.polling = 10000;	// set polling interval to 10000 ms or 10 secs instead of the default 5000 ms
       tcp.send({port:port, topic:tcp.topic, type:'temperature', value:ts})
    });

    server.dataSource('current-temp', (tcp) => {
       let ts = tempSource()
      tcp.send({port:port, topic:tcp.topic, value:ts})
    })
}

main()

```
### 3. Start your application.
```js
$ node device.js
```

## Edge Client Setup

### 1. Create a project directory and install m2m.
```js
$ npm install m2m
```
### 2. Save the code below as client.js in your project directory.
```js
const m2m = require('m2m') 

let client = new m2m.Client()

let edge = new m2m.Edge()

m2m.connect()
.then(async (result) => {
  console.log('connection:', result)

  /***
   * m2m client - access m2m servers through a public internet
   */
  // method 1
  let device1 = client.access(100)
  
  device1.subscribe('m2m-voltage', (data) => {
    console.log('m2m device 100 voltage', data)
  })
  
  let device2 = client.access(200)

  device2.subscribe('m2m-temperature', (data) => {
    console.log('m2m device 200 temperature', data)
  })
    
  // or 
  
  // method 2
  client.subscribe({id:100, channel:'m2m-voltage'}, (data) => {
    console.log('m2m device 100 voltage', data)
  })
  
  client.subscribe({id:200, channel:'m2m-temperature'}, (data) => {
    console.log('m2m device 200 temperature', data)
  })
})
.then(() => {
  /***
   * edge tcp clients - access edge servers through a private local network
   */

  // edge client 1 
  let ec1 = new edge.client(8125, '192.168.0.113')
  
  ec1.subscribe('edge-voltage', (data) => {
    console.log('edge server 1 voltage', data)
  })
  
  ec1.write('data-source-1', 'node-edge', (data) => {
    console.log('edge server 1 data-source-1', data))
  })

  // edge client 2
  let ec2 = new edge.client(8125, '192.168.0.142')
  
  ec2.subscribe('edge-temperature', (data) => {
    console.log('edge server 2 temperature', data))
  })

  let rd = await ec2.read('current-temp')
  console.log('edge server 2 current temperature', rd)
})
.catch((err) => {
  console.log('promise client error:', err) 
})


```
### 3. Start your application.
```js
$ node client.js
```

### 4. The expected output should be similar as shown below.
```js
$

edge server 1 voltage {port:8125, topic:'edge-voltage', type:'voltage', value:16}
edge server 1 data-source-1 {port:8125, topic:tcp.topic, payload:'node-edge'}
m2m device 200 temperature {id:200, topic:'m2m-temperature', type:'temperature', value:25
m2m device 100 voltage {id:100, topic:'m2m-voltage', type:'voltage', value:4
edge server 2 temperature {port:8125, topic:'edge-temperature', type:'temperature', value:25}
edge server 2 current temperature {port:8125, topic:'current-temperature', value:25}

```


