# Object model (UML class diagrams)

Mermaid `classDiagram` blocks; most Markdown viewers render them. Names
match `01-core-abstractions.md`, `07-turn-pipeline.md`, and
`08-walkthrough.md`. Where this note and the walkthrough differ, the
walkthrough is authoritative and this note is kept in sync.

Conventions: `<<interface>>` is a port the core depends on; concrete
adapters are examples, not a fixed list. `*--` composition, `o--`
aggregation, `..>` dependency, `<|..` realizes, `<|--` extends.

## 1. Identity, layers, session

```mermaid
classDiagram
    class AccessToken {
        +String subject
        +Set~String~ scopes
        +String[] groups
        +String issuer
        +Instant expiresAt
        +String raw
        +allows(Layer, verb, resource) boolean
        +allowsAny(Layer) boolean
        +withTeam(TeamId) AccessToken
    }
    class Layer {
        +String name
        +int position
        +Section[] allowedSections
        +boolean mayLock
        +Map~HookName,InnerChain~ hooks
        +run(HookName, TurnContext, payload) Outcome
    }
    class TokenValidator {
        <<interface>>
        +validate(bearer) AccessToken
    }
    class LayerConfig {
        +Layer[] layers
    }
    class Session {
        +SessionId id
        +PrincipalId owner
        +AccessToken token
        +SessionState state
        +TurnState turn
        +SessionStateEnum stateValues
        +TeamId activeTeam
        +String clientKind
        +ToolDefinition[] clientTools
        +ClassifierRule[] rules
        +String instructions
        +Memory[] scratch
        +Map~SkillId,LoadedSkill~ loadedSkills
        +Map~String,PendingClientCall~ pendingClientCalls
        +JobId[] pendingJobResults
        +TurnId[] unminedTurns
        +MemoryCandidate[] unminedCandidates
        +Set~TurnId~ minedTurns
        +boolean frozen
        +createdAt
        +lastSeenAt
    }
    class SessionStateEnum {
        <<enumeration>>
        created
        active
        idle
        exhausted
        ended
    }
    class Conversation {
        +Message[] messages
        +append(Message)
        +window(n) Message[]
    }
    class EventBus {
        +publish(type, payload, level, turnId) int
    }
    class EventLog {
        +append(Event) int
        +since(seq) Event[]
        +oldestSeq() int
        +compactTurn(TurnId)
    }
    class StreamTransport {
        <<interface>>
        +send(Event)
        +close(reason)
    }
    class SSEStream
    class WSStream
    class Listener {
        <<interface>>
        +onEvent(Event)
    }
    class AuditLogListener
    class MetricsListener
    class NotifierListener
    class WebhookListener
    class DeltaCoalescer {
        +flushMs int
        +onDelta(text)
        +flush()
    }
    class ServiceBus {
        +publishToOwner(owner, type, payload)
    }
    class ApprovalStore {
        <<interface>>
        +save(ApprovalRequest)
        +get(approvalId) ApprovalRequest
        +pending(owner, kind) ApprovalRequest[]
        +answer(approvalId, Decision)
    }
    class Decision {
        +String decision
        +String remember
        +String rememberMatch
        +String note
        +PrincipalId by
        +Instant at
    }
    class Event {
        +int v
        +int seq
        +String type
        +SessionId sessionId
        +TurnId turnId
        +Level level
        +Object payload
    }

    TokenValidator ..> AccessToken : produces
    AccessToken ..> Layer : allows?
    LayerConfig "1" *-- "1..*" Layer : ordered = chain order
    Session "1" *-- "1" AccessToken
    Session "1" *-- "1" Conversation
    Session "1" *-- "1" EventBus
    Session "1" *-- "0..1" TurnContext : current turn
    EventBus ..> Event
    EventBus *-- EventLog : durable before delivery
    EventBus o-- "0..1" StreamTransport : the one open stream
    EventBus o-- "0..*" Listener : async
    StreamTransport <|.. SSEStream
    StreamTransport <|.. WSStream
    Listener <|.. AuditLogListener
    Listener <|.. MetricsListener
    Listener <|.. NotifierListener
    Listener <|.. WebhookListener
    EventBus o-- DeltaCoalescer : assistant_delta batching
    ServiceBus ..> EventBus : forwards to the owner's session
    ApprovalStore ..> Decision
    Session ..> ApprovalStore : owner-scoped, memory approvals outlive the session
```

