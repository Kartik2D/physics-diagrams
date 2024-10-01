<script>
  import { onMount } from 'svelte';

  let canvasContainer;
  let contentContainer;

  onMount(() => {
    // Any initialization logic can go here if needed
  });
</script>

<div class="viewer-container">
  <div class="canvas-area" bind:this={canvasContainer}>
    <slot name="canvas">
      <!-- Default content if no canvas is provided -->
      <p>Canvas area - Add your interactive graphics here</p>
    </slot>
  </div>
  <div class="content-area" bind:this={contentContainer}>
    <h2 class="content-title">
      <slot name="title">
        <!-- Default title if no title is provided -->
        <p>Default Title</p>
      </slot>
    </h2>
    <div class="content-body">
      <slot name="body">
        <!-- Default content if no body is provided -->
        <p>Content area - Add your text, controls, and other content here</p>
      </slot>
    </div>
  </div>
</div>

<style>
  :global(body) {
    background-color: #2a2a2a;
    color: #e0e0e0;
    font-family: 'Georgia', serif;
    line-height: 1.6;
  }

  .viewer-container {
    display: flex;
    width: 100%;
    height: 100vh;
    overflow: hidden;
    background-color: #333333;
  }

  .canvas-area {
    flex: 1;
    background-color: #2a2a2a;
    padding: 30px;
    overflow: auto;
    border-right: 1px solid #444444;
  }

  .content-area {
    flex: 1;
    display: flex;
    flex-direction: column;
    background-color: #333333;
    padding: 30px;
    overflow-y: auto;
  }

  .content-title {
    margin-top: 0;
    margin-bottom: 30px;
    font-size: 2.5em;
    color: #ffffff;
    border-bottom: 2px solid #4a4a4a;
    padding-bottom: 15px;
  }

  .content-body {
    flex: 1;
    overflow-y: auto;
    font-size: 1.1em;
  }

  .content-body :global(h3) {
    color: #b8b8b8;
    margin-top: 30px;
    margin-bottom: 15px;
  }

  .content-body :global(p) {
    margin-bottom: 20px;
  }

  .content-body :global(code) {
    background-color: #2a2a2a;
    padding: 2px 5px;
    border-radius: 3px;
    font-family: 'Courier New', monospace;
  }

  @media (max-width: 768px) {
    .viewer-container {
      flex-direction: column;
    }

    .canvas-area,
    .content-area {
      width: 100%;
      height: 50vh;
    }

    .canvas-area {
      border-right: none;
      border-bottom: 1px solid #444444;
    }
  }
</style>
