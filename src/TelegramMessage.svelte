<script lang="ts">
  import { onMount, getContext } from "svelte";
  import { format } from "date-fns";

  interface Props {
    msg: {
      from_name: string;
      t;
      edited;
      group_id: string;
      id: string;
      from_id: string;
      html: string;
    };
    groupinfo: string[][];
    now: Date;
    showDate?: boolean;
    dateText?: string;
    messageId?: string; // Add messageId prop
  }

  let {
    msg,
    groupinfo,
    now,
    showDate = false,
    dateText = "",
    messageId,
  }: Props = $props();

  const formatter = new Intl.DateTimeFormat(undefined, {
    timeStyle: "medium",
    dateStyle: "short",
    hour12: false,
  });

  const fullFormatter = new Intl.DateTimeFormat(undefined, {
    timeStyle: "full",
    dateStyle: "full",
    hour12: false,
  });

  let dt = new Date(msg.t * 1000);

  let edited = msg.edited ? new Date(msg.edited * 1000) : null;

  let title =
    format(dt, "yyyy/MM/dd HH:mm:ss") +
    (edited ? `\n最后编辑于：${format(edited, "yyyy/MM/dd HH:mm:ss")}` : "");

  let relative_dt = format_relative_time(dt, now);

  let iso_date = dt.toISOString();

  let time_display = format(dt, "yyyy/MM/dd HH:mm:ss");

  let msgurl = groupinfo[msg.group_id][0]
    ? `tg://resolve?domain=${groupinfo[msg.group_id][0]}&post=${msg.id}`
    : `tg://privatepost?channel=${msg.group_id}&post=${msg.id}`;

  function format_relative_time(d1: Date, d2: Date) {
    // in miliseconds
    const units = {
      year: 24 * 60 * 60 * 1000 * 365,
      month: (24 * 60 * 60 * 1000 * 365) / 12,
      day: 24 * 60 * 60 * 1000,
      hour: 60 * 60 * 1000,
      minute: 60 * 1000,
      second: 1000,
    };

    const rtf = new Intl.RelativeTimeFormat();
    // https://stackoverflow.com/a/60688789
    const elapsed = d1.valueOf() - d2.valueOf();

    for (const [u, period] of Object.entries(units)) {
      if (Math.abs(elapsed) > period || u === "second") {
        // https://stackoverflow.com/a/64972112
        return rtf.format(
          Math.round(elapsed / period),
          u as Intl.RelativeTimeFormatUnit,
        );
      }
    }
  }
</script>

{#if showDate}
  <div class="date-separator">
    <span>{dateText}</span>
  </div>
{/if}

<div class="telegram-message" data-message-id={messageId}>
  <img
    class="message-avatar"
    src="{getContext('LUOXU_URL')}/avatar/{msg.from_id}.jpg"
    height="40"
    width="40"
    alt="{msg.from_name} 的头像"
  />
  <div class="message-bubble">
    <div class="message-username">{msg.from_name || " "}</div>
    <div class="message-text">{@html msg.html}</div>
    <div class="message-footer">
      <span class="message-id">#{msg.id}</span>
      <div class="message-time-container">
        <a href={msgurl} class="message-time" {title}>
          <span class="time-display">{time_display}</span>
        </a>
        {#if edited}
          <span class="edited-indicator">edited</span>
        {/if}
      </div>
    </div>
  </div>
</div>

<style>
  .telegram-message {
    display: flex;
    padding-left: 0.5rem;
    margin-bottom: 0.5rem;
  }

  .message-avatar {
    width: 40px;
    height: 40px;
    border-radius: 50%;
    margin-right: 0.75rem;
    flex-shrink: 0;
    object-fit: cover;
    align-self: flex-end;
    border: 2px solid var(--color-border);
  }

  .message-bubble {
    flex: 1;
    min-width: 500px;
    max-width: 30%;
    background-color: var(--color-message-bg);
    border: 2px solid var(--color-border);
    border-radius: 0.75rem;
    padding: 0.5rem 0.75rem;
    position: relative;
  }

  /* override on small screens to use ~80% width */
  @media (max-width: 768px) {
    .message-bubble {
      min-width: auto;
      width: 80%;
      max-width: 80%;
    }
  }

  .message-username {
    font-weight: 600;
    color: var(--color-username);
    font-size: 0.85rem;
    margin-bottom: 0.25rem;
  }

  .message-text {
    white-space: pre-wrap;
    word-wrap: break-word;
    line-height: 1.4;
    color: var(--color-text);
    margin-bottom: 0.25rem;
  }

  .message-text :global(.keyword) {
    background-color: var(--color-keyword-bg);
    padding: 0 2px;
    border-radius: 2px;
  }

  .message-footer {
    display: flex;
    justify-content: space-between;
    align-items: flex-end;
    gap: 0.5rem;
  }

  .message-id {
    font-size: 0.7rem;
    color: var(--color-text-dim);
    opacity: 0.5;
  }

  .message-time-container {
    display: flex;
    align-items: center;
    gap: 0.25rem;
  }

  .message-time {
    text-decoration: none;
    color: var(--color-text-dim);
    font-size: 0.75rem;
    transition: color 0.2s ease;
  }

  .message-time:hover {
    color: var(--color-active);
    text-decoration: underline;
  }

  .time-display {
    font-size: 0.75rem;
  }

  .edited-indicator {
    font-size: 0.65rem;
    color: var(--color-text-dim);
    opacity: 0.7;
  }

  .date-separator {
    position: relative;
    text-align: center;
    padding: 0;
    margin: 0.5rem;
  }

  .date-separator::before {
    content: "";
    position: absolute;
    top: 50%;
    left: 0;
    right: 0;
    height: 1px;
    background: var(--color-border);
  }

  .date-separator span {
    background: var(--color-bg);
    padding: 0rem 0.75rem;
    color: var(--color-text-dim);
    font-size: 0.8rem;
    border-radius: 1rem;
    position: relative;
    z-index: 1;
    font-weight: 500;

    outline: 3px solid var(--color-border);
  }

  @media (max-width: 768px) {
    .telegram-message {
      padding: 0.5rem 0.75rem;
    }

    .message-avatar {
      width: 36px;
      height: 36px;
      margin-right: 0.5rem;
    }

    .message-username {
      font-size: 0.85rem;
    }

    .message-text {
      font-size: 0.9rem;
    }
  }
</style>