## 2. The turn pipeline: a chain of chains

An outer chain of `Layer`s; each layer has, per hook point, an inner
chain of `Step`s. Steps read and append to the `TurnContext`
(`07-turn-pipeline.md`). `AccessToken` is the validated OAuth2 token;
there is no other identity object.

```mermaid
classDiagram
    class TurnContext {
        +AccessToken token
        +Session session
        +UserMessage inbound
        +Map~String,PromptFragment~ prompt
        +Map~String,SkillSummary~ skills
        +Map~String,ToolDefinition~ tools
        +ClassifierRule[] rules
        +ScoredMemory[] memories
        +ModelOutput modelOutput
        +Map~String,AuditState~ audit
        +MemoryCandidate[] candidates
        +Map~String,PendingApproval~ pendingApprovals
        +Map~String,ToolHandle~ runningTools
        +ToolCall[] callsThisTurn
        +RecallQuery recallQuery
        +DiscoverQuery discoverQuery
        +Memory[] recalled
        +AssembledPrompt assembled
        +boolean promptDirty
        +boolean memoriesDirty
        +Map~String,Layer[]~ shadowLog
        +Path workspace
        +TurnId turnId
        +put(accumulator, key, value, locked) boolean
        +snapshot() TurnSnapshot
    }
    class AuditState {
        +boolean decided
        +boolean shielded
        +Verdict verdict
        +ClassifierRule askedBy
        +String[] trace
    }
    class OuterChain {
        +Layer[] layers
        +runHook(HookName, TurnContext, payload) Outcome
    }
    class Layer {
        +String name
        +int position
        +Section[] allowedSections
        +boolean mayLock
        +Map~HookName,InnerChain~ hooks
        +run(HookName, TurnContext, payload) Outcome
    }
    class InnerChain {
        +Step[] steps
        +run(TurnContext, payload) Outcome
    }
    class HookName {
        <<enumeration>>
        on_message
        on_tool_call
        on_memory_candidate
        on_turn_end
    }
    class Step {
        <<interface>>
        +StepKind kind
        +run(TurnContext, payload) Outcome
    }
    class StepKind {
        <<enumeration>>
        gating
        contributing
        consuming
    }
    class ContributingStep {
        <<abstract>>
        +fetch(TurnContext, payload) Object
        +apply(TurnContext, Object)
    }
    class ConsumingStep {
        <<abstract>>
        +run(TurnContext, MemoryCandidate) Outcome
    }
    class Outcome {
        <<abstract>>
    }
    class Continue
    class Stop {
        +Action action
    }
    class Action {
        <<abstract>>
    }
    class Deny {
        +String reason
    }
    class Respond {
        +String text
    }
    class RequireApproval {
        +ApprovalRequest request
    }
    class GateStep
    class PromptFragmentsStep {
        +PromptSource source
    }
    class SkillDiscoveryStep {
        +SkillStore source
    }
    class ToolDefinitionsStep {
        +ToolSource source
    }
    class MemorySearchStep {
        +MemoryStore source
    }
    class RuleLoadStep {
        +ClassifierRuleSource source
    }
    class StaticRulesStep
    class RouteToTeamStep {
        +MemoryWriter store
    }
    class RouteToPersonalStep {
        +MemoryWriter store
    }
    class ModelAuditStep {
        +ClassifierEngine engine
    }
    class SkillAutoLoadStep
    class LoadedSkillsStep
    class MiningRetagStep
    class ScratchAcceptStep
    class SessionInstructionsStep
    class ClientToolsStep
    class SessionRulesStep
    class SessionScratchStep
    class FlagStep
    class StepDecorator {
        +Step inner
    }
    class TimedStep
    class FailOpenStep
    class FailClosedStep
    class RequiredStep
    class DryRunStep
    class LayerFactory {
        +forEntry(LayerConfig, AccessToken) Layer
        +chainFor(AccessToken, Session) OuterChain
    }
    class StepFactory {
        +register(stepName, ctor)
        +create(stepName, source, config) Step
    }
    class SourceFactory {
        +register(adapterName, ctor)
        +create(kind, adapterName, config) Source
    }

    OuterChain o-- "1..*" Layer : trust order
    Layer *-- "1..4" InnerChain : one per hook
    Layer ..> HookName
    InnerChain o-- "0..*" Step : configured order
    OuterChain ..> TurnContext
    Step ..> TurnContext : reads and appends
    Step ..> Outcome
    Outcome <|-- Continue
    Outcome <|-- Stop
    Stop --> Action
    Action <|-- Deny
    Action <|-- Respond
    Action <|-- RequireApproval
    Action <|-- Accepted
    Action <|-- Discarded
    Step <|.. ContributingStep
    Step <|.. ConsumingStep
    ConsumingStep <|-- RouteToTeamStep
    ConsumingStep <|-- RouteToPersonalStep
    ConsumingStep <|-- ScratchAcceptStep
    Step ..> StepKind
    Step <|.. SkillAutoLoadStep
    Step <|.. LoadedSkillsStep
    Step <|.. MiningRetagStep
    Step <|.. SessionInstructionsStep
    Step <|.. ClientToolsStep
    Step <|.. SessionRulesStep
    Step <|.. SessionScratchStep
    Step <|.. FlagStep
    StepDecorator <|-- FailClosedStep
    StepDecorator <|-- RequiredStep
    Step <|.. GateStep
    Step <|.. PromptFragmentsStep
    Step <|.. SkillDiscoveryStep
    Step <|.. ToolDefinitionsStep
    Step <|.. MemorySearchStep
    Step <|.. RuleLoadStep
    Step <|.. StaticRulesStep
    Step <|.. ModelAuditStep
    Step <|.. StepDecorator
    StepDecorator o-- Step : wraps
    StepDecorator <|-- TimedStep
    StepDecorator <|-- FailOpenStep
    StepDecorator <|-- DryRunStep
    LayerFactory ..> Layer : creates
    LayerFactory ..> OuterChain : builds per token
    LayerFactory --> StepFactory
    StepFactory --> SourceFactory
    StepFactory ..> Step : creates and decorates
```

