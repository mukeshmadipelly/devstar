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
  <div class="title">Change GIF Loop Count and Speed</div>
  <div class="upload-container">
    <input type="file" accept="image/gif" on:change={handleGifInput} id="gif-upload" style="display: none;">
    <label for="gif-upload" class="upload-button">Upload GIF</label>
  </div>

  {#if inputGif}
    <div class="control-container">
      <label for="loopOption">Loop Option:</label>
      <select id="loopOption" bind:value={loopOption}>
        <option value="infinite">Repeat Infinitely</option>
        <option value="once">Loop Once (Play Twice)</option>
        <option value="specific">Loop Specific Times</option>
      </select>
    </div>

    {#if loopOption === 'specific'}
      <div class="control-container">
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

    <div class="control-container">
      <label for="speedControl">Animation Speed: {speedMultiplier === 1 ? 'Original' : `${speedMultiplier.toFixed(1)}x`}</label>
      <input 
        type="range" 
        id="speedControl" 
        bind:value={speedMultiplier} 
        min="0.1" 
        max="3" 
        step="0.1"
      />
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

    <div class="preview-container">
      <div class="preview">
        <div>Processed GIF</div>
        <canvas bind:this={canvasElement}></canvas>
      </div>
    </div>
  {/if}
</div>

<style>
  .container {
    max-width: 900px;
    margin: auto;
    text-align: center;
  }

  .title {
    font-size: 24px;
    margin-bottom: 20px;
    font-weight: 600;
    color: #333333;
    background-color: #ffffff;
    padding: 10px;
    display: inline-block;
    border-radius: 5px;
    margin-top: 20px;
  }

  .upload-container {
    margin-bottom: 20px;
  }

  .upload-button {
    display: inline-block;
    padding: 12px 24px;
    background-color: #28a745;
    color: white;
    text-decoration: none;
    border-radius: 5px;
    cursor: pointer;
    transition: background-color 0.3s ease;
  }

  .upload-button:hover {
    background-color: #218838;
  }

  .control-container {
    margin: 20px 0;
    color: #333333;
    background-color: #ffffff;
    padding: 10px;
    border-radius: 5px;
    display: inline-block;
    margin-bottom: 10px;
  }

  .control-container label {
    font-weight: 600;
    font-size: 16px;
    color: #333333;
  }

  .button-group {
    margin: 20px 0;
  }

  .button-group button {
    padding: 10px 20px;
    background-color: #007bff;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 16px;
    margin-right: 10px;
    transition: background-color 0.3s ease;
  }

  .button-group button:hover {
    background-color: #0056b3;
  }

  .button-group button:disabled {
    background-color: #ccc;
    cursor: not-allowed;
  }

  .error {
    color: #333333;
    margin-top: 10px;
    font-weight: bold;
    background-color: #ffffff;
    padding: 10px;
    border-radius: 5px;
    display: inline-block;
  }

  .preview-container {
    display: flex;
    justify-content: center;
    margin-top: 20px;
    gap: 20px;
    font-size: 18px;
    color: #333333;
  }

  .preview {
    text-align: center;
    color: #333333;
    background-color: #ffffff;
    padding: 10px;
    border-radius: 5px;
    display: inline-block;
  }

  .preview canvas {
    max-width: 100%;
    height: auto;
    border: 2px solid #ccc;
    border-radius: 5px;
    margin-top: 10px;
  }
</style>
