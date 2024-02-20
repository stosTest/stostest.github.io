---
title: Understanding the Inner Workings of Cheat Engine's encodeFunction()
date: "2024-02-20"
---

Some cheat table authors would like to hide the key part of their script, so CE provides some helper functions. This blog post analyzes the most commonly used one: encodeFunction(), and eventually breaks it.

# Background

In https://forum.cheatengine.org/viewtopic.php?t=595142, Dark Byte says:
> I personally find it silly that people spend so much time hiding their scripts, but since CE 6.6 you can encode functions and decode them in a way that doesn't convert them to pure text inbetween.

Here is an example from him:
```lua
function myscript()
 print('hello')
end

var c_myscript = encodeFunction(myscript)
ASSERT: c_myscript == "c-oWpDNPJ!ketlRCB=/U!NS2(5ypT38s!d+42)bqGnmW70wmZN92guO7#LI;7#P)U8W?.;Vk}S0MVWaeENmI6IXU?@4A:kwWpWC7"

myscript_clone = decodeFunction(c_myscript)
myscript_clone()
ASSERT: "hello" printed
```

# Checking output and Guessing data flow

The output of `encodeFunction()` looks like Base64 encoded strings with some distortion. Trying to base64_decode it fails because of the strange characters.

We know that base64 has variants like base64url, base32, base58, base85, etc., with shorter or longer encoding tables. A hypothesis is that one of these variants is used.

What we have now:

Guessed encodeFunction() data flow: `myfunction --> some internal binary representation --> (baseXX_encode) --> c_myfunction`

# Digging into the source code

Luckily, Cheat Engine is an open-source project. After googling, the repo is located at https://github.dev/cheat-engine/cheat-engine/tree/master/Cheat%20Engine, we could use Github's web editor to search in the repo for `encodeFunction` implementation.

It is located in LuaHandler.pas, and the implementation is surprisingly simple. Here is the content. Cheat Engine is written mainly in Object Pascal, a very old PL once used as a tutorial language; some NOIP students might be familiar with it.

```pascal
function lua_encodeFunction(L: Plua_State): integer; cdecl;
var
  s: TMemoryStream;
  cs: Tcompressionstream;
  i: integer;

  output: pchar;

  new: pchar;
begin
  s:=TMemoryStream.Create;
  cs:=Tcompressionstream.create(clmax, s);


  if (lua_gettop(L)=1) and (lua_isfunction(L, 1)) then
    lua_dump(L, @lwriter, cs, 1);

  cs.free;

  getmem(output, (s.size div 4) * 5 + 5 );
  BinToBase85(pchar(s.Memory), output, s.size);

  lua_pushstring(L, output);
  FreeMemAndNil(output);

  s.free;

  result:=1;
end;
```

So the dataflow is `myfunction --> lua_dump --> BinToBase85 --> c_myfunction`.

We then check the implementation of BinToBase85:

