# Trading Connectivity / FIX Protocol

The financial trading ecosystem uses a variety of connectivity methods.  FIX remains the **dominant standard** for trade messages, especially for low-latency exchange and broker links.  However, many venues and brokers also offer **proprietary interfaces**.  For example, exchanges often provide native binary protocols (Nasdaq’s OUCH/ITCH, CME’s iLink, etc.) for speed, or new “API” gateways (e.g. CME’s REST-based STP platform).  Some brokers and crypto venues offer **REST or WebSocket APIs** for order entry and market data, trading off simplicity for performance.  There are also **RFQ protocols** for negotiated markets: in RFQ trading a buyer requests quotes from multiple sellers before hitting the market.  Trade and order feeds can be duplicated via **drop-copy** channels (see below) for compliance.  Note that *market data* is usually delivered via separate channels (multicast feeds or web streaming), whereas *order entry* uses FIX or equivalent.  Overall, many non-FIX protocols exist because exchanges/brokers optimize for latency, specialize features, or support asset classes that FIX alone cannot easily handle.

- **FIX-based connectivity.** Most broker-dealers and exchanges support FIX (any of v4.x up to FIXT1.1) for orders and trades. FIX provides a unified, open messaging format that reduces the need for custom links.
- **Proprietary broker APIs.** Some brokers (especially in FX, derivatives, or crypto) have their own APIs or bridges.  Firms often supplement FIX with modern REST/JSON or FIX-over-WebSocket endpoints for ease of integration.
- **Exchange-native protocols.** Many markets expose native low-level protocols: e.g. equity markets may use OUCH (binary order entry) and ITCH (market data), CME has iLink/FAST, Eurex uses OUCH or SCTP gateways, etc. These are chosen to minimize latency and support special order types.
- **REST/WebSocket APIs.** Modern venues (especially digital-asset exchanges) often offer REST for order placement and WebSocket for streaming data.  For instance, Coinbase provides REST/WebSocket now plus a new FIX API for legacy systems.
- **RFQ protocols.** In markets like bonds or FX OTC, Request-for-Quote systems let a participant solicit quotes from dealers. RFQ is a formal trading protocol (negotiation process) that many platforms are moving toward.
- **Binary/low-latency feeds.** Ultra-low-latency traders may use specialized binary channels (UDP or kernel-bypass networks) directly to exchange matching engines.  These can bypass general TCP/FIX overhead.
- **Drop-copy connectivity.** Brokers often send copies of all order and trade messages to risk/control systems via a separate FIX session, known as drop-copy.
- **Market-data vs order feeds.** Market data (quotes/trades) is generally delivered through dedicated market-data feeds (multicast/IP), whereas order entry and executions are handled through FIX-like protocols.
- **Why proprietary persists.** When ultra-low latency, new order types, or extra fields are needed, firms build custom protocols or extend FIX (see “Adapter Engineering”).  Exchanges may only certify their own format. In practice, an OMS/EMS supports both FIX and many proprietary connections, hiding this complexity behind a unified interface.

## FIX Protocol Fundamentals

The FIX protocol is an **application-layer messaging standard** for trading. A FIX *message* is a series of `tag=value` fields separated by delimiters. Each field has a numeric **tag** (e.g. 11=ClOrdID, 55=Symbol) and a value; related fields are grouped into **components** or repeating groups. Messages have a header (including fields like BeginString, BodyLength, MsgType, SenderCompID, TargetCompID, MsgSeqNum) and a trailer (Checksum).  
FIX versions (FIX 4.0, 4.2, 4.4, 5.0 SP2, FIXT1.1, etc.) define standard fields and message types; extensions or custom tags (usually in high numeric ranges) are used for non-standard data.  

