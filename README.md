[![CI/CD](https://github.com/TcHaxx/rplc/actions/workflows/cicd.yml/badge.svg?branch=main)](https://github.com/TcHaxx/rplc/actions/workflows/cicd.yml)
# rplc
Invoke `RPC` methods from `TwinCAT PLC` using `ADS`.

# How?

## Prerequisites
A RPC server (here another PLC on same machine `LocalAmsNetId`, but different ADS Port `852`) provides following method signature:
> **NOTE**  
> The method has to have the attribute `TcRpcEnable` in order to be invocable via RPC,  
> see [Beckhoff InfoSys - Attribute 'TcRpcEnable'](https://infosys.beckhoff.com/english.php?content=../content/1033/tc3_plc_intro/7145472907.html)
```
{attribute 'TcRpcEnable'}
METHOD Greet : T_MaxString
VAR_INPUT
	sName : T_MaxString;
END_VAR
VAR_OUTPUT
	nLen : INT;
END_VAR
VAR_IN_OUT CONSTANT
	stLibVersion : ST_LibVersion;
END_VAR
```

## Instantiate client

```
fbRpc : tchaxx.FB_RPC(sAmsNetId:= tchaxx.GCL.cLocalAmsNetId, 
                      nAmsPort:= 852, 
                      sInstancePath:= 'MAIN.fbRpcServer', 
                      sSymbol:= 'Greet');

// Arguments and RetVal
sArg  : T_MaxString := _AppInfo.AppName;
nLen  : INT;
stLibVersion : ST_LibVersion;
sRetVal : T_MaxString;
stRetVal : ST_AdsStatus;
```

## Invoke RPC method

To invoke the RPC method, simply use fluent-api to describe it, e.g.
```
stRetVal := fbRpc.VarInput(sArg)
                 .VarOutput(nLen)
                 .VarInOut(stLibVersion)
                 .ReturnValue(sRetVal)
                 .Invoke();
```

> **NOTE**  
> Unlike the [TwinCAT ADS .NET library](https://infosys.beckhoff.com/english.php?content=../content/1033/tc3_adsnetref/7313454219.html), this implementation does not check the datatypes of the arguments before invoking.  
> It only matches the total size of all arguments (inputs, var-in-out and return value).

## Examples
See example projects `RpcServer` and `RpcClient` or unit-tests in `TESTs/FB_RPC_Tests` for more examples.

### Publisher / Subscriber example
This examples in `RpcServer`/`RpcClient` project, showcases the use of `FB_RPC_Dynamic`.
* `FB_Publisher` in `RpcServer`
  
  A client can subscribe to events, by invoking `Subscribe` method, with following arguments:  
  ```
  VAR_INPUT CONSTANT
	  sAmsNetId		: T_AmsNetID;
	  nPort			: T_AmsPort;
	  sInstancePath	: STRING;
	  sSymbol			: STRING;
  END_VAR
  ```
  These arguments are used to configure a `FB_RPC_Dynamic` instance in the array `aSubscribers` of type `ST_Subscriber`.
  ```
  Subscribe := S_PENDING;

  (* start of critical section *)
  IF fbCrititcalSection.Enter() THEN
      
    FOR i := 0 TO cSubsUpperBound DO	
      IF aSubscribers[i].nHandle > 0 THEN
        CONTINUE;
      END_IF
      nHandleCnt := nHandleCnt + 1;
      aSubscribers[i].nHandle := nHandleCnt;		
      aSubscribers[i].fbRPC.Configure().WithAmsNetId(sAmsNetId).WithAmsPort(nPort).WithSymbolPath(sInstancePath, sSymbol).Build();
      nHandle := nHandleCnt;
      Subscribe := S_OK;
      EXIT;
    END_FOR
    
      (* end of critical section *)
    fbCrititcalSection.Leave();
  END_IF
  ```
  The publisher invokes each registered subscriber:
  ```
  METHOD PRIVATE InvokeSubscribersCallback
  VAR_INPUT
    nTrigger : UDINT;
  END_VAR

  VAR
    i 	: UDINT;
    refSub : REFERENCE TO ST_Subscriber;
  END_VAR
  ```
  ```
  FOR i:= 0 TO cSubsUpperBound DO
	refSub REF= aSubscribers[i];
	
	IF refSub.nHandle = 0 THEN
		CONTINUE;
	END_IF
	
	refSub.stAdsStatus := refSub.fbRPC.VarInput(nTrigger).NoReturnValue().Invoke();

  END_FOR
  ```
* `FB_Subscriber` in `RpcClient`  
  In order to subscribe to events, a client must register with the publisher, which in this case is the `FB_Publisher` of the `RpcServer`:
  ```
  VAR
    fbRpcPublisherSubscribe 	: tchaxx.FB_RPC(sAmsNetId:= tchaxx.GCL.cLocalAmsNetId, nAmsPort:= 852, sInstancePath:= 'MAIN.fbPublisher', sSymbol:= 'Subscribe');
  END_VAR
  ```
  With the variable `sSymbolCallback` "pointing" to the callback-method `PublisherEventCallback`:
  ```
  VAR
    sSymbolCallback			: STRING := 'PublisherEventCallback';
  END_VAR
  ```
  This method is invoked by the publisher, in case of an event.
  ```
  IF NOT bSubscribed AND bSubscribe THEN
    stAdsStatus := fbRpcPublisherSubscribe.VarInput(anyArg:= sAmsNetId)
            .VarInput(anyArg:= _AppInfo.AdsPort)
            .VarInput(anyArg:= sInstancePath)
            .VarInput(anyArg:= sSymbolCallback)
            .VarOutput(anyArg:= nHandle)
            .ReturnValue(anyRetVal:= hr)
            .Invoke();
            
    bSubscribed := NOT stAdsStatus.bBusy AND NOT stAdsStatus.bError;
    IF bSubscribed THEN
      bSubscribe := FALSE;
    END_IF
  END_IF;
  ```

### Call external RPC methods

* See [TcHaxx/Snappy](https://github.com/tchaxx/snappy).

# Acknowledgments

* [TcUnit](https://github.com/tcunit/TcUnit)  
  An unit testing framework for Beckhoff's TwinCAT 3
