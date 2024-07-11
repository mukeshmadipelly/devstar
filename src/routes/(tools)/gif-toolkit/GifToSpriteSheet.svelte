<script>
  import { onMount } from 'svelte';
  import { parseGIF, decompressFrames } from 'gifuct-js';

  let gifFile;
  let spriteSheet;
  let frameCount = 0;
  let rows = 1;
  let columns;
  let frameRate = 250;
  let playbackReversed = false;
  let skipFrames = '';
  let gifBackgroundColor = '';
  let newBackgroundColor = '';
  let colorToneThreshold = 10;
  let padding = 0;

  let canvas;
  let ctx;
  let frames = [];
  let status = '';

  onMount(() => {
    canvas = document.getElementById('spriteSheetCanvas');
    ctx = canvas.getContext('2d');
  });

  function handleFileInput(event) {
    gifFile = event.target.files[0];
    status = `File selected: ${gifFile.name}`;

    // Create a preview
    const reader = new FileReader();
    reader.onload = (e) => {
      const img = document.getElementById('gifPreview');
      img.src = e.target.result;
    };
    reader.readAsDataURL(gifFile);
  }

  async function generateSpriteSheet() {
    if (!gifFile) {
      alert('Please select a GIF file first.');
      return;
    }

    status = 'Generating sprite sheet...';

    // Clear previous sprite sheet
    spriteSheet = null;
    frames = [];
    frameCount = 0;

    try {
      frames = await extractFrames(gifFile);
      frameCount = frames.length;
      status = `Extracted ${frameCount} frames. Rendering sprite sheet...`;
      await renderSpriteSheet();
      status = 'Sprite sheet generated successfully!';
    } catch (error) {
      console.error('Error generating sprite sheet:', error);
      status = `Error: ${error.message}`;
      alert('An error occurred while generating the sprite sheet. Check the console for details.');
    }
  }

  async function extractFrames(file) {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = async (e) => {
        try {
          const arrayBuffer = e.target.result;
          const gif = parseGIF(arrayBuffer);
          const frames = decompressFrames(gif, true);
          resolve(frames);
        } catch (error) {
          reject(error);
        }
      };
      reader.onerror = () => reject(new Error('File reading error'));
      reader.readAsArrayBuffer(file);
    });
  }

  async function renderSpriteSheet() {
    const layout = calculateLayout(frameCount, rows, columns);

    const frameWidth = frames[0].dims.width;
    const frameHeight = frames[0].dims.height;

    canvas.width = layout.width * frameWidth + padding * 2;
    canvas.height = layout.height * frameHeight + padding * 2;

    ctx.clearRect(0, 0, canvas.width, canvas.height);

    let frameIndex = 0;
    for (let y = 0; y < layout.height; y++) {
      for (let x = 0; x < layout.width; x++) {
        if (frameIndex < frameCount) {
          const frame = frames[frameIndex];
          const imageData = new ImageData(frame.patch, frame.dims.width, frame.dims.height);
          ctx.putImageData(
            imageData,
            x * frameWidth + padding + frame.dims.left,
            y * frameHeight + padding + frame.dims.top
          );
          frameIndex++;
        }
      }
    }

    canvas.style.margin = 'auto';
    canvas.style.display = 'block';

    spriteSheet = canvas.toDataURL('image/png');
  }

  function calculateLayout(frameCount, rows, columns) {
    if (rows && columns) {
      return { width: columns, height: rows };
    } else if (rows) {
      return { width: Math.ceil(frameCount / rows), height: rows };
    } else if (columns) {
      return { width: columns, height: Math.ceil(frameCount / columns) };
    } else {
      return { width: frameCount, height: 1 };
    }
  }
</script>

<main>
  <div class="container">
    <div class="title">GIF to SpriteSheet Converter</div>
    <div class="upload-container">
      <input id="gifFileInput" type="file" accept="image/gif" on:change={handleFileInput} style="display: none;">
      <label for="gifFileInput" class="upload-button">Select GIF File</label>
    </div>

    {#if gifFile}
      <div class="control-container">
        <label for="rowsInput">Rows:</label>
        <input id="rowsInput" type="number" bind:value={rows} min="1">
      </div>
      <div class="control-container">
        <label for="columnsInput">Columns:</label>
        <input id="columnsInput" type="number" bind:value={columns} min="1">
      </div>
      <div class="control-container">
        <label for="frameRateInput">Frame Rate (ms):</label>
        <input id="frameRateInput" type="number" bind:value={frameRate} min="1">
      </div>
      <div class="control-container">
        <label for="playbackReversedInput">Reverse Playback:</label>
        <input id="playbackReversedInput" type="checkbox" bind:checked={playbackReversed}>
      </div>
      <div class="control-container">
        <label for="skipFramesInput">Skip Frames:</label>
        <input id="skipFramesInput" type="text" bind:value={skipFrames} placeholder="e.g. 1,3,5">
      </div>
      <div class="control-container">
        <label for="paddingInput">Padding:</label>
        <input id="paddingInput" type="number" bind:value={padding} min="0">
      </div>
      <div class="control-container">
        <label for="colorToneThresholdInput">Color Tone Threshold (%):</label>
        <input id="colorToneThresholdInput" type="number" bind:value={colorToneThreshold} min="0" max="100">
      </div>

      <div class="button-group">
        <button on:click={generateSpriteSheet}>Generate Sprite Sheet</button>
      </div>

      <p>{status}</p>

      <div class="preview-container">
        <div class="preview">
          {#if spriteSheet}
            <div class="highlighted-text">Generated Sprite Sheet</div>
            <img src={spriteSheet} alt="Sprite Sheet" class="sprite-sheet">
            <div>
              <a href={spriteSheet} download="sprite_sheet.png" class="download-link">Download Sprite Sheet</a>
            </div>
          {/if}
        </div>
      </div>
    {/if}
  </div>
  <canvas id="spriteSheetCanvas"></canvas>
</main>

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
    width: calc(50% - 20px);
  }

  .control-container label {
    font-weight: 600;
    font-size: 16px;
    color: #333333;
    margin-bottom: 5px;
    display: block;
  }

  .control-container input {
    width: calc(100% - 20px);
    padding: 8px;
    font-size: 16px;
    border: 1px solid #ddd;
    border-radius: 5px;
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

  .preview-container {
    margin-top: 20px;
  }

  .preview {
    text-align: center;
  }

  .highlighted-text {
    color: #ff5722;
    font-weight: 600;
    font-size: 20px;
    margin-bottom: 10px;
  }

  .sprite-sheet {
    max-width: 100%;
    margin-top: 10px;
    border: 1px solid #ddd;
    border-radius: 5px;
  }

  .download-link {
    display: inline-block;
    padding: 10px 20px;
    background-color: #007bff;
    color: white;
    text-decoration: none;
    border-radius: 4px;
    margin-top: 10px;
    transition: background-color 0.3s ease;
  }

  .download-link:hover {
    background-color: #0056b3;
  }
</style>
