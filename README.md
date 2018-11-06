<h1 align="center">
  Datasheet Server
</h1>

<p align="center">
  <strong>Turn spreadsheet data into a structured, dynamic API. </strong><br>
</p>
<!-- <p align="center">
  <a href="https://github.com/gatsbyjs/gatsby/blob/master/LICENSE">
    <img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="Gatsby is released under the MIT license." />
  </a>
  <a href="#configru">
    <img src="https://circleci.com/gh/gatsbyjs/gatsby.svg?style=shield" alt="Current CircleCI build status." />
  </a>
  <a href="https://www.npmjs.org/package/gatsby">
    <img src="https://img.shields.io/npm/v/gatsby.svg" alt="Current npm package version." />
  </a>
  <a href="https://npmcharts.com/compare/gatsby?minimal=true">
    <img src="https://img.shields.io/npm/dm/gatsby.svg" alt="Downloads per month on npm." />
  </a>
  <a href="https://gatsbyjs.org/docs/how-to-submit-a-pr/">
    <img src="https://img.shields.io/badge/PRs-welcome-brightgreen.svg" alt="PRs welcome!" />
  </a>
</p> -->

<h3 align="center">
  <a href="#overview">Overview</a>
  <span> · </span>
  <a href="#configuration">Configuration</a>
  <span> · </span>
  <a href="#quickstart">Quickstart</a>
</h3>

Datasheet server makes resources from a spreadsheet available as a structured API.

- **Manage structured data without developers**. Allows anyone to dynamically manage data, while simultaneously making this data available in a reliably structured format for frontend interfaces and other programmatic applications.
- **Designed for a dynamic workflow**. References data in a spreadsheet source as a ground truth, but adds a layer of indirection that keep API routes from breaking when changes are made.
- **Customisable data transformation**. Easily create new blueprints to specify the API structure that should be presented from source data.
- **Extensible architecture**. Currently supports Google Sheet as a source and a REST-like query language, but structured modularly with an intention to support other sources and query languages.

