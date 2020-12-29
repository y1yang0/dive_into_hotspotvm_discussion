# 线程执行状态切换(VM&lt;-&gt;Java&lt;-&gt;Native)

<a name="c2N1T"></a>
## 1.ThreadInVMfromJava
<a name="nLp2P"></a>
### 1.1 Java->VM
假如解释器一些代码比较复杂，或者因为其他原因，需要C++的支持，那么他会jmp到interpreterRuntime。比如anewarray创建Object[]，这个字节码就会从jit code跳到InterpreterRuntime::anewarray。这个函数有一个特别的地方是它用JRT_ENTRY和JRT_END包裹着，这个宏展开是一个ThreadInVMfromJava结构。
```cpp
JRT_ENTRY(void, InterpreterRuntime::anewarray(JavaThread* thread, ConstantPool* pool, int index, jint size))
  ...
JRT_END

#define JRT_ENTRY(result_type, header)                               \
  result_type header {                                               \
    ThreadInVMfromJava __tiv(thread);                                \
    VM_ENTRY_BASE(result_type, header, thread)                       \
    debug_only(VMEntryWrapper __vew;)
```
ThreadInVMfromJava是指线程从Java代码部分走到了VM代码部分，虚拟机精确的知道当前线程在执行什么代码：
```cpp
class ThreadInVMfromJava : public ThreadStateTransition {
 public:
  ThreadInVMfromJava(JavaThread* thread) : ThreadStateTransition(thread) {
    // 下面其实就是thread->set_thread_state(_thread_in_vm)
    // 给线程设置一个状态_thread_in_vm
    trans_from_java(_thread_in_vm);
  }
  ~ThreadInVMfromJava()  {
    if (_thread->stack_overflow_state()->stack_yellow_reserved_zone_disabled()) {
      _thread->stack_overflow_state()->enable_stack_yellow_reserved_zone();
    }
    trans(_thread_in_vm, _thread_in_Java);
    // Check for pending. async. exceptions or suspends.
    if (_thread->has_special_runtime_exit_condition()) _thread->handle_special_runtime_exit_condition();
  }
};
```
ThreadInVMfromJava的构造函数——相当于进入从Java代码进入VM代码的InterpreterRuntime::anewarray——只是简单的给线程设置一个状态，但是它的析构函数——相当于从VM代码的InterpreterRuntime::anewarray进入Java代码——却稍微复杂一些。
<a name="27nq3"></a>
### 1.2 Java->VM->Java
析构函数里面有一个trans call：
```cpp
static inline void transition(JavaThread *thread, JavaThreadState from, JavaThreadState to) {
    ... // some asserts
    thread->set_thread_state((JavaThreadState)(from + 1));

    InterfaceSupport::serialize_thread_state(thread);

    SafepointMechanism::block_if_requested(thread);
    thread->set_thread_state(to);

    CHECK_UNHANDLED_OOPS_ONLY(thread->clear_unhandled_oops();)
  }
```
这个trans大概做了三件事情：

1. 将线程状态设置为_thread_in_vm_trans ，表示线程正处于vm转移到其他状态这个过程中
1. 检查安全点
1. 将线程状态设置为_thread_in_Java，表示线程进入Java代码。


<br />最重要的是处理safepoint:
```cpp
void SafepointSynchronize::block(JavaThread *thread) {
  ...
  JavaThreadState state = thread->thread_state();
  switch(state) {
    case _thread_in_vm_trans:
    case _thread_in_Java:        // From compiled code
      // We are highly likely to block on the Safepoint_lock. In order to avoid blocking in this case,
      // we pretend we are still in the VM.
      thread->set_thread_state(_thread_in_vm);

      // 如果正在开启安全点中，将_waiting_to_block--
      // VMThread先拿到safepoint lock再修改——waiting_to_block，所以这里也需要拿到锁再改
      if (is_synchronizing()) {
         Atomic::inc (&TryingToBlock) ;
      }

      Safepoint_lock->lock_without_safepoint_check();
      if (is_synchronizing()) {
        assert(_waiting_to_block > 0, "sanity check");
        _waiting_to_block--;
        thread->safepoint_state()->set_has_called_back(true);

        if (thread->in_critical()) {
          increment_jni_active_count();
        }

        if (_waiting_to_block == 0) {
          Safepoint_lock->notify_all();
        }
      }

      // 将线程状态设置为_thread_blocked
      thread->set_thread_state(_thread_blocked);
      Safepoint_lock->unlock();
      // 因为Theads_lock已经被VMThread线程拿了，所以当前线程走到这里就会阻塞。
      Threads_lock->lock_without_safepoint_check();
      // 恢复状态为_thread_in_vm_trans
      thread->set_thread_state(state);
      Threads_lock->unlock();
      break;

    case _thread_in_native_trans:
    case _thread_blocked_trans:
    case _thread_new_trans:
      ...
      break;

    default:
     fatal("Illegal threadstate encountered: %d", state);
  }
  ...
}

```
其实，安全点无非就三种情况：还没开，正在开，已经开了。[1]

