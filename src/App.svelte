<script>
  import Paho from "paho-mqtt";
  import HslToHex from "hsl-to-hex";
  import { v4 as uuidv4 } from "uuid";
  const clientId = "mobile-" + uuidv4();
  const client = new Paho.Client("wss://mqtt.4hcomputers.club/mqtt", clientId);
  const channel = "hat-draw-1";
  window.client = client;
  let connected = false;
  function mqttConnect() {
    client.connect({
      onSuccess: (_) => {
        console.log("connected");
        connected = true;
        client.subscribe(channel);
      },
      onFailure: (_) => {
        console.log("failed");
        connected = false;
        setTimeout(() => {
          mqttConnect();
        }, 3000);
      },
      cleanSession: true,
    });
  }
  client.onConnectionLost = (_) => {
    console.log("disconnected");
    connected = false;
    setTimeout(() => {
      mqttConnect();
    }, 1000);
  };
  mqttConnect();
  let updateIds = [];
  let lastRecieved = "";
  let firstUpdate = false;
  client.onMessageArrived = (message) => {
    if (message.destinationName === channel) {
      const data = JSON.parse(message.payloadString);
      console.log(data);
      // console.log("Comparing update Ids: " + updateId + " " + data.updateId);
      // debugger;
      if (updateIds.indexOf(data.updateId) === -1 && data.grid && JSON.stringify(data.grid) != JSON.stringify(grid)) {
        console.log("Updating grid");
        grid = [...data.grid];
        lastRecieved = JSON.stringify(grid);
      }
      if (updateIds.indexOf(data.updateId) > -1) updateIds.splice(updateIds.indexOf(data.updateId), 1);
      firstUpdate = true;
    }
  };
  $: {
    if (connected && (firstUpdate || !(JSON.stringify(grid) == JSON.stringify(clear)))) {
      const updateId = Math.round(Math.random() * 1000000);
      updateIds.push(updateId);
      console.log("Creating new updateId: " + updateId);
      if (lastRecieved != JSON.stringify(grid)) client.publish(channel, JSON.stringify({ updateId, grid }), 0, true);
    }
  }
  let grid = new Array(64).fill("#000000");
  const clear = new Array(64).fill("#000000");
  let elements = new Array(64).fill(null);
  let drawing = false;
  let color = "#00FF00";
  let colorMode = "solid";
  let mode = "draw";
  let temporaryErase = false;
  let lastHue = 0;
  $: console.log(color);
  function onDraw(index) {
    let newColor = color;
    if (colorMode === "random") {
      newColor = "#" + ((Math.random() * 0xffffff) << 0).toString(16).padStart(6, "0");
    } else if (colorMode === "rainbow") {
      newColor = HslToHex(lastHue, 100, 50);
      lastHue += 360 / 64;
      if (lastHue > 360) lastHue = 0;
    }
    if (mode === "erase" || temporaryErase) grid[index] = "#000000";
    else if (mode === "draw") grid[index] = newColor;
  }
  window.onmouseup = (e) => {
    drawing = false;
  };
  let lastTouchedIndex = -1;
</script>

<main>
  <div class="tools" style="flex-direction: column;">
    <div class="tools">
      <button class:toolSelected={mode === "draw"} on:click={(_) => (mode = "draw")}>Draw</button>
      <button class:toolSelected={mode === "erase"} on:click={(_) => (mode = "erase")}>Erase</button>
      <button on:click={(_) => (grid = new Array(64).fill("#000000"))}>Clear</button>
    </div>
    <div class="tools">
      <input type="color" name="" id="" bind:value={color} />
      <button class:toolSelected={colorMode === "solid"} on:click={(_) => (colorMode = "solid")}>Solid</button>
      <button class:toolSelected={colorMode === "rainbow"} on:click={(_) => (colorMode = "rainbow")}>Rainbow</button>
      <button class:toolSelected={colorMode === "random"} on:click={(_) => (colorMode = "random")}>Random</button>
    </div>
  </div>
  <div
    class="grid"
    on:mousedown={(e) => {
      if (e.button === 2) {
        temporaryErase = true;
      }
      drawing = true;
    }}
    on:contextmenu={(e) => e.preventDefault()}
    on:mouseup={(_) => {
      temporaryErase = false;
      drawing = false;
    }}
    on:touchmove={(e) => {
      e.preventDefault();
      const newIndex = elements.indexOf(document.elementFromPoint(e.touches[0].pageX, e.touches[0].pageY));
      if (newIndex != lastTouchedIndex) {
        onDraw(newIndex);
        lastTouchedIndex = newIndex;
      }
    }}
    on:touchend={(_) => (lastTouchedIndex = -1)}
    on:dragstart={(e) => e.preventDefault()}
  >
    {#each grid as item, i}
      <div
        bind:this={elements[i]}
        class="box"
        style={`background: ${item}`}
        on:mouseenter={(_) => {
          if (drawing) onDraw(i);
        }}
        on:touchstart={(_) => onDraw(i)}
        on:touchenter={(_) => onDraw(i)}
        on:mousedown={(e) => {
          console.log(e);
          if (e.button === 2 && mode !== "erase") {
            temporaryErase = true;
          }
          onDraw(i);
        }}
        onmous
      />
    {/each}
  </div>
  <div style="display: flex; gap: 12px; align-items: center;">
    <div style={`height: 1em; aspect-ratio: 1; background: ${connected ? "#0F0" : "#F00"}; border-radius: 50%;`} />
    <code style="font-size: 0.7rem;"> {connected ? "connected" : "disconnected"}<br />{clientId}</code>
  </div>
</main>

<style>
  main {
    display: flex;
    flex-direction: column;
    align-items: center;
    gap: 16px;
    width: 100%;
    max-width: 600px;
  }

  .tools {
    display: flex;
    flex-direction: row;
    align-items: center;
    gap: 8px;
  }

  .tools > *:not(.tools) {
    border: none;
    padding: 4px 6px;
    border-radius: 4px;
    cursor: pointer;
    font-size: 24px;
    height: 100%;
  }

  .tools > *:not(.tools).toolSelected {
    border: 2px solid lightgreen;
  }

  .grid {
    width: 100%;
    display: grid;
    gap: 4px;
    aspect-ratio: 1;
    grid-template-columns: repeat(8, auto);
  }
  .box {
    box-shadow: 3px 3px 0 0 #222;
    border-radius: 15%;
  }
</style>
