Google Translate version :)) 
Old version of V8

"Chrome V8 Source Code" 33. Technical details of Lazy Compile


# 1. Summary
This article is the eighth in the Builtin topic.  This article will track the execution process of Bytecode, explain the startup method, workflow, and important data structures of Lazy Compile, and also introduce Builtins related to Lazy Compile.

# 2. Starting Lazy Compile
Before entering Lazy Compile, you must first understand the Bytecode execution process, and use this process to understand how to start Lazy Compile.  The source code is as follows:
```cpp
```text
1.  function ignition(s) {
2.      this.slogan=s;
3.      this.start=function(){eval('console.log(this.slogan);')}
4.  }
5.  worker = new ignition("here we go!");
6.  worker.start();
7.  //............分隔线................
8.  --- AST ---
9.  . . FUNCTION "ignition" = function ignition
10.   . EXPRESSION STATEMENT at 106
11.   . . ASSIGN at 113
12.   . . . VAR PROXY unallocated (0000016B96A17A78) (mode = DYNAMIC_GLOBAL, assigned = true) "worker"
13.   . . . CALL NEW at 115
14.   . . . . VAR PROXY unallocated (0000016B96A17790) (mode = VAR, assigned = true) "ignition"
15.   . . . . LITERAL "here we go!"//...........省略..............
16.   //...............分隔线.................
17.   0000025885361EAE @    0 : 13 00             LdaConstant [0]
18.   0000025885361EB0 @    2 : c2                Star1
19.   0000025885361EB1 @    3 : 19 fe f8          Mov <closure>, r2
20.   0000025885361EB4 @    6 : 64 51 01 f9 02    CallRuntime [DeclareGlobals], r1-r2
21.   0000025885361EB9 @   11 : 21 01 00          LdaGlobal [1], [0]
22.   0000025885361EBC @   14 : c2                Star1
23.   0000025885361EBD @   15 : 13 02             LdaConstant [2]
24.   0000025885361EBF @   17 : c1                Star2
25.   0000025885361EC0 @   18 : 0b f9             Ldar r1
26.   0000025885361EC2 @   20 : 68 f9 f8 01 02    Construct r1, r2-r2, [2]
27.   0000025885361EC7 @   25 : 23 03 04          StaGlobal [3], [4]
28.   0000025885361ECA @   28 : 21 03 06          LdaGlobal [3], [6]
29.   0000025885361ECD @   31 : c1                Star2
30.   0000025885361ECE @   32 : 2d f8 04 08       LdaNamedProperty r2, [4], [8]
31.   0000025885361ED2 @   36 : c2                Star1
32.   0000025885361ED3 @   37 : 5c f9 f8 0a       CallProperty0 r1, r2, [10]
33.   0000025885361ED7 @   41 : c3                Star0
34.   0000025885361ED8 @   42 : a8                Return
35.  //..............省略................
36.  - length: 5
37.          0: 0x02841b4e1d31 <FixedArray[2]>
38.          1: 0x02841b4e1c09 <String[8]: #ignition>
39.          2: 0x02841b4e1c51 <String[11]: #here we go!>
40.          3: 0x02841b4e1c39 <String[6]: #worker>
41.          4: 0x02841b4e1c71 <String[5]: #start>
``````

The above code is divided into three parts. 
The first part (lines 1-6) is the test code used in this article, where line 5 starts Lazy Compile; 
the second part (lines 8-15) is the AST of the test code;  
the third part (lines 17-41) is the Bytecode of the test code. 

Let's start with the Bytecode: (1) LdaGlobal [1], [0] (line 21) uses the string in the constant pool [1] as the key to obtain the global object, that is, to obtain the ignition function; 
Star1 (line 22) stores ignition in r1; 
Ldar r1 (line 25) takes ignition from r1 and stores it in the accumulator register; 
(2) LdaConstant [2] (line 23) and Star2 (line 24) store the string "here we go!" in r2. Construct r1, r2-r2, [2] (26 lines) The Compiler is started when constructing the ignition function. The source code is as follows:

