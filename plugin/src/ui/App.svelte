<script lang="ts">
  import { onMount } from "svelte";
  import ActivityPanel from "./panels/ActivityPanel.svelte";
  import TimelinePanel from "./panels/TimelinePanel.svelte";
  import DebugPanel from "./panels/DebugPanel.svelte";
  import SessionPanel from "./panels/SessionPanel.svelte";
  import type { ActiveRequest, DebugEvent, SessionSummary, TimelineEvent } from "./model";

  let connected = false;
  let currentSessionId = "—";
  let fileName = "—";
  let pageName = "—";
  let selectionCount = 0;
  let knownSessions: SessionSummary[] = [];
  let catalogReady = false;

  let activeRequestMap: Record<string, ActiveRequest> = {};
  let debugEvents: DebugEvent[] = [];
  let timelineEvents: TimelineEvent[] = [];
  let showDebug = false;
  let showTimeline = false;
  let showLogs = false;
  let showSessions = false;

  $: activeRequests = Object.values(activeRequestMap).sort((a, b) => a.startedAt - b.startedAt);
  $: isWorking = activeRequests.length > 0;
  $: sessionCount = catalogReady ? knownSessions.length : 0;
  $: activeSessionMeta =
    knownSessions.find((session) => session.sessionId === currentSessionId) ??
    ({ sessionId: currentSessionId, fileName, pageName, selectionCount } as SessionSummary);
  $: defaultRouteLabel =
    catalogReady && activeSessionMeta?.fileName
      ? `Default route -> ${activeSessionMeta.fileName}`
      : "Default route chưa được xác nhận";
  $: adminURL = `http://${serverHost}:${serverPort}/admin`;
  $: statusLabel = connected ? (isWorking ? "Đang điều phối" : "Sẵn sàng") : "Mất kết nối";
  $: activitySummary = isWorking
    ? `${activeRequests.length} capability đang chạy`
    : connected
      ? "Đang chờ toolcall"
      : "Chờ kết nối plugin";

  let serverHost = "127.0.0.1";
  let serverPort = "1802";

  let showSettings = false;
  let editHost = serverHost;
  let editPort = serverPort;

  const RECONNECT_DELAY_MS = 1500;
  const MAX_DEBUG_EVENTS = 36;
  const MAX_TIMELINE_EVENTS = 16;
  const REQUEST_SETTLE_DELAY_MS = 900;

  let socket: WebSocket | null = null;
  let reconnectTimer: ReturnType<typeof setTimeout> | null = null;
  let sessionRetryTimer: ReturnType<typeof setTimeout> | null = null;
  let configLoaded = false;
  const cleanupTimers = new Map<string, ReturnType<typeof setTimeout>>();

  const reassignRequests = () => {
    activeRequestMap = { ...activeRequestMap };
  };

  const addDebugEvent = (
    tone: DebugEvent["tone"],
    tool: string,
    stage: string,
    message: string,
    requestId?: string,
  ) => {
    const now = new Date();
    debugEvents = [
      ...debugEvents,
      {
        id: `${Date.now()}-${Math.random().toString(36).slice(2, 8)}`,
        tone,
        tool,
        stage,
        requestId,
        message,
        timestamp: now.toLocaleTimeString(),
      },
    ].slice(-MAX_DEBUG_EVENTS);
  };

  const clearCleanupTimer = (requestId: string) => {
    const timer = cleanupTimers.get(requestId);
    if (timer) {
      clearTimeout(timer);
      cleanupTimers.delete(requestId);
    }
  };

  const pushTimelineEvent = (
    tool: string,
    requestId: string,
    status: TimelineEvent["status"],
    message: string,
    durationMs: number,
    sessionId?: string,
  ) => {
    const now = new Date();
    timelineEvents = [
      {
        id: `${requestId}-${Date.now()}`,
        tool,
        requestId,
        status,
        message,
        durationMs,
        timestamp: now.toLocaleTimeString(),
        sessionId,
      },
      ...timelineEvents.filter((item) => item.requestId !== requestId),
    ].slice(0, MAX_TIMELINE_EVENTS);
  };

  const scheduleFinishRequest = (requestId: string) => {
    clearCleanupTimer(requestId);
    cleanupTimers.set(
      requestId,
      setTimeout(() => {
        delete activeRequestMap[requestId];
        cleanupTimers.delete(requestId);
        reassignRequests();
      }, REQUEST_SETTLE_DELAY_MS),
    );
  };

  const ensureActiveRequest = (requestId: string, tool: string) => {
    if (!activeRequestMap[requestId]) {
      activeRequestMap[requestId] = {
        requestId,
        tool,
        stage: "queued",
        progress: 0,
        message: "Runtime đã nhận toolcall",
        startedAt: Date.now(),
        lastDebugMessage: "",
        lastDebugProgress: -1,
      };
      reassignRequests();
    }
    return activeRequestMap[requestId];
  };

  const maybeLogProgress = (entry: ActiveRequest, message: string, progress: number) => {
    const crossedThreshold =
      entry.lastDebugProgress < 0 || Math.abs(progress - entry.lastDebugProgress) >= 10;
    const messageChanged = message !== entry.lastDebugMessage;
    if (!crossedThreshold && !messageChanged) return;
    entry.lastDebugProgress = progress;
    entry.lastDebugMessage = message;
    addDebugEvent("info", entry.tool, "progress", message, entry.requestId);
  };

  const requestSessionCatalog = () => {
    if (socket?.readyState !== WebSocket.OPEN) return;
    addDebugEvent("info", "runtime", "catalog-request", "Requesting session catalog from bridge");
    socket.send(JSON.stringify({ type: "session_request_catalog" }));
  };

  const openAdmin = () => {
    const url = adminURL;
    addDebugEvent("info", "runtime", "open-admin", `Opening ${url}`);
    parent.postMessage({ pluginMessage: { type: "open_external", url } }, "*");
  };

  const copyAdminURL = async () => {
    try {
      await navigator.clipboard.writeText(adminURL);
      addDebugEvent("success", "runtime", "copy-admin-url", `Copied ${adminURL}`);
    } catch {
      addDebugEvent("warn", "runtime", "copy-admin-url", `Could not copy ${adminURL}`);
    }
  };

  const switchSession = (sessionId: string) => {
    if (!sessionId || sessionId === currentSessionId || socket?.readyState !== WebSocket.OPEN) return;
    addDebugEvent("info", "runtime", "session-switch", `Switching active session to ${sessionId}`);
    socket.send(JSON.stringify({ type: "session_switch", sessionId }));
  };

  const switchSessionAndTest = (sessionId: string) => {
    if (!sessionId || socket?.readyState !== WebSocket.OPEN) return;
    switchSession(sessionId);
    addDebugEvent("info", "runtime", "route-test", `Switching and probing ${sessionId}`);
    setTimeout(() => {
      requestSessionCatalog();
    }, 240);
  };

  const announceSession = () => {
    if (socket?.readyState !== WebSocket.OPEN || currentSessionId === "—") return;
    addDebugEvent("info", "runtime", "session-announce", `Announcing ${currentSessionId}`);
    socket.send(
      JSON.stringify({
        type: "session_announce",
        sessionId: currentSessionId,
        fileName,
        pageName,
        selectionCount,
      }),
    );
    requestSessionCatalog();
  };

  const stopSessionRetry = () => {
    if (sessionRetryTimer !== null) {
      clearTimeout(sessionRetryTimer);
      sessionRetryTimer = null;
    }
  };

  const scheduleSessionRetry = () => {
    stopSessionRetry();
    if (catalogReady || socket?.readyState !== WebSocket.OPEN || currentSessionId === "—") return;
    sessionRetryTimer = setTimeout(() => {
      sessionRetryTimer = null;
      if (!catalogReady) {
        announceSession();
        scheduleSessionRetry();
      }
    }, 1800);
  };

  function connect() {
    if (socket) {
      socket.onclose = null;
      socket.close();
    }
    const ws = new WebSocket(`ws://${serverHost}:${serverPort}/ws`);
    socket = ws;

    ws.onopen = () => {
      connected = true;
      addDebugEvent("success", "bridge", "connected", `Connected to ${serverHost}:${serverPort}`);
      parent.postMessage({ pluginMessage: { type: "ui-ready" } }, "*");
      announceSession();
      requestSessionCatalog();
      scheduleSessionRetry();
    };

    ws.onclose = () => {
      if (socket !== ws) return;
      connected = false;
      socket = null;
      catalogReady = false;
      knownSessions = [];
      stopSessionRetry();
      activeRequestMap = {};
      cleanupTimers.forEach((timer) => clearTimeout(timer));
      cleanupTimers.clear();
      addDebugEvent("warn", "bridge", "disconnected", `Disconnected from ${serverHost}:${serverPort}`);
      if (reconnectTimer === null) {
        reconnectTimer = setTimeout(() => {
          reconnectTimer = null;
          connect();
        }, RECONNECT_DELAY_MS);
      }
    };

    ws.onerror = () => {
      connected = false;
      addDebugEvent("error", "bridge", "socket-error", `Socket error at ${serverHost}:${serverPort}`);
    };

    ws.onmessage = (event) => {
      try {
        const payload = JSON.parse(event.data);

        if (payload.type === "session_catalog") {
          const active = payload.data?.activeSession ?? currentSessionId;
          const sessions = Array.isArray(payload.data?.sessions) ? payload.data.sessions : [];
          catalogReady = true;
          stopSessionRetry();
          addDebugEvent(
            "success",
            "runtime",
            "catalog-received",
            `Bridge returned ${sessions.length} session(s)`,
          );
          currentSessionId = active;
          knownSessions = sessions
            .map((session: any) => ({
              sessionId: session.sessionId,
              fileName: session.fileName ?? "Untitled file",
              pageName: session.pageName ?? "Unknown page",
              selectionCount: session.selectionCount ?? 0,
            }))
            .sort((a, b) => a.fileName.localeCompare(b.fileName));
          const resolvedActive = sessions.find((session: any) => session.sessionId === active);
          if (resolvedActive?.fileName) {
            addDebugEvent(
              "success",
              "runtime",
              "default-route",
              `Default route -> ${resolvedActive.fileName}`,
            );
          }
          return;
        }

        if (payload.type === "execution_report") {
          const report = payload.data ?? {};
          const capability = report.capability ?? "tool";
          const requestId = payload.requestId ?? report.requestId ?? `exec-${Date.now()}`;
          const durationMs = Number(report.durationMs ?? 0);
          const resultClass = report.resultClass ?? "complete";
          const fallbackUsed = !!report.fallbackUsed;
          const attempts = Array.isArray(report.attempts) ? report.attempts.length : 0;
          const fallbackPath = Array.isArray(report.fallbackPath) ? report.fallbackPath.join(" → ") : "";
          const status: TimelineEvent["status"] =
            resultClass === "failed" ? "error" : fallbackUsed || resultClass === "partial" || resultClass === "fallback" ? "warn" : "success";
          const message = fallbackPath
            ? `${resultClass} · ${attempts} attempt(s) · ${fallbackPath}`
            : fallbackUsed
              ? `${resultClass} · ${attempts} attempt(s) · fallback`
              : `${resultClass} · ${attempts} attempt(s)`;
          pushTimelineEvent(capability, requestId, status, message, durationMs, payload.sessionId);
          addDebugEvent(
            status === "error" ? "error" : status === "warn" ? "warn" : "success",
            capability,
            "execution-report",
            message,
            requestId,
          );
          return;
        }

        if (payload.requestId) {
          ensureActiveRequest(payload.requestId, payload.type ?? "tool");
          addDebugEvent("info", payload.type ?? "tool", "received", "Server request received", payload.requestId);
        }
        parent.postMessage({ pluginMessage: { type: "server-request", payload } }, "*");
      } catch {
        addDebugEvent("warn", "bridge", "malformed", "Ignored malformed frame from server");
      }
    };
  }

  function handleMessage(event: MessageEvent) {
    const msg = event.data?.pluginMessage;
    if (!msg) return;

    if (msg.type === "ws_config") {
      serverHost = msg.host ?? "127.0.0.1";
      serverPort = msg.port ?? "1802";
      if (!configLoaded) {
        configLoaded = true;
        connect();
      }
      return;
    }

    if (msg.type === "plugin-status") {
      currentSessionId = msg.payload.sessionId ?? currentSessionId;
      fileName = msg.payload.fileName;
      pageName = msg.payload.pageName ?? "—";
      selectionCount = msg.payload.selectionCount;
      announceSession();
      scheduleSessionRetry();
      return;
    }

    if (msg.type === "request_event") {
      const payload = msg.payload ?? {};
      const requestId = payload.requestId;
      const tool = payload.type ?? "tool";
      if (typeof requestId === "string") {
        clearCleanupTimer(requestId);
        const entry = ensureActiveRequest(requestId, tool);
        entry.stage = msg.stage;
        if (msg.stage === "start") {
          entry.message = "Plugin bắt đầu xử lý";
          addDebugEvent("info", tool, "start", "Plugin started handling request", requestId);
        } else if (msg.stage === "success") {
          entry.progress = 100;
          entry.message = `Hoàn tất trong ${payload.durationMs ?? 0} ms`;
          addDebugEvent("success", tool, "success", entry.message, requestId);
          pushTimelineEvent(tool, requestId, "success", entry.message, Number(payload.durationMs ?? 0), currentSessionId);
          scheduleFinishRequest(requestId);
        } else if (msg.stage === "error") {
          entry.progress = 100;
          entry.message = payload.error ?? "Plugin request failed";
          addDebugEvent("error", tool, "error", entry.message, requestId);
          pushTimelineEvent(tool, requestId, "error", entry.message, Number(payload.durationMs ?? 0), currentSessionId);
          scheduleFinishRequest(requestId);
        }
        reassignRequests();
      }
      return;
    }

    if (msg.type === "progress_update" && typeof msg.requestId === "string") {
      clearCleanupTimer(msg.requestId);
      const entry = ensureActiveRequest(msg.requestId, activeRequestMap[msg.requestId]?.tool ?? "tool");
      entry.stage = "progress";
      entry.progress = typeof msg.progress === "number" ? msg.progress : entry.progress;
      entry.message = msg.message ?? "Đang xử lý";
      maybeLogProgress(entry, entry.message, entry.progress);
      reassignRequests();
      if (socket?.readyState === WebSocket.OPEN) {
        socket.send(JSON.stringify(msg));
      }
      return;
    }

    if ("requestId" in msg) {
      const requestId = typeof msg.requestId === "string" ? msg.requestId : "";
      const tool = msg.type ?? activeRequestMap[requestId]?.tool ?? "tool";
      if (requestId) {
        clearCleanupTimer(requestId);
        const entry = ensureActiveRequest(requestId, tool);
        entry.progress = 100;
        entry.stage = msg.error ? "error" : "done";
        entry.message = msg.error ? msg.error : "Đã phản hồi về server";
        addDebugEvent(msg.error ? "error" : "success", tool, entry.stage, entry.message, requestId);
        reassignRequests();
        scheduleFinishRequest(requestId);
      }
      if (socket?.readyState === WebSocket.OPEN) {
        socket.send(JSON.stringify(msg));
      }
    }
  }

  function openSettings() {
    editHost = serverHost;
    editPort = serverPort;
    showSettings = true;
  }

  function applySettings() {
    serverHost = editHost.trim() || "127.0.0.1";
    const parsed = parseInt(editPort, 10);
    serverPort = parsed > 0 && parsed <= 65535 ? String(parsed) : "1802";
    parent.postMessage(
      { pluginMessage: { type: "save_ws_config", host: serverHost, port: serverPort } },
      "*",
    );
    showSettings = false;
    if (reconnectTimer !== null) {
      clearTimeout(reconnectTimer);
      reconnectTimer = null;
    }
    connect();
  }

  function handleKeydown(event: KeyboardEvent) {
    if (event.key === "Enter") applySettings();
    if (event.key === "Escape") showSettings = false;
  }

  onMount(() => {
    window.addEventListener("message", handleMessage);
    parent.postMessage({ pluginMessage: { type: "get_ws_config" } }, "*");

    const fallback = setTimeout(() => {
      if (!configLoaded) {
        configLoaded = true;
        connect();
      }
    }, 500);

    return () => {
      clearTimeout(fallback);
      window.removeEventListener("message", handleMessage);
      if (reconnectTimer !== null) clearTimeout(reconnectTimer);
      stopSessionRetry();
      cleanupTimers.forEach((timer) => clearTimeout(timer));
      cleanupTimers.clear();
      if (socket) socket.close();
    };
  });
