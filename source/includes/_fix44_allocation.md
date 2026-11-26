## FIX 4.4 Post-Trade Allocation

### 1. Overview

This section specifies how a client sends post-trade allocation instructions to Clear Street over FIX 4.4, and the messages Clear Street sends back.

| Direction | MsgType | Message | Purpose |
| --- | --- | --- | --- |
| Client → Clear Street | J | AllocationInstruction | New allocation or cancel of an earlier allocation |
| Clear Street → Client | P | AllocationInstructionAck | Received, accepted or rejected |
| Clear Street → Client | AK | Confirmation | One per allocation leg, with the economics Clear Street will clear |
| Client → Clear Street | AU | ConfirmationAck | Client affirms or rejects a Confirmation |

All other application messages are out of scope. Execution reports (35=8) can share the same FIX session (see [section 2](#2-connectivity-and-session)).

### 1.1 Processing modes

Clear Street processes each allocation instruction in one of two modes. The client does not choose the mode on the message. Clear Street sets it per client during onboarding, from the executing broker on the instruction.

* **Matched.** The trade was executed at Clear Street, or the client is set up to match away trades. Clear Street matches the instruction to its own executed block before it accepts. Accepted allocations are confirmed leg by leg (`35=AK`), and the client affirms them (`35=AU`) unless the client is set up without affirmation.
* **As-given.** The trade was executed away and the client is not set up for matching. Clear Street books the allocation as sent. There is no match step and no `35=AK`.

### 1.2 Allocation types

| AllocType (626) | Meaning | Economics Used |
| --- | --- | --- |
| `1` | Calculated: client sends final economics | Commission (`12`/`13`) on every leg, plus optional fees. Clear Street compares the client's commission with its own, within the client's agreed tolerance. |
| `2` | Preliminary: client sends quantities and accounts only | Clear Street calculates commission and fees. Any `12`, `13` or `136` sent on the legs is ignored. Matched mode only. |

### 2. Connectivity and session

* FIX version: BeginString `8=FIX.4.4`.
* Clear Street connects as the session **initiator**. The client is the acceptor. Host, port, CompIDs and session hours are agreed during onboarding.
* HeartBtInt `108=30` seconds.
* Allocations can use a dedicated session or the client's existing execution session. `SenderCompID` (`49`) identifies the session and must be registered with Clear Street. The client is identified by the **Client ID party** (`452=3`) on each `35=J` (see [section 4.3](#4-3-parties-nopartyids-453)), so one session can carry allocations for several clients.
* Standard FIX 4.4 session-level messages apply (Logon, Heartbeat, TestRequest, ResendRequest, Reject, SequenceReset, Logout). Use PossDupFlag (`43`) and PossResend (`97`) as normal. See [section 9](#9-duplicates-and-resends) for how duplicates are handled at the application level.

### Standard header

| Name | FIX Tag | Allowable Values | Type | Length | Required? | Description |
| - | - | - | - | - | - | - |
| `BeginString` | `8` | `FIX.4.4` | `String` |  | `R` | FIX version |
| `MsgType` | `35` | `J` `P` `AK` `AU` | `String` | `2` | `R` | Message type |
| `SenderCompID` | `49` |  | `String` |  | `R` | On client messages: the registered CompID of the sending session |
| `TargetCompID` | `56` |  | `String` |  | `R` | On client messages: Clear Street's CompID for the session |
| `MsgSeqNum` | `34` |  | `Integer` |  | `R` | Message sequence number |
| `SendingTime` | `52` |  | `UTCTimestamp` |  | `R` | Time the message was sent |

On messages from Clear Street, `49` and `56` are swapped: `49` is Clear Street and `56` is the client.

### 3. Message flows

Solid arrows are client messages. Dashed arrows are Clear Street messages. Notes describe Clear Street processing that sends no message.

### Matched

<pre class="mermaid">
sequenceDiagram
    participant C as Client
    participant CS as Clear Street
    C->>CS: 35=J 70=A1 71=0<br/>626=1 or 2
    alt Format invalid
        CS-->>C: 35=P 70=A1 87=1<br/>with 88 and 58
    else Format valid
        CS-->>C: 35=P 70=A1 87=3
        alt Business reject
            CS-->>C: 35=P 70=A1 87=1<br/>with 88 and 58
        else Accepted for matching
            Note over CS: Match to Clear Street's<br/>executed block with the<br/>same settlement date
            alt Not matched
                Note over C,CS: Stays at 87=3.<br/>Operations resolve it<br/>with the client.
            else Matched
                CS-->>C: 35=P 70=A1 87=0
                opt Affirmation (default)
                    loop Each leg
                        CS-->>C: 35=AK 664=Cn<br/>70=A1 467=leg ID
                    end
                    loop Each 35=AK
                        C->>CS: 35=AU 664=Cn<br/>940=3
                    end
                    Note over CS: All legs affirmed:<br/>Clear Street clears.<br/>No reply to 35=AU.
                end
            end
        end
    end
</pre>

1. Client sends `35=J 71=0` with `626=1` or `626=2`.
2. If the message fails [format validation](#10-1-format-validation), Clear Street sends `35=P 87=1` with `88` and `58`, and nothing further happens.
3. Otherwise Clear Street sends `35=P 87=3` (Received).
4. If the instruction fails a [business check](#10-2-business-rejects-after-87-3), Clear Street sends `35=P 87=1` with `88` and `58`.
5. Otherwise Clear Street matches the instruction to its executed block with the same settlement date.
    * Not matched: the instruction stays at `87=3`, and Clear Street operations resolve it with the client.
    * Matched: Clear Street sends `35=P 87=0` (Accepted).
6. With affirmation (default), Clear Street sends one `35=AK` per leg, each with its own ConfirmID (`664`), the client's AllocID (`70`) and the leg ID (`467`).
7. Client replies to each `35=AK` with `35=AU 940=3` quoting its `664`. When all legs are affirmed, Clear Street clears the allocation. Clear Street sends no reply to a `35=AU`.

If the client is set up without affirmation, Clear Street sends no `35=AK`, and `87=0` is final. See [section 7](#7-confirmationack-35-au-client-to-clear-street) for `940=1` and `940=2`. When the client is set up with affirmation, Clear Street does not clear a matched allocation until every leg that received a `35=AK` is affirmed (`940=3`). Clear Street's end-of-day posting (4:15 PM ET) does not include these allocations. An allocation that is not fully affirmed by then stays unposted, and Clear Street operations follow up with the client.

### As-given

<pre class="mermaid">
sequenceDiagram
    participant C as Client
    participant CS as Clear Street
    C->>CS: 35=J 70=A1 71=0<br/>626=1
    alt Format invalid
        CS-->>C: 35=P 70=A1 87=1<br/>with 88 and 58
    else Format valid
        CS-->>C: 35=P 70=A1 87=3
        alt Business reject
            CS-->>C: 35=P 70=A1 87=1<br/>with 88 and 58
        else Booked as sent
            Note over CS: Book the allocation<br/>as given
            CS-->>C: 35=P 70=A1 87=0
        end
    end
</pre>

1. Client sends `35=J 71=0 626=1`.
2. If the message fails [format validation](#10-1-format-validation), Clear Street sends `35=P 87=1` with `88` and `58`.
3. Otherwise Clear Street sends `35=P 87=3` (Received).
4. If the instruction fails a [business check](#10-2-business-rejects-after-87-3), Clear Street sends `35=P 87=1` with `88` and `58`.
5. Otherwise Clear Street books the allocation as given and sends `35=P 87=0` (Accepted).

As-given allocations get no `35=AK`. Preliminary (`626=2`) is a business reject in as-given mode (`PRELIMINARY_REQUIRES_MATCHED`).

### Cancel

<pre class="mermaid">
sequenceDiagram
    participant C as Client
    participant CS as Clear Street
    C->>CS: 35=J 70=A2 71=2<br/>72=A1 75=TD of A1
    alt Format invalid
        CS-->>C: 35=P 70=A2 87=1<br/>with 88 and 58
    else Format valid
        CS-->>C: 35=P 70=A2 87=3
        alt Cancel refused
            CS-->>C: 35=P 70=A2 87=1<br/>88=15, 7 or 0, with 58
        else Cancel allowed
            opt Matched mode
                Note over CS: Undo the match to<br/>Clear Street's block
                opt Affirmation (default)
                    loop Each leg with a 35=AK
                        CS-->>C: 35=AK 666=2<br/>772=Cn
                    end
                end
            end
            CS-->>C: 35=P 70=A2 87=0
        end
    end
</pre>

1. Client sends `35=J` with a new AllocID (`70=A2`), `71=2`, `72=A1` (the AllocID being cancelled) and `75` = the trade date of `A1`.
2. If the message fails [format validation](#10-1-format-validation), Clear Street sends `35=P 70=A2 87=1` with `88` and `58`.
3. Otherwise Clear Street sends `35=P 70=A2 87=3` (Received).
4. If the cancel is [refused](#8-cancel-71-2), Clear Street sends `35=P 70=A2 87=1` with `88=15`, `7` or `0`, and `58`.
5. Otherwise, in matched mode, Clear Street undoes the match to its block. With affirmation (default), it sends one `35=AK 666=2` per leg that received a `35=AK`, with `772` = the ConfirmID being cancelled.
6. Clear Street sends `35=P 70=A2 87=0`.

The `35=P` for a cancel always carries the **cancel's own AllocID** (`70`), not the AllocID of the allocation it cancels.

### 4. AllocationInstruction (35=J), client to Clear Street

`AllocationInstruction` is sent by the client to Clear Street. Required: `R` = required, `CR` = conditionally required, `O` = optional. Tags not listed are accepted but ignored.

### 4.1 Block fields

| Name | FIX Tag | Allowable Values | Type | Length | Required? | Description |
| - | - | - | - | - | - | - |
| `AllocID` | `70` |  | `String` |  | `R` | Client's ID for this message. Unique per client, SenderCompID, trade date and AllocTransType. The same AllocID on two sessions, or for two clients on one session, is two separate allocations. Clear Street sends no `35=P` for a message without `70` |
| `AllocTransType` | `71` | `0` `2` | `String` | `1` | `R` | `0-New` `2-Cancel`. `1` (Replace) is rejected: send a Cancel, then a New. `3`, `4` and `5` are rejected on FIX 4.4: use AllocType `626` |
| `RefAllocID` | `72` |  | `String` |  | `CR` | Required when `71=2`: the AllocID (`70`) of the New being cancelled |
| `AllocType` | `626` | `1` `2` | `Integer` | `1` | `R` | `1-Calculated` `2-Preliminary`. Required on Cancel too |
| `AllocNoOrdersType` | `857` | `0` `1` | `Integer` | `1` | `R` | `0-Not specified` `1-Explicit list provided`. Validated, not otherwise used. NoOrders (`73`) and NoExecs (`124`) are not read |
| `Side` | `54` | `1` `2` `3` `4` `5` `6` | `String` | `1` | `R` | `1-Buy` `2-Sell` `3-Buy minus` (booked as Buy) `4-Sell plus` (booked as Sell) `5-Sell Short` `6-Sell Short Exempt` (booked as Sell Short). Other values are rejected |
| `Symbol` | `55` |  | `String` |  | `R` | Ticker symbol. For options, the underlying symbol |
| `SecurityID` | `48` |  | `String` |  | `O` | Instrument identifier based on tag `22`. Recommended |
| `SecurityIDSource` | `22` | `1` `2` `4` `8` `H` | `String` | `1` | `CR` | Required when `48` is sent. `1-CUSIP` `2-SEDOL` `4-ISIN` `8-Exchange Symbol` `H-Clearing house` (OCC/OSI for options) |
| `SecurityType` | `167` | `CS` `PS` `WAR` `OPT` | `String` |  | `CR` | Other values are accepted as an unspecified type. Required for options (`OPT`) |
| `CountryOfIssue` | `470` |  | `String` |  | `O` | ISO country code |
| `Quantity` | `53` |  | `Decimal` |  | `R` | Block quantity. Must be greater than 0 |
| `AvgPx` | `6` |  | `Decimal` |  | `R` | Block average price |
| `Currency` | `15` |  | `String` | `3` | `O` | ISO 4217 trade currency |
| `TradeDate` | `75` |  | `Integer` | `8` | `R` | Trade date in `YYYYMMDD` format |
| `TransactTime` | `60` |  | `UTCTimestamp` |  | `O` | Rejected if malformed |
| `SettlDate` | `64` |  | `Integer` | `8` | `O` | Settlement date in `YYYYMMDD` format. Rejected (`88=18`) if malformed |
| `GrossTradeAmt` | `381` |  | `Decimal` |  | `O` | Gross trade amount |
| `NetMoney` | `118` |  | `Decimal` |  | `O` | Net money |
| `PositionEffect` | `77` | `O` `C` | `String` | `1` | `O` | `O-Open` `C-Close` |
| `NoPartyIDs` | `453` |  | `Integer` |  | `R` | Repeating group. Must include the Client ID party (`452=3`), see [section 4.3](#4-3-parties-nopartyids-453) |
| `NoAllocs` | `78` |  | `Integer` |  | `R` | Number of legs. Must be at least 1 and must equal the number of legs actually sent. Optional on a cancel: send no legs, or every leg of the New (see [section 8](#8-cancel-71-2)) |

**Instrument.** Send `55` on every instruction. Also send `48` with `22` where you can. If Clear Street cannot identify the instrument in its reference data, the instruction is booked on the identifiers the client sent:

* Matched: the instruction is not rejected. Clear Street can pair it with its executed block only on the identifier sent. If no block pairs, the instruction stays at `87=3` (see [section 10.5](#10-5-instructions-that-stay-at-87-3)).
* As-given: the allocation is booked only if the instruction carries CountryOfIssue (`470`) and Currency (`15`). Without either, it is rejected with `88=17` `UNKNOWN_INSTRUMENT` (see [section 10.2](#10-2-business-rejects-after-87-3)).

An instrument that Clear Street identifies as fixed income is rejected with `88=17` `UNSUPPORTED_SECURITY_TYPE`, whatever `167` says.

### Options (167=OPT)

| Name | FIX Tag | Allowable Values | Type | Length | Required? | Description |
| - | - | - | - | - | - | - |
| `Symbol` | `55` |  | `String` |  | `R` | Underlying root, for example `AAPL` |
| `PutOrCall` | `201` | `0` `1` | `Integer` | `1` | `R` | `0-Put` `1-Call`. Other values are rejected |
| `StrikePrice` | `202` |  | `Decimal` |  | `R` | Strike price for the option |
| `MaturityDate` | `541` |  | `Integer` | `8` | `R` | Option expiry in `YYYYMMDD` format. MaturityDay (`205`) is not in Clear Street's FIX 4.4 dictionary, so send the expiry in `541` |
| `MaturityMonthYear` | `200` |  | `Integer` | `6` | `O` | `YYYYMM`. Not used to derive the expiry |
| `ContractMultiplier` | `231` |  | `Decimal` |  | `O` | Validated, not otherwise used |

Clear Street derives the OSI symbol from `55`, `201`, `202` and `541`, and books the option on it. If `48` is also sent with `22=H`, a difference from the derived OSI symbol does not reject the instruction. The client's `55`, `48`, `22` and `167` are echoed on the `35=AK`. The other option fields are not.

### 4.2 Allocation legs (NoAllocs, 78)

| Name | FIX Tag | Allowable Values | Type | Length | Required? | Description |
| - | - | - | - | - | - | - |
| `AllocAccount` | `79` |  | `String` |  | `R` | An account Clear Street has set up for the client: a Clear Street account number or an agreed alias. AllocAcctIDSource (`661`) is not read |
| `AllocQty` | `80` |  | `Decimal` |  | `R` | Greater than 0. The sum of `80` over all legs must equal `53` |
| `IndividualAllocID` | `467` |  | `String` |  | `O` | Recommended. Client's leg ID. Must be unique within the message. If absent, Clear Street uses `<AllocID>.<leg number>`, for example `A1.2`. Echoed on the leg's `35=AK` |
| `Commission` | `12` |  | `Decimal` |  | `CR` | Required on every leg of a Calculated (`626=1`) New. Ignored on Preliminary |
| `CommType` | `13` | `1` `2` `3` | `String` | `1` | `CR` | `1-Per unit` `2-Percentage` `3-Absolute`. Required when `12` is sent on a Calculated (`626=1`) New. Other values are rejected. Ignored on Preliminary |
| `AllocNetMoney` | `154` |  | `Decimal` |  | `O` | Net money for the leg |
| `NoMiscFees` | `136` |  | `Integer` |  | `O` | Repeating group: `137` MiscFeeAmt (`Decimal`), `139` MiscFeeType. See [section 4.4](#4-4-fees-miscfeetype-139). Ignored on Preliminary |

**Commission calculation.** Clear Street converts each leg's commission to an absolute amount:

* `13=3`: `12` is the amount.
* `13=1`: `12` × `80`.
* `13=2`: `12` is a ratio (0.001 = 0.1%), so `12` × `6` × `80`.

### 4.3 Parties (NoPartyIDs, 453)

Clear Street reads two entries from NoPartyIDs (`453`). Entries with other PartyRole values are ignored.

| PartyRole (452) | Meaning | Required? | PartyID (448) | PartyIDSource (447) |
| --- | --- | --- | --- | --- |
| `3` | Client ID | `R` | The client's Clear Street client code or alias, agreed during onboarding. Clear Street resolves it to the client | As agreed during onboarding, for example `D` (Proprietary) |
| `1` | Executing firm | `O` | Executing broker MPID | Not read |

Send exactly one `452=3` entry on every `35=J`, including a cancel. There is no fallback to SenderCompID: an instruction without it is rejected (see [section 10.1](#10-1-format-validation)). A client code or alias that is not set up is rejected with `UNKNOWN_CLIENT` (see [section 10.2](#10-2-business-rejects-after-87-3)). The leg accounts (`79`) are resolved against this client. Clear Street sends the same party back on each `35=AK`.

If no executing firm is sent, Clear Street assumes the trade was executed at Clear Street. The executing broker decides the [processing mode](#1-1-processing-modes).

### 4.4 Fees (MiscFeeType, 139)

| MiscFeeType (139) | Meaning |
| --- | --- |
| `1` | Regulatory |
| `2` | Tax |
| `4` | Exchange fees |
| `5` | Stamp duty |
| `6` | Levy (for example PTM levy) |
| other | Other |

In as-given mode, Clear Street calculates its own fees and takes only stamp duty (`5`) and levy (`6`) from the client.

### 5. AllocationInstructionAck (35=P), Clear Street to client

`AllocationInstructionAck` is sent by Clear Street to the client.

| Name | FIX Tag | Allowable Values | Type | Description |
| - | - | - | - | - |
| `AllocID` | `70` |  | `String` | The AllocID of the message being answered (for a cancel, the cancel's own `70`) |
| `TradeDate` | `75` |  | `Integer` | The instruction's trade date. If the instruction had no valid `75`, today's date in New York |
| `TransactTime` | `60` |  | `UTCTimestamp` | Time the ack was generated (UTC) |
| `AllocStatus` | `87` | `3` `0` `1` | `Integer` | `3-Received` `0-Accepted` `1-Block level reject` |
| `AllocRejCode` | `88` |  | `Integer` | Only when `87=1`. See [section 10](#10-reject-reference-35-p-87-1) |
| `Text` | `58` |  | `String` | Only when `87=1`. Machine-readable reason, see [section 10](#10-reject-reference-35-p-87-1) |

Clear Street does not send `72` or the legs on the `35=P`.

### Status sequence

1. `87=3` **Received** is sent as soon as the instruction passes format validation. If validation fails, the client gets `87=1` instead and nothing further happens.
2. After `87=3`, Clear Street can still reject the instruction with `87=1`, for example for an unknown client, account or instrument.
3. `87=0` **Accepted** is sent:
    * as-given: once the allocation is booked;
    * matched: once the instruction matches Clear Street's executed block;
    * cancel: once the cancel has taken effect.

A matched instruction that does not match (including when no executed block has the same settlement date), or matches with a commission difference outside tolerance, stays at `87=3`. Clear Street operations resolve it with the client. When it is matched, the client receives `87=0`. Send allocations after the trades are executed.

### 6. Confirmation (35=AK), Clear Street to client

`Confirmation` is sent by Clear Street to the client in matched mode only, unless the client is set up without affirmation. One `35=AK` is sent per allocation leg, after the `87=0`. The figures are those Clear Street will clear, not the client's own. Each `35=AK` carries the Client ID party from the `35=J`, so a session that carries several clients can route each confirmation.

| Name | FIX Tag | Allowable Values | Type | Description |
| - | - | - | - | - |
| `ConfirmID` | `664` |  | `String` | Clear Street's ID for this leg. Quote it on the `35=AU` |
| `ConfirmTransType` | `666` | `0` | `Integer` | `0-New` |
| `ConfirmType` | `773` | `2` | `Integer` | `2-Confirmation` |
| `ConfirmStatus` | `665` | `4` | `Integer` | `4-Confirmed` |
| `NoPartyIDs` | `453` | `1` | `Integer` | One entry, with `448` PartyID (the client code or alias from the `35=J`), `447` PartyIDSource (as the client sent it; left out if the `35=J` had none) and `452` PartyRole `3` (Client ID) |
| `TransactTime` | `60` |  | `UTCTimestamp` | UTC |
| `AllocID` | `70` |  | `String` | The client's AllocID |
| `IndividualAllocID` | `467` |  | `String` | The client's leg ID (or `<AllocID>.<n>`) |
| `TradeDate` | `75` |  | `Integer` | Trade date in `YYYYMMDD` format |
| `Symbol` / `SecurityID` / `SecurityIDSource` / `SecurityType` | `55` / `48` / `22` / `167` |  | `String` | Instrument, as sent by the client |
| `AllocQty` | `80` |  | `Decimal` | Leg quantity |
| `Side` | `54` | `1` `2` `5` | `String` | `1-Buy` `2-Sell` `5-Sell Short`. A `35=J` sent with `54=3`, `4` or `6` is confirmed as `1`, `2` or `5` |
| `Currency` | `15` |  | `String` | Trade currency |
| `AllocAccount` | `79` |  | `String` | As sent by the client |
| `AvgPx` | `6` |  | `Decimal` | Average price of Clear Street's executed block, the same on every leg. Rounded to 4 decimal places; trailing zeros are not sent |
| `PriceType` | `423` | `2` | `Integer` | `2-Per unit` |
| `GrossTradeAmt` | `381` |  | `Decimal` | Rounded to the currency's precision |
| `NetMoney` | `118` |  | `Decimal` | Net money |
| `SettlDate` | `64` |  | `Integer` | When known |
| `Commission` | `12` |  | `Decimal` | Absolute amount, rounded to the currency's precision |
| `CommType` | `13` | `3` | `String` | `3-Absolute` |
| `NoCapacities` | `862` | `1` | `Integer` | One entry, with `528` OrderCapacity (`A-Agency` `P-Principal` `R-Riskless Principal`) and `863` OrderCapacityQty (= `80`) |
| `NoMiscFees` | `136` |  | `Integer` | Count may be 0. Each fee: `139` (`1-Regulatory` `2-Tax` `4-Exchange` `7-Other`), `137`, `891` MiscFeeBasis `0-Absolute` |

### Confirmation cancel

When a cancel of a matched allocation is accepted, Clear Street withdraws every `35=AK` it sent for that allocation before it sends the cancel's `87=0`. It sends one `35=AK` per leg, with the same leg fields as the original `35=AK` and these differences:

| Name | FIX Tag | Allowable Values | Type | Description |
| - | - | - | - | - |
| `ConfirmID` | `664` |  | `String` | A new ID for this message |
| `ConfirmRefID` | `772` |  | `String` | The ConfirmID (`664`) of the `35=AK` being cancelled |
| `ConfirmTransType` | `666` | `2` | `Integer` | `2-Cancel` |

The cancelled ConfirmIDs are no longer valid, and a `35=AU` for them is ignored. Clients set up without affirmation get no `35=AK`, so they get no `35=AK` cancel either.

### 7. ConfirmationAck (35=AU), client to Clear Street

`ConfirmationAck` is sent by the client to Clear Street to affirm or reject a `35=AK`.

| Name | FIX Tag | Allowable Values | Type | Length | Required? | Description |
| - | - | - | - | - | - | - |
| `ConfirmID` | `664` |  | `String` |  | `R` | From the `35=AK` |
| `AffirmStatus` | `940` | `1` `2` `3` | `Integer` | `1` | `R` | `1-Received` `2-Confirm rejected` `3-Affirmed` |
| `ConfirmRejReason` | `774` | `1` `2` `99` | `Integer` |  | `CR` | Required with `940=2`. `1-Mismatched account` `2-Missing settlement instructions` `99-Other` |
| `Text` | `58` |  | `String` |  | `O` | Free text, recommended with `940=2` |
| `TradeDate` | `75` |  | `Integer` | `8` | `R` | Trade date in `YYYYMMDD` format |
| `TransactTime` | `60` |  | `UTCTimestamp` |  | `R` | Time the ack was generated |

Clear Street sends no FIX reply to a `35=AU`.

| Case | Result |
| --- | --- |
| `940=1` | Recorded. No change |
| `940=3` | Leg affirmed. When every leg that got a `35=AK` is affirmed, Clear Street clears the allocation. Until then the allocation is not included in Clear Street's end-of-day posting (4:15 PM ET) |
| `940=2` before the leg is affirmed | Leg marked rejected. Clear Street operations contact the client. A later `940=3` for the same ConfirmID is still accepted |
| `940=2` after the leg is affirmed | Ignored: the leg stays affirmed. Clear Street operations follow up with the client |
| Unknown `664`, `664` from a different SenderCompID, missing `664` or `940` | Ignored. Clear Street operations investigate |
| The allocation is no longer accepted (for example cancelled), or a cancel of it is being processed | Ignored. If the cancel is refused, send the `35=AU` again |

### 8. Cancel (71=2)

* A cancel (`71=2`) always cancels the **entire** allocation instruction named in RefAllocID (`72`), with all its legs. Cancelling some legs is not supported. To change some legs, cancel the instruction and send a New with a new AllocID.
* Send a new `35=J` with its own unique AllocID (`70`), `71=2` and `72` = the AllocID of the New to cancel.
* Send the cancel on the same session (SenderCompID) as the New, with the same Client ID party (`452=3`). **TradeDate (`75`) must equal the original New's trade date.** Clear Street finds the original by client, SenderCompID, `72` and `75`. A cancel sent on another session or for a different client is refused with `UNKNOWN_REF_ALLOC_ID`.
* The block fields (`626`, `857`, `54`, `55`, `53`, `6` and `75`) must be sent and pass the same format checks as a New. Legs (`78` and the NoAllocs group) are optional on a cancel. Send no legs, or send every leg of the New. Legs that are sent are validated like a New's, and the sum of AllocQty (`80`) must equal Quantity (`53`). Commission (`12`/`13`) is not required. The block fields are not compared with the original.
* A cancel is acknowledged `87=0` on its own AllocID. In matched mode the `87=0` is sent once the match has been undone. If the client is set up with affirmation, Clear Street first withdraws each `35=AK` it sent for the allocation (see [Confirmation cancel](#confirmation-cancel)).
* To correct an allocation: cancel it, then send a New with a **new** AllocID.

A cancel is refused (`87=1`) when:

| AllocRejCode (88) | Text (58) | Case |
| --- | --- | --- |
| `15` | `UNKNOWN_REF_ALLOC_ID` | No New with AllocID = `72` on that trade date from this client and SenderCompID |
| `0` | `UNKNOWN_CLIENT:<448>` | The cancel's Client ID party is not a client code or alias set up with Clear Street |
| `7` | `NOT_CANCELLABLE:<status>` | The New was rejected or already cancelled, or another cancel of it is being processed (`CANCEL_PENDING`) |
| `7` | `CANCEL_AFTER_AFFIRM_NOT_SUPPORTED` | At least one leg has been affirmed (`940=3`) |
| `7` | `CANCEL_AFTER_POST_NOT_SUPPORTED` | Clear Street has already cleared the allocation |
| `7` | `UNMATCH_REFUSED:NO_SETTLE_DATE` | The allocation is matched, but Clear Street has no settlement date for it, so it cannot safely undo the match |
| `7` | `UNMATCH_REFUSED:PAST_SETTLE_DATE:<YYYYMMDD>` | The allocation is matched and its settlement date has passed |
| `7` | `DUPLICATE_CANCEL` | The same cancel AllocID is resent after its earlier refusal no longer applies, for example after `UNKNOWN_REF_ALLOC_ID` once the New has arrived |

For a matched allocation, the `UNMATCH_REFUSED` refusals can also arrive after `87=3`, once Clear Street has tried to undo the match. While a cancel is being processed, `35=AU` messages for that allocation are ignored; send them again if the cancel is refused.

After an `UNKNOWN_REF_ALLOC_ID` refusal, send the cancel again with a new AllocID once the New has been sent. For the other refusals, contact Clear Street operations.

### 9. Duplicates and resends

* Clear Street processes each (client, SenderCompID, TradeDate, AllocID, AllocTransType) once. The same AllocID on two sessions, or for two clients on one session, is two separate allocations.
* A New resent with the same AllocID gets the stored outcome again: `87=0` if accepted, `87=1` with the original `88`/`58` if rejected, or `87=1 88=7` `ALLOC_ID_CANCELLED` if it has since been cancelled. No reply is sent while a matched New is still pending.
* A resent cancel that cancelled the allocation is acknowledged `87=0` again (but not while the cancel is still pending). A resent cancel that was refused gets the refusal again; if that refusal no longer applies, it gets `87=1 88=7` `DUPLICATE_CANCEL`.
* Do not reuse an AllocID for a different allocation.

### 10. Reject reference (35=P 87=1)

Rejects are sent as `35=P 87=1` with AllocRejCode (`88`) and Text (`58`).

### 10.1 Format validation

Rules are checked in this order, and the first failure is returned. `<n>` is a tag number. `(leg n)` is the 1-based leg.

| # | Rule | AllocRejCode (88) | Text (58) |
| --- | --- | --- | --- |
| 1 | AllocTransType | `7` | `MISSING_TAG:71`, `REPLACE_NOT_SUPPORTED_USE_CANCEL_NEW`, `DEPRECATED_ALLOC_TRANS_TYPE:<v>_USE_626` (v = 3, 4 or 5), `UNSUPPORTED_ALLOC_TRANS_TYPE:<v>`, `MISSING_TAG:72` (cancel) |
| 2 | AllocType present | `7` | `MISSING_TAG:626` |
| 3 | AllocType value | `7` | `UNSUPPORTED_ALLOC_TYPE:<v>` |
| 4 | AllocNoOrdersType present | `7` | `MISSING_TAG:857` |
| 5 | AllocNoOrdersType value | `7` | `UNSUPPORTED_ALLOC_NO_ORDERS_TYPE:<v>` |
| 6 | Side and instrument | `7` / `17` | `MISSING_TAG:54`, `UNSUPPORTED_SIDE:<v>` (`88=7`); `UNKNOWN_INSTRUMENT:MISSING` when neither `55` nor `48` is sent (`88=17`) |
| 7 | Block quantity | `1` | `QTY:MISSING`, `QTY:INVALID:<v>` |
| 8 | AvgPx present | `2` | `AVG_PX:MISSING` |
| 9 | AvgPx format | `2` | `AVG_PX:INVALID:<v>` |
| 10 | TradeDate | `7` | `MISSING_TAG:75`, `INVALID_TAG:75` |
| 11 | TransactTime format | `7` | `INVALID_TAG:60` |
| 12 | SettlDate format | `18` | `SETTL_DATE:INVALID:<v>` |
| 13 | Declared leg count | `7` | `INVALID_TAG:78` (`78` differs from the legs sent) |
| 14 | At least one leg (New only) | `1` | `QTY:NO_ALLOCATIONS` |
| 15 | Leg accounts | `0` | `UNKNOWN_ACCOUNT:MISSING(leg n)` |
| 16 | Leg quantities | `1` | `QTY:LEG_<n>` (missing, not numeric or not above 0) |
| 17 | Commission on Calculated | `7` | `MISSING_TAG:12(leg n)` |
| 18 | Commission type | `7` / `2` | `MISSING_TAG:13(leg n)`, `INVALID_TAG:13(leg n)` (`88=7`); `AVG_PX:MISSING` for `13=2` without `6` (`88=2`) |
| 19 | Leg quantities add up | `1` | `QTY:SUM_<sum>_NE_<53>` |
| 20 | Block amounts | `7` | `INVALID_TAG:381`, `INVALID_TAG:118` |
| 21 | PutOrCall | `7` | `INVALID_TAG:201` |
| 22 | Option numbers | `7` | `INVALID_TAG:202`, `INVALID_TAG:231` |
| 23 | Leg numbers | `7` | `INVALID_TAG:<n>(leg n)` for `12`, `154` or `137`. `12` and `137` are not checked on Preliminary (`626=2`) |
| 24 | ProcessCode | `7` | `UNSUPPORTED_PROCESS_CODE:<v>(leg n)` for any `81` other than `0` |
| 25 | Client ID party (New and cancel) | `7` | `MISSING_CLIENT_ID` when no `452=3` entry is sent, or its `448` is empty; `MULTIPLE_CLIENT_IDS` when more than one `452=3` entry is sent |

On a cancel (`71=2`), rule 14 does not apply: a cancel can send no legs. When a cancel sends legs, it must send every leg of the New, and the leg rules apply as on a New. Rules 17 and 18 apply only to a Calculated New.

Numbers must use FIX float syntax: an optional `-`, digits and an optional `.`, with no exponent. A required tag sent with an empty value counts as missing. An optional tag sent with an empty value (for example `60=`, `64=`, `81=` or a numeric tag) counts as malformed.

A message that can only be partly parsed (for example a duplicate tag, or group fields out of order) is rejected with `88=7` and `INVALID_TAG:<n>` or `MALFORMED_MESSAGE`.

### 10.2 Business rejects (after 87=3)

| AllocRejCode (88) | Text (58) | Case |
| --- | --- | --- |
| `0` | `UNKNOWN_CLIENT:<448>` | The Client ID party's PartyID (`448`) is not a client code or alias set up with Clear Street, on a New or a cancel |
| `0` | `UNKNOWN_ACCOUNT:<79,...>` | One or more AllocAccount (`79`) values are not set up for the client |
| `7` | `DUPLICATE_INDIVIDUAL_ALLOC_ID:<467,...>` | Two legs share an IndividualAllocID |
| `7` | `PRELIMINARY_REQUIRES_MATCHED` | Preliminary (`626=2`) sent for an as-given allocation |
| `17` | `UNKNOWN_INSTRUMENT:<48 or 55>` | As-given only: Clear Street cannot identify the instrument, and the instruction has no CountryOfIssue (`470`) or no Currency (`15`) |
| `17` | `UNSUPPORTED_SECURITY_TYPE:<type>` | Clear Street identifies the instrument as fixed income. `<type>` is `CORPORATE_BOND`, `GOVERNMENT_BOND` or `FIXED_INCOME` |
| `7` | `ALLOC_ID_CANCELLED` | A New resent after it was cancelled |

The cancel refusals are listed in [section 8](#8-cancel-71-2).

### 10.3 AllocRejCode (88) values used

| AllocRejCode (88) | Meaning |
| --- | --- |
| `0` | Unknown account |
| `1` | Incorrect quantity |
| `2` | Incorrect average price |
| `7` | Other (see `58`) |
| `15` | Trade not recognized |
| `17` | Incorrect or missing instrument |
| `18` | Incorrect or missing settlement date |

Values `15`, `17` and `18` come from a later FIX version and are added to Clear Street's FIX 4.4 dictionary. Configure your FIX engine to accept them on `88`.

### 10.4 Messages that get no 35=P

Clear Street cannot reply to a message without AllocID (`70`), SenderCompID (`49`) or TargetCompID (`56`), or to a message it cannot parse at all. These are raised to Clear Street operations.

### 10.5 Instructions that stay at 87=3

In matched mode these are not rejects. The instruction stays at `87=3` and Clear Street operations resolve it with the client.

* No executed block matches the instruction.
* No executed block has the same settlement date. When `64` is not sent, Clear Street uses the standard settlement date for the instrument.
* The commission differs from Clear Street's by more than the client's agreed tolerance.
* Clear Street cannot identify the instrument, and no executed block has the identifier sent.

### 11. Not supported

* AllocTransType `1` (Replace), `3`, `4` and `5` on FIX 4.4.
* AllocationReport (`35=AS`) and AllocationInstructionAlert.
* Account-level reject (`87=2`): any failure rejects the whole block.
* Per-leg rejects in the `35=P` NoAllocs group.
* Custom tags.
* Cancelling some legs of an allocation: a cancel always cancels the whole allocation.
* Step-outs: a leg with ProcessCode (`81`) other than `0` is rejected.
* Fixed income (bonds): rejected with `88=17` `UNSUPPORTED_SECURITY_TYPE`. Support will be added in a later version of this specification.

### 12. Examples

`|` stands for SOH. Header tags `9`, `34`, `52` and `10` are left out.

### Calculated New, two legs (matched)

```
BeginString 8=FIX.4.4
MsgType 35=J
SenderCompID 49=CLIENTALLOC
TargetCompID 56=CLSTALLOC
AllocID 70=A1
AllocTransType 71=0
AllocType 626=1
AllocNoOrdersType 857=0
Side 54=1
Symbol 55=AAPL
SecurityID 48=037833100
SecurityIDSource 22=1
SecurityType 167=CS
Quantity 53=1000
AvgPx 6=190.25
Currency 15=USD
TradeDate 75=20261110
TransactTime 60=20261110-20:15:00.000
SettlDate 64=20261112
NoPartyIDs 453=2
  PartyID 448=CLST
  PartyIDSource 447=D
  PartyRole 452=1
  PartyID 448=TC-FUND
  PartyIDSource 447=D
  PartyRole 452=3
NoAllocs 78=2
  AllocAccount 79=ACC001
  AllocQty 80=600
  IndividualAllocID 467=A1-1
  Commission 12=12.00
  CommType 13=3
  AllocAccount 79=ACC002
  AllocQty 80=400
  IndividualAllocID 467=A1-2
  Commission 12=8.00
  CommType 13=3
```

<pre class="fix-message">
8=FIX.4.4|35=J|49=CLIENTALLOC|56=CLSTALLOC|70=A1|71=0|626=1|857=0|54=1|55=AAPL|48=037833100|22=1|167=CS|53=1000|6=190.25|15=USD|75=20261110|60=20261110-20:15:00.000|64=20261112|453=2|448=CLST|447=D|452=1|448=TC-FUND|447=D|452=3|78=2|79=ACC001|80=600|467=A1-1|12=12.00|13=3|79=ACC002|80=400|467=A1-2|12=8.00|13=3
</pre>

Responses:

<pre class="fix-message">
8=FIX.4.4|35=P|49=CLSTALLOC|56=CLIENTALLOC|70=A1|75=20261110|60=20261110-20:15:01.120|87=3
8=FIX.4.4|35=P|49=CLSTALLOC|56=CLIENTALLOC|70=A1|75=20261110|60=20261110-20:16:30.004|87=0
8=FIX.4.4|35=AK|49=CLSTALLOC|56=CLIENTALLOC|664=01JC5Z8Q2W0000000000000001|666=0|773=2|665=4|453=1|448=TC-FUND|447=D|452=3|60=20261110-20:16:30.010|70=A1|467=A1-1|75=20261110|55=AAPL|48=037833100|22=1|167=CS|80=600|54=1|15=USD|79=ACC001|6=190.25|423=2|381=114150|118=114162|64=20261112|12=12|13=3|862=1|528=A|863=600|136=0
</pre>

Affirmation:

<pre class="fix-message">
8=FIX.4.4|35=AU|49=CLIENTALLOC|56=CLSTALLOC|664=01JC5Z8Q2W0000000000000001|940=3|75=20261110|60=20261110-20:20:00.000
</pre>

### Cancel

Neither leg of `A1` has been affirmed yet, so the cancel is allowed. The cancel sends no legs. It could instead send both legs of `A1`; they are then validated like a New's and must add up to `53`. Clear Street withdraws both `35=AK` messages before the `87=0`.

<pre class="fix-message">
8=FIX.4.4|35=J|49=CLIENTALLOC|56=CLSTALLOC|70=A2|71=2|72=A1|626=1|857=0|54=1|55=AAPL|48=037833100|22=1|53=1000|6=190.25|75=20261110|453=1|448=TC-FUND|447=D|452=3
8=FIX.4.4|35=P|49=CLSTALLOC|56=CLIENTALLOC|70=A2|75=20261110|60=20261110-20:30:00.500|87=3
8=FIX.4.4|35=AK|49=CLSTALLOC|56=CLIENTALLOC|664=01JC5Z8Q2W0000000000000011|772=01JC5Z8Q2W0000000000000001|666=2|773=2|665=4|453=1|448=TC-FUND|447=D|452=3|60=20261110-20:30:02.050|70=A1|467=A1-1|75=20261110|55=AAPL|48=037833100|22=1|167=CS|80=600|54=1|15=USD|79=ACC001|6=190.25|423=2|381=114150|118=114162|64=20261112|12=12|13=3|862=1|528=A|863=600|136=0
8=FIX.4.4|35=AK|49=CLSTALLOC|56=CLIENTALLOC|664=01JC5Z8Q2W0000000000000012|772=01JC5Z8Q2W0000000000000002|666=2|773=2|665=4|453=1|448=TC-FUND|447=D|452=3|60=20261110-20:30:02.060|70=A1|467=A1-2|75=20261110|55=AAPL|48=037833100|22=1|167=CS|80=400|54=1|15=USD|79=ACC002|6=190.25|423=2|381=76100|118=76108|64=20261112|12=8|13=3|862=1|528=A|863=400|136=0
8=FIX.4.4|35=P|49=CLSTALLOC|56=CLIENTALLOC|70=A2|75=20261110|60=20261110-20:30:02.100|87=0
</pre>

### Validation reject

<pre class="fix-message">
8=FIX.4.4|35=P|49=CLSTALLOC|56=CLIENTALLOC|70=A3|75=20261110|60=20261110-20:40:00.000|87=1|88=1|58=QTY:SUM_900_NE_1000
</pre>
