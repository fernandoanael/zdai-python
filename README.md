# Zuva DocAI (ZDAI) Python Wrapper

# Wrapper

The API wrapper is designed to reflect the decoupled nature of ZDAI's microservices. Meaning, there is one wrapper 
class for each microservice.

## Setup

The following commands will set up the wrapper and test your credentials to make sure they're valid.

```terminal
python3 -m zdai --set-token <put token here>
python3 -m zdai --set-url <put url here>
python3 -m zdai --test connection
```

Add the zdai folder to your Python Interpreter's site-packages to make it easier to use/import in your projects.

Sometimes Python ```requests``` module throws ```urllib3``` Connection Errors, which may be addressable by installing
the below packages:

```pip3 install pyopenssl ndg-httpsclient pyasn1```

## Authorization

Every API call to the ZDAI requires an Authorization token in the header. You can provide these in the wrapper's
configuration file found in config/access.json and access them using the below method, or provide them from other sources.

### Method 1: Provide the token via the ZDAISDK Constructor

```python
from zdai import ZDAISDK, config

url, token = config.get_access()

sdk = ZDAISDK(url = url, token = token)
```

### Method 2: Provide the token via the config/access.json

```python
from zdai import ZDAISDK

sdk = ZDAISDK(from_config = True)
```

## Files

To create a file in ZDAI:

```python
from zdai import ZDAISDK

sdk = ZDAISDK(from_config = True)

with open('file_zones/upload_files/...', 'rb') as f:
    file, _ = sdk.file.create(content = f.read())
    file_id = file.id
```

## Fields
To get the AI models that can be used for document text extractions:

```python
from zdai import ZDAISDK

sdk = ZDAISDK(from_config = True)

fields, _ = sdk.fields.get()

for field in fields:
    print(f'{field.id}: {field.name}')
```

## Classification

To create a classification request on a file, as well as obtain the request's status:

```python
from zdai import ZDAISDK

sdk = ZDAISDK(from_config = True)

with open('file_zones/upload_files/...', 'rb') as f:
    file, _ = sdk.file.create(content = f.read())

classification_jobs, _ = sdk.classification.create(file_ids = [file.id])
classification_status, _ = sdk.classification.get(request_id = classification_jobs[0].id)
```

Note that the above accepts a list of ```file_ids```.

## Extraction

To create an extraction request on a file, as well as obtain the request's status and results:

```python
from zdai import ZDAISDK

sdk = ZDAISDK(from_config = True)

with open('file_zones/upload_files/...', 'rb') as f:
    file, _ = sdk.file.create(content = f.read())

extraction_jobs, _ = sdk.extraction.create(file_ids = [file.id], field_ids = ['<field_id>', '<field_id>'])
extraction_status, _ = sdk.extraction.get(request_id = extraction_jobs[0].id)

# Only successful if the extraction_status.status is complete or failed
results, _ = sdk.extraction.get_result(request_id = extraction_jobs[0].id)

for result in results:
    print(result)

```

Note that the above accepts a list of ```file_ids``` and a list of ```field_ids```.

## Training

To create a training request for a field, as well as obtain the request's status and accuracy and validation details:

```python
from zdai import ZDAISDK

sdk = ZDAISDK(from_config = True)

new_field_id, _ = sdk.fields.create(field_name = 'test1', description = 'descritpion')

with open('file_zones/upload_files/...', 'rb') as f:
    file1, _ = sdk.file.create(content = f.read())

training_locations = [{"start": 0, "end": 110}]
training_annotations = [{"file_id": file1, "locations": training_locations}]

training_request_id = sdk.fields.train(field_id = new_field_id, annotations = training_annotations)

training_status, _ = sdk.fields.get_training_status(field_id = new_field_id, request_id = training_request_id)

training_accuracy, _ = sdk.fields.get_accuracy(field_id = new_field_id)

validation_details, _, _ = sdk.fields.get_validation_details(field_id = new_field_id)

```

Note that at least 30 files are needed for well trained fields.

## Language

To create an language request on a file, as well as obtain the request's status:

```python
from zdai import ZDAISDK

sdk = ZDAISDK(from_config = True)

with open('file_zones/upload_files/...', 'rb') as f:
    file, _ = sdk.file.create(content = f.read())

language_jobs, _ = sdk.language.create(file_ids = [file.id])
language, _ = sdk.language.get(request_id = language_jobs[0].id)
```

Note that the above accepts a list of ```fild_ids```

# Examples

## How to obtain a document's text, document classification, language and field extractions

```python
from zdai import ZDAISDK, DocumentClassificationRequest, LanguageClassificationRequest, OCRRequest, FieldExtractionRequest
import time

sdk = ZDAISDK(from_config = True)

fields = sdk.fields.get()[0]

with open('path/to/file.ext', 'rb') as f:
    file, _ = sdk.file.create(content=f.read())

print(f'Created {file.id}')

field_names = ['Title', 'Parties', 'Date', 'Governing Law', 'Indemnity']

requests = []
requests.extend(sdk.ocr.create(file_ids = [file.id])[0])
requests.extend(sdk.classification.create(file_ids = [file.id])[0])
requests.extend(sdk.language.create(file_ids = [file.id])[0])
requests.extend(sdk.extraction.create(file_ids = [file.id], field_ids = [f.id for f in fields if f.name in field_names])[0])


while len(requests) > 0:
    for request in requests:
        print(request.type, request.id, request.status)
        request.update()
        if request.is_finished():
            if request.is_type(OCRRequest):
                text = request.get_text()
                print(f'Finished: {text[0:50]}')
            elif request.is_type(DocumentClassificationRequest):
                print(f'Finished: {request.classification}, {request.is_contract}')
            elif request.is_type(LanguageClassificationRequest):
                print(f'Finished: {request.language}')
            elif request.is_type(FieldExtractionRequest):
                for result in request.get_results():
                    print(result.field_id, result.text)

            requests.remove(request)
    time.sleep(2)
```

## Obtain a newly-submitted file's ID

```python
from zdai import ZDAISDK

sdk = ZDAISDK(from_config = True)

with open('file_zones/upload_files/...', 'rb') as f:
    file, _ = sdk.file.create(content = f.read())

print(f'The file was submitted - its file_id is {file.id} and it expires on {file.expiration}')
```
