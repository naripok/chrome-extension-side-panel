# Chrome Extension SidePanel

## Development

1. Run:

```bash
npm install
npm start
```

2. Add the extension, from `/dist` to chrome://extensions/ in developer mode.

3. Click in the extension icon and the sidepanel should open.

## What is this about?

### The sidePanel manifest v3 entry

Every Chrome extension is required to have a manifest file that determines what features the extension will utilize. For the side panel, in this tutorial, we use the action to open the side panel, so you'll need these two entries in your manifest file.

```typescript
action: { default_title: 'Click to open panel' },
side_panel: { default_path: 'assets/sidePanel.html' },
```

The extension uses scripting to build the manifest.json file, which you can find under config/manifest.

In our tutorial, we load the react app via the app.html file, which is loaded by the side panel. In the background script index.ts file you'll see the following code:

```typescript
chrome.sidePanel
  .setPanelBehavior({ openPanelOnActionClick: true })
  .catch((error: any) => console.error(error));
```

### Loading the content scripts when the panel is open

Now getting the content scripts to run when the panel is open is the somewhat tricky part. In order to create a non-intrusive extension experience, we wanted to make sure the app was only registering user mouse events when the side panel was open. Thus, we implemented a solution using Chrome's port messaging functionality.

#### First, the background script loads and listens for a port connection from the side panel and injects the content scripts

```typescript
chrome.runtime.onConnect.addListener((port) => {
  port.onMessage.addListener(async (msg) => {
    if (port.name === PortNames.SidePanelPort) {
      if (msg.type === 'init') {
        console.log('panel opened');

        await storage.setItem('panelOpen', true);

        port.onDisconnect.addListener(async () => {
          await storage.setItem('panelOpen', false);
          console.log('panel closed');
          console.log('port disconnected: ', port.name);
        });

        const tab = await getCurrentTab();

        if (!tab?.id) {
          console.error("Couldn't get current tab");
          return;
        }

        injectContentScript(tab.id);

        port.postMessage({
          type: 'handle-init',
          message: 'panel open',
        });
      }
    }
  });
});
```

#### Secondly, the background script listens for new tab connections and injects the content script only if the panel is open.

```typescript
chrome.tabs.onUpdated.addListener(async (tabId, changeInfo, tab) => {
  if (!(tab.id && changeInfo.status === 'complete')) return;

  console.log('tab connected: ', tab.url, changeInfo);

  if (await storage.getItem('panelOpen')) {
    console.log('panel open');
    injectContentScript(tabId);
  }
});
```

## Notes

This app uses webext-redux to create a redux store that seamlessly provides state updates between the background script and the Side Panel React application.

The app uses typescript and webpack to create the application scripts, and all configurations can be found under the config directory.
