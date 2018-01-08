# Shared Objects

This example demonstrates the use of Remote Shared Objects, which provides a mechanism to share and persist information across multiple clients in real time, as well as sending messages to all clients that are connected to the object.

### Example Code
- ***[PublishTest.java](../PublishTest/PublishTest.java)***
- ***[SharedObjectTest.java](SharedObjectTest.java)***

### Setup
Use of Shared objects requires an active stream - either publishing or subscribing. The content of the stream isn't important to the shared object itself, even a muted audio-only stream will be enough. Also, which stream you are connected to isn't important to which shared object you access, meaning that clients across multiple streams can use the same object, or there could be multiple objects accessed through a single stream.

To run the test, you will need at least two devices running the "Shared Object" example. This example searches active streams for the stream name set as 'stream1' in tests.xml. If a client doesn't find an active stream with that name, it will begin publishing that stream, while any device that finds the published stream will subscribe to it.

### Connection
Shared objects require a successfully started stream to transmit data. There is sometimes a slight delay between receiving the server message that the stream has started and when it will accept connection calls such as the Shared Object connection request - this delay will usually be a small fraction of a second, but still needs to be accounted for.

```Java
public void onConnectionEvent(R5ConnectionEvent r5ConnectionEvent) {

    if(r5ConnectionEvent.name() == R5ConnectionEvent.START_STREAMING.name()){
        final SharedObjectTest safeThis = this;
        callThread = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(250);
                    //start the shared object call
                    sObject = new R5SharedObject("sharedChatTest", stream.connection);
                    sObject.client = safeThis;

                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        callThread.start();
    }
}
```
<sup>
[SharedObjectTest.java #118](SharedObjectTest.java#L118)
</sup>

Instantiating a new R5SharedObject requires a name and the connection used for your stream. After that it will connect and notify the object set as its client that it has connected successfully.

```Java
sObject = new R5SharedObject("sharedChatTest", stream.connection);
sObject.client = safeThis;
```
<sup>
[SharedObjectTest.java #128](SharedObjectTest.java#L128)
</sup>

Once connected successfully, the shared object will attempt to call `onSharedObjectConnect` on the client object - passing a JSONObject with the remote object's current state as a parameter.

To disconnect, simply call `close()` on the object. This should be called before closing the stream whose connection the object shares.

### Persistent Information
Remote Shared Objects use JSON for transmission, meaning that its structure is primarily up to your discretion. The base object will always be a dictionary with string keys, while values can be strings, numbers, booleans, arrays, or other dictionaries - with the same restriction on sub-objects.

This example simply uses a number to keep a count of how many people are connected to the object. As seen in `onSharedObjectConnect`, value can be accessed from the object by name, and set using `setProperty`

```Java
public void onSharedObjectConnect(JSONObject objectValue){
    try {
        addMessage("Connected to object, there are " + ((objectValue.has("count")) ? objectValue.getInt("count") : "no") + " other people connected");
        thisUser = (objectValue.has("count") ? objectValue.getInt("count") + 1 : 1 );
        //set the count property to add yourself
        sObject.setProperty("count", thisUser);
    } catch (JSONException e) { e.printStackTrace(); }
}
```
<sup>
[SharedObjectTest.java #141](SharedObjectTest.java#L141)
</sup>

When one client calls `setProperty` other clients will be notified through `onUpdateProperty` with a JSONObject that holds the single key/value pair that has updated.

```Java
public void onUpdateProperty(JSONObject propertyInfo){
//    propertyInfo.keys().next() can be used to find which property has updated.
    try {
        addMessage("Room update - There are now " + propertyInfo.getInt("count") + " users");
    } catch (JSONException e) {}
}
```
<sup>
[SharedObjectTest.java #151](SharedObjectTest.java#L151)
</sup>

Note that the read-only data property of the R5SharedObject which holds the current state of the remote object is updated before a method is called on the client.

### Messages
The Shared Object interface also allows sending messages to other people watching the object. By sending a JSONObject through the `send` method, that object will be passed to all the listening clients that implement the specified call.

```Java
JSONObject messageOut = new JSONObject();
try {
    messageOut.put("user", "" + thisUser);
    messageOut.put("message", chatInText);
} catch (JSONException e) {}

//Calls for the relevant method with the sent parameters on all clients listening to the shared object
//Note - This includes the client that sends the call
sObject.send("messageTransmit", messageOut);

chatInput.setText("");
```
<sup>
[SharedObjectTest.java #163](SharedObjectTest.java#L163)
</sup>

Which is received by:

```Java
public void messageTransmit( JSONObject messageIn ){

  String user, message;
  try {
      user = messageIn.getString("user");
      message = messageIn.getString("message");
  } catch (JSONException e) { return; }
```
<sup>
[SharedObjectTest.java #176](SharedObjectTest.java#L176)
</sup>

Like with the parameters of the object, the structure of the object sent is up to you, and it will reach the other clients in whole as it was sent.
