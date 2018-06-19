# Verbal interaction with the robot Double
## Natural language generation

---

### Team

* Andrea Antoniazzi
* Luigi Secondo

___

### Introduction
This project is about the generation of natural language sentences by developing a software module which uses the onboard speakers and allows a robot ([Robot Double](https://www.doublerobotics.com)) to compose and utter sentences describing its internal state in response to a request.
___

### Project plan

Work has been divided in two sub-programs:
- Develop an iOS application in swift to be run in the robot
- Develop a python server to be run in a computer.   
We decided to use a separate server for two main reasons: the first is the possibility to reuse the speech generation with any device or sofware and the other reason is to work on the project even without a mac OS device.

Main phases of the project:
1. SWIFT: elaborate internal state
2. PYTHON: create a web socket server
3. SWIFT: send state to server via web socket
4. PYTHON: generate from state a natuaral language sentence and send back
5. SWIFT: speech received sentence

___

### 1. Elaborate internal state
In order to elaborate internal state we decided to take care of these particular infos:
* __Battery level__: the following code is used to obtain battery level to be sent to the server: 
```swift
var batteryL: Float {
    return UIDevice.current.batteryLevel
}
UIDevice.current.isBatteryMonitoringEnabled = true
var batteryLevel : Int
batteryLevel = Int(batteryL * 100)
```
* __Ip Address__: the following code is used to obtain ip address of the device. The function `getIFAddresses` returns a list of all the ip addresses of the device in two forms: exadecimal and decimal. To choose one particular representation we suggest to print the list and see which is the desired.
```swift
var ipAdress : [String]
ipAdress = getIFAddresses()


func getIFAddresses() -> [String] {
    var addresses = [String]()
        
    // Get list of all interfaces on the local machine:
    var ifaddr : UnsafeMutablePointer<ifaddrs>?
    guard getifaddrs(&ifaddr) == 0 else { return [] }
    guard let firstAddr = ifaddr else { return [] }
    
    // For each interface ...
    for ptr in sequence(first: firstAddr, next: { $0.pointee.ifa_next }) {
        let flags = Int32(ptr.pointee.ifa_flags)
        let addr = ptr.pointee.ifa_addr.pointee
        
        // Check for running IPv4, IPv6 interfaces. Skip the loopback interface.
        if (flags & (IFF_UP|IFF_RUNNING|IFF_LOOPBACK)) == (IFF_UP|IFF_RUNNING) {
            if addr.sa_family == UInt8(AF_INET) || addr.sa_family == UInt8(AF_INET6) {
    
                // Convert interface address to a human readable string:
                var hostname = [CChar](repeating: 0, count: Int(NI_MAXHOST))
                if (getnameinfo(ptr.pointee.ifa_addr, socklen_t(addr.sa_len), &hostname, socklen_t(hostname.count),
                                nil, socklen_t(0), NI_NUMERICHOST) == 0) {
                    let address = String(cString: hostname)
                    addresses.append(address)
                }
            }
        }
    }
    
    freeifaddrs(ifaddr)
    return addresses
}
```

___

### 2. Create a web socket server
After a research for possible solutions to implement a web socket server in Phyton, we decided to use [Tornado](http://www.tornadoweb.org/en/stable/). It is a Python web framework and asynchronous networking library. Tornado can scale to tens of thousands of open connections, making it ideal for long polling, WebSockets, and other applications that require a long-lived connection to each user.

The server is developed generating a single class called `server` which implements all the functionalities of a web server.
It is requested to use Python 2.7  and to install Tornado. It is also necessary to import tornado libraries in the code: 
```python
import tornado.httpserver
import tornado.websocket
import tornado.ioloop
import tornado.web
import socket
```
Main server class:
```python
class server(tornado.websocket.WebSocketHandler):
    def open(self):
        # Called when a new connection is enstablished
        localtime = time.asctime(time.localtime(time.time()))
        print (colors.OKBLUE +'[' + localtime + ']'+ colors.ENDC +
            ' - Welcome new client! Ipv6: ' + self.request.remote_ip)

    def on_message(self, message):
        # called when a new message arrives from the clients
        localtime = time.asctime(time.localtime(time.time()))
        print (colors.OKBLUE +'[' + localtime + ']'+ colors.ENDC +
            ' - New Message recieved from: ' + self.request.remote_ip + ' -> ' + message)
        newMessage = messageHandler(message)
        self.write_message(newMessage)

    def on_close(self):
        # called when a connection is closed
        localtime = time.asctime(time.localtime(time.time()))
        print (colors.OKBLUE +'[' + localtime + ']'+ colors.ENDC +
            ' - Bye Bye Client ' + self.request.remote_ip)

    def check_origin(self, origin):
        return True
```

___

### 3. Send state server via web socket

After a research for possible solutions to implement a web socket client in swift, we decided to use the Swift framework called SwiftWebSocket (here is link for [github repository](https://github.com/tidwall/SwiftWebSocket)).
It is a library expressly developed to create a WebSocket client and the API is modeled after the Javascript API.
With a single function, called for each request, it is possible to manage all the aspects relative to a connection with a server.
It is required to add SwiftWebSocket framework to the project ( we suggest to use [Carthage](https://github.com/Carthage/Carthage)) and the directive `import SwiftWebSocket`.
```swift
func echoText(infoText : String){
        // set web server adress and port 
        let ws = WebSocket("ws://192.168.1.1:4040/websocketserver")
        // called to send data to server
        let send : ()->() = {
            ws.send(infoText)
        }
        // called to open a connection
        ws.event.open = {
            print("opened")
            send()
        }
        // called to close a connection
        ws.event.close = { code, reason, clean in
            print("close")
        }
        // connection error management
        ws.event.error = { error in
            print("error \(error)")
        }
        // generation of message to be sent
        ws.event.message = { message in
            let text = message
            print("recv: \(text)")
            // calling speech function (see point 5 for more details)
            self.speechFunc(speech: String(describing: message))
            }
        }
```
Internal state information are stored in a struct:
```swift
struct DoubleMessage : Codable {
    var typeID: String
    var value: String
    var userName: String
}
```

To generate a new message to be sent you simply have to convert the struct in a JSON string using JSON encoder. In orther to do that don't forget to add the key word `Codable` after declaration of the struct.
```swift
// Encoding the struct in json string
let jsonEncoder = JSONEncoder()
let jsonData = try? jsonEncoder.encode(toSpeech)
let jsonString = String(data: jsonData!, encoding: .utf8)
```
___

### 4. Generate a natural language sentence

In order to compose natural language we decided to formulate some sentences dividing them in 3 blocks: first one is realtive to a greeting, then the real information is provided and in the end a suggestion relative to internal state is provided. Each block is randomly chosen in a list of possible utterances.

| Greeting | Information | Suggestion |
|:------:|:-----:|:----------|
| Hello | my battery level is ** | Could you charge me? |
| Good morning | my power is ** | I'm running out of battery! |
| Ciao, | iPad's battery is ** | Please, help! No battery |
| Hi, | battery power is ** | I can still make it for a couple of hours |
| Good evening |  | Don't forget the charger |
| Good to see you |  | As you command Captain! |
| Aloha |  | I'm in perfect shape, let's go! |
| Namaste |  | Charged & Fast! |
___

### 5. Speech received sentence

For speech generation we decided to use AVFoundation framework developed by Apple, because it is easily integrable in an iOS application. To utter a sentence you have to implement this simple code in the main class `ViewController: UIViewController`:
```swift
// Definition of speech variables
let synth = AVSpeechSynthesizer()
var myUtterance = AVSpeechUtterance(string: "")

func speechFunc(speech : String){
        // specify the string to be speeched 
        myUtterance = AVSpeechUtterance( string: speech)
        // set the velocity of the voice
        myUtterance.rate = 0.5
        // call the AVFoundation function which speaks
        synth.speak(myUtterance)
    }
```
___

### Sources
* Text to speech app swift 2014: [Tutorial](https://code.tutsplus.com/tutorials/create-a-text-to-speech-app-with-swift--cms-22229)
* Web socket Swift library: [SwiftWebSocket](https://github.com/tidwall/SwiftWebSocket)
* Web socket Python library: [Tornado](http://www.tornadoweb.org/en/stable/)
* Double APIs repository: [Github](https://github.com/doublerobotics/Basic-Control-SDK-iOS)
* Use Objective-C APIs with swift: [Interacting with Objective-C APIs](https://developer.apple.com/library/content/documentation/Swift/Conceptual/BuildingCocoaApps/InteractingWithObjective-CAPIs.html#//apple_ref/doc/uid/TP40014216-CH4-ID35)