```pascal
procedure BinToBase85(BinValue, outputStringBase85: PChar; BinBufSize: integer);
//
//  example:
//    getmem(outputstring, (BinarySize div 4) * 5 + 5 );
//    BinToBase85(b,outputstring,BinarySize);
//
//  it adds a 0 terminator to outputstring

function Base85ToBin(inputStringBase85, BinValue: PChar): integer;
//
//  example:
//    size:=length(inputstring);
//    if (size mod 5) > 1 then
//      BinarySize:= (size div 5) * 4 + (size mod 5) - 1
//    else
//      BinarySize:= (size div 5) * 4;
//   getmem(b, BinarySize);
//   Base85ToBin(inputstring, b);
//
//  Base85ToBin doesn't support: white space between the characters, line breaks. (HexToBin doesn't support it too)
//  Base85ToBin doesn't check for data corruption.

implementation

const
  customBase85='0123456789'+
                'ABCDEFGHIJKLMNOPQRSTUVWXYZ'+
                'abcdefghijklmnopqrstuvwxyz'+
                '!#$%()*+,-./:;=?@[]^_{}';

procedure BinToBase85(BinValue, outputStringBase85: PChar; BinBufSize: integer);
var
  i : integer;
  a : dword;
begin

  i:=0;
  while i<binbufsize do
  begin

    //first byte ( from 4-tuple )
    a:=pbyte(BinValue+i)^ shl 24;
    inc(i);

    if i<binbufsize then
    begin
      //second byte
      a:=a or pbyte(BinValue+i)^ shl 16;
      inc(i);
    end;

    if i<binbufsize then
    begin
      //third byte
      a:=a or pbyte(BinValue+i)^ shl 8;
      inc(i);
    end;

    if i<binbufsize then
    begin
      //fourth byte
      a:=a or pbyte(BinValue+i)^;
      inc(i);
    end;

    outputStringBase85[4]:= customBase85[a mod 85 + 1]; a:= a div 85;
    outputStringBase85[3]:= customBase85[a mod 85 + 1]; a:= a div 85;
    outputStringBase85[2]:= customBase85[a mod 85 + 1]; a:= a div 85;
    outputStringBase85[1]:= customBase85[a mod 85 + 1]; a:= a div 85;
    outputStringBase85[0]:= customBase85[a mod 85 + 1];
    inc(outputStringBase85,5);

  end;

  //add zero terminator at right place
  a:= (4 - (BinBufSize mod 4)) mod 4;
  dec(outputStringBase85,a);
  outputStringBase85[0]:=#0;
end;
```

It is a modified version of Base85, with a custom character table. Now we have all the information needed to decrypt `encodeFunction()`.

# Reversed data flow

Let me copy the data flow we observed here: 

`myfunction --> lua_dump --> BinToBase85 --> c_myfunction`

We could easily revert the procedure after lua_dump. However, lua_dump produces compiled binary flow, so we need to decompile it. 

Luckily again, Lua is widely used and someone generously writes a decompiler, see https://github.com/viruscamp/luadec

Our workflow would be rather simple:
`c_myfunction --> Base85ToBin --> Writes to a file --> luadec it --> original lua code produced`. I personally prefer to implement a python version of CE's custom Base85ToBin, and use a python script to manage the workflow.

# Conclusion

Someone in CE forum states that,

>encodeFunction only does a simple surface-level encoding on the data. This is easily undone to get the original data back.
>
>lua_encodeFunctionEx does a more in depth means of encoding as it does a few things.
>
>1. It will encode the Lua data down to Lua byte code.
>2. It will encode the data with base85 as well.
>3. It can use an external Lua DLL to do custom byte code compiling.
>
>However, this is still also pretty easily reversed back to normal Lua if the person decompiling your stuff knows what they are doing and how to understand Lua byte code. (Fixing an existing byte code decompiler for shifted/altered opcodes is not hard to make work for custom modifications or just shifted opcode ids which most people do.)
>
>Then there is also the enableDrm feature, I won't go into detail on what this does or how it works, but will mention it is extremely easy to bypass and not really a reliable means of protection either.
>
>If your goal is to protect your work, I don't recommend you rely on CE's built-in trainer maker to do so. Everything you create can be easily dumped/reversed back to its original data fairly easy, even by novices. If you do plan to still use it, then I'd recommend you dig into how encodeFunctionEx works and learn how to understand Lua's byte code, and how to alter it to make use of your own Lua DLL. Modify the bytecode more than just shifting the ids around, add junk code, add cflow obfuscation, etc. and so on if it means that much to you to protect things that much and still use CE's trainer maker.

CE scripts written in lua, interpreted rather than compiled, have a fragile nature when people try to hide or encrypt them. To name a few, enthusiasts may dump the content from CE process.

Maybe stronger protection could be applied at another level, like hiding trainer logic in a fortified executable and base64 packing it into the cheat script. It could communicate with the script to leverage the helper functions and abilities provided by CE.