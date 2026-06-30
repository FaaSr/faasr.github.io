# Function examples

The [FaaSr-Functions] repository has a collection of examples of FaaSr workflows and R/Python functions that you can use as a starting template to develop your own workflows.

Each folder in the repository includes:

- One or more JSON workflow configuration files which can be imported directly into the [FaaSr Workflow Builder Web UI]
- One or more source code functions (R and/or Python)
- A README describing the workflow and functions

These examples assume that you have completed the [FaaSr tutorial] and have set up your FaaSr-workflow repository and secrets as described there.

## Importing an example

To try an example, import its JSON into the Web UI the same way as in the tutorial:

- Open the [FaaSr Workflow Builder Web UI]
- Click *Upload* and paste the GitHub URL of the example's JSON file - for example `https://github.com/FaaSr/FaaSr-Functions/blob/main/WeatherVisualization/WeatherVisualization.json` - then click *Import from GitHub URL*
- Edit the compute servers and data stores as needed (for example, set your GitHub username), then download, register, and invoke it as described in the [tutorial][FaaSr tutorial]

## Available examples

| Example | What it demonstrates |
| --- | --- |
| **tutorial** | The basic two-step workflow used in the tutorial (`tutorial.json`). The folder also includes `tutorialRpy.json` (an R function followed by a Python function), `tutorialLarger.json` (a larger DAG), and `tutorialOW.json` (the workflow running on OpenWhisk). |
| **WeatherVisualization** | A get → process → plot weather pipeline (`WeatherVisualization.json`). Variants demonstrate conditional invocation (`WeatherVisualizationConditional.json`) and using multiple data stores (`WeatherVisualizationDataStores.json`). |
| **WeatherGeographicPlot** | A weather pipeline that produces a geographic plot (`WeatherGeographicPlot.json`), with a ranked/concurrent variant (`WeatherGeographicPlotRanked.json`). |
| **WeatherAPISecrets** | Using workflow secrets to pass an API key (an OpenWeather API key) into a function (`WeatherAPISecrets.json`). |
| **idigbio_media_download** | Downloads media records from the [iDigBio](https://idigbio.org) biodiversity aggregator via the `ridigbio` API, queried by genus, and stores them in an S3 bucket (`idigbio.json`). |
| **nullstart** | The simplest possible FaaSr function - it logs a short string and takes no arguments (`nullstart.json`); useful as a start node that triggers concurrent executions. |

[FaaSr-Functions]: https://github.com/FaaSr/FaaSr-Functions
[FaaSr Workflow Builder Web UI]: workflows.md
[FaaSr tutorial]: tutorial.md