Which steps run at which hook (standard set):

| Hook                  | Steps                                                                 |
|-----------------------|-----------------------------------------------------------------------|
| `on_message`          | GateStep (gating), PromptFragmentsStep, SkillDiscoveryStep, SkillAutoLoadStep, LoadedSkillsStep (implicit), ToolDefinitionsStep, MemorySearchStep, RuleLoadStep (contributing); session layer: SessionInstructionsStep, ClientToolsStep, SessionRulesStep, SessionScratchStep |
| `on_tool_call`        | RiskEscalationStep + ScopeGateStep (service, first); StaticRulesStep per layer; ModelAuditStep (service, last) |
| `on_memory_candidate` | MiningRetagStep (contributing) + RouteToTeamStep (consuming) in team layers, RouteToPersonalStep (consuming) in the user layer, ScratchAcceptStep (consuming) in the session layer; restrictive mining rules run once before this hook |
| `on_turn_end`         | FlagStep (a layer may emit `turn_flagged`); audit persistence is the service's AuditLogListener, not a step |

Precedence per accumulator on `put`:

| Accumulator     | Rule                                                     |
|-----------------|----------------------------------------------------------|
| prompt (by id)  | later step overwrites unless existing value is locked    |
| skills (by name)| same                                                     |
| tools (by name) | same; skill tools are namespaced so they only add        |
| rules           | append in layer order; each layer's step reads its own; restrictive kinds accumulate, locked allow shields |
| memories        | append; merged by `MergeStrategy` after `on_message`     |

## 3. Skills and tools

