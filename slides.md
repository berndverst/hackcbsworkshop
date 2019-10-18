## Using Azure + AI to detect Emotion

#### Bernd Verst (@berndverst)

##### Developer Advocate, Microsoft

> In this workshop we will build a Python app that uses Cognitive Services APIs to detect emotions in faces and then store the emotions detected.


---
## Resources
Everything covered here is available at
### [aka.ms/hackcbsworkshop](https://aka.ms/hackcbsworkshop)

---
## What are we building?

* Client
  * Takes a photo from our webcam
  * Sends the photo to our Backend API

* Server
  * Receives the photo, detects faces and emotions
  * Stores the emotions in a database

---
## Azure Technologies
* [Azure App Service](https://docs.microsoft.com/en-us/azure/app-service/containers/?WT.mc_id=hackcbs-hackathon-beverst)
* [Cognitive Services](https://docs.microsoft.com/azure/cognitive-services/?WT.mc_id=hackcbs-hackathon-beverst)
* [CosmosDB](https://docs.microsoft.com/en-us/azure/cosmos-db/?WT.mc_id=hackcbs-hackathon-beverst)


---
## Open Source Technologies
* [Python 3.7](https://docs.python.org/3.7/)
* [Flask](https://flask.palletsprojects.com/en/1.1.x/)
* [OpenCV](https://docs.opencv.org/master/)

---
## Prerequisites
* [Python 3.7](https://www.python.org/downloads/release/python-374/)
* [Microsoft Azure Account](https://azure.microsoft.com/free/students/?WT.mc_id=hackwithazure-hackathon-beverst)
* [Visual Studio Code](https://code.visualstudio.com/?WT.mc_id=hackcbs-hackathon-beverst)
* [Python Extension for Visual Studio Code](https://marketplace.visualstudio.com/itemdetails?itemName=ms-python.python&WT.mc_id=hackcbs-hackathon-beverst)
* [Azure App Service Extension for Visual Studio Code](https://marketplace.visualstudio.com/itemdetails?itemName=ms-azuretools.vscode-azureappservice&WT.mc_id=hackcbs-hackathon-beverst)
* [Azure Cosmos DB Extension for Visual Studio Code](https://marketplace.visualstudio.com/itemdetails?itemName=ms-azuretools.vscode-cosmosdb&WT.mc_id=hackcbs-hackathon-beverst)

---
## Let's get started

---
## Build an app to take a photo

```
pip3 install opencv-python
```

Create `picturetaker.py` with code:
```python
import cv2

cam = cv2.VideoCapture(0)
cv2.namedWindow('Press space to take a photo')

while True:
  ret, frame = cam.read()
  cv2.imshow('Press space to take a photo', frame)

  key = cv2.waitKey(1)
  if key%256 == 32:
    break

cam.release()
cv2.destroyAllWindows()
```

---
## Create a Flask Web app

Create file `requirements.txt` with content:
```
opencv-python
flask
```

Install requirements by running terminal command
```
pip3 install -r requirements.txt
```

---
## Create a Flask Web app (continued)

Create a file called `app.py` with code:
```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
  return 'Hello World'
```

---
## Deploying to an App Service

Using Visual Code and the App Service extension we are deploying our web app to the cloud.

[Detailed steps](https://github.com/microsoft/computerscience/blob/master/Events%20and%20Hacks/Student%20Hacks/mood-detector-workshop/Steps/DeployTheWebAppToTheCloud.md)

---
## Add a REST (web) API route to the Flask app

*We want to be able to receive image data*

In `app.py` add imports:
```python
from flask import Flask, request
import base64
```

Add code
```python
@app.route('/image', methods=['POST'])
def upload_image():
  json = request.get_json()
  base64_image = base64.b64decode(json['image'])

  return 'OK'
```

---
## Analyse the photo using AI

Using the *Face API* from *Cognitive Services* we will detect a face and emotion.

* Open the Azure portal at [portal.azure.com](https://portal.azure.com)
* Create a resource of type *Face*
* Once completed, take note of the *Endpoint* and *Keys*

---
## Analyse the photo using AI (continued)

Add the Face API SDK to `requirements.txt` by adding the line
```
azure-cognitiveservices-vision-face
```
Install the requirements with
```
pip3 install -r requirements.txt
```

---
## Analyse the photo using AI (continued)

Add imports to `app.py`
```python
from azure.cognitiveservices.vision.face import FaceClient
from msrest.authentication import CognitiveServicesCredentials
import io
import uuid
```

Add two variables:
```python
face_api_endpoint = 'https://centralus.api.cognitive.microsoft.com'
face_api_key = '<key>'
```

---
## Analyse the photo using AI (continued)

Initialize the Face API client and add a helper function

```python
credentials = CognitiveServicesCredentials(face_api_key)
face_client = FaceClient(face_api_endpoint, credentials=credentials)

def best_emotion(emotion):
  emotions = {}
  emotions['anger'] = emotion.anger
  emotions['contempt'] = emotion.contempt
  emotions['disgust'] = emotion.disgust
  emotions['fear'] = emotion.fear
  emotions['happiness'] = emotion.happiness
  emotions['neutral'] = emotion.neutral
  emotions['sadness'] = emotion.sadness
  emotions['surprise'] = emotion.surprise
  return max(zip(emotions.values(), emotions.keys()))[1]
  ```

---
## Analyse the photo using AI (continued)

Update the `upload_image` function by adding this code snippet before the return statement.

```python
image = io.BytesIO(base64_image)
faces = face_client.face.detect_with_stream(image,
    return_face_attributes=['emotion'])

for face in faces:
    doc = {
        'id' : str(uuid.uuid4()),
        'emotion': best_emotion(face.face_attributes.emotion)
        }
```


---
## Save the face details to a database

We will store data in a *Cosmos DB* database. Let's create one.

* Database `workshop`
* Collection `faces`
* Partition key blank
* Initial Throughput 400

[Detailed instructions](https://github.com/microsoft/computerscience/blob/master/Events%20and%20Hacks/Student%20Hacks/mood-detector-workshop/Steps/SaveTheResultsToADatabase.md)


---
## Save the face details to a database (continued)

Add the Cosmos SDK to the `requirements.txt`
```
azure.cosmos
```

Install the requirements
```
pip3 install -r requirements.txt
```

---
## Save the face details to a database (continued)

Import the SDKs

```python
import azure.cosmos.cosmos_client as cosmos_client
```

Initialize the Cosmos DB client

```python
cosmos_url = ''
cosmos_primary_key = ''
cosmos_collection_link = 'dbs/workshop/colls/faces'
client = cosmos_client.CosmosClient(url_connection=cosmos_url,
                                    auth = {'masterKey': cosmos_primary_key})
```

Using the Cosmos DB extension, get the connection string and replace the variable values.

---
## Save the face details to a database (continued)

Now use the Cosmos DB SDK

In the `upload_image` function, inside the loop after the `doc` is created update the code:

```
...
for face in faces:
  ...
  client.CreateItem(cosmos_collection_link, doc)
...
```

Deploy the code using the App Service extension.

---
## Call the Web Api from the photo taking app

Add `requests` to `requirements.txt` and install via
```
pip3 install -r requirements.txt
```

Add these imports to `picturetaker.py`
```
import requests
import base64
```

---
## Call the Web Api from the photo taking app
Add the image upload code

```python
imageUrl = 'https://<Your Web App>.azurewebsites.net/image'

def upload(frame):
  data = {}
  img = cv2.imencode('.jpg', frame)[1]
  data['image'] = base64.b64encode(img).decode()
  requests.post(url=imageUrl, json=data)
```


---
## Call the Web Api from the photo taking app

Add a call to the upload function.

```
...
if k%256 == 32:
  upload(frame)
  break
...
```

Start the Debugger

---
## Create a web page to view the results

Create a template to display the emotions stored in the DB

Create a folder `templates` and file `home.html` within that folder.

```
<!doctype html>
<html>
  <body>
    <table border = 1>
      <tr>
        <td>Emotion</td>
      </tr>
      {% for row in result %}
        <tr>
          <td> {{ row.emotion }} </td>
        </tr>
      {% endfor %}
    </table>
  </body>
</html>
```


---
## Create a web page to view the results

Now update `app.py` and add this import
```python
from flask import Flask, request, render_template
```

Replace the `home` function with this code
```python
@app.route('/')
def home():
  docs = list(client.ReadItems(cosmos_collection_link))
  return render_template('home.html', result = docs)
```

---
## Cleaning up

Delete the resource group we created by visiting the [Azure Portal](https://portal.azure.com).

---
## Thank you!
Everything covered here is available at
### [aka.ms/hackcbsworkshop](https://aka.ms/hackcbsworkshop)

You can follow me on social media at `@berndverst`

