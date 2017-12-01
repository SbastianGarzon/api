# Ship compendia and metadata

Shipments are used to deliver compendia and their metadata to third party repositories or archives.
This section covers shipment related requests, including repository file management.

## Packaging

The packaging of a compendium ensures a recipient can verify the integrity of the transported data.
Currently, the shipment process always creates [BagIt](http://tools.ietf.org/html/draft-kunze-bagit) bags to package a compendium.

## Supported recipients

Use the _recipient_ endpoint to find out, which repositories are available and configured.
The response is list of tuples with `id` and `label` of each repository.
The `id` is the repository identifier to be used in requests to the `/shipment` endpoint, e.g. to define the recipient, while `label` is a human-readable text string suitable for display in user interfaces.

`GET /api/v1/recipient`

```json
200

{
	"recipients": [{
		"id": "download",
		"label": "Download"
	}, {
		"id": "b2share_sandbox",
		"label": "Eudat b2share Sandbox"
	}, {
		"id": "zenodo_sandbox",
		"label": "Zenodo Sandbox"
	}]
}
```

An implementation may support one or more of the following repositories:

- `b2share` - [Eudat b2share](https://b2share.eudat.eu/)
- `b2share_sandbox` - [Eudat b2share Sandbox](https://trng-b2share.eudat.eu/)
- `zenodo` - [Zenodo Sandbox](https://zenodo.org)
- `zenodo_sandbox` - [Zenodo Sandbox](https://sandbox.zenodo.org)

The `download` recipient is a surrogate to enable shipping to the user's local storage.

## List shipments

This is a basic request to list all shipment identifiers.

`GET /api/v1/shipment`

```json
200

["dc351fc6-314f-4947-a235-734ab5971eff", "..."]

```

You can also get only the shipment identifiers belonging to a compendium id (e.g. `4XgD97`).

`GET /api/v1/shipment?compendium_id=4XgD97`

URL parameter:

- `compendium_id` - the identifier of a specific compendium

```json
200 

["dc351fc6-314f-4947-a235-734ab5971eff", "..."]

```

## Get a single shipment

`GET /api/v1/shipment/dc351fc6-314f-4947-a235-734ab5971eff`

```json
200 

{
  "last_modified": "2016-12-12 10:34:32.001475",
  "recipient": "zenodo",
  "id": "dc351fc6-314f-4947-a235-734ab5971eff",
  "deposition_id": "63179",
  "user": "0000-0001-6021-1617",
  "status": "shipped",
  "compendium_id": "4XgD97",
  "deposition_url": "https://zenodo.org/record/63179"
}
```

!!! note
    Returned deposition URLs (property `deposition_url`) from Zenodo as well as Eudat b2share (records) will only be functional after publishing.

## Create a new shipment

You can start a initial creation of a shipment, leading to transmission to a repository and creation of a deposition, using a `POST` request.

`POST /api/v1/shipment`

This **requires** the following parameters as `multipart/form-data` or `application/x-www-form-urlencoded` encoded data:

- `compendium_id` (`string`): the id of the compendium
- `recipient` (`string`): identifier for the repository

The following are **optional parameters**:

- `update_packaging` (`boolean`, default: `false`): the shipment creation only succeeds if a valid package is already present under the provided compendium identifier, or if no packaging is present at all and a new package can be created. In case a partial or invalid package is given, this parameter can control the shipment creation process: If it is set to `true`, the shipment package is updated during the shipment creation in order to make it valid, if set to `false` the shipment creation results in an error.
- `cookie` (`string`): an authentication cookie must be set in the request header, but it may also be provided via a `cookie` form parameter as a fallback
- `shipment_id` (`string`): a user-defined identifier for the shipment (see `id` in response)

!!! note "Required user level"
    The user sending the request to create a shipment must have the required [user level](user.md#user-levels).

### Creation response

The response contains the shipment document, see [Get a single shipment](#get-a-single.shipment).
Some of the fields are not available (have value `null`) until after [publishing](#publish-a-deposition), e.g. `deposition_url`.

```json
201

{
  "id": "9ff3d75e-23dc-423e-a6c6-6987ac5ffc3e",
  "recipient": "zenodo",
  "status": "shipped",
  "deposition_id": "79102"
}
```

If the recipient is the **download** surrogate, the response will be `202` and a zip stream with the Content type `application/zip`.

```
202
```
(zip stream starting point)

The download zip stream is also available under the url of the shipment plus `/dl`, once it has been created, e.g.:  

``` 
http://localhost:8087/api/v1/shipment/22e7b17c-0047-4cb9-9041-bb87f30de388/dl
```


### Shipment status

A shipment can have three possible status:

- `shipped` - a deposition has been created at a repository and completed the necessary metadata for publication.
- `published` - the contents of the shipment are published on the repository, in which case the publishment can not be undone.
- `error` - an error occurred during shipment or publishing.

To get only a shipment's current status you may use the sub-resource `/status`:

`GET api/v1/shipment/<shipment_id>/status`

```json
200

{
"id": "9ff3d75e-23dc-423e-a6c6-6987ac5ffc3e",
"status": "shipped"
}
```

### Publish a deposition

The publishment is supposed to have completed the status `shipped` where metadata requirements for publication have been checked.

!!!note
    Once published, a deposition can no longer be deleted on the supported repositories.

`PUT api/v1/shipment/<shipment_id>/publishment`


```json
200

{
"id": "9ff3d75e-23dc-423e-a6c6-6987ac5ffc3e",
"status": "published"
}
```
Note that a publishment is not possible if the recipient is the download surrogate which immediately results in a zip stream as a response.


### Files in a deposition

#### List deposition files

You can request a list of all files in a deposition and their properties with the sub-resource `/publishment`.

`GET api/v1/shipment/<shipment_id>/publishment`

```json
200

{
"files": [{
	"filesize": 393320,
	"id": "bae2a60c-bd59-47e1-a443-b34bb7d0a981",
	"filename": "4XgD9.zip",
	"checksum": "702f4db3e53b22176d1d5ddcda462a27",
	"links": {
		"self": "https://sandbox.zenodo.org/api/deposit/depositions/71552/files/bae2a60c-bd59-47e1-a443-b34bb7d0a981",
		"download": "https://sandbox.zenodo.org/api/files/31dc8f3d-df00-4d8a-bd99-64ef341372b3/4XgD9.zip"
	}
}]
}
```

You can find the `id` of the file you want to interact with in this json list object at `files[n].id`, where `n` is the position of that file in the array.
Files can be identified in this response by either their id in the depot, their filename or their checksum.

#### Delete a specific file from a deposition

You can delete files from a `shipped` shipment's deposition.
You must state a file's identifier, which can be retrieved from the shipment's deposition files property `id`, as the `file_id` path parameter.
Files for a `published` shipment usually cannot be deleted.

`DELETE api/v1/shipment/<shipment_id>/files/<file_id>`

```
204
```

## Error responses 

```json
400

{"error":"bad request"}
```

```json
403

{"error": "insufficient permissions"}
```