还没开，那没我什么事，继续执行就ok。

正在开，VMThread正在开的时候有一个_waiting_to_block计数，表示要等多少个其他线程block，只有当_waiting_to_block为0时VMThread安全点才能完全打开。所以如果这里遇到VMThread正在开安全点，那么当前线程就将_waiting_to_block减1，告诉开安全点的线程：我马上就阻塞了，你继续执行吧，我不会影响你的。果然马上接下来就那Thread_lock，然后会阻塞在这里。直到安全点关闭。[]<br />
<br />已经开了。那更简单，直接走到那Thread_lock那里阻塞住，直到安全点关闭。<br />
<br />上面有一句话没解释清楚：
> 果然马上接下来就那Thread_lock，然后会阻塞在这里。直到安全点关闭。

为什么拿Thread_lock阻塞？<br />
<br />因为安全点的开关对应两个函数：SafepointSynchronize::begin()和SafepointSynchronize::end()，在begin里面VMThread拿到Threads_lock，然后在end里面释放，只有VMThread可以开/关安全点，所以只要VMThread在<br />begin() .... end() 这个区间执行期间，任何其他尝试拿Threads_lock的线程都会block。

上面这些内容，总结来说就一句话：如果正在开启安全点或者已经开启安全点，那么_thread_in_vm状态的线程不能切换到_thread_in_Java状态，他会block。<br />
<br />

<a name="70YMU"></a>
## 2. ThreadInVMfromNative
<a name="4jlgV"></a>
### 2.1 Native->VM
```cpp
class ThreadInVMfromNative : public ThreadStateTransition {
 public:
  ThreadInVMfromNative(JavaThread* thread) : ThreadStateTransition(thread) {
    trans_from_native(_thread_in_vm);
  }
  ~ThreadInVMfromNative() {
    trans_and_fence(_thread_in_vm, _thread_in_native);
  }
};
```
构造函数的trans_from_native最终调用这个：
```cpp
static inline void transition_from_native(JavaThread *thread, JavaThreadState to) {
    assert((to & 1) == 0, "odd numbers are transitions states");
    assert(thread->thread_state() == _thread_in_native, "coming from wrong thread state");
    // Change to transition state
    thread->set_thread_state(_thread_in_native_trans);

    InterfaceSupport::serialize_thread_state_with_handler(thread);

    // We never install asynchronous exceptions when coming (back) in
    // to the runtime from native code because the runtime is not set
    // up to handle exceptions floating around at arbitrary points.
    if (SafepointMechanism::poll(thread) || thread->is_suspend_after_native()) {
      JavaThread::check_safepoint_and_suspend_for_native_trans(thread);

      // Clear unhandled oops anywhere where we could block, even if we don't.
      CHECK_UNHANDLED_OOPS_ONLY(thread->clear_unhandled_oops();)
    }

    thread->set_thread_state(to);
  }
```
从_thread_in_native到_thread_in_vm要比从_thread_in_Java到_thread_in_vm麻烦一些，后者简单地将状态设置为_thread_in_vm就表示进入了，前者大体上有三步：（和_thread_in_vm到_thread_in_Java一样）

1. 设置状态为_thread_in_native_trans，表示正在从native状态过渡到其他状态
1. 检查安全点
1. 设置状态为_thread_in_vm，表示成功从native进入vm状态

检查状态点和之前有一点区别，只有当安全点已经开启后，才会调用SafepointSynchronize::block阻塞当前线程，如果是正在开，或者关闭的，那么是可以从native转移到vm状态而不需要阻塞的。<br />
<br />既然不会检查是否正在开启安全点，那么SafepointSynchronize::block与之前也有一点小不同：
```cpp
void SafepointSynchronize::block(JavaThread *thread) {
  ...
  switch(state) {
    case _thread_in_vm_trans:
    case _thread_in_Java:        // From compiled code
      ...// 前面已经提到
    case _thread_in_native_trans:
    case _thread_blocked_trans:
    case _thread_new_trans:
      if (thread->safepoint_state()->type() == ThreadSafepointState::_call_back) {
        thread->print_thread_state();
        fatal("Deadlock in safepoint code.  "
              "Should have called back to the VM before blocking.");
      }
      // 设置状态为_thread_blocked
      thread->set_thread_state(_thread_blocked);
      
      Threads_lock->lock_without_safepoint_check();
      thread->set_thread_state(state);
      Threads_lock->unlock();
      
      break;
  }
}
```
没有拿safepoint lock、检查is_ synchronizing()的逻辑，直接阻塞完事。<br />

<a name="RWLPL"></a>
### 2.2 Native->VM->Native
从_thread_in_vm到_thread_in_native的逻辑和从_thread_in_vm到_thread_in_Java几乎一模一样，这里就不贴了。<br />

