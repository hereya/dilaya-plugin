# Dilaya

Dilaya turns a description of how you work into your own software. You describe what you do, in your own words, and Claude builds it: a database of your own, file storage, a web space with email sign-in, a Telegram bot, scheduled automations. Dilaya hosts it and keeps it running.

## What this plugin contains

- **The Dilaya connector** (`.mcp.json`): a remote MCP server at `https://mcp.dilaya.eu`. You sign in with your Dilaya account (e-mail and a one-time code, OAuth). Every tool call goes to that server only, for the organisation you approved.
- **21 skills** (`skills/`): the recipes Claude follows to create and change your apps (`dilaya-create-app`, `dilaya-frontend`, `dilaya-telegram`, …), the user guide and tutorials in French and English (`dilaya-guide`), and the entry point (`dilaya`).

The plugin runs nothing on your computer: no local server, no hook, no script. It sends nothing anywhere except to the Dilaya connector above.

## Requirements

A Dilaya account (https://dilaya.eu, 14-day trial, no card required).

## Support and privacy

- Support: contact@dilaya.eu · https://dilaya.eu/en/guide
- Privacy policy: https://dilaya.eu/en/privacy
- Publisher: Novopattern
