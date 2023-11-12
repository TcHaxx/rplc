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

See example projects `RpcServer` and `RpcClient` or unit-tests in `TESTs/FB_RPC_Tests` for more examples.

> **NOTE**  
> Unlike the [TwinCAT ADS .NET library](https://infosys.beckhoff.com/english.php?content=../content/1033/tc3_adsnetref/7313454219.html), this implementation does not check the datatypes of the arguments before invoking.  
> It only matches the total size of all arguments (inputs, var-in-out and return value).

# Acknowledgments

* [TcUnit](https://github.com/tcunit/TcUnit)  
  An unit testing framework for Beckhoff's TwinCAT 3