## [Overview](#overview)
Datasheet server is a Node server developed at [Forensic Architecture](https://forensic-architecture.org) to make data that is being dynamically modified by researchers concurrently consumable for programmatic applications as an API. We use spreadsheets extensively to keep track of information during [our investigations](http://forensic-architecture.org/cases), and we often want to make this data available via structured queries to make it available in a frontend web application, a game engine, or another use case.

Querying data directly from spreadsheets is brittle, as it relies on the maintenance of a rigid structure in the sheets at all times. By putting Datasheet Server as a proxy that sits in between source sheets and their consumers, it is possible to dynamically modify sheets without breaking applications. A data admin can then use Datasheet Server to ensure that applications always receive eligible data, without foregoing the spreadsheets as sources of truth.  

### Design Concepts
The codebase currently only supports Google Sheets as a source, and a REST-like format as a query language. It is designed, however, with extensibility in mind.

**Sources**     
A source represents a sheet-like collection of data, such as a Google Sheet. A source has one or more **tabs**, each of which contains a 2-dimensional grids of cells. Each cell contains a body of text (a string).

**Resources**     
The data from sources are made available as resources, which are structured blocks of data that are granularly accessible through a query language. Resources are the outfacing aspect of Datasheet Server, and represent the only kind of data that can be queried by applications. Each resource is configured with one or more **query languages**. (Currently only a REST-like query language supported.)

**Blueprints**       
Blueprints are a data structure that represent the way that infromation from **sources** are to be turned into **resources**. For each tab in a source, there is a corresponding Blueprint. Blueprints are created through a [blueprinter function](/src/blueprinters) invoked on the raw data from a source tab.

Blueprints are JSON objects. There have two forms:

1. _desaturated_ -- describes the resources and query languages available on data from a source tab.
2. _saturated_ --  both describes resources available on data from a source etab, and contains that data.

A desaturated Blueprint can be saturated by retrieving its data from the server's **model layer**, which stores tab data from sources.

A JSON catalogue of the available blueprints (desaturated) in a server is available at `/api/blueprints`.

## [Configuration](#configuration)
Copy the [example.config.js](/src/example.config.js) in the [src](/src) directory into a file named 'config.js'. Modify the options in this file accordingly:

| Option  | Description | Type |
| ------- | ----------- | ---- |
| port | The port at which the server will make data available.  | integer |
| googleSheets  | The configuration object for [Google Sheet](https://www.google.co.uk/sheets/about/) data sources. See the [Sources](#source-google-sheets) section below. | object |

#### [Sources](#sources)
###### [Google Sheets](#source-google-sheets)
In order to make the data from a Sheet accessible to the server, you need to [create a service account](https://cloud.google.com/iam/docs/creating-managing-service-accounts). Once created, give the service account email access to each Sheet from which you want to serve data. ('View Only' access is sufficient, as the server never modifies data.)

| Option | Description | Type |
| ------ | ----------- | ---- |
| email | The email address of the service account. This is available in the downloadable service account JSON in the `client_email` field. | string |
| privateKey | The private key associated with the service account. This is available in the downloadable service account JSON in the `private_key` field. | string |
| sheets | A list of objects, one for each sheet that is being used as a source. Each sheet object has a `name` (String), an `id` (String), and a `tabs` (object) field, which are explained below. | object |

Each Google Sheet being used as a as source requires a corresponding object in `sheets`. The object should be structured as follows:

| Option | Description | Type |
| ------ | ----------- | ---- |
| name | Used to refer to data served from this source | string |
| id | The ID of the sheet in Google. (You can find it in the address bar when the Sheet is open in a browser. It is the string that follows 'spreadsheets/d/'). | string |
| tabs | An object that maps each tab in the source to one or more Blueprinters. All of the Blueprinters in the [blueprinters folder](/lib/blueprinters) are available through a single import as at the top of [example.config.js](/src/example.config.js). <br> To correctly associate a Blueprinter, the object key needs to be _the tab name with all lowercase letters, and spaces replaced by a '-'_. For example, if the tab name in Google Sheets is 'Info About SHEEP', the object key should be 'info-about-sheep'. <br> The value should be the Blueprinter function that you want to use for the data in that tab. If you require more than one endpoint for a single tab, you can support multiple blueprinters by making the item an array. See the example of a configuration object below. <br>TODO: no Blueprinter is used by default. | object |

###### Example Configuration Object
```js
import BP from './lib/blueprinters'

export default {
  port: 4040,
  googleSheets: {
    email: 'project-name@reliable-baptist-23338.iam.gserviceaccount.com',
    privateKey: 'SOME_PRIVATE_KEY',
    sheets: [
      {
        name: 'example',
        id: '1s-vfBR8Uy-B-TLO_C5Ozw4z-L0E3hdP8ohMV761ouRI',
        tabs: {
          'objects': BP.byRow,
          'fruit': [BP.byRow, BP.byID],
        }
      },
    ]
  }
}

```


## [Quickstart](#quickstart)
Clone the repository to your local:
```
git clone https://www.github.com/forensic-architecture/datasheet-server
```
### Run with Docker
To create a new instance of the server with [Docker](https://www.docker.com/) installed, clone the repository, create a `config.js`, and build the image:
```sh
docker build -t datasheet-server .
```
You can then run the container and make available the relevant port (`4040` by default):
```sh
docker run -d -p 4040:4040 datasheet-server
```
If running on a cloud server, you'll probably need to zero out the host IP:
```sh
docker run -d -p 0.0.0.0:4040:4040 datasheet-server
```

### Run locally
Install dependencies:
```sh
yarn # npm install
```
And run development server:
```sh
yarn dev # npm run dev
```

## Contribute

### [Code of Conduct](CODE_OF_CONDUCT.md)

Please read our adopted [code of conduct](CODE_OF_CONDUCT.md) before contributing, so that you can understand what actions will and will not be tolerated.

### [Contributing Guide](CONTRIBUTING.md)

Read our [contributing guide](CONTRIBUTING.md) to learn about our development process, how to propose bugfixes and improvements.

## Community
If you have any questions or just want to chat, please join our team [fa_open_source](https://keybase.io/team/fa_open_source) on Keybase for community discussion. Keybase is a great platform for encrypted chat and file sharing.

## License

TimeMap is distributed under the MIT License.
