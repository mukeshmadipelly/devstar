<script>
  import { onMount } from 'svelte';
  import { parseGIF, decompressFrames } from 'gifuct-js';

  let inputGif = null;
  let loopOption = 'infinite';
  let loopCount = 1;
  let error = '';
  let isProcessing = false;
  let canvasElement;
  let ctx;
  let gifFrames = [];
  let currentFrame = 0;
  let loopsDone = 0;
  let animationId = null;
  let gifWidth = 0;
  let gifHeight = 0;
  let lastFrameTime = 0;
  let speedMultiplier = 1; // Start with original speed
  let gifJs = null; // To store the dynamically imported gif.js.optimized module

  onMount(() => {
    // Dynamically import gif.js.optimized only on the client side
    if (typeof window !== 'undefined') {
      import('gif.js.optimized').then((module) => {
        gifJs = module.default;
      });
    }

    return () => {
      if (animationId) {
        cancelAnimationFrame(animationId);
      }
    };
  });

  async function handleGifInput(event) {
    const file = event.target.files[0];
    if (file) {
      inputGif = file;
      error = '';
      // Reset canvas and speed
      if (canvasElement) {
        ctx.clearRect(0, 0, canvasElement.width, canvasElement.height);
      }
      speedMultiplier = 1; // Reset speed to original when new GIF is uploaded
    }
  }

  async function processGif() {
    if (!inputGif) {
      error = 'Please upload a GIF.';
      return;
    }

    isProcessing = true;
    error = '';

    try {
      const arrayBuffer = await inputGif.arrayBuffer();
      const gif = parseGIF(arrayBuffer);
      gifFrames = decompressFrames(gif, true);
      
      gifWidth = gifFrames[0].dims.width;
      gifHeight = gifFrames[0].dims.height;
      
      if (canvasElement) {
        canvasElement.width = gifWidth;
        canvasElement.height = gifHeight;
        ctx = canvasElement.getContext('2d');
      }

      startAnimation();
    } catch (e) {
      error = 'Error processing GIF: ' + e.message;
    } finally {
      isProcessing = false;
    }
  }

  function startAnimation() {
    if (animationId) {
      cancelAnimationFrame(animationId);
    }
    currentFrame = 0;
    loopsDone = 0;
    lastFrameTime = 0;
    animationId = requestAnimationFrame(renderNextFrame);
  }

  function renderNextFrame(timestamp) {
    if (!canvasElement || !ctx || gifFrames.length === 0) return;

    if (!lastFrameTime) lastFrameTime = timestamp;

    const frameDelay = speedMultiplier === 1 
      ? gifFrames[currentFrame].delay * 10  // Use original delay when speedMultiplier is 1
      : (gifFrames[currentFrame].delay * 10) / speedMultiplier; // Adjust speed otherwise
    const elapsedTime = timestamp - lastFrameTime;

    if (elapsedTime > frameDelay) {
      const frame = gifFrames[currentFrame];
      ctx.putImageData(new ImageData(frame.patch, frame.dims.width, frame.dims.height), frame.dims.left, frame.dims.top);

      currentFrame++;
      if (currentFrame >= gifFrames.length) {
        currentFrame = 0;
        loopsDone++;
        if (loopOption === 'infinite' || (loopOption === 'specific' && loopsDone < loopCount) || (loopOption === 'once' && loopsDone < 1)) {
          lastFrameTime = timestamp;
          animationId = requestAnimationFrame(renderNextFrame);
        }
      } else {
        lastFrameTime = timestamp;
        animationId = requestAnimationFrame(renderNextFrame);
      }
    } else {
      animationId = requestAnimationFrame(renderNextFrame);
    }
  }

  async function downloadGif() {
    if (!gifJs) {
      error = 'GIF.js library is not loaded.';
      return;
    }

    const gif = new gifJs({
      workers: 2,
      quality: 10,
      width: gifWidth,
      height: gifHeight
    });

    try {
      for (let i = 0; i < gifFrames.length; i++) {
        const frame = gifFrames[i];

        // Ensure imageData dimensions match the frame dimensions
        const imageData = new ImageData(frame.patch, frame.dims.width, frame.dims.height);

        const canvas = document.createElement('canvas');
        canvas.width = gifWidth;
        canvas.height = gifHeight;
        const context = canvas.getContext('2d');

        // Clear canvas before drawing
        context.clearRect(0, 0, gifWidth, gifHeight);

        // Draw the frame onto the canvas
        context.putImageData(imageData, frame.dims.left, frame.dims.top);

        // Add the frame to gif.js instance
        gif.addFrame(canvas, { delay: frame.delay * 10 }); // Adjusting delay for gif.js format
      }

      gif.on('finished', function(blob) {
        const url = URL.createObjectURL(blob);
        const a = document.createElement('a');
        a.href = url;
        a.download = 'modified.gif'; // Set the filename for download
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
        URL.revokeObjectURL(url);
      });

      gif.render();
    } catch (e) {
      error = 'Error rendering GIF: ' + e.message;
    }
  }
