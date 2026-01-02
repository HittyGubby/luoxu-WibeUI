<script lang="ts">
  import { Search, Send } from "lucide-svelte";

  interface Props {
    query: string;
    onQueryChange: (query: string) => void;
    onSearch: () => void;
    loading?: boolean;
  }

  let { query, onQueryChange, onSearch, loading = false }: Props = $props();
  let inputElement: HTMLInputElement;

  function handleKeydown(event: KeyboardEvent) {
    if (event.key === "Enter" && !event.shiftKey) {
      event.preventDefault();
      onSearch();
    }
  }

  function focusInput() {
    if (inputElement) {
      inputElement.focus();
    }
  }

  // Expose focus method to parent
  function focus() {
    focusInput();
  }
</script>

<div class="bottom-input-bar">
  <div class="input-container">
    <input
      bind:this={inputElement}
      type="text"
      placeholder="搜索消息..."
      bind:value={query}
      oninput={(e) => onQueryChange(e.target.value)}
      onkeydown={handleKeydown}
      class="search-input"
    />
    <button
      class="search-button"
      onclick={() => onSearch()}
      disabled={loading}
      title="搜索"
    >
      {#if loading}
        <div class="loading-spinner"></div>
      {:else}
        <Send size={20} />
      {/if}
    </button>
  </div>
</div>

<style>
  .bottom-input-bar {
    padding: 0.5rem;
    background-color: var(--color-bg);
    border-top: 1px solid var(--color-border);
    z-index: 1000;
    backdrop-filter: blur(10px);
    transition: all 0.3s ease;
  }

  .input-container {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    max-width: 100%;
    margin: 0 auto;
    background-color: var(--color-btn-bg);
    border: 1px solid var(--color-border);
    border-radius: 2rem;
    padding: 0.5rem 0.5rem;
    transition: all 0.2s ease;
  }

  .input-container:focus-within {
    border-color: var(--color-active);
    box-shadow: 0 0 0 3px rgba(0, 136, 204, 0.1);
  }

  .search-input {
    flex: 1;
    border: none;
    background: transparent;
    outline: none;
    font-size: 1rem;
    color: var(--color-text);
  }

  .search-input::placeholder {
    color: var(--color-text-dim);
  }

  .search-button {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 36px;
    height: 36px;
    background-color: var(--color-active);
    border: none;
    border-radius: 50%;
    color: white;
    cursor: pointer;
    transition: all 0.2s ease;
    flex-shrink: 0;
  }

  .search-button:hover:not(:disabled) {
    background-color: #0066aa;
    transform: scale(1.05);
  }

  .search-button:active:not(:disabled) {
    transform: scale(0.95);
  }

  .search-button:disabled {
    background-color: var(--color-text-dim);
    cursor: not-allowed;
    opacity: 0.6;
  }

  .loading-spinner {
    width: 20px;
    height: 20px;
    border: 2px solid rgba(255, 255, 255, 0.3);
    border-radius: 50%;
    border-top-color: white;
    animation: spin 1s ease-in-out infinite;
  }

  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }

  @media (max-width: 768px) {
    .bottom-input-bar {
      padding: 0.75rem;
    }

    .input-container {
      padding: 0.5rem 0.75rem;
    }

    .search-input {
      font-size: 0.9rem;
    }

    .search-button {
      width: 32px;
      height: 32px;
    }
  }
</style>
