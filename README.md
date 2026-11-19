# Calvin for Homey Pro

A [Homey Pro](https://homey.app/) app for [Calvin](https://github.com/osterbergsimon/calvin), the
self-hosted family dashboard. Each Calvin wall screen becomes a Homey device: turn it on or off from
the Homey app, by voice or in a flow, see whether it is held, and get told when it goes offline.

> **Status:** being designed. There is no app code yet. See the
> [design](docs/superpowers/specs/2026-11-20-homey-app-design.md) and the
> [plan](docs/superpowers/plans/2026-11-20-homey-app.md).

## What you need

- A **Homey Pro** (any model) on firmware **12.9.0 or newer**. Homey Cloud (Homey Bridge) is not
  supported: it cannot reach a server on your home network.
- A **Calvin server with screen power from home automation** on the same network as the Homey.
  Older Calvin versions do not have it; the app tells you when Calvin is too old.
- A computer with [Node.js](https://nodejs.org/) and the
  [Homey CLI](https://apps.developer.homey.app/the-basics/getting-started/homey-cli), to install the app.

## Make a Calvin key for the Homey

1. In Calvin, open **Settings › Security › API keys** (sign in with the admin password).
2. **Create key**, name it `Homey`, and pick **Turn screens on and off**.
3. Copy the key. Calvin shows it only once.

This key can only switch screens and read their state. It cannot change settings or read backups.
Details: [Home automation](https://github.com/osterbergsimon/calvin/blob/develop/docs/integrations/HOME_AUTOMATION.md)
in the Calvin docs.

## Install

The app is not in the Homey App Store yet. Install it from this repository:

```bash
npm install --global homey
git clone https://github.com/osterbergsimon/calvin-homey.git
cd calvin-homey
npm ci
homey login        # opens a browser for your Athom account
homey select       # choose your Homey Pro
homey app install
```

Then, in the Homey app, add a device, choose **Calvin**, and enter Calvin's address and the key.
To update, pull the new version and run `homey app install` again.

## Good to know

- Calvin is usually reached over plain HTTP. Anyone who can watch your home network traffic could
  copy the key. Put Calvin behind HTTPS if that matters in your home; the app accepts an `https://`
  address.
- The app checks Calvin about every 30 seconds, so a change made in Calvin can take that long to
  show in Homey.
- If you remove the app, the key still works in Calvin. Revoke it in **Settings › Security › API keys**.

## License

[GPL-3.0](LICENSE).
