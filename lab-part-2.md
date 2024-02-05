# Embedded Systems Lab
## Coursework 1, Part 2: IoT Communications

### 1. MQTT
1. Serialise your sensor data into a byte-encoded JSON message:
   1. Convert it to Python types int, float, string or Boolean, grouped into list or dict if necessary.
   2. Package it into a single Python dict with suitable keys to label each field
   3. Convert it to a JSON message with function dumps() from python module json
2. Send and receive MQTT messages using the Mosquitto test broker and the HiveMQ web client:
   1. Load the [web client](https://www.hivemq.com/blog/full-featured-mqtt-client-browser/)
   2. Connect to the broker at `test.mosquitto.org` using port 8081. Enable SSL
   3. Subscribe to the topic `IC.embedded/GROUP_NAME/#`
   4. Publish a test message to the topic IC.embedded/GROUP_NAME and check it shows in the Messages feed
3. Connect to the broker from your Raspberry Pi
   1. Install the paho module for python

      ```bash
      raspberrypi:~$ pip3 install paho-mqtt
      ```

   2. Connect to the test broker in python using unencrypted port 1883 and publish a message

      1. Connect to the broker. `connect()` returns 0 if the connection was successful. Use `mqtt.error_string(RETURN_CODE)` to decode other error numbers.

         ```python
         >>> import paho.mqtt.client as mqtt
         >>> client = mqtt.Client()
         >>> client.connect("test.mosquitto.org",port=1883)
         ```
  
      2. Publish a message. `publish()` returns a message info object. Use the `rc` attribute of the object to find the result of the publish operation

         ```python
         >>> MSG_INFO = client.publish("IC.embedded/GROUP_NAME/test","hello")
         >>> mqtt.error_string(MSG_INFO.rc)
         ```

      3. Check you have received the message on the web client

4.	Set up encryption, using the instructions at https://test.mosquitto.org/ssl/
    1. Get the broker’s certificate. You can download this straight to the Rapsberry Pi with
    
       ```bash
       raspberrypi:~$ wget https://test.mosquitto.org/ssl/mosquitto.org.crt
       ```
    
    2. Generate a private key

       ```bash
       raspberrypi:~$ openssl genrsa -out client.key
       ```
       
    3. Get a signed certificate from the broker:
       1. Generate a certificate signing request

          ```bash
          Raspberrypi:~$ openssl req -out client.csr -key client.key -new
          ```
          
       2. Paste it into the form on the broker’s website and copy the resulting certificate to the Raspberry Pi

    4. Make a secure connection to the broker. Repeat the stages of step 3, but add the security information before connecting. Use broker port 8884 this time.

       ```python
       >>> client.tls_set(ca_certs="mosquitto.org.crt", certfile="client.crt",keyfile="client.key")
       ```
       
5.	Subscribe to messages on the Raspberry Pi or your laptop
    1. (Option 1) Install mosquitto to publish and subscribe to MQTT messages:	[https://mosquitto.org/download/](https://mosquitto.org/download/). Installation is complicated in Windows, so you may want to use WSL.
    2. (Option 2) Install Paho for desktop Python
    3. Connect to the broker using the Paho library as before
    4. Define a callback function that will be executed when a message is received

       ```python
       >>> def on_message(client, userdata, message) :
       >>> 	print("Received message:{} on topic {}".format(message.payload, message.topic))
       ```

       See https://www.eclipse.org/paho/clients/python/docs/#callbacks
    5. Set the callback attribute of the Client instance to point to your callback function

       ```python
       >>> client.on_message = on_message
       ```
       
       > **Note**
       > 
       > Read about Python syntax for writing functions, loops etc. if you are not already familiar with it.
       > There are no brackets to group lines of code but instead the indentation at the start of the line defines the nesting of statements.
       > Every assignment acts like a C++ reference in Python, except for trivial (immutable) data types.
       > So when you write `client.on_message = on_message`, calling `client.on_message()` calls the function `on_message()` because both names are references for the same function.
       
    6. Subscribe to your group’s topic

       ```python
       >>> client.subscribe("IC.embedded/GROUP_NAME/#")
       ```
       
    7. Run the polling function to process any incoming messages. The callback function will be called for every message that’s received, but only when `loop()` is called

       ```python
       >>> client.loop()
       ```
       
       > **Note**
       > 
       > The `Client.loop()` function is necessary because incoming messages are asynchronous: they can happen at any time.
       > `Client.loop()` allows you to choose when to process new messages (and other events if you have defined other callbacks).
       > You can also run the processing loop continuously in the background using `Client.loop_start()` and `Client.loop_stop()`.
       > This will run your callback as soon as the message arrives.

### 2. Database with REST (HTTP) API

> **Note**
> These instructions use Google Firebase to host a database. You can host a database with other providers and the process will be similar.
       
1. Create a database
   1. Go to the [Firebase Console](https://console.firebase.google.com/) and log in with a Google account.
   2. Click 'Add project' and enter a project name. Enable analytics if you like but they aren't necessary.
   3. Open 'Realtime Database', which is under 'Build' in the sidebar. Click 'Create Database'.
   4. Start in test mode. You can add security rules later.
   5. The database will open with a view of the data, which is currently `null`. Add some test data in the form of a key:value pair `test: true` (Boolean)
      ![Manual entry of data into Firebase](firebase-testdata.png)
   6. Add another entry in a child node `level1/test: true`.
      
      > **Note**
      > 
      > Firebase uses a NoSQL database, which means that the data exists in a tree structure containing key: value pairs.
      > Keys often perform the same function as fields in a traditional database, but while records in a table must have the same fields, there is no requirement for nodes to contain the same keys.

2. Write to the database from embedded Python
   1. The database can be accessed using HTTP requests.
      Writing data will most commonly use PUT or POST methods.
      First, use PUT to create a series of data nodes indexed by their timestamp.
      You'll need to replace the database URL in all the following examples with the URL for your own database, which is shown at the top of the web console where you viewed and editted the database.
      
      ```python
      import requests,time,random
      
      # Define the database URL (change to use your database)
      db = "https://esexample-ccdba-default-rtdb.europe-west1.firebasedatabase.app/"
      
      n = 0
      while n < 10:
          path = "timeseries/{}.json".format(int(time.time()))
          data = {"n":n, "rnd":random.random()}
    
          print("Writing {} to {}".format(data, path))
          response = requests.put(db+path, json=data)
    
          if response.ok:
              print("Ok")
          else:
              raise ConnectionError("Could not write to database: {}".format(response.text))
          time.sleep(1)
          n += 1
      ```
   
   2. Run the Python script on your Raspberry Pi.
      View the database on the Firebase console.
      If the script ran correctly, you'll see the database tree extended with 'timeseries', and below that nodes for each request you submitted.
      Each node contains two key:value pairs: an index 'n' and a random number 'rnd'.
   3. The PUT method created new nodes with a name derived from the system time with `int(time.time())`.
      Each PUT request needs to create a node with a unique name, and that might not always be easy.
      
      The POST method can help because it creates a node with an automatically-generated unique name.
      POST is often used to create lists of data because a new node will be added to a location regardless of what is there already.
      
      Try it with this modified script:
      
      ```python
      import requests,time,random
      db = "https://esexample-ccdba-default-rtdb.europe-west1.firebasedatabase.app/"
      n = 0
      while n < 10:
          path = "postlist.json"
          data = {"n":n, "time":time.time(), "rnd":random.random()}
    
          print("Writing {} to {}".format(data, path))
          response = requests.post(db+path, json=data)
    
          if response.ok:
              print("Created new node named {}".format(response.json()["name"]))
          else:
              raise ConnectionError("Could not write to database: {}".format(response.text))
          time.sleep(1)
          n += 1
      ```
      
      The response for POST request contains the generated name for the node under the `name` key, which is printed if the request is successful.
      
4. Read from the database
   1. Reading a specific node from the database is a simple case of using a GET request with the path to the relevant node:
      
      ```python
      import requests, time
      db = "https://esexample-ccdba-default-rtdb.europe-west1.firebasedatabase.app/"
      path = "level1.json" #This node was created in the Firebase console in step 1
      
      response = requests.get(db+path)
      if response.ok:
          print(response.json())
      else:
          raise ConnectionError("Could not access database: {}".format(response.text))
      ```
      
      If the node has child nodes, they will be returned as a hierarchy of JSON objects within the top-level object.
      Change `path` in the example code to simply `.json`, which will return the entire database.
      
      The Response.json() method returns the JSON object as a Python data structure.
      Retrieve the node accessed previously at `level1` by indexing the object with `level1` as a key: `print(response.json()["level1"])`
      
   2. You may not know the name of the node or nodes you wish to access.
      You could download the entire database and search for the information you need, but that would be inefficient.
      Instead, you can add queries to the database request to select the data you need.
      1. First, add a database rule to define that the contents of the 'timeseries' node will be indexed by key, which was a timestamp.
         Switch to the 'Rules' tab in the console webpage for Firebase Realtime Database and change the database rules to the following:
      
         ```json
         {
            "rules": {
               ".read": true,
               ".write": true,
               "timeseries": {
                  ".indexOn": ".key"
               }  
            }
         }
         ```
         
      2. Add a line to your read script that defines a query. The query contains three parameters, which begin with `?` and are separated by `&`:
         - `orderBy="$key"` means that the query will be matched against node names (`"` is escaped with `\"` in the Python code)
         - `startAt="{}"` is populated with the current system time minus one hour. That will be the earliest node returned
         - `endAt="{}"` is populated with the current time so that every time until now is matched

         ```python
         query = "?orderBy=\"$key\"&startAt=\"{}\"&endAt=\"{}\"".format(int(time.time()-3600), int(time.time()))
         ```
         
         Change the database path to point to the 'timeseries' node and modify the HTTP request to append the query:
         
         ```python
         path = "timeseries.json"
         
         response = requests.get(db+path+query)
         ```
         
         Run the script. You will get an object containing nodes with the matching keys.
         You may need to adjust the `startAt=` parameter depending on when you created the database nodes.
         Pick a value that returns only some of the nodes in 'timeseries'
         
      3. You can search within nodes to run your database queries.
         Modify the `.indexOn` parameter to include the child key 'rnd':
         
         ```json
         "timeseries": {
            ".indexOn": [".key","rnd"]
         }
         ```
         
         Modify the query to return nodes according to their child node 'rnd', returning nodes where 'rnd' is greater or equal to 0.5:
         
         ```python
         query = "?orderBy=\"rnd\"&startAt=0.5&endAt=1.0"
         ```
         
         Try the new query. Querying by child nodes is useful when writing to the database with POST, since you can look past the auto-generated, unique key for the nodes to meaningful data contain within.
         Your query can look multiple levels down the hierarchy — not just at the direct child nodes of the set you are querying.
         See the [Google documentation](https://firebase.google.com/docs/database/rest/retrieve-data#section-rest-filtering) for more information on database queries.
        

3. Your database is currently readable and writable to anyone. The identity of the server is proved by its HTTPS certificate, but you also need to authenticate the client to restrict access to your data.
   1. Switch off public access to your database by changing the `read` and `write` database rules to `false`. Your previous scripts will no longer work.
   2. Get a key that will allow admin access to the database. In the Firebase console, go to Project Settings and switch to the tab for Service Accounts. Click 'Generate Private Key'. Copy the resulting .json file to the Raspberry Pi.
      ![Utility to create an admin key](firebase-genkey.png)
   3. Install the [Google authentication module](https://google-auth.readthedocs.io/en/latest/user-guide.html#making-authenticated-requests) for Python
      
      ```bash
      raspberrypi:~$ pip3 install google-auth
      ```
      
   3. Add authentication to the previous test script for POST access.
      Google provides a wrapper for Requests, [`AuthorizedSession`](https://google-auth.readthedocs.io/en/latest/user-guide.html#requests), which adds the necessary authentication to HTTP requests using the private key you downloaded.
      
      ```python
      import time,random
      from google.oauth2 import service_account
      from google.auth.transport.requests import AuthorizedSession
      
      db = "https://esexample-ccdba-default-rtdb.europe-west1.firebasedatabase.app/"
      
      # Define the private key file (change to use your private key)
      keyfile = "src\esexample-ccdba-firebase-adminsdk-awsaw-482267641d.json"

      # Define the required scopes
      scopes = [
          "https://www.googleapis.com/auth/userinfo.email",
          "https://www.googleapis.com/auth/firebase.database"
      ]

      # Authenticate a credential with the service account (change to use your private key)
      credentials = service_account.Credentials.from_service_account_file(keyfile, scopes=scopes)

      # Use the credentials object to authenticate a Requests session.
      authed_session = AuthorizedSession(credentials)
      
      n = 0
      while n < 10:
          path = "postlist.json"
          data = {"n":n, "time":time.time(), "rnd":random.random()}

          print("Writing {} to {}".format(data, path))
          response = authed_session.post(db+path, json=data)

          if response.ok:
              print("Created new node named {}".format(response.json()["name"]))
          else:
              raise ConnectionError("Could not write to database: {}".format(response.text))
          time.sleep(1)
          n += 1
      ```
      
      > *Note*
      > 
      > It wouldn't be appropriate to embed admin access to the database into a real IoT device.
      > Instead, the device should probably use some user credentials assoicated with the user of the device.
      > If you would like to explore this further, Firebase provides [user authentication](https://firebase.google.com/docs/database/rest/auth#python) through the Firebase SDK. It will add some complexity to your system because you'll need to think about how a user will sign in on their embedded device.
