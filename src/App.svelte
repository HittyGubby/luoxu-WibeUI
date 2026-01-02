<script lang="ts">
  import { run } from "svelte/legacy";
  import { onMount, setContext } from "svelte";
  import TelegramMessage from "./TelegramMessage.svelte";
  import TopBar from "./TopBar.svelte";
  import BottomInputBar from "./BottomInputBar.svelte";
  import { sleep } from "./util.js";
  import { format } from "date-fns";
  import "./global.css";

  const LUOXU_URL = "https://apps.archlinuxcn.org/luoxu";
  const islocal = LUOXU_URL.startsWith("http://localhost");
  let groups: { group_id: string; name: string }[] = $state([]);
  let group: string = $state();
  let query: string = $state();
  let error: string = $state();
  let result: {
    messages: {
      from_name: string;
      t: any;
      edited: any;
      group_id: string;
      id: string;
      from_id: string;
      html: string;
    }[];
    has_more: boolean;
    groupinfo: string[][];
  } = $state();
  let now = $state(new Date());
  let loading = $state(false);
  let need_update_title = $state(false);
  let sender: string = $state();
  let selected_init: string = $state();
  let our_hash_change = $state(false);
  let abort = new AbortController();
  let messagesContainer: HTMLElement;
  let isLoadingMore = $state(false);
  let scrollThreshold = 100; // Pixels from top to trigger loading

  setContext("LUOXU_URL", LUOXU_URL);

  // Load saved preferences from localStorage
  function loadPreferences() {
    const savedGroup = localStorage.getItem("selectedGroup");
    const savedAvatar = localStorage.getItem("selectedAvatar");
    const savedQuery = localStorage.getItem("lastQuery");

    if (savedGroup) group = savedGroup;
    if (savedAvatar) sender = savedAvatar;
    if (savedQuery) query = savedQuery;
  }

  // Save preferences to localStorage
  function savePreferences() {
    if (group) localStorage.setItem("selectedGroup", group);
    if (sender) localStorage.setItem("selectedAvatar", sender);
    if (query) localStorage.setItem("lastQuery", query);
  }

  function parse_hash() {
    const hash = location.hash;
    if (hash) {
      return new URLSearchParams(hash.substring(1));
    }
  }

  onMount(async () => {
    loadPreferences();
    do_hash_search();
    while (true) {
      try {
        const res = await fetch(`${LUOXU_URL}/groups`);
        groups = (await res.json()).groups;
        need_update_title = true;
        if (!group) {
          group = "";
        }
        break;
      } catch (e) {
        console.error("failed to fetch group info, will retry", e);
        await sleep(1000);
      }
    }
  });

  run(() => {
    // only update title on hash change (doing a search)
    if (need_update_title && groups) {
      let group_name: string;
      for (const g of groups) {
        if (g.group_id === group) {
          group_name = g.name;
        }
      }
      if (query && group_name) {
        document.title = `搜索：${query} 于 ${group_name} - 落絮`;
      } else if (query) {
        document.title = `搜索：${query} - 落絮`;
      } else if (group_name) {
        document.title = `搜索 ${group_name} - 落絮`;
      } else {
        document.title = "落絮";
      }
      need_update_title = false;
    }
  });

  function do_hash_search() {
    const info = parse_hash();
    if (info) {
      query = "";
      group = "";
      result = null;
      if (info.has("g")) {
        group = info.get("g");
      }
      if (info.has("q")) {
        query = info.get("q");
      }
      if (info.has("sender")) {
        sender = info.get("sender");
        selected_init = sender;
      }
      if ((group || islocal) && query) {
        result = null;
        do_search();
      }
    }
  }

  let concurrency = 0;
  async function do_search(more?: any) {
    concurrency += 1;
    try {
      abort.abort();
      abort = new AbortController();

      // Check if 'more' is actually an event object (like a click event) rather than a timestamp
      // If it looks like an event object, ignore it and treat this as an initial search
      let actualMore = more;
      if (more && typeof more === "object" && (more.type || more.target)) {
        // This looks like an event object, so this is likely a call from a button click
        // Treat this as an initial search by ignoring the 'more' parameter
        actualMore = undefined;
      }

      // Only require group if not local and not searching for all messages
      if (!group && !islocal && (query || sender)) {
        error = "请选择要搜索的群组";
        return;
      }

      // If all parameters are empty, don't perform search
      if (!query && !sender && !group) {
        error = "请输入搜索关键字或选择群组/用户";
        return;
      }

      error = "";
      our_hash_change = true;
      console.log(
        `searching ${query} for group ${group}, older than ${actualMore}, from ${sender}`,
      );

      const q = new URLSearchParams();
      if (group) {
        q.append("g", group);
      }
      if (query) {
        q.append("q", query);
      }
      if (sender) {
        q.append("sender", sender);
      }

      let url: RequestInfo | URL;
      const qstr = q.toString();
      if (!actualMore) {
        // Update hash only for initial searches, not for pagination
        location.hash = `#${qstr}`;
        need_update_title = true;
        if (result) {
          result.messages = [];
        }
        url = `${LUOXU_URL}/search?${qstr}`;
      } else {
        // For subsequent searches, use the same parameters as the original search
        // Don't update the hash during pagination
        url = `${LUOXU_URL}/search?${q}&end=${actualMore}`;
      }

      now = new Date();
      loading = true;
      try {
        const res = await fetch(url, { signal: abort.signal });
        const r = await res.json();
        loading = false;
        if (abort.signal.aborted) {
          return;
        }
        if (actualMore) {
          // For loading more, append older messages
          // In column-reverse, the last items in the array appear at the top visually
          // So to add older messages that appear at the visual top, we append to the array
          if (result) {
            // Only add messages that are not already in our list
            // To avoid reordering existing messages, we'll just append new ones
            const existingIds = new Set(result.messages.map((m) => m.id));
            const newMessages = r.messages.filter(
              (newMsg) => !existingIds.has(newMsg.id),
            );

            if (newMessages.length > 0) {
              // Add older messages to the end of the array to appear above visually
              result.messages = [...result.messages, ...newMessages];
            }
            result.has_more = r.has_more;
          }
        } else {
          result = r;
          // Scroll to bottom (newest messages) after new search results
          setTimeout(() => {
            if (messagesContainer) {
              messagesContainer.scrollTop = messagesContainer.scrollHeight;
            }
          }, 100);
        }
      } catch (e) {
        if (concurrency <= 1) {
          error = e;
          loading = false;
        }
      }

      our_hash_change = false;
    } finally {
      concurrency -= 1;
    }
  }

  async function on_group_change(newGroup: string) {
    console.log("on_group_change called with:", newGroup);
    group = newGroup;
    console.log("Group state updated to:", group);
    savePreferences();
    error = "";
    // Only search if we have a query or sender
    if (query || sender) {
      await do_search();
    }
  }

  async function do_search_more() {
    if (!result || result.messages.length === 0) return;

    // In column-reverse layout, the oldest messages are at the end of the array
    // So result.messages[result.messages.length - 1] is the oldest (visually at the top)
    const oldestMessage = result.messages[result.messages.length - 1];
    const more = oldestMessage.t;

    try {
      // Call do_search with the timestamp to load older messages
      await do_search(more);
    } catch (e) {
      console.error("Failed to load more messages:", e);
      error = e;
    }
  }

  function format_date(timestamp: number) {
    const d = new Date(timestamp * 1000);
    return format(d, "yyyy/MM/dd");
  }

  function should_show_date_separator(
    current_msg: any,
    previous_msg: any,
    index: number,
    total: number,
  ) {
    // Don't show date separator before the first (newest) message
    if (index === 0) return false;
    if (!previous_msg) return false;

    const current_date = format(new Date(current_msg.t * 1000), "yyyy/MM/dd");
    const previous_date = format(new Date(previous_msg.t * 1000), "yyyy/MM/dd");

    return current_date !== previous_date;
  }

  function handleScroll(event?: Event) {
    if (!messagesContainer || isLoadingMore || !result?.has_more) return;

    // When user scrolls near the top (older messages), load more messages
    // With column-reverse, the top of the scroll is where old messages are
    const scrollTop = messagesContainer.scrollTop;
    const threshold = 100; // pixels from top to trigger loading

    if (scrollTop <= threshold) {
      loadMoreMessages();
    }
  }

  async function loadMoreMessages() {
    if (isLoadingMore || !result?.has_more) return;

    isLoadingMore = true;

    // Record the current scroll position before loading
    const currentScrollTop = messagesContainer.scrollTop;
    const currentHeight = messagesContainer.scrollHeight;

    try {
      await do_search_more();

      // Calculate the height difference after loading new messages
      const newHeight = messagesContainer.scrollHeight;
      const heightDiff = newHeight - currentHeight;

      // Adjust scroll position to maintain the same visual position
      // Add a small buffer to avoid triggering load again immediately
      messagesContainer.scrollTop = currentScrollTop + heightDiff + 50;
    } catch (e) {
      console.error("Error loading more messages:", e);
    } finally {
      isLoadingMore = false;
    }
  }