```cpp
```text
1.  RUNTIME_FUNCTION(Runtime_NewObject) {
2.    HandleScope scope(isolate);
3.    DCHECK_EQ(2, args.length());
4.    CONVERT_ARG_HANDLE_CHECKED(JSFunction, target, 0);
5.    CONVERT_ARG_HANDLE_CHECKED(JSReceiver, new_target, 1);
6.    RETURN_RESULT_OR_FAILURE(
7.        isolate,
8.        JSObject::New(target, new_target, Handle<AllocationSite>::null()));
9.  }
10.  //.............分隔线...............
11.  int JSFunction::CalculateExpectedNofProperties(Isolate* isolate,
12.                                                 Handle<JSFunction> function) {
13.    int expected_nof_properties = 0;
14.    for (PrototypeIterator iter(isolate, function, kStartAtReceiver);
15.         !iter.IsAtEnd(); iter.Advance()) {
16.      Handle<JSReceiver> current =
17.          PrototypeIterator::GetCurrent<JSReceiver>(iter);
18.      if (!current->IsJSFunction()) break;
19.      Handle<JSFunction> func = Handle<JSFunction>::cast(current);
20.      // The super constructor should be compiled for the number of expected
21.      // properties to be available.
22.      Handle<SharedFunctionInfo> shared(func->shared(), isolate);
23.      IsCompiledScope is_compiled_scope(shared->is_compiled_scope(isolate));
24.      if (is_compiled_scope.is_compiled() ||
25.          Compiler::Compile(isolate, func, Compiler::CLEAR_EXCEPTION,
26.                            &is_compiled_scope)) {
27.      } else {
28.      }
29.    }
30.  }
``````

The above code is divided into two parts. New() (line 8) in Runtime_NewObject creates a new object, that is, creates the ignition function. The second part of the code (lines 11-30) is called in New(). When line 24 calculates the properties of ignition, the Compiler is started to generate and execute bytecode. The source code is as follows:

```
00000258853621BE @    0 : 82 00 04          CreateFunctionContext [0], [4]
         00000258853621C1 @    3 : 1a f9             PushContext r1
//...省略............
         00000258853621E7 @   41 : a8                Return
```

The Compiler will not be started when the above code is executed, so the Return instruction will return to the test code and execute line 32 CallProperty0 r1, r2, [10]. The source code is as follows:


```cpp
1.  IGNITION_HANDLER(CallProperty0, InterpreterJSCallAssembler) {
2.    JSCallN(0, ConvertReceiverMode::kNotNullOrUndefined);
3.  }
4.  //.............分隔线......................
5.    void JSCallN(int arg_count, ConvertReceiverMode receiver_mode) {
6.      Comment("sea node1");
7.      const int kFirstArgumentOperandIndex = 1;
8.      const int kReceiverOperandCount = (receiver_mode == ConvertReceiverMode::kNullOrUndefined) ? 0 : 1;
9.      const int kReceiverAndArgOperandCount = kReceiverOperandCount + arg_count;
10.      const int kSlotOperandIndex = kFirstArgumentOperandIndex + kReceiverAndArgOperandCount;
11.      TNode<Object> function = LoadRegisterAtOperandIndex(0);
12.      LazyNode<Object> receiver = [=] {return receiver_mode == ConvertReceiverMode::kNullOrUndefined
13.                   ? UndefinedConstant() : LoadRegisterAtOperandIndex(1); };
14.      TNode<UintPtrT> slot_id = BytecodeOperandIdx(kSlotOperandIndex);
15.      TNode<HeapObject> maybe_feedback_vector = LoadFeedbackVector();
16.      TNode<Context> context = GetContext();
17.      CollectCallFeedback(function, receiver, context, maybe_feedback_vector,
18.                          slot_id);
19.      switch (kReceiverAndArgOperandCount) {
20.        case 0:
21.          CallJSAndDispatch(function, context, Int32Constant(arg_count),
22.                            receiver_mode);
23.          break;
24.        case 1:
25.          CallJSAndDispatch(
26.              function, context, Int32Constant(arg_count), receiver_mode,
27.              LoadRegisterAtOperandIndex(kFirstArgumentOperandIndex));
28.          break;//....省略.......
29.        default:
30.          UNREACHABLE();
31.      }
32.    }
33.  };
```