Different **MsgType** values identify message purposes. For example, NewOrderSingle (35=D) is used to place orders, OrderCancelReplaceRequest (35=G) modifies orders, OrderCancelRequest (35=F) cancels orders, and ExecutionReport (35=8) reports fills or cancellations.  A CancelReject (35=9) indicates a failed cancel/replace.  Each ExecutionReport carries fields like `OrdStatus(39)` and `ExecType(150)` to show order state (New, PartiallyFilled, Filled, Canceled, Rejected, etc.). Other FIX message categories include Market Data (e.g. MarketDataRequest, MarketDataSnapshot), Trade Capture Reports, and Post-Trade messages.  In short, FIX provides a rich set of application-level message types so that “each action [is] a FIX message” – e.g. new order, cancel, replace, cancel-reject, etc..  Firms often extend FIX with *custom fields/tags* for proprietary features; these are negotiated and documented in each counterparty’s interface spec.  

- **Tags & fields:** Every field is defined by a numeric tag ID. FIX is tag-oriented (e.g. 54=Side, 38=OrderQty). Components (reusable groups of fields) and *repeating groups* (e.g. leg details) allow arrays of fields. All parties must agree on message structure (via a FIX dictionary or spec).
- **Message types:** FIX messages are categorized (order, market data, trade capture, etc).  Important examples: 35=D (NewOrderSingle), 35=8 (ExecutionReport), 35=G (OrderCancelReplace), 35=F (OrderCancel), 35=3 (Reject) and 35=AE (TradeCaptureReport).  Market-data messages (35=W Snapshot, 35=X Incremental Refresh) are separate category.
- **Versions and extensions:** FIX 4.2/4.4 are common for equities/FX; FIXT1.1 (based on 5.0) separates session from application, allowing mixed application versions in one session.  Over time, FIX has evolved via **extension packs** for backward compatibility. Each version defines a “FIX dictionary” of fields and message types, but brokers/vendors may maintain dialects (see Adapter Engineering).
- **Custom tags:** Firms can define extra tags (usually tag numbers above 5000) for proprietary use.  These appear in the FIX spec under “User Defined.” An adapter must translate or ignore unknown tags. 
- **Example structure:** A new order message might look like `8=FIX.4.4|9=...|35=D|49=CLIENT|56=BROKER|11=12345|55=IBM|54=1|38=100|40=2|44=150.50|59=0|10=...|` (BeginString, BodyLength, MsgType, SenderCompID, TargetCompID, ClOrdID, Symbol, Side, OrderQty, OrdType, Price, TimeInForce, CheckSum).

## FIX Session Layer

FIX’s session layer ensures ordered delivery and recovery.  A **FIX session** is a bi-directional stream of sequential messages (each with a MsgSeqNum) between two parties.  It typically begins with a Logon (MsgType=A) exchange: the **Initiator** sends Logon to authenticate/establish parameters, the **Acceptor** replies with its own Logon.  The Logon header can include a `ResetSeqNumFlag (141)` to reset sequence numbers to 1 (used to start a new session, often daily).  After successful login, the session is **ESTABLISHED** and normal message exchange begins. During the session each side sends periodic **Heartbeats (35=0)** – “I’m alive” messages – to maintain the connection. If no message arrives within the heartbeat interval, a **TestRequest (35=1)** may be sent to provoke a heartbeat.

Both parties track an **incoming** and **outgoing MsgSeqNum** (each starts at 1).  Every message carries MsgSeqNum (34). When a message arrives, the receiver checks that its sequence number matches the expected count.  If the incoming SeqNum is higher than expected (i.e. one or more messages were missed), the receiver sends a **ResendRequest (35=2)** for the missing range.  The sender then retransmits the missing messages **with PossDupFlag=Y** (indicating “possible duplicate”) using their original sequence numbers.  If the gap is large, the sender may instead send a **SequenceReset (35=4) – GapFill** message to skip ahead (also marked PossDup=Y).  In an extreme case (e.g. a disaster restart), a SequenceReset-Reset (35=4 with Reset mode) can force both sides back in sync. Importantly, administrative messages like Logon, Logout, Heartbeat, TestRequest, ResendRequest, and previous SeqResets are **not** normally replayed during recovery; they are gap-filled instead. 

