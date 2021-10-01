# Bonsai Cartpole Visualizer Example

## Intro

The Bonsai App visualizer plugin API is intended to allow you to create custom visualizations of your simulation state and action space. These visualizations can be loaded into the Bonsai App when displaying episode data, making it easier to understand.

![Example Visualizer In Use](ExampleViz.png "The Visualizer in use.")

The Bonsai App will send updated state that follows the user’s cursor when they hover over chart data.

## Overview

Plugins are URLs to a JavaScript web application hosted somewhere that the user’s browser can access. These URLs are loaded into an IFrame by the Bonsai App. You will need to provide your own hosting for your plugins.

You can use pure JavaScript, or a web framework like React or Angular. We use the Window.postMessage() API to communicate with the plugin in the IFrame.

The plugin will also be passed a variety of query parameters when it is loaded. Some of them are reserved by the plugin API, but a developer can have the user pass in additional configuration parameters as necessary.

## Basic Example

Let’s make a visualizer for the classic “cart & pole” problem.

First we’ll parse the \_theme query parameter and update the colors to match. We’ll also allow the end user to override what variables they want to use for the cart position and pole angle.

Then, we’ll register an event handler and use the contents of the event sent from Bonsai App via Window.postMessage to update the html.

Lastly, we’ll use the state to render a simple drawing of a cart and pole system.

```html
<html>
  <head>
    <script language="JavaScript">
      const params = new URLSearchParams(window.location.search);
      const darkMode = params.get("_theme") === "dark";

      // optional overrides for parameter names
      const posKey = params.has("pos") ? params.get("pos") : "cart_position";
      const angleKey = params.has("angle") ? params.get("angle") : "pole_angle";

      function init() {
        const pole = document.getElementById("pole");
        const cart = document.getElementById("cart");

        // adjust our colors based upon theme.
        if (darkMode) {
          document.body.style.backgroundColor = "#333";
          document.body.style.color = "white";
          pole.style.fill = "white";
        }

        window.addEventListener(
          "message",
          (event) => {
            // convert message to formatted JSON text for display
            const data = JSON.parse(event.data);
            const str = JSON.stringify(data, null, 4);
            const out = document.getElementById("out");
            out.textContent = str;

            // read state out of message and convert units
            const pos = data.state[posKey];
            const angle = (data.state[angleKey] * 180.0) / Math.PI;

            // update positions of graphical elements
            cart.setAttribute("transform", `translate(${pos} 0)`);
            pole.setAttribute(
              "transform",
              `translate(${pos} 0) rotate(${angle} 0 0)`
            );
          },
          false
        );
      }
    </script>
  </head>
  <body>
    <pre id="out" style="display: box; position: absolute">Waiting...</pre>
    <svg
      preserveAspectRatio="xMidYMid meet"
      height="100%"
      width="100%"
      viewBox="-1 -1 2 2"
    >
      <rect
        id="track"
        x="-0.5"
        y="-0.025"
        width="1"
        height="0.05"
        fill="grey"
      />
      <rect
        id="cart"
        x="-0.15"
        y="-0.05"
        width="0.3"
        height="0.1"
        fill="grey"
      />
      <rect
        id="pole"
        x="-0.025"
        y="-0.5"
        width="0.05"
        height="0.5"
        fill="#333"
      />
    </svg>
    <script language="JavaScript">
      init();
    </script>
  </body>
</html>
```

### Using the Visualizer

When served from the local machine, this will render the contents of `message` into a text block. We can use npm to serve the html doc like so:

```bash
npm install -g http-server
npx http-server & open "http://localhost:8080/"
```

In the settings for the concept, we will then change set the visualizer URL to “http://localhost:8080” and then start training.

![Picture of Visualizer Settings Panel]("")

You can also use a visualizer directly from Github Pages once it has been set up.

Example:

```javascript
...
// Place this in your Inkling file and update with your github pages location.
const SimulatorVisualizer =
  "https://{yourgithub}.github.io/bonsai-viz-example/";
...
```

