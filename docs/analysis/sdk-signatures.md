// vendor/mindwire/packages/sdk/src/client.ts
56 export interface MindwireOptions {
     /**
      * Default agent type for agent-scoped calls (e.g. `"claude-code"`). When omitted, the daemon
      * uses its own default. Override per call with the `agent` option, or scope a whole client
      * with {@link Mindwire.withAgent}.
      */
     agent?: string;
     /**
      * **Where the daemon runs and how the SDK reaches it.** A {@link Target} factory: {@link local}
      * (the default — an embedded loopback daemon, auto-spawned on a server runtime), {@link remote}
      * (a daemon you already run, required in the browser), or {@link import("./target/ssh.js").ssh} /
      * {@link import("./target/docker.js").docker} / {@link import("./target/oblien.js").oblien} — or any
      * object implementing `Target`. One `new Mindwire` = one target = one daemon = one environment; to
      * isolate agents, create another `Mindwire` with its own target. Omit for the zero-config default.
      */
     target?: Target;
     /**
      * Receives a step {@link EnsureEvent} while a target provisions (connect → probe → upload → launch →
      * ready, or skip). Handy to surface the seconds-long "upload the daemon + launch + health-poll" of a
      * fresh SSH/Docker/Oblien box. A throwing callback can't abort provisioning.
      */
     logger?: (e: EnsureEvent) => void;
     /** Custom fetch. Defaults to the global `fetch` (Node 18+, Bun, Deno, browsers). */
     fetch?: FetchLike;
     /** Extra headers merged into every request. */
     headers?: Record<string, string>;
     /**
      * Deadline in milliseconds for a single unary request (health, config, `turn` creation, memory,
      * …). Guards against a hung daemon or a dead tunnel blocking a call forever — on expiry the call
      * rejects with {@link import("./errors.js").TimeoutError}. Does **not** apply to the event stream,
      * which is long-lived. Defaults to 120000; pass `0` to disable.
      */
     requestTimeoutMs?: number;
   }
96 export interface AgentScoped {
     agent?: string;
   }
125 export class Mindwire {
      constructor(opts: MindwireOptions = {});
      async ensure(): Promise<void>;
      withAgent(agent: string): Mindwire;
      agentParam(scoped?: AgentScoped): { agent: string } | undefined;
      async close(): Promise<void>;
      health(): Promise<Health>;
      stats(): Promise<Stats>;
      catalog(): Promise<Catalog>;
      agent(scoped?: AgentScoped): Promise<AgentInfo>;
      software(opts: AgentScoped & { refresh?: boolean } = {}): Promise<HarnessSoftware>;
      models(scoped?: AgentScoped): Promise<ModelInfo[]>;
      commands(opts: { directory?: string; refresh?: boolean } & AgentScoped = {}): Promise<CommandCatalog>;
      doctor(scoped?: AgentScoped): Promise<DoctorReport>;
      setup(scoped?: AgentScoped): Promise<SetupStatus>;
      update(scoped?: AgentScoped): Promise<SetupStatus>;
      setupStatus(scoped?: AgentScoped): Promise<SetupStatus>;
      getConfig(scoped?: AgentScoped): Promise<Record<string, string>>;
      async setConfig(values: Record<string, string>, scoped?: AgentScoped): Promise<void>;
      chats(options: { cwd?: string; projectId?: string; refresh?: boolean } = {}): Promise<ChatSummary[]>;
      renameChat(chatId: string, title: string): Promise<ChatSummary>;
      uploadAttachmentChunk(chatId: string, id: string, chunk: AttachmentUploadChunk, signal?: AbortSignal): Promise<AttachmentUploadState>;
      async uploadAttachment(chatId: string, file: { name: string; mime?: string; data: Uint8Array }, options: { signal?: AbortSignal; onProgress?: (fraction: number) => void } = {}): Promise<Attachment>;
      deleteChat(chatId: string): Promise<DeleteResult>;
      forkChat(chatId: string, opts: ForkChatOptions = {}): Promise<ChatForkResult>;
      messages(chatId: string, opts: AgentScoped & { limit?: number; before?: string } = {}): Promise<Message[]>;
      async messagePage(chatId: string, opts: AgentScoped & { limit?: number; before?: string; maxBytes?: number; ifRevision?: string } = {}): Promise<MessagePage>;
      async latestRun(chatId: string): Promise<Run | null>;
      async turn(input: {
          /** Keep this ID when retrying an unacknowledged turn. Requires turnRequestVersion >= 1. */
          requestId?: string;
          /** Queue native follow-up input when this chat is active. Requires requestId and queuedInputVersion >= 1. */
          queueIfRunning?: boolean;
          chatId: string;
          message: string;
          cwd?: string;
          options?: TurnOptions;
          mode?: "turn" | "resolve";
          resolve?: ResolveOptions;
          gitAuth?: ProjectAuth;
        } & AgentScoped): Promise<Run>;
      async resolve(input: {
          requestId?: string;
          chatId: string;
          message: string;
          cwd?: string;
          options?: TurnOptions;
          resolve?: ResolveOptions;
          gitAuth?: ProjectAuth;
        } & AgentScoped): Promise<Run>;
      async run(id: string): Promise<Run>;
      async compact(chatId: string, opts: { instructions?: string; requestId?: string } & AgentScoped = {}): Promise<Run>;
      getNotifyConfig(): Promise<NotifyConfigStatus>;
      async setNotifyConfig(input: NotifyConfigInput): Promise<void>;
      async *notifications(opts: { signal?: AbortSignal } = {}): AsyncGenerator<Notification>;
      async *processes(opts: { agent?: string; signal?: AbortSignal } = {}): AsyncGenerator<ProcessFrame>;
    }