In the above code, the value of register r1 is JSFunction start, and the value of register r2 is ignition map.  Line 2 calls JSCallN(); line 9 calls kReceiverAndArgOperandCount to 2; line 11 calls function to JSFunction start; line 25 calls CallJSAndDIspatch(), which uses TailCallN() to complete the function call and finally enters Lazy Compile.  Figure 1 shows the call stack at this time.

![enter image description here](https://pica.zhimg.com/v2-0faaa1cc4a827287242880f78bc17ad6_1440w.jpg)

# 3. Lazy Compile
The way to start Lazy Compile in the test code is Runtime. The source code is as follows:
```cpp
1.  RUNTIME_FUNCTION(Runtime_CompileLazy) {
2.    HandleScope scope(isolate);
3.    DCHECK_EQ(1, args.length());
4.    CONVERT_ARG_HANDLE_CHECKED(JSFunction, function, 0);
5.    Handle<SharedFunctionInfo> sfi(function->shared(), isolate);
6.  #ifdef DEBUG
7.    if (FLAG_trace_lazy && !sfi->is_compiled()) {
8.      PrintF("[unoptimized: ");
9.      function->PrintName();
10.      PrintF("]\n");
11.    }
12.  #endif
13.    StackLimitCheck check(isolate);
14.    if (check.JsHasOverflowed(kStackSpaceRequiredForCompilation * KB)) {
15.      return isolate->StackOverflow();
16.    }
17.    IsCompiledScope is_compiled_scope;
18.    if (!Compiler::Compile(isolate, function, Compiler::KEEP_EXCEPTION,
19.                           &is_compiled_scope)) {
20.      return ReadOnlyRoots(isolate).exception();
21.    }
22.    DCHECK(function->is_compiled());
23.    return function->code();
24.  }
```

The value of function in line 3 of the above code is JSFunction start; line 18 starts the compilation process. The source code is as follows:
```cpp
1.  bool Compiler::Compile(...省略....) {
2.   Handle<Script> script(Script::cast(shared_info->script()), isolate);
3.   UnoptimizedCompileFlags flags =
4.       UnoptimizedCompileFlags::ForFunctionCompile(isolate, *shared_info);
5.   UnoptimizedCompileState compile_state(isolate);
6.   ParseInfo parse_info(isolate, flags, &compile_state);
7.   LazyCompileDispatcher* dispatcher = isolate->lazy_compile_dispatcher();
8.    if (dispatcher->IsEnqueued(shared_info)) {
9.    }
10.    if (shared_info->HasUncompiledDataWithPreparseData()) {
11.    }
12.    if (!parsing::ParseAny(&parse_info, shared_info, isolate,
13.                           parsing::ReportStatisticsMode::kYes)) {
14.      return FailWithPendingException(isolate, script, &parse_info, flag);
15.    }//..........省略........
16.    FinalizeUnoptimizedCompilationDataList
17.        finalize_unoptimized_compilation_data_list;
18.    if (!IterativelyExecuteAndFinalizeUnoptimizedCompilationJobs(
19.            isolate, shared_info, script, &parse_info, isolate->allocator(),
20.            is_compiled_scope, &finalize_unoptimized_compilation_data_list,
21.            nullptr)) {
22.      return FailWithPendingException(isolate, script, &parse_info, flag);
23.    }
24.    FinalizeUnoptimizedCompilation(isolate, script, flags, &compile_state,
25.                                   finalize_unoptimized_compilation_data_list);
26.    if (FLAG_always_sparkplug) {
27.      CompileAllWithBaseline(isolate, finalize_unoptimized_compilation_data_list);
28.    }
29.    return true;
30.  }
```

The above code is consistent with the compilation process described above, please analyze it yourself.  Note: Line 27 is the new compilation component added by V8, which is located between Ignition and Turbofan.  Figure 2 shows the call stack at this time.
![enter image description here](https://pic2.zhimg.com/v2-efe9f8970088437a76c5a8616692e30f_1440w.jpg)

Technical summary (1) This article involves two compiles, one for calculating object properties and the other for Lazy Compile; (2) TailCallN() is used to add a Node to the end of the current Block and complete the function call, see sea of ​​nodes for details.
