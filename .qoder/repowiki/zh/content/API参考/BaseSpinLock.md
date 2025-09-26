# BaseSpinLock

<cite>
**本文档中引用的文件**
- [base.rs](file://src/base.rs)
- [lib.rs](file://src/lib.rs)
</cite>

## 目录
1. [泛型参数G约束与中断控制机制](#泛型参数g约束与中断控制机制)
2. [new构造函数实现细节](#new构造函数实现细节)
3. [lock方法执行逻辑分析](#lock方法执行逻辑分析)
4. [Send和Sync实现条件](#send和sync实现条件)
5. [共享计数器使用示例](#共享计数器使用示例)

## 泛型参数G约束与中断控制机制

`BaseSpinLock<G: BaseGuard, T: ?Sized>` 结构体的泛型参数 `G` 必须实现 `BaseGuard` trait，该trait定义了在获取锁前后对系统状态的控制行为。通过不同的 `BaseGuard` 实现，可以控制锁获取过程中的中断和抢占行为。

具体来说，`G::acquire()` 方法在尝试获取锁之前被调用，用于保存当前上下文状态并根据需要禁用本地中断（IRQ）或内核抢占。例如：
- 当 `G` 为 `NoOp` 时，不进行任何操作，适用于已处于禁止抢占和中断的上下文中
- 当 `G` 为 `NoPreempt` 时，在获取锁前禁用内核抢占，在释放锁后恢复
- 当 `G` 为 `NoPreemptIrqSave` 时，在获取锁前同时禁用内核抢占和本地中断，并在释放锁时恢复原始状态

这种设计使得 `BaseSpinLock` 能够适应不同的执行环境需求，既可以在完全受控的环境中最小化开销，也可以在复杂环境中提供更强的安全保证。

**Section sources**
- [base.rs](file://src/base.rs#L21-L42)
- [lib.rs](file://src/lib.rs#L5)

## new构造函数实现细节

`new()` 构造函数用于创建一个新的 `BaseSpinLock` 实例并包装提供的数据。其核心实现逻辑如下：

在单核环境（未启用"smp"特性）下，由于不存在并发竞争，锁状态被优化移除，仅保留 `UnsafeCell<T>` 来包裹数据。而在多核环境（启用"smp"特性）下，构造函数会初始化一个 `AtomicBool` 类型的锁标志，初始值设为 `false` 表示锁处于空闲状态。

构造过程中，传入的数据通过 `UnsafeCell::new(data)` 进行封装，确保即使在不可变引用下也能进行内部可变性操作。`PhantomData<G>` 的引入则用于标记泛型参数 `G`，避免编译器因未实际使用该类型而产生警告，同时不影响运行时大小。

此构造函数被标记为 `const fn` 和 `#[inline(always)]`，允许在编译期常量上下文中使用，并确保内联以减少函数调用开销。

**Section sources**
- [base.rs](file://src/base.rs#L44-L68)

## lock方法执行逻辑分析

`lock()` 方法的实现根据是否启用 SMP 特性表现出不同的行为模式：

在单核环境下，由于不存在真正的并发竞争，方法直接调用 `G::acquire()` 获取上下文状态后立即返回锁守卫，无需任何等待逻辑。

在多核环境下，方法采用经典的自旋等待机制：
1. 首先调用 `G::acquire()` 保存当前中断/抢占状态
2. 进入循环，使用 `compare_exchange_weak` 原子操作尝试将锁状态从 `false`（空闲）更改为 `true`（占用）
3. 若交换失败（返回 `Err`），则进入辅助等待循环，通过 `is_locked()` 检查锁状态并调用 `core::hint::spin_loop()` 提示 CPU 进行忙等待优化
4. 成功获取锁后，构建并返回 `BaseSpinLockGuard`

`compare_exchange_weak` 的使用允许在某些平台上发生虚假失败，但配合外层循环可提高整体性能。内存顺序上采用 `Acquire` 语义确保后续内存访问不会被重排序到锁获取之前，保证同步正确性。

**Section sources**
- [base.rs](file://src/base.rs#L70-L100)
- [base.rs](file://src/base.rs#L102-L109)

## Send和Sync实现条件

`BaseSpinLock` 对 `Send` 和 `Sync` trait 的实现基于以下安全假设：

```rust
unsafe impl<G: BaseGuard, T: ?Sized + Send> Sync for BaseSpinLock<G, T> {}
unsafe impl<G: BaseGuard, T: ?Sized + Send> Send for BaseSpinLock<G, T> {}
```

线程安全保证的边界条件包括：
1. 内部数据 `T` 必须满足 `Send` 约束，确保可以在不同线程间安全转移所有权
2. 锁本身的引用可以通过原子操作在多个线程间共享（`Sync`）
3. 多核环境下的 `AtomicBool` 提供了必要的原子性保证
4. 单核环境通过正确的使用约定（proper guard in use）来保证安全性

需要注意的是，虽然 `BaseSpinLock` 本身实现了 `Sync`，但实际的线程安全性还依赖于正确的使用方式和底层 `BaseGuard` 实现的正确性。特别是在中断处理等特殊场景中，必须遵循文档规定的使用约束。

**Section sources**
- [base.rs](file://src/base.rs#L44-L45)

## 共享计数器使用示例

以下是使用 `BaseSpinLock<T, NoPreempt>` 保护共享计数器的完整代码示例：

```rust
use kspin::{BaseSpinLock, NoPreempt};

// 创建一个被锁保护的共享计数器
let counter = BaseSpinLock::<NoPreempt, usize>::new(0);

// 获取锁并修改数据
{
    let mut guard = counter.lock();
    *guard += 1;
    println!("Counter value: {}", *guard);
    // 锁在此处自动释放
}

// 在另一个作用域中再次获取锁
{
    let mut guard = counter.lock();
    *guard += 1;
    assert_eq!(*guard, 2);
}
```

该示例展示了完整的生命周期：首先通过 `new()` 创建带锁的计数器，然后通过 `lock()` 方法获取独占访问权，在作用域结束时锁自动释放。`NoPreempt` 作为 `BaseGuard` 实现在获取锁时会禁用内核抢占，防止当前线程被调度出去导致死锁风险。

**Section sources**
- [lib.rs](file://src/lib.rs#L10-L12)
- [base.rs](file://src/base.rs#L70-L100)