```mermaid
classDiagram
    class Skill {
        +SkillId id
        +String name
        +String description
        +String instructions
        +Layer layer
        +SourceId source
        +Capability[] capabilities
        +String[] triggers
        +String version
        +SkillRef derivedFrom
        +boolean locked
        +boolean auto
        +boolean pinned
    }
    class SkillRef {
        +SkillId id
        +Layer layer
        +String version
    }
    class SkillSummary {
        +SkillId id
        +String name
        +String description
        +String[] toolNames
        +Layer layer
        +boolean locked
        +String version
        +SkillRef derivedFrom
        +String[] triggers
        +boolean auto
        +boolean pinned
        +SkillId shadows
        +boolean stale
        +boolean suggested
        +boolean loaded
    }
    class LoadedSkill {
        +SkillId id
        +Layer layer
        +String version
        +Instant lastUsed
        +ToolDefinition[] tools
        +PromptFragment[] fragments
    }
    class ResourceRef {
        +String path
        +String mediaType
    }
    class ToolDefinition {
        +String name
        +String description
        +JSONSchema inputSchema
        +Risk risk
        +ExecuteOn executeOn
        +ToolMode mode
        +boolean locked
        +Layer layer
        +ToolOrigin origin
        +Capability[] requires
        +Capability[] granted
        +Risk effectiveRisk
    }
    class ToolOrigin {
        +OriginKind kind
        +String skillName
        +Layer skillLayer
    }
    class ToolImpl {
        <<abstract>>
    }
    class ScriptImpl {
        +String runtime
        +ResourceRef entry
    }
    class HttpImpl {
        +String url
        +String method
    }
    class BuiltinImpl {
        +String ref
    }
    class ClientImpl {
        +int timeoutS
    }
    class ClientToolDeclaration {
        +String name
        +JSONSchema inputSchema
        +Risk risk
        +Capability[] requires
        +ToolMode mode
    }
    class ClientGate {
        +RuleKind kind
        +Match match
    }
    class SkillStore {
        <<interface>>
        +discover(query, AccessToken) SkillSummary[]
        +load(SkillId) Skill
        +resource(SkillId, path) bytes
        +loadVersion(SkillId, version) Skill
        +has(name) boolean
        +summary(SkillId) SkillSummary
        +version(SkillId) String
    }
    class SkillWriter {
        <<interface>>
        +put(Skill) SkillId
        +update(SkillId, patch)
        +delete(SkillId)
    }
    class ToolSource {
        <<interface>>
        +tools(AccessToken) ToolDefinition[]
    }
    class ToolAccumulator {
        +Map~String,ToolDefinition~ tools
        +put(ToolDefinition, locked) boolean
        +resolve(name) ToolDefinition
    }
    class ToolRunner {
        <<interface>>
        +run(ToolDefinition, args) Result
    }
    class ToolRunnerFactory {
        +for(ToolImpl) ToolRunner
    }
    class RunnerDecorator {
        +ToolRunner inner
    }
    class SandboxedRunner
    class CapabilityCheckedRunner
    class TimeoutRunner
    class RateLimitedRunner
    class ConcurrencyRunner
    class AuditedRunner
    class SecretsProvider {
        <<interface>>
        +get(name) String
        +redactionPatterns() Pattern[]
    }
    class ToolDispatcher {
        +dispatch(ToolCall) Result
    }
    class Job {
        +JobId id
        +PrincipalId owner
        +JobOrigin origin
        +JobState state
        +Progress progress
        +ToolResult result
        +DeliveryMode delivery
    }
    class JobRunner {
        +adopt(Job, handle)
        +onProgress(Job, Progress)
        +onFinish(Job, ToolResult)
        +cancel(Job)
        +reattach(Job)
    }
    class JobStore {
        <<interface>>
        +save(Job)
        +load(JobId) Job
        +list(owner, state) Job[]
    }
    class Notifier {
        <<interface>>
        +send(owner, kind, payload)
    }

    Skill "1" *-- "0..*" ToolDefinition
    Skill "0..1" --> "1" SkillRef : derivedFrom
    LoadedSkill ..> Skill : pinned version, cached per session
    Skill "1" *-- "0..*" ResourceRef
    Skill "1" *-- "0..*" PromptFragment
    ToolDefinition "1" *-- "1" ToolImpl
    ToolImpl <|-- ScriptImpl
    ToolImpl <|-- HttpImpl
    ToolImpl <|-- BuiltinImpl
    ToolImpl <|-- ClientImpl
    ClientToolDeclaration ..> ToolDefinition : registered as, at session layer
    ClientGate --|> ClassifierRule : unlocked, restrictive
    ScriptImpl --> ResourceRef
    SkillStore ..> SkillSummary
    SkillStore ..> Skill
    ToolSource ..> ToolAccumulator : the layer's tools step puts into
    Skill ..> ToolAccumulator : loaded skill tools put at skill's layer
    ToolAccumulator ..> ToolDefinition
    ToolDispatcher --> ToolAccumulator : resolve
    ToolDispatcher --> ToolRunnerFactory
    ToolDispatcher ..> Job : creates for background mode
    JobRunner o-- "0..*" Job
    JobRunner --> JobStore
    JobRunner --> Notifier : owner has no session
    JobRunner ..> AgentCore : agent-initiated turn
    ToolDispatcher --> ClassifierEngine : audit
    ToolRunnerFactory ..> ToolRunner : creates + decorates
    ToolRunner <|.. RunnerDecorator
    RunnerDecorator o-- ToolRunner : wraps
    RunnerDecorator <|-- SandboxedRunner
    RunnerDecorator <|-- CapabilityCheckedRunner
    RunnerDecorator <|-- TimeoutRunner
    RunnerDecorator <|-- RateLimitedRunner
    RunnerDecorator <|-- ConcurrencyRunner
    ToolRunner ..> SecretsProvider : resolves by name at run time, redacts results with its patterns
    RunnerDecorator <|-- AuditedRunner
```