608 export class AuthApi {
      constructor(private readonly mw: Mindwire);
      methods(scoped?: AgentScoped): Promise<AuthMethod[]>;
      begin(method: string, scoped?: AgentScoped): Promise<AuthState>;
      step(input: Record<string, string>, scoped?: AgentScoped): Promise<AuthState>;
      poll(flowId: string, scoped?: AgentScoped): Promise<AuthState>;
      cancel(flowId: string, scoped?: AgentScoped): Promise<AuthState>;
      status(scoped?: AgentScoped): Promise<AuthStatus>;
      logout(scoped?: AgentScoped): Promise<AuthStatus>;
    }
669 export class PromptsApi {
      constructor(private readonly mw: Mindwire);
      private query(scoped: AgentScoped | undefined, extra?: Record<string, string | undefined>);
      memory(opts: AgentScoped & { dir?: string } = {}): Promise<MemoryDoc[]>;
      setMemory(input: { scope: MemoryScope; content: string }, opts: AgentScoped & { dir?: string } = {}): Promise<MemoryDoc>;
      deleteMemory(opts: AgentScoped & { scope?: MemoryScope; dir?: string } = {}): Promise<MemoryDoc>;
      list(opts: AgentScoped & { dir?: string } = {}): Promise<PromptTemplate[]>;
      get(name: string, opts: AgentScoped & { scope?: MemoryScope; dir?: string } = {}): Promise<PromptTemplate>;
      set(name: string, content: string, opts: AgentScoped & { scope?: MemoryScope; dir?: string } = {}): Promise<PromptTemplate>;
      async delete(name: string, opts: AgentScoped & { scope?: MemoryScope; dir?: string } = {}): Promise<void>;
      subagents(opts: AgentScoped & { dir?: string } = {}): Promise<Subagent[]>;
      subagent(name: string, opts: AgentScoped & { scope?: MemoryScope; dir?: string } = {}): Promise<Subagent>;
      setSubagent(name: string, content: string, opts: AgentScoped & { scope?: MemoryScope; dir?: string } = {}): Promise<Subagent>;
      async deleteSubagent(name: string, opts: AgentScoped & { scope?: MemoryScope; dir?: string } = {}): Promise<void>;
    }
805 export class McpApi {
      constructor(private readonly mw: Mindwire);
      private query(scoped: AgentScoped | undefined, extra?: Record<string, string | undefined>);
      list(opts: AgentScoped & { dir?: string } = {}): Promise<Partial<Record<MemoryScope, Record<string, MCPServer>>>>;
      get(name: string, opts: AgentScoped & { scope?: MemoryScope; dir?: string } = {}): Promise<MCPServer>;
      set(name: string, server: MCPServer, opts: AgentScoped & { scope?: MemoryScope; dir?: string } = {}): Promise<MCPServer>;
      async delete(name: string, opts: AgentScoped & { scope?: MemoryScope; dir?: string } = {}): Promise<void>;
    }