</script>

<div class="shell">
  <section class="console-bar">
    <div class="console-title-row">
      <div class="console-title">
        <div class="eyebrow">Za-talk-to-figma Runtime</div>
        <h1>Runtime Console</h1>
      </div>
      <div class:status-pill={true} class:connected class:disconnected={!connected}>
        <span class="status-dot"></span>
        <span>{statusLabel}</span>
      </div>
    </div>
    <div class="console-status">
      <div class="mini-stat">
        <span>Sessions</span>
        <strong>{sessionCount}</strong>
      </div>
      <div class="mini-stat">
        <span>Sel</span>
        <strong>{selectionCount}</strong>
      </div>
    </div>
  </section>

  <section class="route-strip">
    <div class="route-block">
      <div class="route-kv">
        <span class="route-label">File</span>
        <span class="route-name" title={fileName}>{fileName}</span>
      </div>
      <div class="route-kv">
        <span class="route-label">Page</span>
        <span class="route-page" title={pageName}>{pageName}</span>
      </div>
    </div>
    <div class="route-block route-block-right">
      <div class="route-kv">
        <span class="route-label">Default route</span>
        <span class="route-default" title={defaultRouteLabel}>{defaultRouteLabel}</span>
      </div>
      <div class="route-kv">
        <span class="route-label">Active session</span>
        <span class="route-active" title={activeSessionMeta.fileName}>
          {activeSessionMeta.fileName} · {activeSessionMeta.pageName} · {activeSessionMeta.selectionCount ?? 0} sel
        </span>
      </div>
    </div>
  </section>

  <section class="action-strip">
    <button class="ghost-btn" on:click={requestSessionCatalog}>Refresh</button>
    <button class="ghost-btn" on:click={() => (showSessions = true)}>
      Sessions
      <span class="inline-badge">{sessionCount}</span>
    </button>
    <button class="ghost-btn" on:click={() => (showLogs = true)}>
      Logs
      {#if timelineEvents.length + debugEvents.length > 0}
        <span class="inline-badge">{timelineEvents.length + debugEvents.length}</span>
      {/if}
    </button>
    <button class="ghost-btn" on:click={() => (showSettings = true)}>Endpoint</button>
    <button class="primary-btn" on:click={openAdmin}>Open admin</button>
  </section>

  <section class="activity-shell">
    <div class="activity-shell-head">
      <span>Tool Activity</span>
      <span class="panel-badge">{activeRequests.length}</span>
    </div>
    <ActivityPanel {activeRequests} showHeader={false} compact={true} />
  </section>

  {#if showSessions}
    <div class="overlay" role="button" tabindex="0" on:click={() => (showSessions = false)} on:keydown={(event) => {
      if (event.key === "Escape" || event.key === "Enter" || event.key === " ") {
        event.preventDefault();
        showSessions = false;
      }
    }}>
      <div class="modal-sheet modal-sheet-wide" role="dialog" aria-modal="true" aria-label="Runtime sessions" tabindex="-1" on:click|stopPropagation on:keydown|stopPropagation>
        <div class="modal-head">
          <div>
            <div class="eyebrow">Route control</div>
            <h2>Sessions</h2>
          </div>
          <div class="modal-actions">
            <button class="ghost-btn" on:click={requestSessionCatalog}>Refresh</button>
            <button class="primary-btn" on:click={() => (showSessions = false)}>Close</button>
          </div>
        </div>
        <div class="modal-body">
          <SessionPanel
            sessions={knownSessions}
            activeSessionId={currentSessionId}
            onSwitch={switchSession}
            onSwitchAndTest={switchSessionAndTest}
          />
        </div>
      </div>
    </div>
  {/if}

  {#if showSettings}
    <div class="overlay" role="button" tabindex="0" on:click={() => (showSettings = false)} on:keydown={(event) => {
      if (event.key === "Escape" || event.key === "Enter" || event.key === " ") {
        event.preventDefault();
        showSettings = false;
      }
    }}>
      <div class="modal-sheet" role="dialog" aria-modal="true" aria-label="Endpoint settings" tabindex="-1" on:click|stopPropagation on:keydown|stopPropagation>
        <div class="modal-head">
          <div>
            <div class="eyebrow">Bridge endpoint</div>
            <h2>Endpoint</h2>
          </div>
          <div class="modal-actions">
            <button class="ghost-btn" on:click={copyAdminURL}>Copy admin URL</button>
            <button class="primary-btn" on:click={() => (showSettings = false)}>Close</button>
          </div>
        </div>
        <div class="modal-body">
          <div class="endpoint-summary">
            <div class="endpoint-line">
              <span class="info-label">Admin</span>
              <span class="info-value" title={adminURL}>{adminURL}</span>
            </div>
            <div class="endpoint-line">
              <span class="info-label">Socket</span>
              <span class="info-value">{serverHost}:{serverPort}</span>
            </div>
          </div>
          <div class="settings-strip">
            <label class="field">
              <span>Host</span>
              <input bind:value={editHost} placeholder="127.0.0.1" on:keydown={handleKeydown} />
            </label>
            <label class="field">
              <span>Port</span>
              <input bind:value={editPort} placeholder="1802" on:keydown={handleKeydown} />
            </label>
            <div class="settings-actions">
              <button class="primary-btn" on:click={applySettings}>Apply</button>
              <button class="ghost-btn" on:click={openAdmin}>Open admin</button>
            </div>
          </div>
        </div>
      </div>
    </div>
  {/if}

  {#if showLogs}
    <div class="overlay" role="button" tabindex="0" on:click={() => (showLogs = false)} on:keydown={(event) => {
      if (event.key === "Escape" || event.key === "Enter" || event.key === " ") {
        event.preventDefault();
        showLogs = false;
      }
    }}>
      <div class="modal-sheet modal-sheet-wide" role="dialog" aria-modal="true" aria-label="Runtime logs" tabindex="-1" on:click|stopPropagation on:keydown|stopPropagation>
        <div class="modal-head">
          <div>
            <div class="eyebrow">Runtime diagnostics</div>
            <h2>Logs</h2>
          </div>
          <div class="modal-actions">
            <button class="ghost-btn" on:click={() => (showTimeline = !showTimeline)}>
              {showTimeline ? "Hide timeline" : "Show timeline"}
            </button>
            <button class="ghost-btn" on:click={() => (showDebug = !showDebug)}>
              {showDebug ? "Hide debug" : "Show debug"}
            </button>
            <button class="primary-btn" on:click={() => (showLogs = false)}>Close</button>
          </div>
        </div>
        <div class="modal-body logs-body">
          <TimelinePanel {timelineEvents} {showTimeline} onToggle={() => (showTimeline = !showTimeline)} />
          <DebugPanel {debugEvents} {showDebug} onToggle={() => (showDebug = !showDebug)} />
        </div>
      </div>
    </div>
  {/if}
</div>

<style>
  :global(*) {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }

  :global(body) {
    font-family: "Plus Jakarta Sans", Inter, "Segoe UI", system-ui, sans-serif;
    font-size: 12px;
    color: #13304a;
    height: 100vh;
    background: #f7f9fc;
  }

  .shell {
    height: 100%;
    display: flex;
    flex-direction: column;
    gap: 8px;
    padding: 10px 12px;
    background: #f7f9fc;
  }

  .console-bar,
  .route-strip,
  .action-strip,
  .activity-shell {
    border-radius: 16px;
    border: 1px solid #dde5f0;
    background: rgba(255, 255, 255, 0.96);
  }

  .console-bar {
    padding: 12px 14px 10px;
    display: flex;
    flex-direction: column;
    gap: 10px;
    align-items: stretch;
  }

  .console-title-row {
    display: flex;
    justify-content: space-between;
    gap: 12px;
    align-items: flex-start;
  }

  .console-title {
    min-width: 0;
  }

  .console-status {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
  }

  .mini-stat {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 7px 10px;
    border-radius: 999px;
    background: #eef4ff;
    color: #274690;
    font-size: 11px;
    line-height: 1;
  }

  .mini-stat strong {
    color: #0f172a;
    font-size: 12px;
  }

  .inline-badge {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-width: 18px;
    height: 18px;
    padding: 0 5px;
    border-radius: 999px;
    background: #dce8ff;
    color: #1d4ed8;
    font-size: 10px;
    font-weight: 700;
  }

  .eyebrow {
    font-size: 10px;
    letter-spacing: 0.12em;
    text-transform: uppercase;
    font-weight: 700;
    color: #6b7b94;
  }

  h1 {
    margin-top: 4px;
    font-size: 18px;
    line-height: 1.08;
    letter-spacing: -0.03em;
    color: #0f172a;
  }

  .route-strip {
    padding: 10px 12px;
    display: grid;
    grid-template-columns: minmax(0, 1fr) minmax(0, 1fr);
    gap: 10px 14px;
  }

  .route-block {
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .route-block-right {
    align-items: flex-start;
  }

  .route-kv {
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 3px;
  }

  .route-label {
    font-size: 10px;
    font-weight: 700;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: #7b8aa3;
  }

  .route-name {
    font-size: 13px;
    font-weight: 700;
    color: #0f172a;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .route-page {
    font-size: 12px;
    color: #64748b;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .route-default,
  .route-active {
    min-width: 0;
    font-size: 12px;
    color: #0f172a;
    font-weight: 600;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }

  .action-strip {
    display: grid;
    grid-template-columns: repeat(5, minmax(0, 1fr));
    gap: 8px;
    padding: 8px;
  }

  .status-dot {
    width: 9px;
    height: 9px;
    border-radius: 999px;
    background: #86efac;
    box-shadow: 0 0 0 6px rgba(134, 239, 172, 0.14);
  }

  .status-pill.disconnected .status-dot {
    background: #fda4af;
    box-shadow: 0 0 0 6px rgba(253, 164, 175, 0.16);
  }

  .activity-shell {
    display: flex;
    flex-direction: column;
    min-height: 88px;
    max-height: 140px;
    overflow: hidden;
  }

  :global(.activity-shell .panel) {
    border: 0;
    border-radius: 0;
    background: transparent;
  }

  .activity-shell-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 10px 12px 6px;
    font-size: 12px;
    font-weight: 700;
    color: #0f172a;
    letter-spacing: 0.06em;
    text-transform: uppercase;
  }

  .panel-badge {
    min-width: 24px;
    height: 24px;
    border-radius: 999px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    font-size: 12px;
    color: #1447e6;
    background: rgba(20, 71, 230, 0.08);
  }

  .info-label {
    font-size: 10px;
    font-weight: 700;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: #7b8aa3;
  }

  .ghost-btn,
  .primary-btn {
    border: 0;
    cursor: pointer;
    transition: background 140ms ease, border-color 140ms ease;
    min-width: 0;
  }

  .ghost-btn {
    padding: 8px 10px;
    border-radius: 10px;
    background: #eff4fb;
    color: #274690;
    font-weight: 700;
  }

  .primary-btn {
    padding: 10px 12px;
    border-radius: 10px;
    background: #0f6eff;
    color: white;
    font-weight: 700;
  }

  .ghost-btn,
  .primary-btn {
    height: 36px;
  }

  .ghost-btn:hover,
  .primary-btn:hover {
    background: #e7eef8;
  }

  .primary-btn:hover {
    background: #0459d8;
  }

  .overlay {
    position: fixed;
    inset: 0;
    z-index: 20;
    background: rgba(15, 23, 42, 0.22);
    display: flex;
    align-items: flex-end;
    justify-content: center;
    padding: 12px;
  }

  .modal-sheet {
    width: min(100%, 520px);
    max-height: min(78vh, 760px);
    border-radius: 18px;
    border: 1px solid #d7e0ec;
    background: #ffffff;
    display: flex;
    flex-direction: column;
    overflow: hidden;
    box-shadow: 0 20px 48px rgba(15, 23, 42, 0.18);
  }

  .modal-sheet-wide {
    width: min(100%, 720px);
  }

  .modal-head {
    display: flex;
    justify-content: space-between;
    gap: 12px;
    align-items: flex-start;
    padding: 14px 16px 10px;
    border-bottom: 1px solid #e8eef6;
  }

  h2 {
    margin-top: 3px;
    font-size: 18px;
    color: #0f172a;
  }

  .modal-actions {
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
    justify-content: flex-end;
  }

  .modal-body {
    min-height: 0;
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 12px;
    overflow: auto;
    background: #f8fafc;
  }

  .logs-body {
    gap: 10px;
  }

  .endpoint-summary {
    display: flex;
    flex-direction: column;
    gap: 6px;
    padding-bottom: 10px;
  }

  .endpoint-line {
    display: flex;
    align-items: center;
    gap: 8px;
  }

  .settings-strip {
    display: grid;
    grid-template-columns: 1fr 1fr auto;
    gap: 10px;
    align-items: end;
  }

  .field {
    display: flex;
    flex-direction: column;
    gap: 6px;
    min-width: 0;
  }

  .field span {
    font-size: 11px;
    font-weight: 700;
    color: #475569;
  }

  .field input {
    width: 100%;
    border: 1px solid rgba(191, 219, 254, 0.9);
    border-radius: 14px;
    padding: 11px 12px;
    font: inherit;
    color: #0f172a;
    background: rgba(248, 250, 252, 0.96);
    outline: none;
  }

  .field input:focus {
    border-color: rgba(37, 99, 235, 0.48);
    box-shadow: 0 0 0 4px rgba(37, 99, 235, 0.08);
  }

  .settings-actions {
    display: flex;
    gap: 10px;
  }

  @media (max-width: 620px) {
    .shell {
      padding: 10px;
      gap: 10px;
    }

    .console-title-row {
      flex-direction: column;
      align-items: flex-start;
    }

    h1 {
      font-size: 16px;
      max-width: none;
    }

    .console-status {
      width: 100%;
    }

    .action-strip,
    .settings-strip {
      grid-template-columns: 1fr 1fr;
    }

    .route-strip {
      grid-template-columns: 1fr;
    }

    .modal-head {
      flex-direction: column;
    }

    .modal-actions {
      width: 100%;
      justify-content: flex-start;
    }
  }

  @media (max-width: 440px) {
    .shell {
      padding: 8px;
      gap: 8px;
    }

    .console-bar,
    .route-strip,
    .action-strip,
    .activity-shell {
      border-radius: 14px;
    }

    h1 {
      font-size: 15px;
    }

    .eyebrow {
      font-size: 9px;
    }

    .info-label {
      font-size: 9px;
    }

    .route-default,
    .route-active {
      font-size: 11px;
    }

    .action-strip,
    .settings-strip {
      grid-template-columns: 1fr;
    }

    .ghost-btn,
    .primary-btn {
      width: 100%;
      justify-content: center;
      text-align: center;
    }
  }
</style>