## 4. Memory

```mermaid
classDiagram
    class Memory {
        +MemoryId id
        +MemoryKind kind
        +String title
        +String body
        +String[] tags
        +MemoryId[] links
        +Layer layer
        +Audience suggestedAudience
        +MemoryId promotedFrom
        +MemoryId[] supersedes
        +MemoryId supersededBy
        +String hash
        +PrincipalId owner
        +float confidence
        +boolean pinned
        +SourceId source
        +createdAt
        +updatedAt
        +usedAt
        +float[] embedding
    }
    class ScoredMemory {
        +Memory memory
        +float score
        +int rank
        +boolean pinned
        +Layer layer
        +Layer[] alsoIn
    }
    class MemoryCandidate {
        +MemoryKind kind
        +String body
        +String rationale
        +float confidence
        +Audience audience
        +String[] tags
        +MemoryId[] links
        +MemoryId[] supersedes
        +Provenance provenance
        +Declined[] declined
        +Audience suggestedAudience
        +MemoryId promotedFrom
        +TurnId sourceTurn
        +String hash
        +String retaggedBy
        +boolean explicit
    }
    class MemoryStore {
        <<interface>>
        +search(query, AccessToken, opts) ScoredMemory[]
        +get(MemoryId, AccessToken) Memory
        +list(AccessToken, filter) Memory[]
        +touch(MemoryId, AccessToken, usedAt)
    }
    class MemoryWriter {
        <<interface>>
        +put(Memory) MemoryId
        +update(MemoryId, patch)
        +delete(MemoryId)
    }
    class MemoryMerge {
        +merge(TurnContext) ScoredMemory[]
    }
    class MergeStrategy {
        <<interface>>
        +merge(ScoredMemory[][] byLayer) ScoredMemory[]
    }
    class ReciprocalRankFusion
    class LayerBoostedRank
    class CrossEncoderRerank
    class RouteToTeamStep {
        +MemoryWriter store
        +TeamId teamId
    }
    class RouteToPersonalStep {
        +MemoryWriter store
    }
    class RecallPolicy {
        +select(ScoredMemory[], tokenBudget) Memory[]
    }
    class RecallStep {
        <<interface>>
        +apply(Memory[]) Memory[]
    }

    MemoryMerge *-- MergeStrategy
    MemoryMerge ..> ScoredMemory : from ctx.memories (a strategy the core invokes, not a step)
    MergeStrategy <|.. ReciprocalRankFusion
    MergeStrategy <|.. LayerBoostedRank
    MergeStrategy <|.. CrossEncoderRerank
    RouteToTeamStep --> MemoryWriter : accept with positive signal
    RouteToPersonalStep --> MemoryWriter : fallback
    RouteToTeamStep ..> MemoryCandidate
    RouteToPersonalStep ..> MemoryCandidate
    MemoryStore ..> ScoredMemory
    ScoredMemory --> Memory
    RecallPolicy *-- "1..*" RecallStep : pipeline
    RecallPolicy ..> ScoredMemory
```

## 5. System prompt

