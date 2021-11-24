# Solver testdata directory

This directory contains credentials that are used when running the test suite.
Because the tests actually perform live updates via the Name.com API, you need to supply valid Name.com credentials.

## Name.com username

Copy [`config.json.sample`](config.json.sample) to `config.json` and update the `username` field to use your Name.com username.

## Name.com API token

Copy [`namedotcom-credentials.yml.sample`](namedotcom-credentials.yml.sample) to `namedotcom.credentials` and update the `api-token` field to use your Name.com API token.
