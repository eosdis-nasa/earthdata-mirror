# Earthdata Mirror

Scripts to mirror all URLs from a NASA CMR query to a filesystem / Amazon S3.

These were developed as quickly as possible as part of a data rescue effort, with refinements only
happening as the scripts were moving data. Apologies for code quality, but I hope they're helpful
to someone.

# Installation Prerequisites

The scripts list their Python dependencies inline, following [PEP-723](https://peps.python.org/pep-0723/), supported by [PDM](https://pdm-project.org/en/latest/) and [Hatch](https://hatch.pypa.io/latest/).

Or just use a relatively recent Python 3 (tested on 3.11) and run

`pip3 install earthaccess asyncio aiohttp aiotools`

# bin/harvest

```
usage: harvest [-h] [-c CONCURRENCY] [--start-i START_I] [--end-i END_I] [--whitelist-file WHITELIST_FILE] config_file output_dir

Mirror files from a CMR query to an output directory

positional arguments:
  config_file           Configuration file defining non-host-specific parameters. See sedac/config.json
  output_dir            Output directory to place mirrored files in

options:
  -h, --help            show this help message and exit
  -c CONCURRENCY, --concurrency CONCURRENCY
                        Maximum number of concurrent requests
  --start-i START_I     Index of the first result to mirror
  --end-i END_I         Index of the last result to mirror, exclusive. -1 (default) = no end index
  --whitelist-file WHITELIST_FILE
                        File containing a newline-delimited list of all URLs to get
```

Command-line options are ones that tend to vary between machines and processes for the same job.
See [config/sedac-all.json](config/sedac-all.json) for an example config file of more static options.

Config keys:
 * `name`: A unique name that will be used for a folder for caching and output generation
 * `query`: An [earthaccess](https://earthaccess.readthedocs.io/en/latest/) query for collections and granules
    corresponding to data that needs to be downloaded
 * `hosts_to_paths`: A mapping of hostnames to the directory path they should be mirrored under. Required for all hosts that are
   not ignored by the `ignore` key
 * `task_class`: (Optional) The class to instantiate to perform downloads. Useful when some URLs have unconventional logic.
 * `ignore`: (Optional) A list of strings which, if present in a URL, should cause that URL to not be mirrored
 * `fixes`: (Optional) A list of two-element lists corresponding to string replacements that should be performed on URLs

## Outputs

In addition to mirroring data, it outputs cache and logs to the local directory specified by the `name` in the config.

 - `collections.json` and `granules.json` cache the CMR search, which can be expensive. If new data arrives, delete the cache.
 - `success.txt` contains URLs that succeeded in transfer, one per line. These are skipped on subsequent runs.
 - `missing.txt` contains URLs that could not be found, one per line. These are skipped on subsequent runs.
 - `data_error.jsonl` contains JSON entries of errors encountered in accessing data files. These will be retried on subsequent runs.
 - `non_data_error.jsonl` contains JSON entries of errors encountered in accessing non-data files like documentation and browse, which
    tend to be more common and less severe than data errors. These will be retried on subsequent runs.

## Other Notes

Implements some features that proved necessary for rapid iteration and data transfer:
  - Graceful stopping. Be patient on Ctrl-C. It will try to complete downloads rather than
    leaving half-done downloads to reconcile later
  - Skipping successful prior downloads
  - Skipping URLs found to be 404s, of which there are sadly many
  - Skipping anything that redirects to Earthdata, including DOIs and DAAC sites
  - Caching all CMR results to avoid re-crawling CMR each run
  - Splitting the work by numeric ranges to allow running on multiple machines
  - Buffering responses to avoid costly multipart uploads to S3

In practice, I ran this in an EC2 instance with an Amazon S3 Mountpoint in the target directory, which moved all
downloaded files to S3. I strongly recommend running in a detached `screen` session.