</script>

<div class="container">
  <h2>Change GIF Loop Count</h2>
  
  <div class="input-group">
    <label for="gifInput">Select GIF:</label>
    <input 
      type="file" 
      id="gifInput" 
      accept="image/gif" 
      on:change={handleGifInput}
    />
  </div>

  <div class="input-group">
    <label for="loopOption">Loop Option:</label>
    <select id="loopOption" bind:value={loopOption}>
      <option value="infinite">Repeat Infinitely</option>
      <option value="once">Loop Once (Play Twice)</option>
      <option value="specific">Loop Specific Times</option>
    </select>
  </div>

  {#if loopOption === 'specific'}
    <div class="input-group">
      <label for="loopCount">Loop Count:</label>
      <input 
        type="number" 
        id="loopCount" 
        bind:value={loopCount} 
        min="1" 
        step="1"
      />
    </div>
  {/if}

  <div class="input-group">
    <label for="speedControl">Animation Speed:</label>
    <input 
      type="range" 
      id="speedControl" 
      bind:value={speedMultiplier} 
      min="0.1" 
      max="3" 
      step="0.1"
    />
    <span>{speedMultiplier === 1 ? 'Original' : `${speedMultiplier.toFixed(1)}x`}</span>
  </div>

  <div class="button-group">
    <button on:click={processGif} disabled={isProcessing || !inputGif}>
      {isProcessing ? 'Processing...' : 'Process GIF'}
    </button>
    <button on:click={downloadGif} disabled={!gifFrames.length || isProcessing}>
      Download GIF
    </button>
  </div>

  {#if error}
    <div class="error">{error}</div>
  {/if}

  <div class="gif-container">
    <h3>Processed GIF:</h3>
    <canvas bind:this={canvasElement}></canvas>
  </div>
</div>

<style>
  .container {
    max-width: 600px;
    margin: auto;
    padding: 20px;
    border: 1px solid #ccc;
    border-radius: 5px;
    background: #f9f9f9;
  }
  .input-group {
    margin-bottom: 15px;
  }
  .input-group label {
    display: block;
    margin-bottom: 5px;
    font-weight: bold;
  }
  .input-group input,
  .input-group select {
    width: 100%;
    padding: 8px;
    font-size: 16px;
    border: 1px solid #ccc;
    border-radius: 3px;
  }
  .button-group {
    margin-bottom: 15px;
  }
  .button-group button {
    padding: 10px 20px;
    background-color: #4CAF50;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 16px;
  }
  .button-group button:disabled {
    background-color: #cccccc;
    cursor: not-allowed;
  }
  .error {
    color: red;
    margin-top: 5px;
  }
  .gif-container {
    margin-top: 20px;
    text-align: center;
  }
  .gif-container canvas {
    max-width: 100%;
    height: auto;
    border: 1px solid #ddd;
    border-radius: 4px;
  }
</style>
