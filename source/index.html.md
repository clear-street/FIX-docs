---
title: FIX Trade Specification


toc_footers:
  - [Powered By Slate](https://github.com/slatedocs/slate)

search: true
---

# Introduction
FIX trade specification details the tags and values along with description of values for each trade field for processing incoming trades from OMS clients. Clear Street currently utilizes FIX 4.2 format.

# Configuring FIX Session

```
// Example Config
[DEFAULT]
ReconnectInterval=5
SenderCompID=TEST
FileLogPath=/tmp
HeartBtInt=30
SocketAcceptPort=31234
TargetCompID=CLST
HeartBtInt=30
SocketConnectPort=31234
SocketConnectHost=100.0.0.0
SessionQualifier=1
BeginString=FIX.4.2

[SESSION]
StartTime=10:00:00
EndTime=02:00:00
SQLStoreDriver=postgres
```
A FIX session is defined as a unique combination of a `BeginString` (the FIX version number 4.2), a
`SenderCompID` (OMS Client ID as defined by Clear Street), and a `TargetCompID` (Clear Street's ID CLST). A `SessionQualifier` can also be used to disambiguate otherwise identical sessions.

A FIX session has an active time period during which all data is transmitted. Clear Street currently accepts connections between 5 AM EST to 9 PM EST on all business days.

A Logon message (Tag 35=A) must be sent to Clear Street on every business day to indicate the beginning of activity and a Logout message (Tag 35=5)must be sent to Clear Street indicating end of activity for that day.

Clear Street also recommends exchanging heartbeats every 30 seconds by sending a message with tag 35=0.

Clear Street expects every message sent to have a unique continuous sequence number as part of message with tag 34=. If there are any gaps in sequence numbers, a sequence reset message will sent to OMS client with tag 34=4.

All trade messages must have tag 35=8 indicating that each message is an execution report. Please note that we have added few custom tags starting with 9001 to allow clients to send specific information needed for trades to be processed at our end. These tags are listed as part of the inbound specification.


# FIX Inbound Trade Specification

## Allocation Trade


```
8=FIX.4.29=25335=849=OMS_CLIENT56=0000913234=12914352=20201021-21:42:34
20=09001=A1=10007817=CLIENT_TRADE_ID75=2020102122=448=US70450Y1038
421=USA15=USD31=000213.48000032=0000000298754=263=064=20201023
60=20201021-13:42:34.12347=A76=ABCD79=10001710=180
```


```
BeginString 8=FIX.4.2
BodyLength9=253
MsgType 35=8
SenderCompID49=TEST
TargetCompID 56=CLST
MsgSeqNum 34=129143
SendingTime52=20201021-21:42:34
ExecutionTransactionType20=0
TradeType 9001=A
AccountID1=100078
ClientTradeID17=CLIENT_TRADE_ID
TradeDate75=20201021
InstrumentIdentifierType 22=4
InstrumentIdentifier48=US70450Y1038
InstrumentCountry421=USA
InstrumentCurrency15=USD
Price31=000213.480000
Quantity32=00000002987
Side54=2
SettlementDateType 63=0
SettlementDate 64=20201023
TradeExecutionTime60=20201021-13:42:34.123
Capacity 47=A
TargetAccountID 79=100017
CheckSum 10=180
```

This trade type is used to facilitate average-price workflows, i.e. averaging many
trades for a customer and allocating it to them as a single trade.

| Name | FIX Tag | Allowable Values | Type | Length | Required? | Description |
| - | - | - | - | - | - | - |
| `ExecutionTransactionType` | `20` | `0` `1` | `Integer` | `1` | `R` | `0-New` `1-Cancel` |
| `TradeType` | `9001` | `A` | `String` | `1` | `R` | `A-Allocation` |
| `AccountID` | `1` |  | `Integer` | `6` | `R` | Clear Street provided account id |
| `BehalfOfAccountID` | `109` |  | `Integer` | `6` | `O` | Clear Street provided account ID if this trade is on behalf of another account |
| `BranchOffice` | `9003` |  | `String` |  | `O` | Branch office for this trade |
| `TradeID` | `17` |  | `String` |  | `R` | Unique Trade ID for this trade. Must be unique across days |
| `CancelTradeID` | `9009` |  | `String` |  | `CR` | Original trade ID to cancel; Required for all Cancel trades |
| `TradeDate` | `75` |  | `Integer` | `8` | `R` | Trade Date in `YYYYMMDD` format |
| `Instrument Identifier Type` | `22` | `1` `2` `4` `8` | `Integer` | `1` | `CR` | `1-CUSIP 2-SEDOL 4-ISIN 8-TICKER` Required unless the instrument is an Option and tags 167, 55, 200, 201, 202, and 205 are provided. | `8` |
| `Instrument Identifier` | `48` |  | `String` |  | `CR` | Instrument Identifier based on tag 22 Required unless the instrument is an Option and tags 167, 55, 200, 201, 202, and 205 are provided. | `AAPL` |
| `Security Type` | `167` | | `String` | | `CR` | Indicates type of security. Required for Options where tags 22 or 48 are not provided. | `OPT` |
| `Symbol` | `55` | | `String` | `6` | `CR` | Ticker symbol. Required for Options where tags 22 or 48 are not provided. | `SPY` |
| `Maturity Month Year` | `200` | | `Integer` | `6` | `CR` | Month and year of the maturity for an Option. Maturity Month Year in format `YYYYMM`. Required for Options where tags 22 or 48 are not provided. | `202107` |
| `Put Or Call` | `201` | `0` `1` | `Integer` | `1` | `CR` | Indicates whether an Option is for a put or a call `0 = Put` `1 = Call` Required for Options where tags 22 or 48 are not provided. | `1` |
| `Strike Price` | `202` | | `Decimal` | `8` | `CR` | Strike Price for an Option. Required for Options where tags 22 or 48 are not provided. | `100.50` |
| `Maturity Day` | `205` | | `Integer` | `2` | `CR` | To be used in conjunction with Maturity Month Year `200` to specify a particular maturity date for an Option. Required for Options where tags 22 or 48 are not provided. | `30` | 
| `InstrumentCountry` | `421` |  | `String` | `3` | `R` | ISO 3166 alpha-3 country code where the instrument trades |
| `InstrumentCurrency` | `15` |  | `String` | `3` | `R` | ISO 4217 alpha-3 currency code in which the instrument trades |
| `Price` | `31` |  | `Decimal` |  | `R` | The price of the trade |
| `Quantity` | `32` |  | `Decimal` |  | `R` | The quantity of the trade (supports fractional quantities) |
| `RegisteredRep` | `9002` |  | `String` |  | `O` | Registered rep on this trade |
| `Side` | `54` | `1` `2` `5` `6` | `Integer` | `1` | `R` | `1-Buy` `2-Sell` `5-Sell Short` `6-Sell Exempt` |
| `PositionType` | `77` | `C` `O` | `String` | `1` | `O` | `C-Close ` `O-Open` |
| `SettlementCurrency` | `120` |  | `String` | `3` | `O` | ISO 4217 alpha-3 currency code in which the instrument trades |
| `SettlementDateType` | `63` | `0` `7` | `Integer` | `1` | `R` | `0-Regular` `7-When_Issued` |
| `SettlementDate` | `64` |  | `Integer` | `8` | `CR` | Defaults to 99991231 for when issued trades and not required for when issued trades. Settlement date in `YYYYMMDD` format |
| `Solicited` | `325` | `F` `T` | `String` | `1` | `O` | `F-Solicited` `T-Not Solicited` |
| `TradeExecutionTime` | `60` |  | `UTCTimestamp` |  | `R` | Timestamp of when the trade occurred in YYYYMMDD-HH:MM:SS.sss |
| `Capacity` | `47` | `A` `M` `P` `R` | `String` | `1` | `R` | `A-Agency` `M-Mixed` `P-Principal` `R-Riskless Principal` |
| `ContraSideQualifier` | `9004` | `5` `6` | `Integer` |  | `CR` | `5-Sell Short` `6-Sell Exempt` |
| `Commission` | `12` |  | `Decimal` |  | `O` | Commission charged or paid |
| `TargetAccountID` | `79` |  | `Integer` |  | `R` | Clear Street provided account ID  |
| `OmitSECFee` | `9005` | `F` `T` | `String` | `1` | `O` | True if SEC fees should not be applied |
| `OmitTAFFee` | `9006` | `F` `T` | `String` | `1` | `O` | True if TAF fees should not be applied |
| `OrderID` | `37` |  | `String` |  | `O` | Order ID to link all the executions in the average price account |


## Away Trade

```
8=FIX.4.29=26135=849=OMS_CLIENT56=0000913234=12914552=20201021-21:42:34
20=09001=W1=10007817=CLIENT_TRADE_ID75=2020102122=448=US70450Y1038
421=USA15=USD31=000213.48000032=0000000298754=263=064=20201023
60=20201021-13:42:34.12347=M440=0295375=ABCD76=WXYZ10=180
```

```
BeginString 8=FIX.4.2
BodyLength9=261
MsgType35=8
SenderCompID49=OMS_CLIENT
TargetCompID56=CLST
MsgSeqNum34=129145
SendingTime52=20201021-21:42:34
ExecutionTransactionType20=0
TradeType9001=W
AccountID1=100078
ClientTradeID17=CLIENT_TRADE_ID
TradeDate75=20201021
InstrumentIdentifierType22=4
InstrumentIdentifier 48=US70450Y1038
InstrumentCountry421=USA
InstrumentCurrency15=USD
Price31=000213.480000
Quantity32=00000002987
Side54=2
SettlementDateType63=0
SettlmenetDate64=20201023
TradeExecutionTime60=20201021-13:42:34.123
Capacity47=M
ContraClearingNum 440=0295
ContraMPID375=ABCD
ExecutingMPID 76=WXYZ
Checksum10=180
```
This trade type represents a customer executing away from Clear Street LLC. For example, direct customer of CLST routes
order for execution to Goldman.

| Name | FIX Tag | Allowable Values | Type | Length | Required? | Description |
| - | - | - | - | - | - | - | 
| `ExecutionTransactionType` | `20` | `0` `1` | `Integer` | `1` | `R` | `0-New` `1-Cancel` |
| `TradeType` | `9001` | `W` | `String` | `1` | `R` | `W-Away` |
| `AccountID` | `1` |  | `Integer` | `6` | `R` | Clear Street provided account id |
| `BehalfOfAccountID` | `109` |  | `Integer` | `6` | `O` | Clear Street provided account ID if this trade is on behalf of another account | 
| `BranchOffice` | `9003` |  | `String` |  | `O` | Branch office for this trade |
| `TradeID` | `17` |  | `String` |  | `R` | Unique Trade ID for this trade. Must be unique across days |
| `CancelTradeID` | `9009` |  | `String` |  | `CR` | Original trade ID to cancel; Required for all Cancel trades |
| `TradeDate` | `75` |  | `Integer` | `8` | `R` | Trade Date in `YYYYMMDD` format |
| `Instrument Identifier Type` | `22` | `1` `2` `4` `8` | `Integer` | `1` | `CR` | `1-CUSIP 2-SEDOL 4-ISIN 8-TICKER` Required unless the instrument is an Option and tags 167, 55, 200, 201, 202, and 205 are provided. | `8` |
| `Instrument Identifier` | `48` |  | `String` |  | `CR` | Instrument Identifier based on tag 22 Required unless the instrument is an Option and tags 167, 55, 200, 201, 202, and 205 are provided. | `AAPL` |
| `Security Type` | `167` | | `String` | | `CR` | Indicates type of security. Required for Options where tags 22 or 48 are not provided. | `OPT` |
| `Symbol` | `55` | | `String` | `6` | `CR` | Ticker symbol. Required for Options where tags 22 or 48 are not provided. | `SPY` |
| `Maturity Month Year` | `200` | | `Integer` | `6` | `CR` | Month and year of the maturity for an Option. Maturity Month Year in format `YYYYMM`. Required for Options where tags 22 or 48 are not provided. | `202107` |
| `Put Or Call` | `201` | `0` `1` | `Integer` | `1` | `CR` | Indicates whether an Option is for a put or a call `0 = Put` `1 = Call` Required for Options where tags 22 or 48 are not provided. | `1` |
| `Strike Price` | `202` | | `Decimal` | `8` | `CR` | Strike Price for an Option. Required for Options where tags 22 or 48 are not provided. | `100.50` |
| `Maturity Day` | `205` | | `Integer` | `2` | `CR` | To be used in conjunction with Maturity Month Year `200` to specify a particular maturity date for an Option. Required for Options where tags 22 or 48 are not provided. | `30` | 
| `InstrumentCountry` | `421` |  | `String` | `3` | `R` | ISO 3166 alpha-3 country code where the instrument trades |
| `InstrumentCurrency` | `15` |  | `String` | `3` | `R` | ISO 4217 alpha-3 currency code in which the instrument trades |
| `Price` | `31` |  | `Decimal` |  | `R` | The price of the trade |
| `Quantity` | `32` |  | `Decimal` |  | `R` | The quantity of the trade (supports fractional quantities) |
| `RegisteredRep` | `9002` |  | `String` |  | `O` | Registered rep on this trade |
| `Side` | `54` | `1` `2` `5` `6` | `Integer` | `1` | `R` | `1-Buy` `2-Sell` `5-Sell Short` `6-Sell Exempt` |
| `PositionType` | `77` | `C` `O` | `String` | `1` | `O` | `C-Close` `O-Open` |
| `SettlementCurrency` | `120` |  | `String` | `3` | `O` | ISO 4217 alpha-3 currency code in which the instrument trades |
| `SettlementDateType` | `63` | `0` `7` | `Integer` | `1` | `R` | `0-Regular` `7-When_Issued` |
| `SettlementDate` | `64` |  | `Integer` | `8` | `CR` | Defaults to 99991231 for when issued trades and not required for when issued trades. Settlement date in `YYYYMMDD` format |
| `Solicited` | `325` | `F` `T` | `String` | `1` | `O` | `F-Solicited` `T-Not Solicited` |
| `TradeExecutionTime` | `60` |  | `UTCTimestamp` |  | `R` | Timestamp of when the trade occurred in YYYYMMDD-HH:MM:SS.sss |
| `Capacity` | `47` | `A` `M` `P` `R` | `String` | `1` | `R` | `A-Agency` `M-Mixed` `P-Principal` `R-Riskless Principal` |
| `ContraClearingNumber` | `440` |  | `Integer` | `4` | `O` | Contra-party's clearing number, If not supplied the value will be derived from an internal MPID to clearing number mapping |
| `ContraMPID` | `375` |  | `String` | `4` | `R` | Contra-party's MPID |
| `ExecutingMPID` | `76` |  | `String` | `4` | `R` | Executing party's MPID |
| `ContraSideQualifier` | `9004` | `5` `6` | `Integer` |  | `CR` | `5-Sell Short` `6-Sell Exempt` |
| `MIC` | `30` |  | `String` | `4` | `O` | ISO 10383 Market Identifer Code for the exchange |
| `Commission` | `12` |  | `Decimal` |  | `O` | Commission charged or paid |
| `OmitSECFee` | `9005` | `F` `T` | `String` | `1` | `O` | True if SEC fees should not be applied |
| `OmitTAFFee` | `9006` | `F` `T` | `String` | `1` | `O` | True if TAF fees should not be applied |
| `LocateID` | `9007` |  | `String` |  | `CR` | Locate ID obtained for a short sale; Required for Sell Short |
| `LocateSource` | `9008` |  | `String` |  | `CR` | Firm supplying the locate (usually MPID); Required for Sell Short |
| `OrderID` | `37` |  | `String` |  | `O` | Order ID to link all the executions in the average price account |
| `NSCCClearing` | `9010` | `agu` `contra` `corr` `corr_fees` `qsr` | `String` |  | `O` | `agu` `contra` `corr` `corr_fees` `qsr` |



## Bilateral Trade

```
8=FIX.4.29=26135=849=OMS_CLIENT56=0000913234=12914152=20201021-21:42:34
20=09001=B1=10007817=CLIENT_TRADE_ID75=2020102122=448=US70450Y1038
421=USA15=USD31=000213.48000032=0000000298754=263=064=20201023
60=20201021-13:42:34.12347=M440=0295375=ABCD76=WXYZ10=180
```

```
BeginString 8=FIX.4.2
BodyLength9=261
MsgType35=8
SenderCompID49=OMS_CLIENT
TargetCompID56=CLST
MsgSeqNum34=129145
SendingTime52=20201021-21:42:34
ExecutionTransactionType20=0
TradeType9001=B
AccountID1=100078
ClientTradeID17=CLIENT_TRADE_ID
TradeDate75=20201021
InstrumentIdentifierType22=4
InstrumentIdentifier 48=US70450Y1038
InstrumentCountry421=USA
InstrumentCurrency15=USD
Price31=000213.480000
Quantity32=00000002987
Side54=2
SettlementDateType63=0
SettlmenetDate64=20201023
TradeExecutionTime60=20201021-13:42:34.123
Capacity47=M
ContraClearingNum 440=0295
ContraMPID375=ABCD
ExecutingMPID 76=WXYZ
Checksum10=180
```

This trade represents a trade between two trading entities. For example, trading firm XYZ buys 100 share of AAPL from
trading firm ABC.

| Name | FIX Tag | Allowable Values | Type | Length | Required? | Description |
| - | - | - | - | - | - | - |
| `ExecutionTransactionType` | `20` | `0` `1` | `Integer` | `1` | `R` | `0-New` `1-Cancel` |
| `TradeType` | `9001` | `B` | `String` | `1` | `R` | `B-Bilateral` |
| `AccountID` | `1` |  | `Integer` | `6` | `R` | Clear Street provided account id |
| `BehalfOfAccountID` | `109` |  | `Integer` | `6` | `O` | Clear Street provided account ID if this trade is on behalf of another account |
| `BranchOffice` | `9003` |  | `String` |  | `O` | Branch office for this trade |
| `TradeID` | `17` |  | `String` |  | `R` | Unique Trade ID for this trade. Must be unique across days |
| `CancelTradeID` | `9009` |  | `String` |  | `CR` | Original trade ID to cancel; Required for all Cancel trades |
| `Trade Date` | `75` |  | `Integer` | `8` | `R` | Trade Date in `YYYYMMDD` format |
| `Instrument Identifier Type` | `22` | `1` `2` `4` `8` | `Integer` | `1` | `CR` | `1-CUSIP 2-SEDOL 4-ISIN 8-TICKER` Required unless the instrument is an Option and tags 167, 55, 200, 201, 202, and 205 are provided. | `8` |
| `Instrument Identifier` | `48` |  | `String` |  | `CR` | Instrument Identifier based on tag 22 Required unless the instrument is an Option and tags 167, 55, 200, 201, 202, and 205 are provided. | `AAPL` |
| `Security Type` | `167` | | `String` | | `CR` | Indicates type of security. Required for Options where tags 22 or 48 are not provided. | `OPT` |
| `Symbol` | `55` | | `String` | `6` | `CR` | Ticker symbol. Required for Options where tags 22 or 48 are not provided. | `SPY` |
| `Maturity Month Year` | `200` | | `Integer` | `6` | `CR` | Month and year of the maturity for an Option. Maturity Month Year in format `YYYYMM`. Required for Options where tags 22 or 48 are not provided. | `202107` |
| `Put Or Call` | `201` | `0` `1` | `Integer` | `1` | `CR` | Indicates whether an Option is for a put or a call `0 = Put` `1 = Call` Required for Options where tags 22 or 48 are not provided. | `1` |
| `Strike Price` | `202` | | `Decimal` | `8` | `CR` | Strike Price for an Option. Required for Options where tags 22 or 48 are not provided. | `100.50` |
| `Maturity Day` | `205` | | `Integer` | `2` | `CR` | To be used in conjunction with Maturity Month Year `200` to specify a particular maturity date for an Option. Required for Options where tags 22 or 48 are not provided. | `30` | 
| `InstrumentCountry` | `421` |  | `String` | `3` | `R` | ISO 3166 alpha-3 country code where the instrument trades |
| `InstrumentCurrency` | `15` |  | `String` | `3` | `R` | ISO 4217 alpha-3 currency code in which the instrument trades |
| `Price` | `31` |  | `Decimal` |  | `R` | The price of the trade |
| `Quantity` | `32` |  | `Decimal` |  | `R` | The quantity of the trade (supports fractional quantities) |
| `RegisteredRep` | `9002` |  | `String` |  | `O` | Registered rep on this trade |
| `Side` | `54` | `1` `2` `5` `6` | `Integer` | `1` | `R` | `1-Buy` `2-Sell` `5-Sell Short` `6-Sell Exempt` |
| `PositionType` | `77` | `C` `O` | `String` | `1` | `O` | `C-Close ` `O-Open` |
| `SettlementCurrency` | `120` |  | `String` | `3` | `O` | ISO 4217 alpha-3 currency code in which the instrument trades |
| `SettlementDateType` | `63` | `0` `7` | `Integer` | `1` | `R` | `0-Regular` `7-When_Issued` |
| `SettlementDate` | `64` |  | `Integer` | `8` | `CR` | Defaults to 99991231 for when issued trades and not required for when issued trades. Settlement date in `YYYYMMDD` format |
| `Solicited` | `325` | `F` `T` | `String` | `1` | `O` | `F-Solicited` `T-Not Solicited` |
| `TradeExecutionTime` | `60` |  | `UTCTimestamp` |  | `R` | Timestamp of when the trade occurred in YYYYMMDD-HH:MM:SS.sss |
| `Capacity` | `47` | `A` `M` `P` `R` | `String` | `1` | `R` | `A-Agency` `M-Mixed` `P-Principal` `R-Riskless Principal` |
| `ContraClearing Number` | `440` |  | `Integer` | `4` | `O` | Contra-party's clearing number, If not supplied the value will be derived from an internal MPID to clearing number mapping |
| `ContraMPID` | `375` |  | `String` | `4` | `R` | Contra-party's MPID |
| `ExecutingMPID` | `76` |  | `String` | `4` | `R` |Executing party's MPID |
| `ContraSideQualifier` | `9004` | `5` `6` | `Integer` |  | `CR` | `5-Sell Short` `6-Sell Exempt` |
| `MIC` | `30` |  | `String` | `4` | `O` | ISO 10383 Market Identifer Code for the exchange |
| `Commission` | `12` |  | `Decimal` |  | `O` | Commission charged or paid |
| `OmitSECFee` | `9005` | `F` `T` | `String` | `1` | `O` | True if SEC fees should not be applied |
| `OmitTAFFee` | `9006` | `F` `T` | `String` | `1` | `O` | True if TAF fees should not be applied |
| `LocateID` | `9007` |  | `String` |  | `CR` | Locate ID obtained for a short sale; Required for Sell Short |
| `LocateSource` | `9008` |  | `String` |  | `CR` | Firm supplying the locate (usually MPID); Required for Sell Short |
| `OrderID` | `37` |  | `String` |  | `O` | Order ID to link all the executions in the average price account |
| `NSCC Clearing` | `9010` | `agu` `contra` `corr` `corr_fees` `qsr` | `String` | `O` | `agu` `contra` `corr` `corr_fees` `qsr` |
| `LastLiquidityInd` | `851` | `1` `2` `3` `4` | `Integer` |  | `O` | `1 - Added Liquidity` `2 - Removed Liquidity` `3 - Liquidity Routed Out` `4 - Netted Liquidity` | `1` |


## Exchange Trade

```
8=FIX.4.29=25135=849=OMS_CLIENT56=0000913234=12914452=20201021-21:42:34
20=09001=E1=10007817=CLIENT_TRADE_ID75=2020102122=448=US70450Y1038
421=USA15=USD31=000213.48000032=0000000298754=263=064=20201023
60=20201021-13:42:34.12347=R76=ABCD30=NYSE10=180
```

```
BeginString 8=FIX.4.2
BodyLength9=261
MsgType35=8
SenderCompID49=OMS_CLIENT
TargetCompID56=CLST
MsgSeqNum34=129145
SendingTime52=20201021-21:42:34
ExecutionTransactionType20=0
TradeType9001=E
AccountID1=100078
ClientTradeID17=CLIENT_TRADE_ID
TradeDate75=20201021
InstrumentIdentifierType22=4
InstrumentIdentifier 48=US70450Y1038
InstrumentCountry421=USA
InstrumentCurrency15=USD
Price31=000213.480000
Quantity32=00000002987
Side54=2
SettlementDateType63=0
SettlementDate64=20201023
TradeExecutionTime60=20201021-13:42:34.123
Capacity47=R
ExecutingMPID 76=WXYZ
MIC 30=NYSE
Checksum10=180
```

This trade represents a trade between a trading entity and an exchange. For example, trading firm XYX buys 100 shares of
AAPL directly on Nasdaq

| Name | FIX Tag | Allowable Values | Type | Length | Required? | Description |
| - | - | - | - | - | - | - |
| `ExecutionTransactionType` | `20` | `0` `1` | `Integer` | `1` | `R` | `0-New` `1-Cancel` |
| `TradeType` | `9001` | `E` | `String` | `1` | `R` | `E-Exchange` |
| `AccountID` | `1` |  | `Integer` | `6` | `R` | Clear Street provided account id |
| `BehalfOfAccountID` | `109` |  | `Integer` | `6` | `O` | Clear Street provided account ID if this trade is on behalf of another account |
| `BranchOffice` | `9003` |  | `String` |  | `O` | Branch office for this trade |
| `TradeID` | `17` |  | `String` |  | `R` | Unique Trade ID for this trade. Must be unique across days |
| `CancelTradeID` | `9009` |  | `String` |  | `CR` | Original trade ID to cancel; Required for all Cancel trades |
| `TradeDate` | `75` |  | `Integer` | `8` | `R` | Trade Date in `YYYYMMDD` format |
| `Instrument Identifier Type` | `22` | `1` `2` `4` `8` | `Integer` | `1` | `CR` | `1-CUSIP 2-SEDOL 4-ISIN 8-TICKER` Required unless the instrument is an Option and tags 167, 55, 200, 201, 202, and 205 are provided. | `8` |
| `Instrument Identifier` | `48` |  | `String` |  | `CR` | Instrument Identifier based on tag 22 Required unless the instrument is an Option and tags 167, 55, 200, 201, 202, and 205 are provided. | `AAPL` |
| `Security Type` | `167` | | `String` | | `CR` | Indicates type of security. Required for Options where tags 22 or 48 are not provided. | `OPT` |
| `Symbol` | `55` | | `String` | `6` | `CR` | Ticker symbol. Required for Options where tags 22 or 48 are not provided. | `SPY` |
| `Maturity Month Year` | `200` | | `Integer` | `6` | `CR` | Month and year of the maturity for an Option. Maturity Month Year in format `YYYYMM`. Required for Options where tags 22 or 48 are not provided. | `202107` |
| `Put Or Call` | `201` | `0` `1` | `Integer` | `1` | `CR` | Indicates whether an Option is for a put or a call `0 = Put` `1 = Call` Required for Options where tags 22 or 48 are not provided. | `1` |
| `Strike Price` | `202` | | `Decimal` | `8` | `CR` | Strike Price for an Option. Required for Options where tags 22 or 48 are not provided. | `100.50` |
| `Maturity Day` | `205` | | `Integer` | `2` | `CR` | To be used in conjunction with Maturity Month Year `200` to specify a particular maturity date for an Option. Required for Options where tags 22 or 48 are not provided. | `30` | 
| `InstrumentCountry` | `421` |  | `String` | `3` | `R` | ISO 3166 alpha-3 country code where the instrument trades |
| `InstrumentCurrency` | `15` |  | `String` | `3` | `R` | ISO 4217 alpha-3 currency code in which the instrument trades |
| `Price` | `31` |  | `Decimal` |  | `R` | The price of the trade |
| `Quantity` | `32` |  | `Decimal` |  | `R` | The quantity of the trade (supports fractional quantities) |
| `RegisteredRep` | `9002` |  | `String` |  | `O` | Registered rep on this trade |
| `Side` | `54` | `1` `2` `5` `6` | `Integer` | `1` | `R` | `1-Buy` `2-Sell` `5-Sell Short` `6-Sell Exempt` |
| `PositionType` | `77` | `C` `O` | `String` | `1` | `O` | `C-Close` `O-Open` |
| `SettlementCurrency` | `120` |  | `String` | `3` | `O` | ISO 4217 alpha-3 currency code in which the instrument trades |
| `SettlementDateType` | `63` | `0` `7` | `Integer` | `1` | `R` | `0-Regular` `7-When_Issued` |
| `SettlementDate` | `64` |  | `Integer` | `8` | `CR` | Defaults to 99991231 for when issued trades and not required for when issued trades. Settlement date in `YYYYMMDD` format |
| `Solicited` | `325` | `F` `T` | `String` | `1` | `O` | `F-Solicited` `T-Not Solicited` |
| `TradeExecutionTime` | `60` |  | `UTCTimestamp` |  | `R` | Timestamp of when the trade occurred in YYYYMMDD-HH:MM:SS.sss |
| `Capacity` | `47` | `A` `M` `P` `R` | `String` | `1` | `R` | `A-Agency` `M-Mixed` `P-Principal` `R-Riskless Principal` |
| `ExecutingMPID` | `76` |  | `String` | `4` | `R` | Executing party's MPID |
| `ContraSideQualifier` | `9004` | `5` `6` | `Integer` |  | `CR` | `5-Sell Short` `6-Sell Exempt` |
| `MIC` | `30` |  | `String` | `4` | `R` | ISO 10383 Market Identifer Code for the exchange |
| `Commission` | `12` |  | `Decimal` |  | `O` | Commission charged or paid |
| `OmitSECFee` | `9005` | `F` `T` | `String` | `1` | `O` | True if SEC fees should not be applied |
| `OmitTAFFee` | `9006` | `F` `T` | `String` | `1` | `O` | True if TAF fees should not be applied |
| `LocateID` | `9007` |  | `String` |  | `CR` | Locate ID obtained for a short sale; Required for Sell Short |
| `LocateSource` | `9008` |  | `String` |  | `CR` | Firm supplying the locate (usually MPID); Required for Sell Short |
| `OrderID` | `37` |  | `String` |  | `O` | Order ID to link all the executions in the average price account |
| `LastLiquidityInd` | `851` | `1` `2` `3` `4` | `Integer` |  | `O` | `1 - Added Liquidity` `2 - Removed Liquidity` `3 - Liquidity Routed Out` `4 - Netted Liquidity` | `1` |

## Transfer Trade

```
8=FIX.4.29=25335=849=OMS_CLIENT56=0000913234=12914252=20201021-21:42:34
20=09001=T1=10007817=CLIENT_TRADE_ID75=2020102122=448=US70450Y1038
421=USA15=USD31=000213.48000032=0000000298754=263=064=20201023
60=20201021-13:42:34.12347=P76=ABCD79=10001710=180
```

```
BeginString 8=FIX.4.2
BodyLength9=261
MsgType35=8
SenderCompID49=OMS_CLIENT
TargetCompID56=CLST
MsgSeqNum34=129145
SendingTime52=20201021-21:42:34
ExecutionTransactionType20=0
TradeType9001=T
AccountID1=100078
ClientTradeID17=CLIENT_TRADE_ID
TradeDate75=20201021
InstrumentIdentifierType22=4
InstrumentIdentifier 48=US70450Y1038
InstrumentCountry421=USA
InstrumentCurrency15=USD
Price31=000213.480000
Quantity32=00000002987
Side54=2
SettlementDateType63=0
SettlmenetDate64=20201023
TradeExecutionTime60=20201021-13:42:34.123
Capacity47=P
ExecutingMPID 76=WXYZ
TargetAccountID 79=100017
Checksum10=180
```

This trade type is to facilitate trade movement between Clear Street internal accounts. For example, trade movement from a proprietary account to an average price account.

| Name | FIX Tag | Allowable Values | Type | Length | Required? | Description |
| - | - | - | - | - | - | - |
| `ExecutionTransactionType` | `20` | `0` `1` | `Integer` | `1` | `R` | `0-New` `1-Cancel` |
| `TradeType` | `9001` | `T` | `String` | `1` | `R` | `T-Transfer` |
| `AccountID` | `1` |  | `Integer` | `6` | `R` | Clear Street provided account id |
| `BehalfOfAccountID` | `109` |  | `Integer` | `6` | `O` | Clear Street provided account ID if this trade is on behalf of another account |
| `BranchOffice` | `9003` |  | `String` |  | `O` | Branch office for this trade |
| `TradeID` | `17` |  | `String` |  | `R` | Unique Trade ID for this trade. Must be unique across days |
| `CancelTradeID` | `9009` |  | `String` |  | `CR` | Original trade ID to cancel; Required for all Cancel trades |
| `TradeDate` | `75` |  | `Integer` | `8` | `R` | Trade Date in `YYYYMMDD` format |
| `Instrument Identifier Type` | `22` | `1` `2` `4` `8` | `Integer` | `1` | `CR` | `1-CUSIP 2-SEDOL 4-ISIN 8-TICKER` Required unless the instrument is an Option and tags 167, 55, 200, 201, 202, and 205 are provided. | `8` |
| `Instrument Identifier` | `48` |  | `String` |  | `CR` | Instrument Identifier based on tag 22 Required unless the instrument is an Option and tags 167, 55, 200, 201, 202, and 205 are provided. | `AAPL` |
| `Security Type` | `167` | | `String` | | `CR` | Indicates type of security. Required for Options where tags 22 or 48 are not provided. | `OPT` |
| `Symbol` | `55` | | `String` | `6` | `CR` | Ticker symbol. Required for Options where tags 22 or 48 are not provided. | `SPY` |
| `Maturity Month Year` | `200` | | `Integer` | `6` | `CR` | Month and year of the maturity for an Option. Maturity Month Year in format `YYYYMM`. Required for Options where tags 22 or 48 are not provided. | `202107` |
| `Put Or Call` | `201` | `0` `1` | `Integer` | `1` | `CR` | Indicates whether an Option is for a put or a call `0 = Put` `1 = Call` Required for Options where tags 22 or 48 are not provided. | `1` |
| `Strike Price` | `202` | | `Decimal` | `8` | `CR` | Strike Price for an Option. Required for Options where tags 22 or 48 are not provided. | `100.50` |
| `Maturity Day` | `205` | | `Integer` | `2` | `CR` | To be used in conjunction with Maturity Month Year `200` to specify a particular maturity date for an Option. Required for Options where tags 22 or 48 are not provided. | `30` | 
| `InstrumentCountry` | `421` |  | `String` | `3` | `R` | ISO 3166 alpha-3 country code where the instrument trades |
| `InstrumentCurrency` | `15` |  | `String` | `3` | `R` | ISO 4217 alpha-3 currency code in which the instrument trades |
| `Price` | `31` |  | `Decimal` |  | `R` | The price of the trade |
| `Quantity` | `32` |  | `Decimal` |  | `R` | The quantity of the trade (supports fractional quantities) |
| `RegisteredRep` | `9002` |  | `String` |  | `O` | Registered rep on this trade |
| `Side` | `54` | `1` `2` `5` `6` | `Integer` | `1` | `R` | `1-Buy` `2-Sell` `5-Sell Short` `6-Sell Exempt` |
| `PositionType` | `77` | `C` `O` | `String` | `1` | `O` | `C-Close` `O-Open` |
| `SettlementCurrency` | `120` |  | `String` | `3` | `O` | ISO 4217 alpha-3 currency code in which the instrument trades |
| `SettlementDateType` | `63` | `0` `7` | `Integer` | `1` | `R` | `0-Regular` `7-When_Issued` |
| `SettlementDate` | `64` |  | `Integer` | `8` | `CR` | Defaults to 99991231 for when issued trades and not required for when issued trades. Settlement date in `YYYYMMDD` format |
| `Solicited` | `325` | `F` `T` | `String` | `1` | `O` | `F-Solicited` `T-Not Solicited` |
| `TradeExecutionTime` | `60` |  | `UTCTimestamp` |  | `R` | Timestamp of when the trade occurred in YYYYMMDD-HH:MM:SS.sss |
| `Capacity` | `47` | `A` `M` `P` `R` | `String` | `1` | `R` | `A-Agency` `M-Mixed` `P-Principal` `R-Riskless Principal` |
| `ContraSideQualifier` | `9004` | `5` `6` | `Integer` |  | `CR` | `5-Sell Short` `6-Sell Exempt` |
| `Commission` | `12` |  | `Decimal` |  | `O` | Commission charged or paid |
| `TargetAccountID` | `79` |  | `Integer` |  | `R` | Clear Street provided account ID  |
| `OmitSECFee` | `9005` | `F` `T` | `String` | `1` | `O` | True if SEC fees should not be applied |
| `OmitTAFFee` | `9006` | `F` `T` | `String` | `1` | `O` | True if TAF fees should not be applied |


# FIX Outbound (Response) Specification

| Name             | FIX Tag | Allowable Values | Type       | Description                                 | Example           |
| ---------------- | ------- | ---------------- | ---------- | ------------------------------------------- | ----------------- |
| `Trade Details` |  |  |  | FIX message received from OMS |  |
| `Response` | `9011` |  | `String` | ACK or NACK with reason | `accepted` |
