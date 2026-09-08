# Privacy Policy — dbmysql

This Kiro power does not collect, store, transmit, or process any personal data, telemetry, or usage information on behalf of its author.

## What this power does

`dbmysql` connects the Kiro agent directly to a MySQL database that you configure, using your own credentials, supplied exclusively through system environment variables that you define and control. The connection runs entirely on your machine, between your local Kiro installation and the database server you specify.

## What this power does NOT do

- It does not send data to any server, service, or endpoint controlled by the author.
- It does not log, store, or transmit your database credentials anywhere. Credentials exist only as environment variables on your own system.
- It does not read credential-bearing files (`.env`, `application.properties`, etc.).
- It does not collect telemetry, analytics, or usage statistics.
- It does not share data with third parties.

## Third-party components

This power uses the third-party npm package [`@benborla29/mcp-server-mysql`](https://www.npmjs.com/package/@benborla29/mcp-server-mysql) to communicate with MySQL via the Model Context Protocol. That package's own privacy practices are governed by its own license and documentation, independent of this power. Review it directly before using this power against production data.

## Data flow

All data flows directly between your local Kiro agent and the MySQL instance you configure — nothing passes through, or is stored by, any infrastructure controlled by this power's author.

## Contact

For questions about this policy, see the contact information in the [README](./README.md).

## Changes

This policy may be updated as the power evolves. Changes will be reflected in this file's version history in the repository.
