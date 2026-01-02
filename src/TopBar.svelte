<script lang="ts">
  import { onMount, getContext } from "svelte";
  import { Moon, Sun, X, ChevronDown } from "lucide-svelte";

  // --- Types ---
  interface Group {
    group_id: string;
    name: string;
  }

  interface Props {
    groups: Group[];
    selectedGroup: string;
    onGroupChange: (group: string) => void;
    selectedAvatar: string;
    onAvatarChange: (avatar: string) => void;
  }

  let {
    groups,
    selectedGroup,
    onGroupChange,
    selectedAvatar,
    onAvatarChange,
  }: Props = $props();

  // --- Global Context & Refs ---
  const url = getContext("LUOXU_URL");
  let groupWrapperRef: HTMLDivElement;
  let avatarWrapperRef: HTMLDivElement;
  let isDarkMode = $state(false);

  // --- Group Logic ---
  let showGroupDropdown = $state(false);
  let groupSearchQuery = $state("");

  // Derived state for local filtering is cleaner and faster
  let filteredGroups = $derived(
    groups.filter((g) =>
      g.name.toLowerCase().includes(groupSearchQuery.toLowerCase()),
    ),
  );

  const selectedGroupName = $derived(
    groups.find((g) => g.group_id === selectedGroup)?.name || "选择群组",
  );

  function handleSelectGroup(group: Group) {
    onGroupChange(group.group_id);
    showGroupDropdown = false;
    groupSearchQuery = "";
  }

  // --- Avatar Logic (Refactored) ---
  let showAvatarDropdown = $state(false);
  let avatarQuery = $state("");
  let avatarResults = $state<[string, string][]>([]); // Tuple: [id, name]
  let isAvatarLoading = $state(false);

  // Create a persistent AbortController and Timer
  let searchController: AbortController | null = null;
  let debounceTimer: ReturnType<typeof setTimeout>;

  let latestSearchId = 0;

  async function performAvatarSearch(query: string) {
    // Increment the ID for every new search attempt
    const currentSearchId = ++latestSearchId;

    if (!query || !selectedGroup) {
      avatarResults = [];
      isAvatarLoading = false;
      return;
    }

    if (searchController) searchController.abort();
    searchController = new AbortController();

    isAvatarLoading = true;

    try {
      const res = await fetch(
        `${url}/names?g=${selectedGroup}&q=${encodeURIComponent(query)}`,
        { signal: searchController.signal },
      );

      if (!res.ok) throw new Error("Network response was not ok");

      const data = await res.json();

      // --- CRITICAL FIX START ---
      // 1. Guard against stale requests (Racing)
      if (currentSearchId !== latestSearchId) return;

      const rawNames = Array.isArray(data.names) ? data.names : [];

      // 2. Deduplicate by ID to prevent Svelte {#each} crashes
      const uniqueMap = new Map();
      rawNames.forEach((item) => {
        // Ensure item is a valid tuple [id, name]
        if (Array.isArray(item) && item[0] != null) {
          const id = item[0];
          if (!uniqueMap.has(id)) {
            uniqueMap.set(id, item);
          }
        }
      });

      avatarResults = Array.from(uniqueMap.values());
      // --- CRITICAL FIX END ---
    } catch (e) {
      if (e.name !== "AbortError") {
        console.error("Avatar search failed:", e);
        if (currentSearchId === latestSearchId) avatarResults = [];
      }
    } finally {
      // Only stop the loading spinner if this is still the most recent request
      if (currentSearchId === latestSearchId) {
        isAvatarLoading = false;
      }
    }
  }

  // The input handler - Decoupled from binding for safety
  function onAvatarInput(e: Event) {
    const target = e.currentTarget as HTMLInputElement;
    const value = target.value;

    // Update state manually to ensure sync
    avatarQuery = value;

    // Clear previous timer
    clearTimeout(debounceTimer);

    if (!value) {
      avatarResults = [];
      return;
    }

    // Debounce the search by 150ms to prevent UI freezing and API spam
    debounceTimer = setTimeout(() => {
      performAvatarSearch(value);
    }, 150);
  }

  function handleSelectAvatar(id: string, name: string) {
    onAvatarChange(id);
    avatarQuery = name; // Set input to the name
    showAvatarDropdown = false;
  }

  // --- Lifecycle & Events ---

  // Theme logic
  onMount(() => {
    const saved = localStorage.getItem("theme");
    const sysDark = window.matchMedia("(prefers-color-scheme: dark)").matches;
    if (saved === "dark" || (!saved && sysDark)) {
      setTheme(true);
    }
  });

  function setTheme(dark: boolean) {
    isDarkMode = dark;
    if (dark) {
      document.documentElement.setAttribute("data-theme", "dark");
      localStorage.setItem("theme", "dark");
    } else {
      document.documentElement.removeAttribute("data-theme");
      localStorage.setItem("theme", "light");
    }
  }

  function handleWindowClick(e: MouseEvent) {
    const target = e.target as Node;

    // Robust check using DOM references instead of class names
    if (
      showGroupDropdown &&
      groupWrapperRef &&
      !groupWrapperRef.contains(target)
    ) {
      showGroupDropdown = false;
      groupSearchQuery = "";
    }

    if (
      showAvatarDropdown &&
      avatarWrapperRef &&
      !avatarWrapperRef.contains(target)
    ) {
      showAvatarDropdown = false;
    }
  }