```mermaid
classDiagram
    class PromptFragment {
        +String id
        +Layer layer
        +SourceId source
        +int priority
        +Section section
        +String body
        +boolean locked
        +boolean optional
        +boolean shrinkable
        +Predicate condition
    }
    class PromptSource {
        <<interface>>
        +fragments(AccessToken, Session) PromptFragment[]
    }
    class PromptAssembler {
        +assemble(TurnContext, budget) AssembledPrompt
        +fragmentsUsed() PromptFragment[]
    }
    class RenderStrategy {
        <<interface>>
        +render(SectionMap fragmentsBySection) String
    }
    class PlainTextRender
    class TaggedSectionRender
    class PromptBudgetStep {
        +apply(PromptFragment[], tokenBudget) PromptFragment[]
    }

    PromptAssembler ..> PromptFragment : from ctx.prompt
    PromptSource ..> PromptFragment : the layer's prompts step puts into ctx.prompt
    Skill ..> PromptFragment : loaded skill contributes at its layer
    PromptAssembler *-- RenderStrategy
    PromptAssembler *-- "0..*" PromptBudgetStep : pipeline
    RenderStrategy <|.. PlainTextRender
    RenderStrategy <|.. TaggedSectionRender
    PromptSource ..> PromptFragment
```

## 6. Classifier

```mermaid
classDiagram
    class ClassifierEngine {
        <<interface>>
        +audit(AuditInput) AuditOutput
        +mine(turns, hints, instructions) MemoryCandidate[]
        +summarise(Session) MemoryCandidate
    }
    class ClassifierRule {
        +String id
        +Layer layer
        +Role role
        +RuleKind kind
        +Match match
        +boolean locked
        +String body
    }
    class Role {
        <<enumeration>>
        tool_audit
        memory_mining
        memory_recall
    }
    class AuditStore {
        <<interface>>
        +write(AuditRecord)
        +query(filter, AccessToken) AuditRecord[]
        +retention
    }
    class AuditRecord {
        +String id
        +Instant ts
        +PrincipalId owner
        +SessionId sessionId
        +TurnId turnId
        +String kind
        +Object ref
        +Layer layer
        +Object payload
    }
    class ClassifierRuleSource {
        <<interface>>
        +rules(AccessToken) ClassifierRule[]
    }
    class StaticRulesStep {
        +Layer layer
    }
    class ModelAuditStep {
        +ClassifierEngine engine
        +Thresholds thresholds
        +Risk alwaysAuditRiskGte
    }
    class RiskEscalationStep
    class ScopeGateStep
    class AuditState {
        +boolean decided
        +boolean shielded
        +Verdict verdict
        +ClassifierRule askedBy
    }
    class ModelEngine {
        +ModelClient client
        +audit(AuditInput) AuditOutput
        +mine(turns, hints, instructions) MemoryCandidate[]
        +summarise(Session) MemoryCandidate
    }
    class RulesOnlyEngine
    class RemoteEngine {
        +String url
    }
    class AuditOutput {
        +float pRequested
        +String effectSummary
        +String reason
        +Mismatch mismatch
    }
    class MemoryMiner {
        +mine(turns, hints, instructions) MemoryCandidate[]
    }
    class Verdict {
        +Decision decision
        +float confidence
        +String reason
        +ClassifierRule rule
    }
    class ClassifierDecorator {
        +ClassifierEngine inner
    }
    class DryRunClassifier
    class AuditedClassifier

    TurnContext ..> AuditState : per tool call
    StaticRulesStep ..> ClassifierRule : reads this layer's rules from ctx.rules
    ModelAuditStep --> ClassifierEngine
    ModelAuditStep ..> Verdict : thresholds turn AuditOutput into a Verdict
    ModelAuditStep --> AuditStore : audit.write
    AuditStore ..> AuditRecord
    ClassifierRule ..> Role
    Step <|.. StaticRulesStep
    Step <|.. ModelAuditStep
    ClassifierEngine <|.. ModelEngine
    ClassifierEngine <|.. RulesOnlyEngine
    ClassifierEngine <|.. RemoteEngine
    ModelEngine --> ModelClient : own instance, own model
    ModelEngine ..> AuditOutput : structured output
    ClassifierEngine o-- "0..1" MemoryMiner
    ClassifierEngine --> PromptAssembler : own system prompt
    ClassifierRuleSource ..> TurnContext : the layer's rules step appends to ctx.rules
    ClassifierRuleSource ..> ClassifierRule
    Step <|.. RiskEscalationStep
    Step <|.. ScopeGateStep
    ClassifierEngine ..> MemoryCandidate
    MemoryMiner ..> RouteToTeamStep : candidates walk on_memory_candidate
    ClassifierEngine <|-- ClassifierDecorator
    ClassifierDecorator o-- ClassifierEngine : wraps
    ClassifierDecorator <|-- DryRunClassifier
    ClassifierDecorator <|-- AuditedClassifier
```

