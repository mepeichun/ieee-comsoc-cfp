# Academic Call for Papers Tracker

Live site: <https://mepeichun.github.io/ieee-comsoc-cfp/>

This project collects special-issue calls for papers from selected engineering
and science journals and magazines and generates a sortable static HTML page.
The interface groups sources into IEEE ComSoc, IEEE Computer Society, and
other publishers.

## Supported publications

### IEEE Communications Society

- IEEE Communications Magazine
- IEEE Network
- IEEE Wireless Communications
- IEEE Internet of Things Magazine
- IEEE Journal on Selected Areas in Communications
- IEEE Transactions on Cognitive Communications and Networking
- IEEE Transactions on Green Communications and Networking
- IEEE Transactions on Network Science and Engineering
- IEEE Transactions on Network and Service Management
- IEEE Internet of Things Journal

### IEEE Computer Society

The tracker collects all non-ongoing journal CFPs from the IEEE Computer
Society's aggregated calls-for-papers page. Results are grouped by their actual
journal name in the generated interface.

### Other publishers and societies

- IEEE Transactions on Vehicular Technology (TVT)
- IEEE Journal of Selected Topics in Signal Processing (JSTSP)
- *npj Wireless Technology*

## Project structure

- `config.py`: journal catalogue and runtime defaults.
- `models.py`: shared CFP and journal result models.
- `parsers.py`: site-specific HTML parsers and the parser registry.
- `guest_editor_extractor.py`: DeepSeek guest-editor extraction and cache logic.
- `guest_editor_cache.json`: committed results for already parsed detail pages.
- `renderer.py`: static page rendering.
- `main.py`: command-line orchestration.
- `utils.py`: SMTP notification and Git publishing helpers.

## Running

Install dependencies:

```console
python -m pip install -r requirements.txt
```

Run one update and exit:

```console
python main.py --once
```

`--once` is implicit, so `python main.py` does the same thing.

Run continuously every six hours:

```console
python main.py --watch
```

Optional flags:

- `--output PATH`: choose the generated HTML file.
- `--interval-hours HOURS`: change the watch interval.
- `--notify`: email newly discovered CFPs.
- `--publish`: commit and push `index.html`.

Email notifications read these environment variables:

- `CFP_SMTP_USER`
- `CFP_SMTP_PASSWORD`
- `CFP_SMTP_HOST` (optional, defaults to `smtp.163.com`)
- `CFP_SMTP_PORT` (optional, defaults to `465`)

IEEE Computer Society and JSTSP detail pages, plus TVT CFP PDFs, use DeepSeek
only when a CFP is not already present in `guest_editor_cache.json`:

- `DEEPSEEK_API_KEY`: required for new, uncached CFP detail pages.
- `DEEPSEEK_MODEL`: optional; defaults to `deepseek-v4-flash`.

The cache key is the SHA-256 hash of the detail-page URL. Keep the cache file
when deploying or restarting watch mode so the same CFP is not charged twice.
For TVT, only entries inside the `Open Call for Papers` section are collected;
PDF text is extracted locally with `pypdf` before it is sent to DeepSeek.

The JSTSP parser follows the newest-first listing and stops as soon as it sees
the first expired deadline. Only open calls have their detail pages fetched and
their Guest Editors extracted, so older pages do not incur network or API work.

The npj Wireless Technology parser discovers the total number of listing pages,
visits every open collection, and extracts editor names directly from Nature's
structured HTML. It does not use DeepSeek.

## Debian background deployment

Install the dependencies with Python 3:

```console
python3 -m pip install -r requirements.txt
```

Create a private environment file for the SMTP account:

```console
printf "CFP_SMTP_USER='%s'\nCFP_SMTP_PASSWORD='%s'\nDEEPSEEK_API_KEY='%s'\n" \
  'findlab@163.com' \
  'your-smtp-authorization-code' \
  'your-deepseek-api-key' > ~/.cfp-env
chmod 600 ~/.cfp-env
```

For 163 Mail, `CFP_SMTP_PASSWORD` should normally be the SMTP client
authorization code rather than the mailbox login password. Do not commit this
file or its contents to Git.

Load the variables and run the tracker continuously in the background:

```console
set -a
source ~/.cfp-env
set +a
nohup python3 main.py --watch --notify > cfp.log 2>&1 & echo $! > cfp.pid
```

To also commit and push the generated `index.html`, add `--publish`:

```console
set -a
source ~/.cfp-env
set +a
nohup python3 main.py --watch --notify --publish > cfp.log 2>&1 & echo $! > cfp.pid
```

Follow the background process log:

```console
tail -f cfp.log
```

Stop the background process:

```console
kill "$(cat cfp.pid)"
```

## Adding a journal

If the journal uses a currently supported page layout, add a `JournalConfig`
entry to `JOURNALS` in `config.py`. Set its `collection` to `comsoc`,
`computer`, or `others`; omitted values default to `comsoc`.

For a new layout:

1. Add a parser in `parsers.py` that returns `CallForPaper` objects.
2. Register it in the `PARSERS` dictionary.
3. Add the corresponding `JournalConfig` entry in `config.py`.

This is the intended extension point for additional journals and publishers.

## Tests

The tests use local HTML fixtures and do not access the network:

```console
python -m unittest discover -s tests -v
```