</script>

<svelte:window
  onhashchange={() => {
    if (!our_hash_change) do_hash_search();
  }}
/>

<div class="app-container">
  <TopBar
    {groups}
    selectedGroup={group}
    onGroupChange={on_group_change}
    selectedAvatar={sender}
    onAvatarChange={(avatar) => {
      sender = avatar;
      savePreferences();
      // Don't automatically trigger search here - only search when user clicks the search button
    }}
  />

  <main
    class="messages-container"
    bind:this={messagesContainer}
    onscroll={handleScroll}
  >
    {#if result && result.messages.length > 0}
      <div class="messages-list">
        {#each result.messages as message, index}
          {@const showDate = should_show_date_separator(
            message,
            result.messages[index - 1],
            index,
            result.messages.length,
          )}
          {@const dateText = showDate ? format_date(message.t) : ""}
          <TelegramMessage
            msg={message}
            groupinfo={result.groupinfo}
            {now}
            {showDate}
            {dateText}
            messageId={message.id}
          />
        {/each}
      </div>
    {:else if !loading && !error}
      <div class="welcome-screen">
        <div class="welcome-content">
          <h2>落絮消息搜索</h2>
          <p>
            搜索消息时，搜索字符串不区分简繁（会使用 OpenCC
            自动转换），也不进行分词（请手动将可能不连在一起的词语以空格分开）。
          </p>
          <p>搜索字符串支持以下功能：</p>
          <ul>
            <li>以空格分开的多个搜索词是「与」的关系</li>
            <li>使用 OR（全大写）来表达「或」条件</li>
            <li>使用 - 来表达排除，如 落絮 - 测试</li>
            <li>使用小括号来分组</li>
          </ul>
          <p>人名补全支持上下方向键和 Alt+N/P 进行选择。</p>
          <p>
            搜索结果右下角的时间，悬停可查看绝对时间、最后编辑时间（如编辑过），点击可跳转到
            Telegram 中展示该消息。
          </p>
        </div>
      </div>
    {/if}

    {#if loading && !result?.messages.length}
      <div class="loading-indicator">
        <div class="loading-spinner"></div>
        <p>正在加载...</p>
      </div>
    {/if}

    {#if !loading || result?.messages.length > 0}
      {#if error}
        <div class="error-message">
          <p>{error}</p>
        </div>
      {:else if result && result.messages.length === 0 && !loading}
        <div class="no-results">
          <p>没有匹配的消息。</p>
        </div>
      {/if}
    {/if}
  </main>

  <BottomInputBar
    {query}
    onQueryChange={(q) => {
      query = q;
      savePreferences();
    }}
    onSearch={do_search}
    {loading}
  />
</div>

<style>
  .app-container {
    height: 100vh;
    display: flex;
    flex-direction: column;
    background-color: var(--color-bg);
    background-image: url("/topography.svg");
  }

  .messages-container {
    flex: 1;
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    min-height: 0; /* Important: allows flex child to shrink properly */
  }

  .messages-list {
    display: flex;
    flex-direction: column-reverse; /* Messages flow from bottom to top */
    flex: 1;
  }

  .welcome-screen {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 2rem;
  }

  .welcome-content {
    max-width: 600px;
    text-align: center;
    line-height: 1.6;
  }

  .welcome-content h2 {
    color: var(--color-text);
    margin-bottom: 1.5rem;
    font-size: 1.5rem;
  }

  .welcome-content p {
    color: var(--color-text);
    margin-bottom: 1rem;
  }

  .welcome-content ul {
    text-align: left;
    color: var(--color-text);
    margin: 1rem 0;
  }

  .loading-indicator {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 2rem;
    gap: 1rem;
  }

  .loading-spinner {
    width: 40px;
    height: 40px;
    border: 3px solid var(--color-border);
    border-radius: 50%;
    border-top-color: var(--color-active);
    animation: spin 1s ease-in-out infinite;
  }

  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }

  .error-message,
  .no-results {
    display: flex;
    justify-content: center;
    padding: 1rem;
  }

  .error-message p {
    color: var(--color-error);
    text-align: center;
  }

  .no-results p {
    color: var(--color-text-dim);
    text-align: center;
    background-color: var(--color-btn-bg);
    padding: 0.5rem 1rem;
    border-radius: 2rem;
    border: 1px solid var(--color-border);
  }

  @media (max-width: 768px) {
    .welcome-content {
      padding: 1rem;
    }

    .welcome-content h2 {
      font-size: 1.3rem;
    }
  }
</style>