## 7. Agent core, API, and clients

```mermaid
classDiagram
    class AgentCore {
        +runTurn(Session, UserMessage)
    }
    class OuterChain {
        +runHook(HookName, TurnContext, payload) Outcome
    }
    class ModelClient {
        <<interface>>
        +complete(ModelRequest) Stream~ModelEvent~
        +countTokens(ModelRequest) int
        +capabilities() Capabilities
    }
    class ModelRequest {
        +String model
        +SystemBlock[] system
        +Message[] messages
        +ToolSpec[] tools
        +int maxOutput
        +Effort effort
        +JSONSchema structured
        +Duration deadline
    }
    class AnthropicModelClient
    class OpenAICompatibleModelClient
    class LocalModelClient
    class FakeModelClient {
        +ModelEvent[] script
    }
    class ApiLayer {
        +createSession(token) Session
        +postMessage(sessionId, text)
        +streamEvents(sessionId) SSE
        +postToolResult(sessionId, result)
        +postApproval(sessionId, approvalId, decision)
    }
    class TokenValidator {
        <<interface>>
        +validate(bearer) AccessToken
    }
    class OAuth2JwtValidator
    class DevIdpValidator
    class Client {
        <<abstract>>
        +ownsSession
    }
    class CliClient
    class WebClient
    class AppClient
    class SessionStore {
        <<interface>>
        +save(Session)
        +load(SessionId) Session
    }

    ApiLayer --> AgentCore
    ApiLayer *-- TokenValidator
    ApiLayer ..> AccessToken : passes down
    ApiLayer o-- "0..*" Session
    ApiLayer o-- SessionStore
    TokenValidator <|.. OAuth2JwtValidator
    TokenValidator <|.. DevIdpValidator
    AgentCore --> LayerFactory : chainFor(token, session)
    AgentCore --> OuterChain : on_message, on_tool_call, on_memory_candidate, on_turn_end
    AgentCore --> ModelClient : after on_message
    ModelClient ..> ModelRequest
    ModelClient <|.. AnthropicModelClient
    ModelClient <|.. OpenAICompatibleModelClient
    ModelClient <|.. LocalModelClient
    ModelClient <|.. FakeModelClient
    AgentCore --> PromptAssembler : merge step
    AgentCore --> MemoryMerge : merge strategy
    AgentCore --> RecallPolicy : merge step
    AgentCore --> ToolDispatcher
    OuterChain ..> ModelAuditStep : appended after last layer
    Client "1" --> "1" Session : owns
    Client <|-- CliClient
    Client <|-- WebClient
    Client <|-- AppClient
    Client --> ApiLayer : HTTP + SSE
```

## Shared vs. per-session, at a glance

| Shared (one per service)                          | Per session / per turn               |
|---------------------------------------------------|--------------------------------------|
| all `Source`s, `Layer`s and their `Step`s (cached per layer/team) | `Session`, `Conversation`, `EventBus` |
| `ClassifierEngine`, `ModelAuditStep`              | `AccessToken` (validated per request)|
| `LayerFactory`, `StepFactory`, `SourceFactory`, `ToolRunnerFactory` | `OuterChain` (built per token) |
| `PromptAssembler`, `MemoryMerge`, `ModelClient`, `AuditStore`, `SecretsProvider` | `TurnContext` and its accumulators |

## Viewing

The diagrams render inline in JetBrains IDEs with the Mermaid plugin, in
VS Code with a Mermaid preview extension, and on GitHub/GitLab. No
pre-rendered copies are kept; the Mermaid source in this file is the
single version.
