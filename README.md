# YouTube Auto Unsubscribe Script

A JavaScript script to automatically unsubscribe from non-verified YouTube channels using your browser's console. The script scrolls through your subscribed channels, detects non-verified channels, and unsubscribes from them while ensuring smooth scrolling and focusing on the current channel being processed.

## Features

- Automatically scrolls through your subscription list to load channels.
- Detects non-verified channels and unsubscribes from them.
- Smoothly scrolls to focus on the channel being processed.
- Handles confirmation dialogs for unsubscribing.
- Retries scrolling to ensure all channels are processed.

## Instructions

1. **Open YouTube Subscriptions Management Page:**
   - Navigate to `https://www.youtube.com/feed/channels`.

2. **Open the Browser Console:**
   - Right-click on the page, select `Inspect`, and go to the `Console` tab.

3. **Paste and Run the Script:**
   - Copy the script below and paste it into the console, then press `Enter`.

4. **Watch the Script Work:**
   - The script will scroll through your subscriptions, unsubscribe from non-verified channels, and confirm the unsubscription dialog.

---

## Script

```javascript
(async function () {
  const scrollAndWaitForNewContent = async () => {
    return new Promise((resolve) => {
      const distance = 1000; 
      const delay = 200; 
      let reachedBottom = false;

      const interval = setInterval(() => {
        window.scrollBy(0, distance);

        if (
          window.innerHeight + window.scrollY >= document.body.offsetHeight
        ) {
          reachedBottom = true;
          clearInterval(interval);
          resolve(reachedBottom);
        }
      }, delay);
    });
  };

  const delay = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

  const extractNonVerifiedChannels = (processedChannels) => {
    const channelElements = document.querySelectorAll("ytd-channel-renderer");
    console.log(`Found ${channelElements.length} total channels.`);
    const channelsToUnsubscribe = [];

    channelElements.forEach((el) => {
      const channelNameElement = el.querySelector("#channel-title yt-formatted-string");
      const isVerified = el.querySelector("ytd-badge-supported-renderer:not([hidden])");
      const unsubscribeButton = el.querySelector("#subscribe-button button[aria-label^='Unsubscribe']");

      if (channelNameElement) {
        const channelName = channelNameElement.textContent.trim();
        if (!processedChannels.has(channelName)) {
          console.log(
            `Channel: ${channelName}, Verified: ${!!isVerified}, Unsubscribe Button: ${
              unsubscribeButton ? "Found" : "Not Found"
            }`
          );

          if (!isVerified && unsubscribeButton) {
            channelsToUnsubscribe.push({
              name: channelName,
              button: unsubscribeButton,
              element: el, 
            });
          }

          processedChannels.add(channelName);
        }
      }
    });

    return channelsToUnsubscribe;
  };

  console.log("Starting to collect and unsubscribe channels...");
  const processedChannels = new Set();

  while (true) {
    let newChannels = extractNonVerifiedChannels(processedChannels);

    for (const { name, button, element } of newChannels) {
      console.log(`Unsubscribing from: ${name}`);

      element.scrollIntoView({ behavior: "smooth", block: "center" });
      await delay(500); 

      button.click(); 

      await new Promise((resolve) => {
        const interval = setInterval(() => {
          const confirmButton = document.querySelector(
            "#confirm-button button[aria-label='Unsubscribe']"
          );
          if (confirmButton) {
            console.log(`Confirming unsubscribe for: ${name}`);
            confirmButton.click();
            clearInterval(interval);
            resolve();
          }
        }, 200); 
      });

      await delay(1000);
    }

    console.log("Scrolling to load more channels...");
    let retries = 0;
    let previousChannelCount = document.querySelectorAll(
      "ytd-channel-renderer"
    ).length;

    while (retries < 5) {
      await scrollAndWaitForNewContent();

      await delay(3000);

      let currentChannelCount = document.querySelectorAll(
        "ytd-channel-renderer"
      ).length;

      if (currentChannelCount > previousChannelCount) {
        console.log("New channels loaded. Rechecking...");
        break;
      } else {
        console.log(
          `No new channels loaded after scroll. Retry ${retries + 1}/5...`
        );
        retries++;
      }
    }

    if (retries === 5) {
      console.log("No more channels to process. Exiting.");
      break;
    }
  }

  console.log("Unsubscribe process completed!");
})();
