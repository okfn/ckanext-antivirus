[![Tests](https://github.com/okfn/ckanext-antivirus/workflows/Tests/badge.svg?branch=main)](https://github.com/okfn/ckanext-antivirus/actions)

# ckanext-antivirus

Scan all files with an antivirus before publishing them.

Files uploaded as CKAN resources are sent to an external scanning service in the background. They can't be downloaded until the service reports them as clean. Uploads stay fast, because users don't wait for the scan to finish.

**Status:** under development. This README describes the intended behavior, and not all of it is implemented yet.

## How it works

1. A user uploads a file through the UI or the API. CKAN stores it, records it as `pending` and queues a background job.
2. The job sends the file to the scanning service, which answers with a `scan_id`.
3. The job checks the result periodically and records it.
4. The file can be downloaded only if it is `clean`.

Each uploaded file has one of these statuses:

| Status     | Meaning                                              | Can be downloaded?                       |
| ---------- | ---------------------------------------------------- | ---------------------------------------- |
| `pending`  | Waiting for the result of a scan                     | Only with `allow_unverified_downloads`   |
| `clean`    | No malware found                                     | Yes                                      |
| `infected` | Malware found                                        | Never                                    |
| `failed`   | The scan could not be completed                      | Only with `allow_unverified_downloads`   |
| `legacy`   | Uploaded before installing this extension, unscanned | Only with `allow_unverified_downloads`   |

Files with no recorded status are also blocked. Scan results are recorded in a database table and in the application logs. Administrators can review blocked files.

The status is stored in the database, not in Redis. If queued jobs are lost, `ckan antivirus requeue` creates them again.

## Requirements

Compatibility with core CKAN versions:

| CKAN version    | Compatible?   |
| --------------- | ------------- |
| 2.10 and earlier | not tested   |
| 2.11            | yes           |
| 2.12            | not yet       |

This extension also requires:

- **A scanning service.** Any service that implements the [scanning service API](#scanning-service-api) works.
  The extension doesn't depend on a specific antivirus.
- **A CKAN background worker** listening on the `antivirus` queue.

## Installation

To install ckanext-antivirus:

1. Activate your CKAN virtual environment, for example:

       . /usr/lib/ckan/default/bin/activate

2. Clone the source and install it on the virtualenv:

       git clone https://github.com/okfn/ckanext-antivirus.git
       cd ckanext-antivirus
       pip install -e .
       pip install -r requirements.txt

3. Add `antivirus` to the `ckan.plugins` setting in your CKAN config file (by default the config file is located at `/etc/ckan/default/ckan.ini`).

4. Set `ckanext.antivirus.service_url` in the same file.

5. Create the database tables:

       ckan -c /etc/ckan/default/ckan.ini db upgrade -p antivirus

6. Start a background worker for the `antivirus` queue:

       ckan -c /etc/ckan/default/ckan.ini jobs worker antivirus

7. Schedule the command that recreates lost jobs, for example every 10 minutes:

       ckan -c /etc/ckan/default/ckan.ini antivirus requeue

### Installing on a site with existing files

Register existing files before enabling the plugin. Otherwise they have no status, and all of them are blocked:

    ckan -c /etc/ckan/default/ckan.ini antivirus backfill

Then scan them in batches:

    ckan -c /etc/ckan/default/ckan.ini antivirus rescan

## Config settings

    # URL of the scanning service (required).
    ckanext.antivirus.service_url = http://antivirus:8000

    # Allow downloading files with status pending, failed or legacy. Files with
    # status infected are always blocked (optional, default: false).
    ckanext.antivirus.allow_unverified_downloads = false

    # Seconds between checks for the result of a scan (optional, default: 15).
    ckanext.antivirus.poll_interval = 15

    # Attempts before a file is marked as failed (optional, default: 3).
    ckanext.antivirus.max_retries = 3

    # Seconds allowed for a scan, including sending the file
    # (optional, default: 3600).
    ckanext.antivirus.scan_timeout = 3600

    # Minutes without activity before a pending file gets a new job
    # (optional, default: 30).
    ckanext.antivirus.stale_minutes = 30

The maximum file size of the scanning service must be equal to or greater than `ckan.max_resource_size`. Otherwise large files end up as `failed`.

## Commands

| Command                       | Description                                              |
| ----------------------------- | -------------------------------------------------------- |
| `ckan antivirus requeue`      | Creates jobs again for `pending` files that lost theirs  |
| `ckan antivirus backfill`     | Records existing files as `legacy`                       |
| `ckan antivirus rescan`       | Queues `legacy` files for scanning, in batches           |
| `ckan antivirus retry-failed` | Queues `failed` files for scanning again                 |

## Scanning service API

A scanning service must provide these endpoints:

| Request                | Response                                                                                  |
| ---------------------- | ----------------------------------------------------------------------------------------- |
| `POST /scans`          | `202` with `{"scan_id": "..."}`. The request body is the file, with a `Content-Length`.   |
| `GET /scans/<scan_id>` | `200` with the scan details, or `404` if the service doesn't know this scan.              |

`POST /scans` can also answer `413` if the file is too large, and `429` or `507` if the service has no capacity at the moment.

Scan details look like this:

```json
{
  "scan_id": "970972aa-3630-482d-9af9-4c2e3be73984",
  "status": "done",
  "verdict": "infected",
  "signature": "Eicar-Test-Signature",
  "engine": "ClamAV 1.5.4/28171/Sun Nov  1 06:27:34 2026",
  "size": 68,
  "sha256": "275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f",
  "details": null
}
```

- `status` is `queued`, `scanning` or `done`.
- `verdict` is `clean`, `infected` or `error` when `status` is `done`.

CKAN accepts a `clean` verdict only if `size` and `sha256` match the file it sent. If the service answers `404`, CKAN sends the file again.

The service is expected to run in a private network. Authentication is not supported yet.

## Developer installation

To install ckanext-antivirus for development, activate your CKAN virtualenv and do:

    git clone https://github.com/okfn/ckanext-antivirus.git
    cd ckanext-antivirus
    pip install -e .
    pip install -r dev-requirements.txt

## Tests

To run the tests, do:

    pytest --ckan-ini=test.ini

## License

[AGPL](https://www.gnu.org/licenses/agpl-3.0.en.html)