867 export class ProvidersApi {
      constructor(private readonly mw: Mindwire);
      private query(scoped: AgentScoped | undefined, extra?: Record<string, string | undefined>);
      list(opts: AgentScoped & { dir?: string } = {}): Promise<Partial<Record<MemoryScope, Record<string, CustomProvider>>>>;
      get(id: string, opts: AgentScoped & { scope?: MemoryScope; dir?: string } = {}): Promise<CustomProvider>;
      set(id: string, provider: Omit<CustomProvider, "id" | "hasKey"> & Partial<Pick<CustomProvider, "hasKey">>, opts: AgentScoped & { scope?: MemoryScope; dir?: string; apiKey?: string; secrets?: Record<string, string> } = {}): Promise<CustomProvider>;
      async delete(id: string, opts: AgentScoped & { scope?: MemoryScope; dir?: string } = {}): Promise<void>;
    }
933 export class NotifyApi {
      constructor(private readonly mw: Mindwire);
      channels(): Promise<NotifyChannel[]>;
      createChannel(input: NotifyChannelInput): Promise<NotifyChannel>;
      setChannel(id: string, input: NotifyChannelInput): Promise<NotifyChannel>;
      async deleteChannel(id: string): Promise<void>;
      testChannel(id: string): Promise<NotifyChannelTestResult>;
      rules(): Promise<NotifyRule[]>;
      createRule(input: NotifyRuleInput): Promise<NotifyRule>;
      setRule(id: string, input: NotifyRuleInput): Promise<NotifyRule>;
      async deleteRule(id: string): Promise<void>;
    }
995 type Mutable<T> = { -readonly [K in keyof T]: T[K] };

// vendor/mindwire/daemon/internal/orchestrator/orchestrator.go
60 type ResolveOptions struct {
    MaxIterations int           // cap on auto-continued iterations before StopReason "capped"
    Deadline      time.Duration // overall wall-clock budget for the whole resolve
   }
67 type Agent struct {
    Adapter agent.Adapter
    Auth    agent.AuthModule
    Runner  *runner.Runner
    Creds   *CredView
   }
75 func (a *Agent) ID() string
77 func (a *Agent) Capabilities() agent.Capabilities
92 type CredView struct {
    store  *session.Store
    prefix string
   }
97 func newCredView(store *session.Store, agentType string) *CredView
110 func sharedCred(bareKey string) bool
114 func (v *CredView) ns(key string) string
121 func (v *CredView) Get(key string) string
122 func (v *CredView) Set(key, val string) error
124 func (v *CredView) SetMany(values map[string]string) error
136 func (v *CredView) All() map[string]string
154 type Supervisor struct {
        nativeSessions *nativesessions.Service
        authSource     *agent.AuthSource
        prepareGit     GitPreparation
        store          *session.Store
        hub            *stream.Hub
        notifier       notify.Notifier
        notes          *notify.Stream   // local SSE broadcaster (in-app live notifications)
        mon            *procmon.Monitor // on-demand per-turn CPU/mem sampler (idle until a client subscribes)
        cwd            string
        turnTimeout    time.Duration // zero: native harness lifetime, independent of connected viewers
        agents map[string]*Agent // by agent type
        def    string            // default agent type when a request omits ?agent=
        mu                      sync.Mutex
        notificationPreferences notify.Preferences
        active                  map[string]string // chatId -> active turn's working directory
        activeAgents            map[string]int    // harness -> live turns, under the same lock as setup admission
        authChanging            map[string]bool
        setup                   *setup.Tracker
        toolchain               *toolchain.Manager
        cancels                 map[string]context.CancelFunc // runId -> cancel the running turn
        // User-in-loop ingress, parallel to cancels (same register-before-return + defer close). inputs
        // holds each running turn's inbound channel (present only for agents declaring an ingress
        // capability); pending records the run's respondable interactions so Respond can echo their
        // adapter correlators (Interaction.Meta) back on the Inbound without the adapter keeping state.
        inputs             map[string]chan agent.Inbound
        voices             map[string]*agent.VoiceBridge
        inputQueues        map[string]*runInputQueue
        pending            map[string]map[string]agent.Interaction // runId -> interactionId -> interaction
        answering          map[string]bool
        runClosed          map[string]chan struct{}
        interactionContext InteractionContext
        // Drain accounting: inflight counts live execRun/execResolve goroutines (each decrements in a
        // defer that runs AFTER its terminal SaveRun), and idle broadcasts when it reaches zero. Wait()
        // parks on idle so a caller can Cancel every run and then tear down the state directory without
        // racing a late persist. Guarded by mu, like the maps above.
        surfaces          *surface.Service
        serviceReplies    map[string]map[string]chan agent.InteractionResponse
        serviceEmit       map[string]agent.Emit
        inflight          int
        serviceOperations int
        updateLease       *ServiceUpdateLease
        stopping          bool
        idle              *sync.Cond
    }