If a message arrives with a **lower-than-expected SeqNum** (and not marked as PossDup), it indicates a serious protocol error.  In such cases the session should be terminated (typically by sending a Logout with explanation) and manual intervention is needed.  At any time, either side can send **Logout (35=5)** to end the session cleanly.   

When the network drops, a reconnect can occur and the session will **recover**.  The FIX engine often moves through states (CONNECTED → LOGON_SENT → ESTABLISHED → HEARTBEATING; on disconnect it may go to DISCONNECTED then RECOVERING, sending ResendRequests, until it returns to ESTABLISHED).  Notably, a single FIX session can span multiple TCP connections.  For example, upon reconnect the initiator re-sends Logon (with NewSeqNo/Reset flag if needed) and gap-fill exchange resumes.  Some platforms use “reset-on-logon” (reset sequences daily) while others maintain persistent sequence numbers across days; these policies are negotiated at logon.  In summary, the FIX session layer (logon/logout, heartbeats, MsgSeqNum, gap fill, possDup) provides reliable, sequenced messaging with recovery in case of gaps.

- **Logon/Logout:** Start a session (35=A) and end it (35=5).  Logon may request sequence reset.  Both sides agree on credentials and heartbeat interval.
- **Heartbeats/TestRequest:** Sent periodically to verify connectivity.  A missing heartbeat triggers a TestRequest.
- **MsgSeqNum:** Each message has a sequence number (tag 34).  Both parties track incoming/outgoing counters. 
- **ResendRequest (35=2):** If an incoming message has a gap (incoming Seq > expected +1), send ResendRequest for the missing range. 
- **SequenceReset (35=4):** Used in gap recovery. GapFill resets a range (skips retransmission); Reset mode forces re-synchronization.
- **PossDupFlag (43):** Marked “Y” on retransmitted messages.  On receipt, duplicates are used only to advance sequence numbers, not re-processed.
- **ResetSeqNumFlag (141):** Setting this on Logon resets both sides’ sequence counters to 1, typically at start of trading day.
- **Gap detection:** If incoming Seq < expected (not duplicate), this is a fatal error (session logout recommended).
- **Session recovery:** After disconnect, reconnects resume using last sequence numbers.  The Accepting side’s Logon can include a NextExpectedMsgSeqNum to request any missed post-login messages.
- **Administrative vs application:** Session-level messages (Logon, Heartbeat, ResendRequest, etc.) follow these protocol rules.  Business messages (NewOrderSingle, ExecReport, etc.) are application-level and have their own error fields (e.g. business-level Rejects).

## Order/Execution Lifecycle over FIX

In a trade workflow, client software (OMS/EMS) issues orders via FIX and handles the resulting messages to update order state.  A new order is sent with **NewOrderSingle (35=D)**.  The broker or exchange first acknowledges receipt (via an ExecutionReport with `ExecType=0/New`), or immediately rejects if invalid (ExecType=8/Rejected, or a business-level OrderCancelReject if responding to a cancellation).  Once accepted, any fills are reported with ExecutionReports (`ExecType=1/PartialFill` or `2/Fill`).  The final fill (or a complete cancel) generates a terminal ExecutionReport (`ExecType=2` or `4/Cancelled`).  For example, the OMS sees a sequence of ExecReports updating the order status from “New” to “PartiallyFilled” to “Filled” (or “Cancelled”).  

If the client wishes to change the order, it sends **OrderCancelReplaceRequest (35=G)** with a new ClOrdID and modified parameters; the exchange replies with an ExecReport reflecting the updated order or a CancelReject (35=9) if the change is refused.  To cancel an order, the client sends **OrderCancelRequest (35=F)**; success is indicated by an ExecReport with `ExecType=4/Canceled`, or an OrderCancelReject if already filled or invalid.  In each ExecReport, the tags OrdStatus(39) and ExecType(150) indicate the status (e.g. New, PartiallyFilled, Filled, Canceled, Rejected).  