</script>

<svelte:window onclick={handleWindowClick} />

<header class="topbar">
  <div class="topbar-content">
    <div class="group-section">
      <div class="custom-select-wrapper" bind:this={groupWrapperRef}>
        <div
          class="custom-select-trigger group-selector"
          role="button"
          tabindex="0"
          onclick={() => {
            showGroupDropdown = !showGroupDropdown;
            if (showGroupDropdown) groupSearchQuery = "";
          }}
          onkeydown={(e) => {
            if (e.key === "Enter" || e.key === " ") {
              showGroupDropdown = !showGroupDropdown;
              if (showGroupDropdown) groupSearchQuery = "";
            }
          }}
        >
          <span class="selected-value">{selectedGroupName}</span>
          <ChevronDown
            class={`chevron ${showGroupDropdown ? "rotated" : ""}`}
            size={16}
          />
        </div>

        {#if showGroupDropdown}
          <div class="group-select-dropdown group-search-wrapper">
            <div class="search-input-wrapper">
              <input
                type="text"
                placeholder="搜索群组..."
                class="search-input"
                bind:value={groupSearchQuery}
                onkeydown={(e) =>
                  e.key === "Escape" && (showGroupDropdown = false)}
                onclick={(e) => e.stopPropagation()}
              />
              {#if groupSearchQuery}
                <button
                  class="clear-button"
                  onclick={(e) => {
                    e.stopPropagation();
                    groupSearchQuery = "";
                  }}
                >
                  <X size={14} />
                </button>
              {/if}
            </div>

            <div class="options-list">
              {#each filteredGroups as group (group.group_id)}
                <div
                  class="option-item"
                  class:active={group.group_id === selectedGroup}
                  role="option"
                  aria-selected={group.group_id === selectedGroup}
                  tabindex="0"
                  onclick={() => handleSelectGroup(group)}
                  onkeydown={(e) =>
                    (e.key === "Enter" || e.key === " ") &&
                    handleSelectGroup(group)}
                >
                  {group.name}
                </div>
              {/each}
              {#if filteredGroups.length === 0 && groupSearchQuery}
                <div class="no-results">没有找到匹配的群组</div>
              {/if}
            </div>
          </div>
        {/if}
      </div>
    </div>

    <div class="right-controls">
      <div class="avatar-section">
        <div class="custom-select-wrapper" bind:this={avatarWrapperRef}>
          <div class="search-input-wrapper avatar-input-wrapper">
            <input
              type="text"
              placeholder="搜索用户..."
              class="search-input avatar-input"
              value={avatarQuery}
              oninput={onAvatarInput}
              onfocus={() => {
                showAvatarDropdown = true;
                if (avatarQuery && avatarResults.length === 0)
                  performAvatarSearch(avatarQuery);
              }}
              onkeydown={(e) => {
                if (e.key === "Escape") showAvatarDropdown = false;
                if (e.key === "Enter" && avatarResults.length > 0) {
                  handleSelectAvatar(avatarResults[0][0], avatarResults[0][1]);
                  e.preventDefault();
                }
              }}
            />

            {#if avatarQuery}
              <button
                class="clear-button"
                onclick={() => {
                  avatarQuery = "";
                  onAvatarChange("");
                  avatarResults = [];
                }}
              >
                <X size={14} />
              </button>
            {/if}
          </div>

          {#if showAvatarDropdown && (avatarResults.length > 0 || avatarQuery)}
            <div class="avatar-select-dropdown avatar-search-wrapper">
              <div class="options-list">
                {#each avatarResults as [id, name] (id)}
                  <div
                    class="option-item user-option"
                    role="option"
                    aria-selected={id === selectedAvatar}
                    tabindex="0"
                    onclick={() => handleSelectAvatar(id, name)}
                    onkeydown={(e) =>
                      (e.key === "Enter" || e.key === " ") &&
                      handleSelectAvatar(id, name)}
                  >
                    <img
                      src={`${url}/avatar/${id}.jpg`}
                      alt="avatar"
                      class="result-avatar"
                    />
                    <span class="result-name">{name}</span>
                  </div>
                {/each}

                {#if avatarResults.length === 0 && avatarQuery}
                  <div class="no-results">
                    {isAvatarLoading ? "Searching..." : "没有找到匹配的用户"}
                  </div>
                {/if}
              </div>
            </div>
          {/if}
        </div>

        <button
          class="theme-toggle"
          onclick={() => setTheme(!isDarkMode)}
          title={isDarkMode ? "切换到浅色模式" : "切换到深色模式"}
        >
          {#if isDarkMode}
            <Sun size={20} />
          {:else}
            <Moon size={20} />
          {/if}
        </button>
      </div>
    </div>
  </div>
</header>

<style>
  .topbar {
    background-color: var(--color-topbar-bg);
    border-bottom: 1px solid var(--color-border);
    z-index: 1000;
    backdrop-filter: blur(10px);
    transition: all 0.3s ease;
  }

  .topbar-content {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 1rem;
    padding-left: 1rem;
    padding-right: 1rem;
  }

  .group-section {
    display: flex;
    align-items: center;
    gap: 0.75rem;
    flex: 1;
    max-width: 500px;
  }

  .custom-select-wrapper {
    position: relative;
    width: 100%;
    max-width: 300px;
  }

  .custom-select-trigger {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0.5rem 1rem;
    background: var(--color-btn-bg);
    border: 1px solid var(--color-border);
    border-radius: 2rem;
    cursor: pointer;
    transition: all 0.2s ease;
    color: var(--color-text);
    min-width: 150px;
  }

  .custom-select-trigger:hover {
    background-color: var(--color-hover);
  }

  .custom-select-trigger:focus {
    outline: none;
    border-color: var(--color-active);
    box-shadow: 0 0 0 3px rgba(0, 136, 204, 0.1);
  }

  .selected-value {
    flex: 1;
    text-align: left;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .group-select-dropdown {
    position: absolute;
    top: calc(100% + 0.5rem);
    left: 0;
    right: 0;
    background: var(--color-topbar-bg);
    border: 1px solid var(--color-border);
    border-radius: 0.5rem;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    z-index: 10000;
    overflow: hidden;
    min-width: 200px;
  }

  .avatar-select-dropdown {
    position: absolute;
    top: calc(100% + 0.5rem);
    left: 0;
    right: 0;
    background: var(--color-topbar-bg);
    border: 1px solid var(--color-border);
    border-radius: 0.5rem;
    box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
    z-index: 10000;
    overflow: hidden;
    min-width: 200px;
  }

  .search-input-wrapper {
    position: relative;
    display: flex;
    align-items: center;
    flex: 1;
    padding: 0.5rem;
    border-bottom: 1px solid var(--color-border);
  }

  .search-input {
    flex: 1;
    padding: 0.5rem 1rem;
    border: 1px solid var(--color-border);
    border-radius: 2rem;
    background-color: var(--color-btn-bg);
    color: var(--color-text);
    font-size: 0.9rem;
    transition: all 0.2s ease;
    box-sizing: border-box;
    width: 1%;
  }

  .search-input:focus {
    outline: none;
    border-color: var(--color-active);
    box-shadow: 0 0 0 3px rgba(0, 136, 204, 0.1);
  }

  .clear-button {
    position: absolute;
    right: 1rem;
    background: none;
    border: none;
    color: var(--color-text-dim);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    width: 24px;
    height: 24px;
    padding: 0.25rem;
    border-radius: 50%;
    transition: all 0.2s ease;
  }

  .clear-button:hover {
    background-color: var(--color-hover);
    color: var(--color-text);
  }

  .avatar-input-wrapper {
    position: relative;
    display: flex;
    align-items: center;
    flex: 1;
  }

  .avatar-input {
    flex: 1;
    padding: 0.5rem 1rem;
    border: 1px solid var(--color-border);
    border-radius: 2rem;
    background-color: var(--color-btn-bg);
    color: var(--color-text);
    font-size: 0.9rem;
    transition: all 0.2s ease;
    box-sizing: border-box;
  }

  .options-list {
    max-height: 40vh;
    overflow-y: auto;
  }

  .option-item {
    padding: 0.75rem 1rem;
    cursor: pointer;
    transition: background-color 0.2s ease;
    color: var(--color-text);
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }

  .option-item:hover,
  .option-item.active {
    background-color: var(--color-hover);
  }

  .user-option {
    gap: 0.75rem;
  }

  .result-avatar {
    width: 32px;
    height: 32px;
    border-radius: 50%;
    object-fit: cover;
  }

  .result-name {
    flex: 1;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .no-results {
    padding: 0.75rem 1rem;
    text-align: center;
    color: var(--color-text-dim);
    font-style: italic;
  }

  .right-controls {
    display: flex;
    align-items: center;
    gap: 1rem;
    transition: all 0.3s ease;
  }

  .avatar-section {
    display: flex;
    align-items: center;
    gap: 0.75rem;
  }

  .theme-toggle {
    padding: 0.7rem;
    display: flex;
    align-items: center;
    justify-content: center;
    width: 40px;
    height: 40px;
    background: transparent;
    border: none;
    border-radius: 50%;
    cursor: pointer;
    transition: all 0.2s ease;
    color: var(--color-text);
  }

  .theme-toggle:hover {
    background-color: var(--color-hover);
  }

  @media (max-width: 768px) {
    .topbar-content {
      padding: 0 0.5rem;
    }

    .custom-select-trigger {
      padding: 0.5rem;
    }
  }
</style>