205 func New(store *session.Store, hub *stream.Hub, notifier notify.Notifier, cwd, defaultAgent string, options ...Option) *Supervisor
240 func (s *Supervisor) Resolve(agentType string) (*Agent, bool)
249 func (s *Supervisor) Default() string
252 func (s *Supervisor) Notes() *notify.Stream
256 func (s *Supervisor) ProcMon() *procmon.Monitor
259 func (s *Supervisor) CWD() string
263 type StartTurnInput struct {
        RequestID      string // stable client receipt; retrying returns the same durable run
        QueueIfRunning bool   // native follow-up when busy; ordinary turn when idle
        ChatID         string
        Message        string
        CWD            string // run this turn in a specific dir (a project workdir); else the daemon default
        Options        agent.TurnOptions
        GitAuth        *gitaccess.Auth // write-only; never session or transcript state
        gitLease       *gitaccess.Lease
        reply          *session.InteractionReply // internal: atomically commit a resumed async answer with the turn
    }
278 func (s *Supervisor) StartTurn(a *Agent, req StartTurnInput) (session.Run, bool)
288 func (s *Supervisor) StartCompact(a *Agent, req StartTurnInput) (session.Run, bool)
301 func (s *Supervisor) StartResolve(a *Agent, req StartTurnInput, ro ResolveOptions) (session.Run, bool)
306 func (s *Supervisor) StartResolveChecked(a *Agent, req StartTurnInput, ro ResolveOptions) (session.Run, error)
380 func (s *Supervisor) start(a *Agent, req StartTurnInput, compact bool) (session.Run, bool)
385 func (s *Supervisor) startChecked(a *Agent, req StartTurnInput, compact bool) (session.Run, error)
556 func (s *Supervisor) Busy(chatID string) bool
563 func (s *Supervisor) activePath(path string) string
575 func (s *Supervisor) BusyPath(path string) bool
589 func (s *Supervisor) runDone()
603 func (s *Supervisor) Wait()
613 func (s *Supervisor) Cancel(runID string) bool
634 func (s *Supervisor) Respond(runID, interactionID, decision string, options []string, text string) bool
639 func (s *Supervisor) RespondInteraction(runID string, reply agent.InteractionResponse) error
643 func (s *Supervisor) RespondInteractionWithGitAuth(runID string, reply agent.InteractionResponse, gitAuth *gitaccess.Auth) error
676 func (s *Supervisor) SendInput(runID, text string) bool
691 func (s *Supervisor) Interrupt(runID string) bool
700 func (s *Supervisor) SetModel(runID, model string) bool
708 func (s *Supervisor) SetPermissionMode(runID, mode string) bool
716 func (s *Supervisor) recordPending(runID string, it agent.Interaction)
735 func (s *Supervisor) trySend(runID string, in agent.Inbound) bool
750 func metaStrings(m map[string]any) map[string]string
768 func (s *Supervisor) execRun(ctx context.Context, cancel context.CancelFunc, a *Agent, run session.Run, req StartTurnInput, inbound chan agent.Inbound, compact bool)
958 func (s *Supervisor) execResolve(ctx context.Context, cancel context.CancelFunc, a *Agent, parent session.Run, req StartTurnInput, ro ResolveOptions)
1151 func (s *Supervisor) emit(a *Agent, run session.Run, cond agent.Condition, body string)
1185 func (s *Supervisor) SetNotificationPreferences(preferences notify.Preferences)
1191 func (s *Supervisor) notificationOutcome(run session.Run, cond agent.Condition, summary string)
1204 func (s *Supervisor) watchInteractions(a *Agent, run session.Run)
1242 func (s *Supervisor) buildNote(a *Agent, cond agent.Condition, body, chatID, runID string) agent.Notification
1259 func newID() string
1266 func nowISO() string
1269 func snippet(s string) string