Other messages include **TradeCaptureReport (35=AE)** for off-exchange (“give-up” or matched) trades, and **BusinessMessageReject (35=j)** or session-level Rejects (35=3) for errors.  Drops copies of fills can be sent on a separate FIX session (drop-copy) to risk systems.  In a modern trading system, the OMS maps these FIX messages to its internal order-state machine.  Each FIX message carries a *ClOrdID* (client order ID) and an *OrderID* (broker order ID) that the OMS uses to correlate events to its own records.  Idempotency is handled by not re-sending messages when possible, and duplicate ClOrdIDs are managed.  

In summary, a typical lifecycle over FIX is: **NewOrderSingle → (Reject) or ExecReport(New)** → (zero or more) **ExecReport(PartialFill)** → **ExecReport(Fill)** or **OrderCancelRequest → ExecReport(Canceled)** (or Rejects along the way).  As one author notes, “there’s a FIX message for every action,” so understanding these core types (New Order, Cancel, ExecReport) is key.  For audit/compliance, the same flow of ExecutionReport messages is often sent to a drop-copy port.

- *NewOrderSingle (35=D):* Submit a new order (specify ClOrdID, Symbol, Side, Quantity, Price, etc.).
- *ExecutionReport (35=8):* Sent by market; ExecType/OrdStatus indicate New (accepted), PartialFill, Fill, Cancelled, Rejected, etc.
- *OrderCancelReplace (35=G):* Modify existing order (new ClOrdID); success yields updated ExecReports, failure a CancelReject.
- *OrderCancelRequest (35=F):* Request to cancel; a final ExecReport(Canceled) confirms cancel or a CancelReject if too late.
- *CancelReject (35=9):* Response when a cancel/replace cannot be honored (e.g. already filled).
- *Reject (35=3 or j):* Session- or business-level reject of a message (e.g. bad format or business rule).
- *Partial/Full Fills:* Multiple ExecReports may be sent until the order is fully filled.
- *TradeCaptureReport (35=AE):* Reporting of trades, often used by clearing venues or to confirm off-exchange trades.
- *Drop-copy:* Another session (with protocol mirroring) that receives the same ExecReport messages for order-tracking.
- *Internal mapping:* The OMS/EMS uses the FIX fields (ClOrdID, OrderID, ExecID, etc.) and timestamps to correlate and update internal order state.

## Internal Normalized Connectivity Model

An OMS/EMS typically implements a **canonical trading model** internally.  All external adapters translate to this unified schema. For example, internally an order might be represented by fields (Symbol, Side, Quantity, Price, TIF, etc.) in a standard format.  Similarly, executions and fills are normalized into an **execution model** (FilledQty, CumQty, LastQty, LastPrice, Status, etc.). Adapters convert between this canonical model and each counterparty’s API.  

Key principles include **event normalization** and correlation.  When a FIX ExecReport or JSON fill arrives, the adapter emits a uniform internal *OrderUpdate* or *FillEvent*. Each order is tracked by a unique internal ID; the OMS assigns a **ClOrdID** (or uses an internal reference) and maps it to each venue’s identifiers. This ensures that all events (acceptances, fills, cancels) can be tied back to the original order request.  **Idempotency** is enforced by the OMS (e.g. by not re-sending a cancel twice) and by de-duplicating any repeated messages based on ExecID or ClOrdID. 

The system also handles **sequencing and timing**: it stamps events with authoritative exchange timestamps and preserves message order for each session. Errors from different sources are mapped to a common error model (e.g. “OrderRejected”, “CancelDenied”).  Often a **capability model** is maintained: the OMS knows which destinations support which order types, TIFs, securities, etc., and can route or adjust orders accordingly. In effect, FIX and other connectivity become adapters feeding a single, normalized trading API inside the OMS.

## Non-FIX Adapters

Beyond FIX adapters, trading systems include adapters for any specialized protocol: 

