# Object model (UML class diagrams)

Mermaid `classDiagram` blocks; most Markdown viewers render them. Names
match `01-core-abstractions.md` and `05-design-patterns.md`. Attributes and
operations are the ones already sketched there; anything not shown is
undecided, not omitted on purpose.

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
        +allows(Layer, verb, resource) boolean
    }
    class Layer {
        +String name
        +int precedence
        +boolean writable
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
        +TeamId activeTeam
        +createdAt
        +sendUserMessage(text)
        +cancel(turnId)
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
        +Map~String,Skill~ skills
        +Map~String,ToolDefinition~ tools
        +ClassifierRule[] rules
        +ScoredMemory[] memories
        +ModelOutput modelOutput
        +Map~String,Verdict~ verdicts
        +MemoryCandidate[] candidates
        +put(accumulator, key, value, locked) boolean
    }
    class OuterChain {
        +Layer[] layers
        +runHook(HookName, TurnContext, payload) Outcome
    }
    class Layer {
        +String name
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
        +run(TurnContext, payload) Outcome
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
    class StepDecorator {
        +Step inner
    }
    class TimedStep
    class FailOpenStep
    class DryRunStep
    class LayerFactory {
        +forEntry(LayerConfig, AccessToken) Layer
        +chainFor(AccessToken) OuterChain
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
    Step <|.. GateStep
    Step <|.. PromptFragmentsStep
    Step <|.. SkillDiscoveryStep
    Step <|.. ToolDefinitionsStep
    Step <|.. MemorySearchStep
    Step <|.. RuleLoadStep
    Step <|.. StaticRulesStep
    Step <|.. RouteToTeamStep
    Step <|.. RouteToPersonalStep
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
| `on_message`          | GateStep, PromptFragmentsStep, SkillDiscoveryStep, ToolDefinitionsStep, MemorySearchStep, RuleLoadStep |
| `on_tool_call`        | StaticRulesStep per layer; ModelAuditStep appended by the service     |
| `on_memory_candidate` | RouteToTeamStep (team layers), RouteToPersonalStep (user layer)       |
| `on_turn_end`         | audit / metrics steps                                                 |

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
        +Capability[] permissions
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
        +SkillId shadows
        +boolean stale
        +boolean suggested
        +boolean pinned
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
        +resources(SkillId) ResourceReader
    }
    class SkillWriter {
        <<interface>>
        +put(Skill) SkillId
        +delete(SkillId)
        +fork(SkillId base, Layer target) SkillId
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
    class AuditedRunner
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
        +send(owner, Job)
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
    ToolSource ..> ToolAccumulator : handler puts into
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
        +SourceId source
        +createdAt
        +updatedAt
        +float[] embedding
    }
    class ScoredMemory {
        +Memory memory
        +float score
        +Layer layer
    }
    class MemoryCandidate {
        +MemoryKind kind
        +String body
        +String rationale
        +float confidence
        +Audience audience
    }
    class MemoryRouter {
        +route(MemoryCandidate, AccessToken, Session) Layer
    }
    class MemoryStore {
        <<interface>>
        +search(query, AccessToken, opts) ScoredMemory[]
        +get(MemoryId) Memory
        +list(AccessToken, filter) Memory[]
    }
    class MemoryWriter {
        <<interface>>
        +put(Memory) MemoryId
        +update(MemoryId, patch)
        +delete(MemoryId)
    }
    class MemoryMergeStep {
        +merge(TurnContext) ScoredMemory[]
    }
    class MergeStrategy {
        <<interface>>
        +merge(ScoredMemory[][] byLayer) ScoredMemory[]
    }
    class ReciprocalRankFusion
    class LayerBoostedRank
    class CrossEncoderRerank
    class MemoryWriteChain {
        +offer(MemoryCandidate, AccessToken, Session) MemoryId
        +promote(MemoryId, TeamId) MemoryId
    }
    class RecallPolicy {
        +select(ScoredMemory[], tokenBudget) Memory[]
    }
    class RecallStep {
        <<interface>>
        +apply(Memory[]) Memory[]
    }

    MemoryMergeStep *-- MergeStrategy
    MemoryMergeStep ..> ScoredMemory : from ctx.memories
    MergeStrategy <|.. ReciprocalRankFusion
    MergeStrategy <|.. LayerBoostedRank
    MergeStrategy <|.. CrossEncoderRerank
    MemoryWriteChain *-- MemoryRouter
    MemoryWriteChain o-- "1..*" MemoryWriter : team layers, then user
    MemoryWriteChain ..> MemoryCandidate
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
        +Predicate condition
    }
    class PromptSource {
        <<interface>>
        +fragments(AccessToken, Session) PromptFragment[]
    }
    class PromptAssembler {
        +assemble(AccessToken, Session) String
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
    PromptSource ..> PromptFragment : handler puts into ctx.prompt
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
        +audit(ToolCall, Conversation, AccessToken) Verdict
        +mine(Turn, AccessToken) MemoryCandidate[]
    }
    class ClassifierRule {
        +String id
        +Layer layer
        +Role role
        +RuleKind kind
        +Predicate match
        +boolean locked
        +String body
    }
    class ClassifierRuleSource {
        <<interface>>
        +rules(AccessToken) ClassifierRule[]
    }
    class ClassifierPolicy {
        +rulesFor(Role, TurnContext) ClassifierRule[]
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
        +observe(Turn) MemoryCandidate[]
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

    ClassifierEngine *-- ClassifierPolicy
    TurnContext ..> AuditState : per tool call
    ClassifierEngine <|.. ModelEngine
    ClassifierEngine <|.. RulesOnlyEngine
    ClassifierEngine <|.. RemoteEngine
    ModelEngine --> ModelClient : own instance, own model
    ModelEngine ..> AuditOutput : structured output
    ClassifierEngine o-- "0..1" MemoryMiner
    ClassifierEngine --> PromptAssembler : own system prompt
    ClassifierPolicy ..> ClassifierRuleSource : rules arrive via ctx.rules
    ClassifierRuleSource ..> ClassifierRule
    Step <|.. RiskEscalationStep
    Step <|.. ScopeGateStep
    ClassifierEngine ..> Verdict
    ClassifierEngine ..> MemoryCandidate
    MemoryMiner --> MemoryWriteChain : offers candidates
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
    AgentCore --> LayerFactory : chainFor(token)
    AgentCore --> OuterChain : on_message, on_tool_call, on_memory_candidate, on_turn_end
    AgentCore --> ModelClient : after on_message
    ModelClient ..> ModelRequest
    ModelClient <|.. AnthropicModelClient
    ModelClient <|.. OpenAICompatibleModelClient
    ModelClient <|.. LocalModelClient
    ModelClient <|.. FakeModelClient
    AgentCore --> PromptAssembler : merge step
    AgentCore --> MemoryMergeStep : merge step
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
| `PromptAssembler`, `MemoryMergeStep`, `ModelClient` | `TurnContext` and its accumulators |

## Viewing

The diagrams render inline in JetBrains IDEs with the Mermaid plugin, in
VS Code with a Mermaid preview extension, and on GitHub/GitLab. No
pre-rendered copies are kept; the Mermaid source in this file is the
single version.