<a name="gbyYJ"></a>
## 3. ThreadToNativeFromVM
除了2之外还有一个VM->Native->VM，他相当与将2的构造和析构换了个位置，代码几乎一样，可以对照着看：
```cpp
class ThreadInVMfromNative : public ThreadStateTransition {
 public:
  ThreadInVMfromNative(JavaThread* thread) : ThreadStateTransition(thread) {
    trans_from_native(_thread_in_vm); //1
  }
  ~ThreadInVMfromNative() {
    trans_and_fence(_thread_in_vm, _thread_in_native); //2
  }
};


class ThreadToNativeFromVM : public ThreadStateTransition {
 public:
  ThreadToNativeFromVM(JavaThread *thread) : ThreadStateTransition(thread) {
    assert(!thread->owns_locks(), "must release all locks when leaving VM");
    thread->frame_anchor()->make_walkable(thread);
    trans_and_fence(_thread_in_vm, _thread_in_native); //2
    if (_thread->has_special_runtime_exit_condition()) _thread->handle_special_runtime_exit_condition(false);
  }

  ~ThreadToNativeFromVM() {
    trans_from_native(_thread_in_vm); //1
    assert(!_thread->is_pending_jni_exception_check(), "Pending JNI Exception Check");
    // We don't need to clear_walkable because it will happen automagically when we return to java
  }
};
```


<a name="8tbyq"></a>
## 4. ~~ThreadToJavaFromVM~~
哈，其实是没有这个ThreadToJavaFromVM结构的。但是可以确定，vm->java->vm其中vm到java肯定需要安全点检查的。<br />
<br />vm到java是由JavaCalls::call_helper完成的，它的逻辑如下：
```cpp
void JavaCalls::call_helper(...) {
    ...
    // do call
  { JavaCallWrapper link(method, receiver, result, CHECK);
    { HandleMark hm(thread);  // HandleMark used by HandleMarkCleaner

      // NOTE: if we move the computation of the result_val_address inside
      // the call to call_stub, the optimizer produces wrong code.
      intptr_t* result_val_address = (intptr_t*)(result->get_value_addr());
      intptr_t* parameter_address = args->parameters();
      StubRoutines::call_stub()(
        (address)&link,
        // (intptr_t*)&(result->_value), // see NOTE above (compiler problem)
        result_val_address,          // see NOTE above (compiler problem)
        result_type,
        method(),
        entry_point,
        parameter_address,
        args->size_of_parameters(),
        CHECK
      );

      result = link.result();  // circumvent MS C++ 5.0 compiler bug (result is clobbered across call)
      // Preserve oop return value across possible gc points
      if (oop_result_flag) {
        thread->set_vm_result((oop) result->get_jobject());
      }
    }
  } // Exit JavaCallWrapper (can block - potential return oop must be preserved)
}
```
call java的真正逻辑在call_stub里面（实际上还隔了比较远），在call_stub外面有个JavaCallWrapper，这个包装对象会负责状态的转换，同时也包括安全点检查：
```cpp
JavaCallWrapper::JavaCallWrapper(const methodHandle& callee_method, Handle receiver, JavaValue* result, TRAPS) {
  ...
  ThreadStateTransition::transition(thread, _thread_in_vm, _thread_in_Java);
  ...
}
JavaCallWrapper::~JavaCallWrapper() {
  ...
  ThreadStateTransition::transition_from_java(_thread, _thread_in_vm);
  ...
}
```
所以，JavaCallWrapper才是上面三种结构的等价物。<br />

<a name="ehbBD"></a>
## 5. 总结

<br />TL;DR. 总结一下上面的状态转换：

|Wrapper|Trans|Desc|
| --- | --- | --- |
| (ThreadInVMfromJava)Java->VM->Java | Java->VM:一定可以转换，只需要改变线程状态**(0)** | VM->Java:如果安全点正在开，或者已经开了，那么不能转换，线程阻塞。**（1）** |
| (ThreadInVMfromNative)Native->VM->Native | Native->VM:如果安全点已经开了，那么不能转换，线程阻塞。(2) | VM -> Native：与（1）一致 |
| (ThreadToNativeFromVM)VM->Native->VM | VM->Native:与（1）一致 | Native->VM：与（2）一致 |
| (JavaCallWrapper)VM->Java->VM | VM->Java：与(1)一致 | Java->VM:与（0）一致 |

note：这里说的一致实际上是“几乎一致”，严格来说还是有一些区别的，只是不影响主要流程。<br />
<br />它们相当于将JVM执行的代码划分成了三部分：Java代码、VM代码、native代码，对于安全点是有重要意义的。比如说，当VMThread请求开启安全点的时候，他要求java线程停止执行，那么java线程怎么停止呢？一个最简单的方式就是发生Java->VM->Java的转换，比如上面java代码执行anewarray字节码的时候，就会在析构里面检查是否开启安全点，然后停止。

<a name="4pYrp"></a>
## footnote
[1] 狭义的说，安全点的开、关只是将一片内存的读写权限进行修改，所以不会存在开、正在开、关这种状态划分，这里的安全点开、关其实是对应SafepointSynchronize::begin()和end()两个函数。