- **Broker proprietary APIs:** For example, EBS’s older API for FX was replaced by CME’s STP API (FIX and REST).  Many brokers offer single-dealer platforms (SDPs) with their own messages.
- **Exchange protocols:** Exchanges may require native APIs.  For instance, CME’s new *STP** API (certificate-based FIX/REST) is used for EBS FX and other services, and LSEG has native OUCH order ports.
- **RFQ systems:** Adaptors handle RFQ messages, which may be built over FIX or proprietary network messages (e.g. Bloomberg’s credit RFQ engines).
- **WebSocket/REST APIs:** Crypto venues (Binance, Coinbase, Kraken, etc.) and some FX platforms offer WebSocket for market data and REST for orders.  These adapters accept JSON streams and translate them into order events.  As noted by Coinbase, they support WebSocket/REST for all users but added a FIX API for legacy tools.
- **Binary feeds:** Market data feeds like ITCH/OUCH are parsed by special handlers.  For example, an adapter reads a UDP multicast and emits market events; similarly, some order-entry protocols (e.g. LME’s Pro@Market) have their own binary formats.
- **Protocol translation:** The OMS acts as a translator. For instance, a REST order placement is transformed into an internal NewOrder event, and incoming JSON trade reports become internal fills.  Field/enum translation is done here (e.g. mapping WebSocket JSON fields to FIX tag equivalents).

In each case, the adapter encapsulates the venue’s specific handshake, sequencing, and field conventions, converting to the internal canonical model.  The goal is that the internal engine treats all markets uniformly, with FIX just being *one* of many adapter protocols.

## Adapter Engineering

Building reliable adapters requires accounting for **dialects and quirks**. Each FIX counterpart may use slightly different versions or custom fields. For example, one broker might use FIX 4.2 while another uses FIXT1.1, or they may define non-standard enum values.  Adapters thus include **configurable dictionaries**: they translate tag names/types, handle custom tags, and map enum values (e.g. some brokers use non-standard algorithm or order-type codes).   Often, a broker adds proprietary tags (e.g. Tag 8000+) for extra instructions; the adapter must insert/read these for that broker only. 

**Session differences:** Some brokers require special Logon parameters or heartbeat rules. The adapter’s session logic often needs to support these idiosyncrasies (e.g. setting particular Logon fields, handling mandatory heartbeats).  Field/enum translation: for example, FIX uses Side=1/2 for buy/sell, but a custom API might use “B”/“S” strings; the adapter must convert. 

**Broker-specific behavior:** Some venues exhibit odd cases (e.g. sending an ExecutionReport with ExecType=OrderCanceled even if no cancel was requested). Adapters often contain rules or fixes (or configuration options) to handle these. 

**Capability negotiation:** Adapters may use the FIX Logon’s <MsgTypeGrp> to advertise supported message versions, or simply document capabilities. The OMS might restrict certain order types if a venue doesn’t support them. 

**Configuration vs code:** Good adapter design keeps behavior configurable (dialect files or settings) rather than hard-coded. Tools like QuickFIX or commercial engines often use “dialect files” to define each partner’s messages. 

**Certification and testing:** Every adapter must be thoroughly tested. Typically, each broker/exchange provides a certification environment and a test harness. Many trading firms maintain simulators (“FIX simulators” or mock servers) to emulate counterparty behavior for offline testing. 

**Version management:** Protocol versions evolve (e.g. FIX 5.0 SP2 added extension packs). Adapters must manage versions per session (using ApplVerID, etc.) and be updated when counterparties upgrade.  Regression testing is critical whenever a new FIX release is adopted by a venue. 

In short, adapter engineering is meticulous: one mis-mapped tag or unexpected field can break the workflow. Mature systems isolate all these differences so the core engine remains clean. 

## Connectivity Infrastructure

Behind the adapters is an infrastructure ensuring **robustness and continuity**. Key elements include:

