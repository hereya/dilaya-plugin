# Connect Dilaya to your AI

You drive Dilaya **from your AI**: you talk to it, and Dilaya gives it the hands to build your software, store your information and put it online. Your software itself runs outside: it has its own address and keeps running once you close the conversation. For that, the two need to be linked **once**. It takes two minutes, and there is nothing to install on your computer.

**Works with Claude and ChatGPT.** You need a subscription to your AI, paid separately: Claude Pro or ChatGPT Plus at least.

## In Claude

Dilaya installs as a **plugin**: nothing to copy, nothing to configure.

1. In Claude (claude.ai or the app), open **"Customize"** then **"Plugins"**, search for **Dilaya** and click **"Add"**. You can also get there from the "Connect Dilaya" button of your customer area on dilaya.eu.
2. In the plugin's **"Connectors"** tab, click **"Connect"** next to Dilaya: the dilaya.eu authorisation page opens (see below).
3. In Claude's **settings**, at the very bottom, **"Allowed domains"**: choose **"Package managers"** and add `*.dilaya.eu`. In a **Claude Team or Enterprise** organisation, the organisation's owner does this step.

On a **phone**, the plugin is in the same place; if you cannot see it, install it once from a computer: it counts for all your devices.

## In ChatGPT

Dilaya is added as a **custom MCP server** (ChatGPT Plus at least recommended). Do it from the **browser** or the **desktop app**: the mobile app cannot add a custom MCP server.

1. Open **chatgpt.com/plugins?view=personal&directoryTab=personal** (the Plugins page, **"Personal"** tab), click **"+"** then **"Add custom MCP server"**.
2. Name: **Dilaya**.
3. Connection: **Public endpoint** → paste `https://app.dilaya.eu/mcp`.
4. Authentication is detected on its own: the dilaya.eu authorisation page opens (see below).
5. Accept the warning (**"I understand and want to continue"**) then **"Create as a plugin"**.
6. Open a **new conversation** and say: "Use Dilaya's guide tool".

## Allow access (the dilaya.eu page)

- **Sign in** with the email address of your Dilaya account: you receive a **code by email**, no password to remember. No account yet? That code creates it.
- **Tick your organisation** (your workshop): the one whose software your AI may use. If you have several, tick every one you want. No organisation yet? The page lets you **create it right there**: a name, an identifier, done.
- Click **"Approve"**.

You are back in the conversation: it is connected, and you will not have to do it again.

## Say what you want

Open a conversation and talk normally. For example:

> "With Dilaya, I want a notebook to keep track of my customers and their orders."

Your AI recognises Dilaya and gets to work.

## Frequently asked questions

**Which plan do I need?** Claude Pro or ChatGPT Plus at least. That is a quality recommendation, not a technical limit: below it the connection works, but the AI lacks the mode that lets it carry real work from start to finish.

**What if I switch AI?** Your software, your data, your files and their instructions stay at Dilaya. Connect your new AI and it picks up where the other one left off. What follows: your software, your data, your files, your sites, your instructions, your Telegram bot and your scheduled automations. What stays tied to Claude: live dashboards (Claude Cowork) and the code agent (Claude Code).

**Another AI?** Any AI that can add an MCP server can connect to Dilaya through the same authorisation page. Ask us for the steps.

**Several people in the same workspace?** Each person connects Dilaya in **their own** AI, with **their own** Dilaya account, once invited into the workspace.

**Disconnecting?** In Claude: "Customize" → "Plugins", then remove Dilaya. In ChatGPT: the Plugins page, then remove the Dilaya server. Your software and your information stay safe at Dilaya; you can reconnect whenever you like.
