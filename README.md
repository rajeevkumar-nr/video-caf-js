[![New Relic Experimental header](https://github.com/newrelic/opensource-website/raw/master/src/images/categories/Experimental.png)](https://opensource.newrelic.com/oss-category/#new-relic-experimental)

# New Relic CAF Tracker

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

The New Relic CAF Tracker provides comprehensive video analytics for Chromecast receiver applications built with the Cast Application Framework (CAF). Track playback events, monitor quality, identify errors, and gain deep insights into streaming performance and user experience on Chromecast devices.

## Features

- **Automatic Event Detection** - Captures CAF PlayerManager events automatically without manual instrumentation
- **Ad Break Tracking** - Full support for CAF ad breaks including quartile progress events
- **Sender User Agent** - Automatically captures the sender device user agent
- **Quality Monitoring** - Bitrate and rendition change tracking
- **Error Tracking** - Detailed error codes with CAF-specific error classification
- **Easy Integration** - Single script tag, no build system required on the receiver
- **Seek & Buffer Tracking** - Complete seek and buffer event lifecycle

## Table of Contents

- [Installation](#installation)
- [Prerequisites](#prerequisites)
- [Usage](#usage)
- [Best Practices](#best-practices)
- [Configuration Options](#configuration-options)
- [API Reference](#api-reference)
- [Data Model](#data-model)
- [Known Limitations](#known-limitations)
- [Support](#support)
- [Contribute](#contribute)
- [License](#license)

## Installation

Include the CAF Tracker script and the CAF Receiver SDK in your Chromecast receiver HTML:

```html
<!DOCTYPE html>
<html>
  <head>
    <!-- New Relic CAF Tracker -->
    <script src="path/to/newrelic-caf.min.js"></script>

    <!-- CAF Receiver SDK -->
    <script src="//www.gstatic.com/cast/sdk/libs/caf_receiver/v3/cast_receiver_framework.js"></script>
  </head>
  <body>
    <cast-media-player></cast-media-player>

    <script>
      const receiverContext = cast.framework.CastReceiverContext.getInstance();

      // Configure New Relic tracker with credentials from one.newrelic.com
      const options = {
        info: {
          licenseKey:    'YOUR_LICENSE_KEY',
          beacon:        'bam.nr-data.net',
          applicationID: 'YOUR_APPLICATION_ID',
        },
      };

      // Initialize tracker BEFORE calling receiverContext.start()
      const tracker = new CAFTracker(receiverContext, options);

      receiverContext.start();
    </script>
  </body>
</html>
```

**Setup Steps:**

1. **Get Configuration** - Visit [one.newrelic.com](https://one.newrelic.com) and follow the Streaming Video & Ads onboarding flow to get your `licenseKey`, `beacon`, and `applicationID`.
2. **Integrate** - Include the script in your receiver HTML and initialize with your configuration **before** calling `receiverContext.start()`.

## Prerequisites

Before using the tracker, ensure you have:

- **New Relic Account** - Active New Relic account with valid credentials (`beacon`, `applicationID`, `licenseKey`)
- **CAF Receiver SDK** - The Cast Application Framework Receiver SDK loaded in your receiver page
- **Chromecast Device or Emulator** - A real Chromecast device or the [Chromecast Device Emulator](https://github.com/ajaidanial/chromecast-device-emulator) for local testing

## Usage

### Getting Your Configuration

Before initializing the tracker, obtain your New Relic configuration:

1. Log in to [one.newrelic.com](https://one.newrelic.com)
2. Navigate to the video agent onboarding flow
3. Copy your credentials: `licenseKey`, `beacon`, and `applicationID`

### Basic Setup

```javascript
// 1. Get the CastReceiverContext singleton
const receiverContext = cast.framework.CastReceiverContext.getInstance();

// 2. Configure tracker with credentials from one.newrelic.com
const options = {
  info: {
    licenseKey:    'YOUR_LICENSE_KEY',
    beacon:        'bam.nr-data.net',
    applicationID: 'YOUR_APPLICATION_ID',
  },
};

// 3. Initialize tracker BEFORE receiverContext.start()
const tracker = new CAFTracker(receiverContext, options);

// 4. Start the receiver
receiverContext.start();
```

> **Important:** The tracker must be initialized before calling `receiverContext.start()` to ensure all playback events are captured from the beginning of the session.

### Advanced Configuration

```javascript
const options = {
  info: {
    licenseKey:    'YOUR_LICENSE_KEY',
    beacon:        'bam.nr-data.net',
    applicationID: 'YOUR_APPLICATION_ID',
  },
  config: {
    qoeAggregate: true,        // Enable QoE event aggregation
    qoeIntervalFactor: 2,      // Send QoE events every 2 harvest cycles
  },
  customData: {
    contentTitle:     'My Video Title',
    customPlayerName: 'MyCustomPlayer',
    customAttribute:  'customValue',
  },
};

const tracker = new CAFTracker(receiverContext, options);
```

## Best Practices

### 1. Setting `contentTitle`

The `contentTitle` attribute is populated from the media metadata title if present. For best results, explicitly set it via `customData`:

```javascript
const tracker = new CAFTracker(receiverContext, {
  info: {
    licenseKey:    'YOUR_LICENSE_KEY',
    beacon:        'bam.nr-data.net',
    applicationID: 'YOUR_APPLICATION_ID',
  },
  customData: {
    contentTitle: 'My Video Title',
  },
});
```

If your content changes dynamically (e.g., a playlist or queue):

```javascript
tracker.sendOptions({
  customData: {
    contentTitle: 'New Video Title',
  },
});
```

### 2. Setting `userId`

Track analytics per user by setting a user identifier:

```javascript
// Set userId during initialization
const tracker = new CAFTracker(receiverContext, {
  info: {
    licenseKey:    'YOUR_LICENSE_KEY',
    beacon:        'bam.nr-data.net',
    applicationID: 'YOUR_APPLICATION_ID',
  },
  customData: {
    userId: 'user-12345',
  },
});

// Or set userId separately after initialization
tracker.setUserId('user-12345');
```

### 3. Adding Custom Attributes

Enrich your analytics data with deployment-specific attributes:

```javascript
const tracker = new CAFTracker(receiverContext, {
  info: {
    licenseKey:    'YOUR_LICENSE_KEY',
    beacon:        'bam.nr-data.net',
    applicationID: 'YOUR_APPLICATION_ID',
  },
  customData: {
    contentTitle:     videoMetadata.title,
    userId:           currentUser.id,
    subscriptionTier: 'premium',
    contentProvider:  'studio-abc',
    region:           'us-west-2',
    appVersion:       '2.1.0',
  },
});
```

**Query custom attributes in New Relic:**

```sql
-- Analyze content starts by subscription tier
SELECT count(*) FROM VideoAction WHERE actionName = 'CONTENT_START'
FACET subscriptionTier SINCE 1 day ago

-- Monitor bitrate by region
SELECT average(contentBitrate) FROM VideoAction
FACET region SINCE 1 hour ago
```

## Configuration Options

### `options.info` (required)

| Field | Type | Description |
|-------|------|-------------|
| `licenseKey` | string | New Relic Browser license key |
| `beacon` | string | Data endpoint — use `bam.nr-data.net` |
| `applicationID` | string | New Relic application ID |

### `options.customData` (optional)

Add custom attributes to all events:

```javascript
customData: {
  contentTitle:        'My Video Title',   // Override video title
  customPlayerName:    'MyPlayer',         // Custom player identifier
  customPlayerVersion: '1.0.0',           // Custom player version
  userId:              '12345',            // User identifier
  // Any additional custom attributes
}
```

> **Limit:** The maximum total number of custom attributes per event is **150**. Attributes beyond this limit will be dropped.

> **Note:** Reserved keywords (`actionName`, `contentId`, `contentTitle`, `playerName`, `playerVersion`, `viewSession`, `viewId`) cannot be used as custom attribute names — they will be dropped. See [DATAMODEL.md](./DATAMODEL.md) for the full reserved keyword list.

## API Reference

### Core Methods

#### `tracker.setUserId(userId)`

Set a unique identifier for the current user.

```javascript
tracker.setUserId('user-12345');
```

#### `tracker.setHarvestInterval(milliseconds)`

Configure how frequently data is sent to New Relic. Accepts values between 1000ms (1 second) and 300000ms (5 minutes).

```javascript
tracker.setHarvestInterval(30000); // Send data every 30 seconds
```

#### `tracker.sendCustom(actionName, state, attributes)`

Send a custom event with arbitrary attributes.

```javascript
tracker.sendCustom('CUSTOM_ACTION', 'state time', {
  test1: 'value1',
  test2: 'value2',
});
```

#### `tracker.sendOptions(options)`

Update custom data after initialization.

```javascript
tracker.sendOptions({
  customData: {
    contentTitle: 'New Video Title',
  },
});
```

### Example: Complete Integration

```javascript
const receiverContext = cast.framework.CastReceiverContext.getInstance();

const tracker = new CAFTracker(receiverContext, {
  info: {
    licenseKey:    'YOUR_LICENSE_KEY',
    beacon:        'bam.nr-data.net',
    applicationID: 'YOUR_APPLICATION_ID',
  },
  customData: {
    contentTitle: 'My Video',
    userId:       'user-12345',
  },
});

// Configure reporting interval
tracker.setHarvestInterval(30000);

receiverContext.start();
```

## Data Model

The tracker captures comprehensive video analytics across four event types:

- **VideoAction** — Playback events (start, pause, buffer, seek, rendition changes, heartbeats)
- **VideoAdAction** — Ad break events (ad start, quartile progress, completions)
- **VideoErrorAction** — Error events (playback failures, ad errors, network errors)
- **VideoCustomAction** — Custom events defined by your application

**Full Documentation:** See [DATAMODEL.md](./DATAMODEL.md) for the complete event and attribute reference.

## Known Limitations

- When a video `END` event fires, the `contentSrc` attribute is incorrect. This is due to how the Chromecast player clears media information before the event is emitted.

## Support

Should you need assistance with New Relic products, you are in good hands with several support channels.

If the issue has been confirmed as a bug or is a feature request, please file a GitHub issue.

### Support Channels

- [New Relic Documentation](https://docs.newrelic.com): Comprehensive guidance for using our platform
- [New Relic Community](https://discuss.newrelic.com): The best place to engage in troubleshooting questions
- [New Relic University](https://learn.newrelic.com): A range of online training for New Relic users of every level
- [New Relic Technical Support](https://support.newrelic.com): 24/7/365 ticketed support. Read more about our [Technical Support Offerings](https://docs.newrelic.com/docs/licenses/license-information/general-usage-licenses/support-plan)

## Contribute

We encourage your contributions to improve the New Relic CAF Tracker! Keep in mind that when you submit your pull request, you'll need to sign the CLA via the click-through using CLA-Assistant. You only have to sign the CLA one time per project.

If you have any questions, or to execute our corporate CLA (which is required if your contribution is on behalf of a company), drop us an email at opensource@newrelic.com.

For more details on how best to contribute, see [CONTRIBUTING.md](./CONTRIBUTING.md).

### A note about vulnerabilities

As noted in our [security policy](../../security/policy), New Relic is committed to the privacy and security of our customers and their data. We believe that providing coordinated disclosure by security researchers and engaging with the security community are important means to achieve our security goals.

If you believe you have found a security vulnerability in this project or any of New Relic's products or websites, we welcome and greatly appreciate you reporting it to New Relic through our [bug bounty program](https://docs.newrelic.com/docs/security/security-privacy/information-security/report-security-vulnerabilities/).

If you would like to contribute to this project, review [these guidelines](./CONTRIBUTING.md).

To all contributors, we thank you! Without your contribution, this project would not be what it is today.

## License

New Relic CAF Tracker is licensed under the [Apache 2.0](http://apache.org/licenses/LICENSE-2.0.txt) License.

The New Relic CAF Tracker also uses source code from third-party libraries. Full details on which libraries are used and the terms under which they are licensed can be found in the [third-party notices document](./THIRD_PARTY_NOTICES.md).