- **Session management:** The FIX engine (or connectivity layer) maintains session state and heartbeats.  It logs all messages for recovery.  Persistent stores (journals) record outgoing and incoming FIX streams so that after a restart the system can replay or resend messages as needed.
- **Persistence/journaling:** Every FIX message (and other API message) is typically journaled to disk or database in real time. This allows the system to reconstruct in-flight orders after a crash, and to replay history for audit.
- **Message replay:** Some FIX engines support automatic replay: upon restart, they can resend unsent messages (from journal) or re-request from counterparty. If a failure occurs mid-session, the system uses the saved sequence state to either send a ResendRequest or replay from its log.
- **Failover/HA:** High availability is achieved via redundant hosts (active/standby). Both machines share a storage of journals and sequence numbers. If the active node dies, the standby takes over with the same session credentials and sequence state, so trading continues with minimal interruption. 
- **Connection recovery:** Beyond session logic, the network stack monitors TCP connections.  If a link is lost, the engine immediately attempts reconnection (often with exponential backoff). Many trading platforms use multiple FIX gateways or redundant paths (e.g. two IB routers).
- **Monitoring:** All connections and threads are monitored. Metrics include message rates, heartbeat latencies, queue depths. Alerting is set up on missing heartbeats, high queue lag, or abnormal disconnects.  Dashboards typically show session status (Up/Down) and sequence number differences.
- **Operational controls:** Admin features include **kill switches** and circuit breakers. An operator can pause or close sessions if something goes wrong (e.g. a flood of rejects or risk violation). Automated checks may throttle order flow if risk limits are exceeded.
- **Audit:** Complete logging of message traffic (sent/received FIX and events) is kept for compliance.  Tools or dashboards may allow replaying a session for post-mortem. Many systems also log all adapter errors and acknowledgements.
- **Network topology:** Often colocated or connected via low-latency links to exchanges. IP multicasting may be used for market data. Firewalls and routers are configured to minimize latency. 

This infrastructure is critical. For example, if a fixengine crashes, its journaling allows reconnect without losing orders. If a market goes down, the OMS can automatically reroute or halt orders.  Proper connectivity design ensures that the dozens of heterogeneous sessions operate as one reliable service from the OMS/EMS standpoint.

## Buy vs Build Connectivity

Firms must choose between building in-house connectivity or using third-party services:

- **Vendor/Connectivity Networks:** Some companies offer managed connectivity (“connectivity-as-a-service”) to multiple venues. E.g. a vendor may provide a single API that internally connects to many exchanges and brokers. This reduces the certification burden on the trading firm, at the cost of subscription fees and some loss of control. It can accelerate rollout, but may add latency or vendor lock-in.
- **Direct broker/venue links:** Connecting directly to each broker or exchange maximizes control and often gives the lowest latency. However, it requires significant development effort: each connection must be implemented, maintained, and certified separately. The firm must handle version upgrades and troubleshooting on its own.
- **Managed services vs DIY:** The economics depend on scale. Large banks often build their own infrastructure (they can amortize cost and need ultra-low latency). Smaller firms might use broker gateways or cloud-based gateways (e.g. FIX gateways offered by execution platform providers).
- **Certification burden:** Direct connectivity means going through each venue’s test and certification process (usually lengthy). Vendors may have already certified, so buying a solution can skip some effort. However, one still typically has to do internal validation and testing.
- **Latency and control:** Building often yields fastest order acknowledgement (removing extra hops). But buying can add extra network hops or unified routing logic. There is also the question of **vendor lock-in**: using a closed platform may limit future flexibility.
- **Hybrid approaches:** Some firms mix approaches (use vendor for smaller markets, and direct connect to critical venues). They also consider connectivity “aaS” options, or cloud-based FIX routers with global presence.
- **Audit and governance:** In build mode, the firm fully owns security, keys, and data. With vendors, one must trust their security and may need audits of the vendor.

In summary, while FIX is a standard, the surrounding connectivity strategy must balance cost, speed, risk, and maintainability. The trend is often to treat **FIX and other protocols as pluggable adapters** – the OMS/EMS exposes one internal API and hides the complexity of many external protocols, whether purchased or built.  

**Sources:** Authoritative industry documentation and observations have been used throughout (FIX Trading Community standards, exchange notices, and specialized technical articles), along with practitioner insights. Notably, CME Group’s EBS notices show migration from FIX to new APIs, and companies like Coinbase explicitly describe offering both REST/WS and FIX. Detailed FIX session logic and drop-copy behavior are documented in FIX technical standards.
