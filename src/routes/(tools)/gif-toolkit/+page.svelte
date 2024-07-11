<script>
	import GifToSpriteSheet from './GifToSpriteSheet.svelte';
	import GifLoopCount from './GifLoopCount.svelte';
	import GifContrast from './GifContrast.svelte';
	import BoomerangGif from './BoomerangGif.svelte'; // Import the BoomerangGif component
	
	let activeComponent = null;
	
	function setActiveComponent(component) {
	  activeComponent = component;
	}
  </script>
  
  <main>
	<nav>
	  <button on:click={() => setActiveComponent('GifToSpriteSheet')}>
		GIF to Sprite Sheet
	  </button>
	  <button on:click={() => setActiveComponent('GifLoopCount')}>
		GIF Loop Count
	  </button>
	  <button on:click={() => setActiveComponent('GifContrast')}>
		GIF Contrast
	  </button>
	  <button on:click={() => setActiveComponent('BoomerangGif')}>
		Boomerang GIF
	  </button> <!-- Button for BoomerangGif component -->
	</nav>
  
	<section>
	  {#if activeComponent === 'GifToSpriteSheet'}
		<GifToSpriteSheet />
	  {:else if activeComponent === 'GifLoopCount'}
		<GifLoopCount />
	  {:else if activeComponent === 'GifContrast'}
		<GifContrast />
	  {:else if activeComponent === 'BoomerangGif'}
		<BoomerangGif />
	  {:else}
		<p>Click a button to start using a tool</p>
	  {/if}
	</section>
  </main>
  
  <style>
	main {
	  display: flex;
	  flex-direction: column;
	  align-items: center;
	  padding: 20px;
	}
  
	nav {
	  display: flex;
	  gap: 10px;
	  margin-bottom: 20px;
	}
  
	button {
	  padding: 10px 20px;
	  font-size: 16px;
	  background-color: #4CAF50;
	  color: white;
	  border: none;
	  border-radius: 5px;
	  cursor: pointer;
	  transition: transform 0.3s ease, box-shadow 0.3s ease;
	}
  
	button:hover, button:focus {
	  transform: translateY(-3px);
	  box-shadow: 0 4px 8px rgba(0,0,0,0.2);
	}
  
	section {
	  width: 100%;
	  max-width: 800px;
	}
  </style>
  