This is a basic example. In practice, your visualizations will want to do more error checking.  
However, you can see how the data passed into the visualizer could be used to draw complicated representations with 2D images, or even a 3D scene.
We have several examples of advanced visualization in our github repo [Link to examples]. You can use them as inspiration for building your own visualizers.

## Configuring Visualizers

Often, it is desirable to have configuration information passed into a visualizer. You may want to provide some sort of a mapping between the concept state space and the internal variables used by the visualizer. Or maybe you want to provide different operating modes or options.

Configuration information is passed in via URL query parameters. They can be parsed out with the URLSearchParams API.

Example:

```typescript
...
const params = new URLSearchParams(window.location.search);
const useLightTheme = params.get("_theme") === "light";
const prefsKey = params.get("_prefsKey");
...
```

In addition to the query parameters specified in the visualizer URL, the Bonsai App will append a set of reserved query parameters when loading a visualizer plugin.

Example:

```
If a user enters:
 http://localhost:8080/?someParam=1&anotherParam=true

The plugin receives:
 http://localhost:8080/?someParam=1&anotherParam=true&_theme=light&_lang=en& &_prefsKey=12345
```

The reserved query parameters start with an underscore. Your plugin is free to use query parameters that do not start with an underscore for its own configuration information, and should ignore reserved parameters it does not recognize.

### Reserved Query Parameters

- ?\_theme=[light|dark]
- ?\_lang=[en]
- ?\_prefsKey=[string]

### Theme

This parameter is used to indicate that the user has selected either dark or light mode in the app. You should choose a palette for your visualizer that fits within these two styles.

### Language

This parameter is updated to match the users currently selected localization. If your visualizer supports localizations, it can use this to automatically switch to match the user’s preferences. You can also use the browsers settings if that is more desirable.

### PrefsKey

This is a unique identifier that can be used as a key by the visualizer to save user preferences for different instances of the visualizer in the Bonsai App. Preferences are the responsibility of the visualizer to store and retrieve.

## Visualizer Event Messages

All messages received from the Bonsai App are JSON text blobs. They will always contain the following header:

```typescript
interface MessageHeader {
    version: “1.0”;
    type: string;
    ...
}
```

The visualizer should ignore messages with a type that it doesn’t recognize. Today there is only one inbound message, “IterationUpdate”.

## IterationUpdate

Currently, this is the only message that visualizers will receive. It contains all the available data for a single iteration within an episode.

This message has the format:

```typescript
export enum MessageType {
  IterationUpdate = "IterationUpdate",
}

export type StructType = { [key: string]: ValueType };
export type ValueType = StructType | ValueType[] | number | string;

export type MetaType = {
  episode: number;
  iteration: number;
  [key: string]: ValueType;
};

// Message received by the visualizer
export interface IterationUpdateMessage {
  version: string;
  type: MessageType.IterationUpdate;
  config: ValueType | undefined;
  meta: MetaType | undefined;
  state: ValueType;
  action: ValueType | undefined;
}
```

### Version

This is currently “1.0”, the plugin should reject major version changes as incompatible. The plugin should ignore fields it does not recognize. We will use the versioning practices defined in Semantic Versioning 2.0.0 | Semantic Versioning (semver.org)

### Type

Currently, this is “IterationUpdate”. A plugin should ignore messages it doesn’t recognize.

### Episode

If the Bonsai App has a sequence of one or more episodes, this index will be used to uniquely identify each episode relative to others in the set. This can be used to note when the visualizer is being asked to display data from different episodes.

### Iteration

Within an episode, this is the sequence index for the iteration data. However, your simulator should be stateless. Iterations can come in any order as the passed in data will follow user interactions.

### Meta

This contains meta-data fields for the iteration. Reward, terminal flags, episode boundaries, etc.

[List of meta-data fields]

### Config

If present, this contains the configuration information that was used to initialize this episode. The variables will match those used in Inkling.

### State

The state for the iteration, if present. The variables will match those in Inkling.

### Action

The resulting action for the iteration, if present. The variables will match those in Inkling.

## Contributing

This project welcomes contributions and suggestions. Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft
trademarks or logos is subject to and must follow
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
